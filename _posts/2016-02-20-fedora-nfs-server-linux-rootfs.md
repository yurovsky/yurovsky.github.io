---
layout: post
title: NFS root file system on Fedora
---

Here are a few notes on making your Fedora workstation serve a root file system
for some embedded Linux target. There are a couple of additional steps when
compared to Ubuntu, namely telling the firewall to allow NFS traffic and
re-enabling NFSv2 support.

First, install the NFS and rpcbind packages:

    sudo dnf install -y nfs-utils rpcbind

Enable and then activate services:

    sudo systemctl enable rpcbind
    sudo systemctl enable nfs-server
    sudo systemctl start rpcbind
    sudo systemctl start nfs-server
    sudo systemctl start rpc-statd
    sudo systemctl start nfs-idmapd

Tell the firewall to allow NFS connections:

    sudo firewall-cmd --permanent --add-service nfs
    sudo firewall-cmd --permanent --add-service rpc-bind
    sudo firewall-cmd --permanent --add-service mountd
    sudo firewall-cmd --reload

Enable NFSv2 support.  This is very important as the Linux kernel explicitly
uses the old NFS version 2 protocol when NFS booting, however NFS version 2 is
disabled by default in Fedora 22, 23, and so on.  Edit `/etc/sysconfig/nfs`
and set `RPCNFSDARGS=` to "-V 2" in order to enable NFSv2.

Restart services for this to take effect:

    sudo systemctl restart nfs-config
    sudo systemctl restart nfs
    sudo systemctl restart rpcbind

Verify that NFSv2 support is enabled by checking `/proc/nfsd/versions`,

    sudo cat /proc/fs/nfsd/versions
    +2 +3 +4 +4.1 +4.2

The `+2` indicates NFSv2 support. You should now be able to add your exported
file systems to `/etc/exports.d/` and then run:

    sudo exportfs -ra

to reload and add the new entries.
