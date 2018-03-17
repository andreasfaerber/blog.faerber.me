---
title: "Proxmox cluster, 2+1 configuration: 2 local and 1 remote quorum node via OpenVPN"
layout: single
date:   2017-05-08 21:47:22 +0100
excerpt: "Proxmox remote quorum: How to setup a Proxmox cluster with a remote quorum node in a 2+1 configuration via OpenVPN"
classes: wide
toc: true
categories:
  - Virtualization
tags:
  - proxmox
  - proxmox cluster
  - high availability
  - remote quorum
  - openvpn
last_modified_at: 2017-05-30 17:41:00 +0100
---
This post describes my setup of a 2+1 node Proxmox remote cluster that utilizes a remote node connected via openvpn as a quorum node for the cluster.
It will guide you through creating your cluster from two directly connected nodes, install and configure openvpn and connect your remote quorum node to your cluster.


For my own little infrastructure i am using two servers that run virtualized machines. Recently i have migrated the servers to Proxmox
as virtualization solution and began to play around with Proxmox and the cluster features.

One of the main reasons for doing so is that Proxmox offers a central firewall that can be managed for all nodes which reduces the effort
of maintaining a firewall on each host or virtual machine seperately (you can still do that, if you want).

### Proxmox cluster, Requirements
In order to get a "2+1" cluster set up you need to have 3 cluster nodes of which 2 ("cluster node") would ideally be next to each other from a network point
of view and your third node ("quorum node") reachable from both of the cluster nodes.

### Starting point
I'll be starting to describe the setup based on the assumption that you have installed Proxomx on each of the three nodes with a working network setup
and connectivity between the nodes.

### Base configuration overview
To showcase the configuration i am running 2 cluster nodes as KVM virtual machines on one of my Proxmox servers and the third node on a remote Proxmox
host (at OVH), configured like this:

````
prx01: 10.10.10.101/24
prx02: 10.10.10.102/24
prx03: 10.10.10.103/24
````

This results in prx01 and prx02 being in the same network on the same host. This would be the same for physical hosts that are connected to the same network or
via an interconnect network for cluster communication for example. prx03 is on the remote host and is not able to see or talk to prx01 or prx02.

All 3 nodes are masqueraded to the outer world by the host they're running on. That's why they all are located in the 10.10.10.0/24 network.

### Step 1: Build a cluster from the first 2 nodes
{% highlight shell %}
root@prx01:~# pvecm create CLUSTER -ring0_addr 10.10.10.101
Corosync Cluster Engine Authentication key generator.
Gathering 1024 bits for key from /dev/urandom.
Writing corosync key to /etc/corosync/authkey.
root@prx01:~#
{% endhighlight %}

This results in the following /etc/pve/corosync.conf:

{% highlight jsonnet linenos %}
totem {
  version: 2
  secauth: on
  cluster_name: CLUSTER
  config_version: 1
  ip_version: ipv4
  interface {
    ringnumber: 0
    bindnetaddr: 10.10.10.101
  }
}

nodelist {
  node {
    ring0_addr: 10.10.10.101
    name: prx01
    nodeid: 1
    quorum_votes: 1
  }
}

quorum {
  provider: corosync_votequorum
}

logging {
  to_syslog: yes
  debug: off
}
{% endhighlight %}



### Step 2: Add unicast transport to /etc/pve/corosync.conf

