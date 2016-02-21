---
layout: post
title: TFTP server on Fedora
---

Here are some quick notes on setting up a TFTP server on Fedora 23. This is
used, for example, to send Linux kernel images and other binaries to a
bootloader on an embedded system.

First, install the packages:

    sudo dnf install -y tftp tftp-server

The `tftp` itself will allow you to test your configuration by attempting a
file transfer.  The default directory for TFTP transfers is
`/var/lib/tftpboot`.

The TFTP server works through xinetd so you will need to add a rule that says
`in.tftpd: ALL` to `/etc/hosts.allow`:

    sudo su -c "echo 'in.tftpd: ALL' >> /etc/hosts.allow"

Enable and start the TFTP server:

    sudo systemctl enable tftp
    sudo systemctl start tftp
    sudo systemctl demon-reload

Tell the firewall to allow TFTP traffic:

    sudo firewall-cmd --permanent --add-service tftp
    sudo firewall-cmd --reload

You should now be able to transfer files via TFTP.
