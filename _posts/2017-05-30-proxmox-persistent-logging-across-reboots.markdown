---
title: "Proxmox: Persistent logging across reboots"
layout: single
date:   2017-05-30 17:35:08 +0100
excerpt: "Proxmox persistent logging: How to enable persistent journal logging on a Proxmox node as persistent logging is not enabled by default."
classes: wide
categories:
  - Virtualization
tags:
  - proxmox
  - logging
---
In case you require persistent logging for your Proxmox node that survives a reboot, it can be enabled with two simple steps:

```console
root@prx01:~# mkdir /var/log/journal; systemctl restart systemd-journald
root@prx01:~#
```

This enables persistent logging via the journal which you can then recall with "journalctl -b" or "journalctl -b 1" to review logfiles
from different reboots.