Add "transport: udpu" to /etc/pve/corosync.conf, increase "config_version" by 1 and reboot the node to activate the new corosync.conf. See [Editing corosync.conf](https://pve.proxmox.com/wiki/Editing_corosync.conf) about editing /etc/pve/corosync.conf.

The updated corosync.conf with **lines 5 and 7 added**:

{% highlight jsonnet linenos %}
totem {
  version: 2
  secauth: on
  cluster_name: CLUSTER
  config_version: 2
  ip_version: ipv4
  transport: udpu
  interface {
    ringnumber: 0
    bindnetaddr: 10.10.10.101
  }
}

nodelist {
  node {
    ring0_addr: prx01
    name: prx01
    nodeid: 1
    quorum_votes: 1
  }
}

quorum {
  provider: corosync_votequorum
}

logging {
  to_syslog: yes
  debug: off
}
{% endhighlight %}

### Step 3: Reboot prx01
Reboot prx01 to activate the new corosync.conf.

### Step 4: Add prx02 to the cluster
Add prx02 to the cluster:

{% highlight shellsession %}
root@prx02:~# pvecm add 10.10.10.101 -ring0_addr 10.10.10.102
Generating public/private rsa key pair.
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
8a:cc:03:7c:6c:1a:b9:41:11:1e:93:f2:05:c9:cc:bb root@prx02
The key''s randomart image is:
+---[RSA 2048]----+
| +*=             |
|..*+.            |
| ooo             |
| ooo             |
|  *.+   S        |
|  E@ . .         |
|  o = .          |
|     .           |
|                 |
+-----------------+
The authenticity of host '10.10.10.101 (10.10.10.101)' can't be established.
ECDSA key fingerprint is ec:fd:a0:cb:32:7e:7f:94:9b:9e:db:5a:34:e8:bc:61.
Are you sure you want to continue connecting (yes/no)? yes
root@10.10.10.101's password:
copy corosync auth key
stopping pve-cluster service
backup old database
waiting for quorum...OK
generating node certificates
merge known_hosts file
restart services
successfully added node 'prx02' to cluster.
root@prx02:~# pvecm status
Quorum information
------------------
Date:             Sun Mar 19 20:07:25 2017
Quorum provider:  corosync_votequorum
Nodes:            2
Node ID:          0x00000002
Ring ID:          1/180
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   2
Highest expected: 2
Total votes:      2
Quorum:           2
Flags:            Quorate

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 10.10.10.101
0x00000002          1 10.10.10.102 (local)
root@prx02:~#
{% endhighlight %}

Resulting /etc/pve/corosync.conf:

{% highlight jsonnet linenos %}
logging {
  debug: off
  to_syslog: yes
}

nodelist {
  node {
    name: prx02
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 10.10.10.102
  }

  node {
    name: prx01
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 10.10.10.101
  }

}

quorum {
  provider: corosync_votequorum
}

totem {
  cluster_name: CLUSTER
  config_version: 3
  ip_version: ipv4
  secauth: on
  transport: udpu
  version: 2
  interface {
    bindnetaddr: 10.10.10.101
    ringnumber: 0
  }

}
{% endhighlight %}

We now have a 2 node Proxmox cluster with an issue: No real quorum. So if one cluster node fails for whatever reason the
other cluster node will not be able to provide services as it cannot determine if the quorum is available and will thus
block activity and services due to lack of quorum (see line 15 and 16):

{% highlight shell linenos %}
root@prx02:~# pvecm status
Quorum information
------------------
Date:             Sun Mar 26 09:26:44 2017
Quorum provider:  corosync_votequorum
Nodes:            1
Node ID:          0x00000002
Ring ID:          2/196
Quorate:          No

Votequorum information
----------------------
Expected votes:   2
Highest expected: 2
Total votes:      1
Quorum:           2 Activity blocked
Flags:

Membership information
----------------------
    Nodeid      Votes Name
0x00000002          1 10.10.10.102 (local)
{% endhighlight %}

As we only have 2 local nodes, we set up a third node in a remote location and use it for quorum only.

### Step 5: Setup OpenVPN on prx03

OpenVPN will be used to provide network connectivity between the initial two (prx01 and prx02) and the remote node (prx03):

{% highlight shellsession %}
root@prx03:~# apt-get install openvpn
{% endhighlight %}

{% highlight shellsession linenos %}
root@prx03:~# cp -a /usr/share/easy-rsa /etc/openvpn/
root@prx03:~# cd /etc/openvpn/easy-rsa
root@prx03:~# vi ./vars
{% endhighlight %}

I suggest to change the following variables in the vars file, even though that is optional:

{% highlight shell %}
export KEY_COUNTRY="DE"
export KEY_PROVINCE="Hamburg"
export KEY_CITY="Hamburg"
export KEY_ORG="PRX-CLUSTER"
export KEY_EMAIL="cluster@email.address"
export KEY_OU="PRX-CLUSTER"
{% endhighlight %}

Afterwards, create your CA and certificates:

{% highlight shell %}
root@prx03:/etc/openvpn/easy-rsa# source ./vars
NOTE: If you run ./clean-all, I will be doing a rm -rf on /etc/openvpn/easy-rsa/keys
root@prx03:/etc/openvpn/easy-rsa# ./clean-all
root@prx03:/etc/openvpn/easy-rsa# ./build-ca
Generating a 2048 bit RSA private key
...............................................+++
....................................................................................................+++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [DE]:
State or Province Name (full name) [Hamburg]:
Locality Name (eg, city) [Hamburg]:
Organization Name (eg, company) [PRX-CLUSTER]:
Organizational Unit Name (eg, section) [PRX-CLUSTER]:
Common Name (eg, your name or your servers hostname) [PRX-CLUSTER CA]:
Name [EasyRSA]:
Email Address [cluster@email.address]:
root@prx03:/etc/openvpn/easy-rsa# ./build-key-server prx03
Generating a 2048 bit RSA private key
.............................................................+++
..........................................+++
writing new private key to 'prx03.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [DE]:
State or Province Name (full name) [Hamburg]:
Locality Name (eg, city) [Hamburg]:
Organization Name (eg, company) [PRX-CLUSTER]:
Organizational Unit Name (eg, section) [PRX-CLUSTER]:
Common Name (eg, your name or your servers hostname) [prx03]:
Name [EasyRSA]:
Email Address [cluster@email.address]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /etc/openvpn/easy-rsa/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subjects Distinguished Name is as follows
countryName           :PRINTABLE:'DE'
stateOrProvinceName   :PRINTABLE:'Hamburg'
localityName          :PRINTABLE:'Hamburg'
organizationName      :PRINTABLE:'PRX-CLUSTER'
organizationalUnitName:PRINTABLE:'PRX-CLUSTER'
commonName            :PRINTABLE:'prx03'
name                  :PRINTABLE:'EasyRSA'
emailAddress          :IA5STRING:'cluster@email.address'
Certificate is to be certified until Apr 20 10:24:50 2027 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
root@prx03:/etc/openvpn/easy-rsa# openvpn --genkey --secret /etc/openvpn/ta.key
root@prx03:/etc/openvpn/easy-rsa# ./build-dh 2048
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
................ [...]

root@prx03:/etc/openvpn/easy-rsa# ./pkitool prx01
Generating a 2048 bit RSA private key
....+++
.........................................................................+++
writing new private key to 'prx01.key'
-----
Using configuration from /etc/openvpn/easy-rsa/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'DE'
stateOrProvinceName   :PRINTABLE:'Hamburg'
localityName          :PRINTABLE:'Hamburg'
organizationName      :PRINTABLE:'PRX-CLUSTER'
organizationalUnitName:PRINTABLE:'PRX-CLUSTER'
commonName            :PRINTABLE:'prx01'
name                  :PRINTABLE:'EasyRSA'
emailAddress          :IA5STRING:'cluster@email.address'
Certificate is to be certified until Apr 20 10:28:37 2027 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
root@prx03:/etc/openvpn/easy-rsa# ./pkitool prx02
Generating a 2048 bit RSA private key
...............................................+++
..........................................+++
writing new private key to 'prx02.key'
-----
Using configuration from /etc/openvpn/easy-rsa/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'DE'
stateOrProvinceName   :PRINTABLE:'Hamburg'
localityName          :PRINTABLE:'Hamburg'
organizationName      :PRINTABLE:'PRX-CLUSTER'
organizationalUnitName:PRINTABLE:'PRX-CLUSTER'
commonName            :PRINTABLE:'prx02'
name                  :PRINTABLE:'EasyRSA'
emailAddress          :IA5STRING:'cluster@email.address'
Certificate is to be certified until Apr 20 10:28:50 2027 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
root@prx03:/etc/openvpn/easy-rsa#
{% endhighlight %}

Create OpenVPN server configuration on prx03. Note: If you also use a test installation on a private network, you need to NAT the
OpenVPN port 11194 to 10.10.10.103 to allow OpenVPN connections. Add the public IP to the client configurations (PRX-03-PUBLIC-IP).

{% highlight shell %}
root@prx03:/etc/openvpn/easy-rsa# cat > /etc/openvpn/pve-server.conf <<EOF
local 10.10.10.103
proto tcp
port 11194
dev tun
status /tmp/openvpn-status.log

server 10.10.100.0 255.255.255.0
tls-server
verb 3
ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/prx03.crt
key /etc/openvpn/easy-rsa/keys/prx03.key
dh /etc/openvpn/easy-rsa/keys/dh2048.pem
tls-auth /etc/openvpn/ta.key 0
key-direction 0
keepalive 10 60
persist-key
persist-tun

route 10.10.10.101 255.255.255.255
route 10.10.10.102 255.255.255.255
client-config-dir /etc/openvpn/ccd

user nobody
group nogroup
EOF
root@prx03:/etc/openvpn/easy-rsa# mkdir -p /etc/openvpn/ccd
root@prx03:/etc/openvpn/easy-rsa# cat > /etc/openvpn/ccd/prx01 <<EOF
iroute "10.10.10.101 255.255.255.255"
push "route 10.10.10.103 255.255.255.255"
EOF
root@prx03:/etc/openvpn/easy-rsa# cat > /etc/openvpn/ccd/prx02 <<EOF
iroute "10.10.10.102 255.255.255.255"
push "route 10.10.10.103 255.255.255.255"
EOF
root@prx03:/etc/openvpn/easy-rsa# 
{% endhighlight %}

Start and enable OpenVPN:

{% highlight shell %}
root@prx03:/etc/openvpn/easy-rsa# systemctl start openvpn@pve-server
root@prx03:/etc/openvpn/easy-rsa# systemctl enable openvpn@pve-server
Created symlink from /etc/systemd/system/multi-user.target.wants/openvpn@pve-server.service to /lib/systemd/system/openvpn@.service.
root@prx03:/etc/openvpn/easy-rsa#
{% endhighlight %}

### Step 6: Install and configure OpenVPN on the clients

{% highlight shell %}
root@prx01:~# apt-get install openvpn
{% endhighlight %}

{% highlight shell %}
root@prx02:~# apt-get install openvpn
{% endhighlight %}

Create OpenVPN client configurations on prx01 and prx02:

{% highlight shell %}
root@prx01:~# cat > /etc/openvpn/pve-prx03.conf <<EOF
client
tls-client
dev tun
proto tcp
remote PRX-03-PUBLIC-IP 11194
resolv-retry infinite
nobind

persist-key
persist-tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/prx01.crt
key /etc/openvpn/prx01.key
tls-auth /etc/openvpn/ta.key 1

keepalive 2 10

verb 3
EOF
root@prx01:~# 
{% endhighlight %}

{% highlight shell %}
root@prx02:~# cat > /etc/openvpn/pve-prx03.conf <<EOF
client
tls-client
dev tun
proto tcp
remote PRX-03-PUBLIC-IP 11194
resolv-retry infinite
nobind

persist-key
persist-tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/prx02.crt
key /etc/openvpn/prx02.key
tls-auth /etc/openvpn/ta.key 1

keepalive 2 10

verb 3
EOF
root@prx02:~# 
{% endhighlight %}

Copy the required files to prx01 and prx02:

{% highlight shell %}
prx03: /etc/openvpn/ta.key -> prx01/prx02: /etc/openvpn/ta.key
prx03: /etc/openvpn/easy-rsa/keys/ca.crt -> prx01/prx02: /etc/openvpn/ca.crt
prx03: /etc/openvpn/easy-rsa/keys/prx01.crt -> prx01: /etc/openvpn/prx01.crt
prx03: /etc/openvpn/easy-rsa/keys/prx01.key -> prx01: /etc/openvpn/prx01.key
prx03: /etc/openvpn/easy-rsa/keys/prx02.crt -> prx01: /etc/openvpn/prx02.crt
prx03: /etc/openvpn/easy-rsa/keys/prx02.key -> prx01: /etc/openvpn/prx02.key
{% endhighlight %}

Start OpenVPN on prx01 and prx02:

{% highlight shell %}
root@prx01:~# systemctl start openvpn@pve-prx03
root@prx01:~# systemctl enable openvpn@pve-prx03
root@prx01:~# 
{% endhighlight %}

{% highlight shell %}
root@prx02:~# systemctl start openvpn@pve-prx03
root@prx02:~# systemctl enable openvpn@pve-prx03
root@prx02:~# 
{% endhighlight %}

Check that you can ping prx03 from prx01 and prx02 and vice versa:

{% highlight shell %}
root@prx01:~# ping -c 2 10.10.10.103
PING 10.10.10.103 (10.10.10.103) 56(84) bytes of data.
64 bytes from 10.10.10.103: icmp_seq=1 ttl=64 time=18.1 ms
64 bytes from 10.10.10.103: icmp_seq=2 ttl=64 time=18.1 ms

--- 10.10.10.103 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 18.159/18.161/18.164/0.134 ms
root@prx01:~# 
{% endhighlight %}

{% highlight shell %}
root@prx02:~# ping -c 2 10.10.10.103
PING 10.10.10.103 (10.10.10.103) 56(84) bytes of data.
64 bytes from 10.10.10.103: icmp_seq=1 ttl=64 time=18.3 ms
64 bytes from 10.10.10.103: icmp_seq=2 ttl=64 time=18.1 ms

--- 10.10.10.103 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 18.130/18.241/18.353/0.175 ms
root@prx02:~# 
{% endhighlight %}

{% highlight shell %}
root@prx03:~# ping -c 2 10.10.10.101
PING 10.10.10.101 (10.10.10.101) 56(84) bytes of data.
64 bytes from 10.10.10.101: icmp_seq=1 ttl=64 time=18.0 ms
64 bytes from 10.10.10.101: icmp_seq=2 ttl=64 time=18.3 ms

--- 10.10.10.101 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 18.072/18.198/18.324/0.126 ms
root@prx03:~# ping -c 2 10.10.10.102
PING 10.10.10.102 (10.10.10.102) 56(84) bytes of data.
64 bytes from 10.10.10.102: icmp_seq=1 ttl=64 time=18.2 ms
64 bytes from 10.10.10.102: icmp_seq=2 ttl=64 time=18.4 ms

--- 10.10.10.102 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 18.231/18.361/18.491/0.130 ms
root@prx03:~#
{% endhighlight %}

### Step 7: Add prx03 to the cluster

{% highlight shell %}
root@prx03:~# pvecm add 10.10.10.101 -ring0_addr 10.10.10.103
The authenticity of host '10.10.10.101 (10.10.10.101)' can't be established.
ECDSA key fingerprint is ec:fd:a0:cb:32:7e:7f:94:9b:9e:db:5a:34:e8:bc:61.
Are you sure you want to continue connecting (yes/no)? yes
root@10.10.10.101's password:
copy corosync auth key
stopping pve-cluster service
backup old database
waiting for quorum...OK
generating node certificates
merge known_hosts file
restart services
successfully added node 'prx03' to cluster.
root@prx03:~#
{% endhighlight %}

Check cluster status:

{% highlight shell %}
root@prx03:~# pvecm status
Quorum information
------------------
Date:             Sat Apr 22 15:10:55 2017
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          0x00000003
Ring ID:          1/260
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2
Flags:            Quorate

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 10.10.10.101
0x00000002          1 10.10.10.102
0x00000003          1 10.10.10.103 (local)
root@prx03:~#
{% endhighlight %}

Voila! If you give it a try, let me know if it works for you.


