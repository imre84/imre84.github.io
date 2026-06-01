---
layout: post
title: "The kernel ip= parameter: what each field actually does"
description: "What each field of the kernel ip= boot parameter actually does, with notes from setting it up for Dropbear-in-initramfs remote LUKS unlock"
date: 2026-06-01 18:00:00 +0200
tags: [linux, initramfs, dropbear, networking]
---

I was writing an automated Proxmox installer with full-disk encryption and remote SSH unlock, which means a Dropbear SSH server inside the initramfs, which means the initramfs needs a working network stack, which means filling in this kernel parameter correctly. The format took some untangling. Here's what I learned.

According to [official documentation](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/nfs/nfsroot.rst), my whole article in a single line is as follows:

```
ip=<client-ip>:<server-ip>:<gw-ip>:<netmask>:<hostname>:<device>:<autoconf>:<dns0-ip>:<dns1-ip>:<ntp0-ip>
```

I've investigated that from the viewpoint of Dropbear LUKS unlocking. This is what I have:

```
ip=<client-ip>::<gw-ip>:<netmask>::<device>:off
```

One caveat though: the Dropbear initramfs config file (`/etc/initramfs-tools/conf.d/dropbear-ip.conf`) uses uppercase `IP=`, while the kernel uses lowercase `ip=`.

On my system this parameter is in the aforementioned file, however providing an alternative `ip=` from kernel commandline using grub overrides the contents of that file. Good for testing.

The fields I've populated:

* `client-ip`: you want to ssh into your initramfs and/or you want your initramfs to generate traffic, you need it.
* `gw-ip`: If you plan to connect only from LAN, you don't need this, however if you're behind NAT and the relevant port is forwarded, in order for the reply packets to reach the gateway (and then the client) you need this. (I tested it, in hindsight it seems obvious, when establishing the connection a SYN ACK packet is being sent with DST:\<external ip of the client>. Unless you have a gateway set up, that packet has nowhere to go)
* `netmask`: it needs to be in the format `255.255.255.0` and not like `24`
* `device`: the name of the ethernet device, in my case it's `enp0s3`, look at the output of `ip a` to figure out yours.
* `autoconf`: the autoconf method, in my case it's `off`. Can be `off` or `none`, `on` or `any` (default, meaning any of the next three), `dhcp`, `bootp`, `rarp`, and `both` meaning both bootp and rarp.

The fields I haven't populated:

* `server-ip`: if your rootfs is an NFS this is the NFS server
* `hostname`: according to the doc it might be used in the dhcp request
* `dns0-ip`, `dns1-ip`: maybe you need dns for the hostname resolution for your nfs host. I can't think of any other reason. Once your machine properly boots up, it might end up overriding all of these settings anyway
* `ntp0-ip`: as per official docs: "Value is exported to `/proc/net/ipconfig/ntp_servers`, but is otherwise unused" and then later on "Note that the kernel will not synchronise the system time with any NTP servers it discovers; this is the responsibility of a user space process". Maybe somehow this value can survive the boot process and get used by the ntp daemon, other than that, seems to be completely superfluous.

## `24` vs `255.255.255.0`
CIDR stands for Classless Inter-Domain Routing, `127.0.0.1/8` is what we call the CIDR style IP notation. However the fourth parameter in `ip` is a dotted netmask. When the ip parsing was written in the kernel, the dotted representation was still the norm, and kernel wants to maintain backwards compatibility.

## Conclusion

In this article we walked through the kernel `ip` parameter field by field. The format is a small fossil, but a useful one to know if you ever want to talk to a kernel before user-space exists. Like in this example, booting up Dropbear from initramfs. In this case we need only half of the fields. Now we know which ones, and why.
