---
published: false
layout: post
title: 'Planning a proof of concept - usairb devlog #1'
comments: true
categories: devlog usairb
date: 2022-02-05 12:43:00 +0200
excerpt: "Entry #1 of the usairb development log (USB/IP and embedded Linux)"
---

{% include usairb.html %}

First development day. Added an [empty
`CHANGELOG`](https://github.com/fnune/usairb/CHANGELOG.md) and a
[`README`](https://github.com/fnune/usairb/CHANGELOG.md), and started this
development log.

## Introduction

I wrote this wishful piece for the `README`:

> The goal of [`usairb`][usairb-repo] (Universal Serial Air-Bus) is to transform
any embedded Linux device with access to the Internet into a multiplexing
transmitter for USB hubs: connect gadgets to it and use them remotely from your
desktop.

A quick tech overview:

> To achieve this, `usairb` uses [USB/IP][usbip]. USB/IP follows a server-client
architecture where the server or host is the device broadcasting its USB
gadgets, and the client can connect to them. USB/IP is available as a native
Kernel module on Linux for the host, and has multi-platform client programs.

And a justification:

> While all planned features of `usairb` are achievable using just USB/IP,
`usairb` aims to provide a no-frills experience, potentially offering both a
client graphical user interface as well as very simple interface for the host
device.

I guess you can call it a no-frills experience if there's no product for you to
use!

[usairb-repo]: https://github.com/fnune/usairb
[usbip]: https://wiki.archlinux.org/title/USB/IP

## Planning features of the host device

To build a PoC that broadcasts devices automatically, and that can be used from
a client alongside the `usbip` command-line, interface, the host needs the
following features:

- Bind only leaf devices. Avoid binding an entire USB hub.
  - Recognize connected USB hubs.
    - Eventually, reconsider the idea of not binding entire USB hubs. I'm
      deciding this right now because I don't know how USB hubs will behave.
  - Eventually, allow the user to bind and unbind specific USB hubs.
    - Remember this across boots. Maybe using `systemd` or a SQLite database?
- Listen to leaf USB devices when they connect and disconnect.
  - Use `libudev` for this.
- Automatically bind and unbind USB devices.
  - Call `usbip bind` for this, using `libusbip`.

## Telling apart leaf USB devices from USB hubs

Running `lsusb` with the `--tree` option returns this:

```diff
--- log-without-hub     2022-02-05 12:26:40.311210251 +0100
+++ log-with-hub        2022-02-05 12:26:35.234559166 +0100
@@ -1,11 +1,15 @@
 /:  Bus 04.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/2p, 10000M
     ID 1d6b:0003 Linux Foundation 3.0 root hub
+    |__ Port 1: Dev 2, If 0, Class=Hub, Driver=hub/4p, 5000M
+        ID 05e3:0620 Genesys Logic, Inc. GL3523 Hub
 /:  Bus 03.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/2p, 480M
     ID 1d6b:0002 Linux Foundation 2.0 root hub
 /:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/10p, 10000M
     ID 1d6b:0003 Linux Foundation 3.0 root hub
 /:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/16p, 480M
     ID 1d6b:0002 Linux Foundation 2.0 root hub
+    |__ Port 4: Dev 5, If 0, Class=Hub, Driver=hub/4p, 480M
+        ID 05e3:0610 Genesys Logic, Inc. Hub
     |__ Port 8: Dev 2, If 3, Class=Video, Driver=uvcvideo, 480M
         ID 04f2:b6be Chicony Electronics Co., Ltd
     |__ Port 8: Dev 2, If 1, Class=Video, Driver=uvcvideo, 480M
```

The Genesys USB hub shows up twice, under two different trees: first the 3.0
root hub, and then the 2.0 root hub. If I connect a USB 3.0 device to the hub, it doesn't matter which port I connect it to, it always shows up under the 3.0 tree of the root hub in my laptop:

```diff
--- log-with-hub        2022-02-05 12:26:35.234559166 +0100
+++ log-with-device     2022-02-05 12:37:10.480333271 +0100
@@ -1,14 +1,16 @@
 /:  Bus 04.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/2p, 10000M
     ID 1d6b:0003 Linux Foundation 3.0 root hub
-    |__ Port 1: Dev 2, If 0, Class=Hub, Driver=hub/4p, 5000M
+    |__ Port 1: Dev 3, If 0, Class=Hub, Driver=hub/4p, 5000M
         ID 05e3:0620 Genesys Logic, Inc. GL3523 Hub
+        |__ Port 2: Dev 7, If 0, Class=Mass Storage, Driver=usb-storage, 5000M
+            ID 090c:1000 Silicon Motion, Inc. - Taiwan (formerly Feiya Technology Corp.) Flash Drive
 /:  Bus 03.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/2p, 480M
     ID 1d6b:0002 Linux Foundation 2.0 root hub
 /:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/10p, 10000M
     ID 1d6b:0003 Linux Foundation 3.0 root hub
 /:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/16p, 480M
     ID 1d6b:0002 Linux Foundation 2.0 root hub
-    |__ Port 4: Dev 5, If 0, Class=Hub, Driver=hub/4p, 480M
+    |__ Port 4: Dev 6, If 0, Class=Hub, Driver=hub/4p, 480M
         ID 05e3:0610 Genesys Logic, Inc. Hub
     |__ Port 8: Dev 2, If 3, Class=Video, Driver=uvcvideo, 480M
         ID 04f2:b6be Chicony Electronics Co., Ltd
```

It looks like `lsusb` has a way to tell apart hubs from leaf devices. I can
probably simply look at the device `Class` and check to see if it's `root_hub`
or `Hub`. This could enable letting the user pick whether to bind a USB hub or
their root hub (any ports on the host device itself).

I'm still not certain as to whether I can get this `Class` information directly
from `libudev` or I'll need to include some other dependency.
