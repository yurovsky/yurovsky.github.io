---
layout: post
title: Listing WiFi (802.11) interfaces with netlink
---

The Linux kernel netlink interface can be used to learn about and manage
network interfaces.  Netlink covers a lot of territory and is quite
sophisticated, check out the [userspace documentation for libnl](http://www.carisma.slowglass.com/~tgr/libnl/doc/core.html) for a good description.

I recently wrote an application that, among other things, needs to know the
names of 802.11 interfaces on the system (ex: `wlan0`) so that it can then do
things with those interfaces.  I was curious how `iw` does this and dug around,
so here are some quick notes.

## API

I built an application that links against the generic netlink library
`libnl-genl` as well as `libnl` itself.  This example will connect via
netlink, send a command asking to dump all 802.11 network interfaces, and then
exit.

The netlink API needed is:

    #include <netlink/netlink.h>
    #include <netlink/genl/genl.h>
    #include <netlink/genl/family.h>
    #include <netlink/genl/ctrl.h>

The ABI contract, `nl80211.h`, can be taken from the iw source tree.  It
provides the 802.11-specific netlink commands and attributes that we use when
talking to the kernel:

    #include "nl80211.h"

And finally we need standard IO and `errno.h` for error codes:

    #include <stdio.h>
    #include <errno.h>

## Initialization

Netlink provides a socket wrapper of type `struct nl_sock` and methods to
allocate and free that.  There is also a unique ID used as a handle.  We can
keep some data structure around for these:

    static struct {
        struct nl_sock *nls;
        int nl80211_id;
    } wifi;

We need one of these sockets in order to talk to the kernel, the buffer sizes
are taken from the `iw` implementation:

    wifi.nls = nl_socket_alloc();
    if (!wifi.nls) {
            fprintf(stderr, "Failed to allocate netlink socket.\n");
            return -ENOMEM;
    }
    
    nl_socket_set_buffer_size(wifi.nls, 8192, 8192);

We can then use the generic netlink to connect to the `nl80211` control
channel:

    if (genl_connect(wifi.nls)) {
            fprintf(stderr, "Failed to connect to generic netlink.\n");
            nl_socket_free(wifi.nls);
            return -ENOLINK;
    }

    wifi.nl80211_id = genl_ctrl_resolve(wifi.nls, "nl80211");
    if (wifi.nl80211_id < 0) {
            fprintf(stderr, "nl80211 not found.\n");
            nl_socket_free(wifi.nls);
            return -ENOENT;
    }

We now have a connected socket and an ID for the `nl80211` channel, so we can
set things up to send commands and receive responses.

## Messages and callbacks

We allocate a netlink message that we will later send:

    struct nl_msg *msg = nlmsg_alloc();
    if (!msg) {
            fprintf(stderr, "Failed to allocate netlink message.\n");
            return -ENOMEM;
    }

We also allocate a callback for which we can register various handlers:

    struct nl_cb *cb = nl_cb_alloc(NL_CB_DEFAULT);
    if (!cb) {
            fprintf(stderr, "Failed to allocate netlink callback.\n");
	    nlmsg_free(msg);
            return -ENOMEM;
    }

This callback will actually list 802.11 interfaces if things work out:

    nl_cb_set(cb, NL_CB_VALID, NL_CB_CUSTOM, list_interface_handler, NULL);

This sets up the generic netlink message:

    genlmsg_put(msg, 0, 0, wifi.nl80211_id, 0,
                    NLM_F_DUMP, NL80211_CMD_GET_INTERFACE, 0);

See `nl80211.h` (and its comments) for a list of commands (`NL80211_CMD_`) and
flags.  We use the `NLM_F_DUMP` flag because we want a dump of all interfaces
(and our callback may be called multiple times), `iw` will do this for certain
commands or for example when printing information about a specified interface
or phy, will instead not set this flag but specify the interface or phy index.

This kicks off the message:

    nl_send_auto_complete(wifi.nls, msg);

And finally, borrowing from `iw`, this lets you know that the `dump` is
finished:

    int err = 1;

    nl_cb_set(cb, NL_CB_FINISH, NL_CB_CUSTOM, finish_handler, &err);

    while (err > 0)
        nl_recvmsgs(wifi.nls, cb);

We will have listed all of the interfaces with calls to
`list_interface_handler` and then we will have been told that we are done by a
call to `finish_handler` when the while loop exits by having `err` cleared.

The `finish_handler` looks like this:

    static int finish_handler(struct nl_msg *msg, void *arg)
    {
        int *ret = arg;
        *ret = 0;
        return NL_SKIP;
    }

Here is a very simple `list_interface_handler` implementation, see
`interface.c` in `iw` for the complete implementation and more ideas:

    static int list_interface_handler(struct nl_msg *msg, void *arg)
    {
        struct nlattr *tb_msg[NL80211_ATTR_MAX + 1];
        struct genlmsghdr *gnlh = nlmsg_data(nlmsg_hdr(msg));

        nla_parse(tb_msg, NL80211_ATTR_MAX, genlmsg_attrdata(gnlh, 0),
                genlmsg_attrlen(gnlh, 0), NULL);

        if (tb_msg[NL80211_ATTR_IFNAME])
            printf("Interface: %s\n", nla_get_string(tb_msg[NL80211_ATTR_IFNAME]));

        return NL_SKIP;
    }

## Cleanup

We should free up the callback, message, and netlink socket:

    nlmsg_free(msg);
    nl_cb_put(cb);

    nl_socket_free(wifi.nls);
