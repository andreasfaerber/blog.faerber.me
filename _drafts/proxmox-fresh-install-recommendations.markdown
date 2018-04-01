---
title: "Proxmox: Fresh install recommendations"
layout: single
date:   2018-03-27 19:27:08 +0100
excerpt: "My recommendations following a fresh Proxmox install"
classes: wide
categories:
  - Virtualization
tags:
  - proxmox
  - lvm
---
Based on my experience so far there are some recommendations for a fresh Proxmox
installation that i would like to share:

## Increase LVM meta data size

The default metadata size of a fresh Proxmox installation seems quite small. Having
run into an issue already i increase the metadata size of a fresh install to a gigabyte:

Check for thin pools metadata:
````
lvs -a | grep -i tmeta
````

Set size of all pools metadata to 1G. Note: This sets ALL pools metadata to 1G. If you
have other pools on the system only set / increase size selectively. For safety reasons
the below command only prints out the commands to increase each pools metadata size.

````
for i in $(lvs -a | grep -i tmeta | sed -e 's/\[//g' | sed -e 's/\]//g' | awk '{print $2"/"$1}'); do echo lvextend -L 1G $i; done
````
Commands to set metadata size to 1G:
````
lvextend -L 1G pve/data_tmeta
````

