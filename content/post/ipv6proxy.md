---
title: "IPv6 to IPv4 proxy with xinetd"
date: 2018-12-14T19:48:02+01:00
categories: ["Projects"]
tags: ["ipv6", "ipv4", "proxy", "xinetd"]
image: "img/code.png"
draft: false
---

If you have to deal with legacy applications that do not have native IPv6
support and you need to access via IPv6 there is a way to (ab-)use xinetd to
listen for the incoming IPv6 traffic and forward it to the IPv4 port.

In this example we are going to use the Web interface of Transmision, which
currently can't be configured to be accessible via IPv6, but other things, like
the Spice proxy of Proxmox suffer the same issue.

The tutorial is for Debian, but other distros should work as well. We further
assume, that transmission is already installed and the webinterface works via
IPv4. It should look something like this:

    # netstat -tlpn | grep 9091
    tcp   0   0 0.0.0.0:9091     0.0.0.0:*        LISTEN   1492/transmission-d

First we need to install `xinetd` if it's not already installed:

    # apt install xinetd

Next we need to create configuration file `/etc/xinet.d/transmission6` for our
service:

    service transmission6
    {
        flags           = IPv6
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = 127.0.0.1 9091
        port            = 9092
    }

And then reload `xinetd`:

    # systemctl reload xinetd

The result will be:

    # netstat -tlpn | grep 9092
    tcp6  0   0 :::9092          :::*             LISTEN   7663/xinetd

You will notice, that for IPv6 we are using a different port number. That is
because otherwise xinetd will complain with something like this:

    xinetd: bind failed (Address already in use (errno = 98)). service = transmission6
    xinetd: Service transmission6 failed to start and is deactivated.

This is because xinetd sees that something is already listening on the (IPv4)
port 9091. The `flags = IPv6` doesn't seem to have much of an effect. Adding:

    bind            = ::

to the service definition (the equivalent of 0.0.0.0 in IPv4) does not work
either.

If you absolutely need to use the same port for IPv4 and IPv6 its necessary to
specify the exact IPv6 address xinetd should bind the service to like this:

        bind            = 2001:DB8:DEAD::BEEF
        ...
        port            = 9091

If your IPv6 address additionally changes over time (changing network prefix
etc.) you have to regulary update the service definition and reload xinetd:

    #!/bin/sh

    address=$(ip -6 addr list scope global dev eth0 | grep -v " fd" | sed -n 's/.*inet6 \([0-9a-f:]\+\).*/\1/p' | head -n 1)
    sed "s/%BIND%/$address/" $HOME/transmission6 > /etc/xinetd.d/transmission6
    systemctl reload xinetd

The `$HOME/transmission6` file in that case is your template for the service
definition, in which `%BIND%` will be replaced with you current IPv6 address.
