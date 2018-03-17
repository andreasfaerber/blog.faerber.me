---
title: "Proxmox cluster: GlusterFS with remote quorum"
layout: single
date:   2017-05-09 20:22:28 +0100
excerpt: "Proxmox glusterfs: Howto setup glusterfs within a proxmox cluster that utilizes a remote node for quorum"
classes: wide
categories:
  - Virtualization
tags:
  - proxmox
  - proxmox cluster
  - glusterfs
  - high availability
  - openvpn
  - remote quorum
last_modified_at: 2017-05-30 17:41:00 +0100
---
To utilize shares storage for virtual machines storage (to allow Proxmox live migration / HA failover) or as shared docker storage (to be tested) i utilized glusterfs
and describe the installation and initial usage in this post. To demonstrate and test the installation this post assumes a running 2+1 node cluster (or a standard three
node cluster) described in the post [Proxmox VE, 2+1 cluster via OpenVPN]({{ site.baseurl }}{% link _posts/2017-05-08-proxmox-ve-21-cluster-2-local-and-1-remote-quorum-node-via-openvpn.markdown %}).

GlusterFS installation (on each node):
{% highlight shell %}
root@prx01:~# wget -O - http://download.gluster.org/pub/gluster/glusterfs/3.8/LATEST/rsa.pub | apt-key add -
--2017-04-26 14:05:31--  http://download.gluster.org/pub/gluster/glusterfs/3.8/LATEST/rsa.pub
Resolving download.gluster.org (download.gluster.org)... 23.253.208.221, 2001:4801:7824:104:be76:4eff:fe10:23d8
Connecting to download.gluster.org (download.gluster.org)|23.253.208.221|:80... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://download.gluster.org/pub/gluster/glusterfs/3.8/LATEST/rsa.pub [following]
--2017-04-26 14:05:31--  https://download.gluster.org/pub/gluster/glusterfs/3.8/LATEST/rsa.pub
Connecting to download.gluster.org (download.gluster.org)|23.253.208.221|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1732 (1.7K)
Saving to: ‘STDOUT’

[...]

2017-04-26 14:05:32 (55.3 MB/s) - written to stdout [1732/1732]

OK
root@prx01:~# DEBID=$(grep 'VERSION_ID=' /etc/os-release | cut -d '=' -f 2 | tr -d '"')
root@prx01:~# DEBVER=$(grep 'VERSION=' /etc/os-release | grep -Eo '[a-z]+')
root@prx01:~# echo deb https://download.gluster.org/pub/gluster/glusterfs/3.8/LATEST/Debian/${DEBID}/apt ${DEBVER} main > /etc/apt/sources.list.d/gluster.list
root@prx01:~# apt-get update
[...]
root@prx01:~# apt-get install glusterfs-server glusterfs-client
[...]
{% endhighlight %}
(Note: Also install on prx02 and prx03)

Probe gluster peers to create gluster cluster (sounds nice!)

{% highlight shell %}
root@prx01:~# gluster peer probe 10.10.10.102
peer probe: success.
root@prx01:~# gluster peer probe 10.10.10.103
peer probe: success.
root@prx01:~# gluster peer status
Number of Peers: 2

Hostname: 10.10.10.102
Uuid: 00e7d290-f6de-4ad3-9572-8ca631f82f1e
State: Peer in Cluster (Connected)

Hostname: 10.10.10.103
Uuid: f89e323b-9884-4b10-8b83-8aa81998df18
State: Peer in Cluster (Connected)
root@prx01:~#
{% endhighlight %}

To create Partitions and volumes i have added two 10GB volumes to prx01 and prx02 to use for glusterfs via the Proxmox GUI. No volume on prx03 as prx03 is the quorum host only.

Create partitions, setup lvm and create file systems:

{% highlight shell %}
root@prx01:~# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.25.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): g
Created a new GPT disklabel (GUID: 92E85B57-0222-402E-83E9-523AE62DC3C9).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-20971486, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-20971486, default 20971486):

Created a new partition 1 of type 'Linux filesystem' and of size 10 GiB.

Command (m for help): w

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

root@prx01:~# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created
root@prx01:~# vgcreate vg_glusterfs /dev/sdb1
  Volume group "vg_glusterfs" successfully created
root@prx01:~# lvcreate -l 100%FREE -n lv_glusterfs vg_glusterfs
  Logical volume "lv_glusterfs" created.
root@prx01:~# mkdir /glusterfs
root@prx01:~# mkfs.ext4 /dev/mapper/vg_glusterfs-lv_glusterfs
mke2fs 1.42.12 (29-Aug-2014)
Discarding device blocks: done
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: 73b89c59-789d-44d3-9856-b1d0d87a1802
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

root@prx01:~# echo "/dev/mapper/vg_glusterfs-lv_glusterfs /glusterfs ext4 defaults 0 0" >> /etc/fstab
root@prx01:~# mount /glusterfs
{% endhighlight %}
(Note: Repeat on prx02)

Create glusterfs volume (on prx01 only):

{% highlight shell %}
root@prx01:~# gluster volume create gfs-testing transport tcp replica 2 10.10.10.101:/glusterfs/testing 10.10.10.102:/glusterfs/testing
volume create: gfs-testing: success: please start the volume to access data
root@prx01:~# gluster volume start gfs-testing
volume start: gfs-testing: success
root@prx01:~# gluster volume status
Status of volume: gfs-testing
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick 10.10.10.101:/glusterfs/testing       49152     0          Y       12034
Brick 10.10.10.102:/glusterfs/testing       49152     0          Y       11746
Self-heal Daemon on localhost               N/A       N/A        Y       12054
Self-heal Daemon on 10.10.10.102            N/A       N/A        Y       11766
Self-heal Daemon on 10.10.10.103            N/A       N/A        Y       22491

Task Status of Volume gfs-testing
------------------------------------------------------------------------------
There are no active volume tasks

root@prx01:~#

{% endhighlight %}

You can then add the gfs-testing Volume via the Proxmox UI for prx01 and prx02 (and not prx03!) as GlusterFS storage (on the data center storage level). The UI is pretty self-explanatory:

{% include figure image_path="/assets/screenshots/proxmox_glusterfs_gfs-testing.png" alt="Screenshot of Proxmox UI gfs-testing configuration" caption="Proxmox gfs-testing configuration" %}

Proxmox will then mount the gfs-testing volume under /mnt/pve/gfs-testing.

Finished.
