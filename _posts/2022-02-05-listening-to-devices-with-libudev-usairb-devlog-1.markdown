---
layout: post
title: 'Listening to devices with libudev - usairb devlog #1'
comments: true
categories: devlog usairb
date: 2022-02-05 12:43:00 +0200
excerpt: "Entry #1 of the usairb development log (USB/IP and embedded Linux)"
---

{% include usairb.html %}

First development day. Added an [empty
`CHANGELOG`](https://github.com/fnune/usairb/blob/main/CHANGELOG.md) and a
[`README`](https://github.com/fnune/usairb/blob/main/README.md), and started
this development log.

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

## Listening to USB devices as they connect and disconnect

I found this helpful post: [`libudev` and Sysfs
Tutorial][libudev-sysfs-tutorial]. I won't be following it so much, but it
seems like a good example. Running `man libudev` also seems helpful.

As a first step, I'm going to implement a busy loop that listens to connection
and disconnection events and just logs any interesting information to the
console.

From `man libudev`:

> To monitor the local system for hotplugged or unplugged devices, a monitor
> can be created via `udev_monitor_new_from_netlink`(3).

Looks like `udev_monitor_new_from_netlink` takes a pointer to `udev` and a
`name`. To get a `udev` context object, I can call `udev_new`.

Aside: to set up a bit of a quicker feedback loop, I'm running
[`watchexec`][watchexec]: `watchexec -c -w src -e c 'make && ./target/usairb'`.

After adding `-ludev` to my Makefile, I wrote this:

```c
#include <libudev.h>
#include <stdio.h>
#include <stdlib.h>

const char *UDEV_MONITOR_NAME = "usairb-udev-monitor";

int main(void) {
  struct udev *udev = udev_new();
  struct udev_monitor *udev_monitor =
      udev_monitor_new_from_netlink(udev, UDEV_MONITOR_NAME);

  if (!udev_monitor) {
    fprintf(stderr, "udev_monitor_new_from_netlink returned NULL\n");
    exit(EXIT_FAILURE);
  }
}
```

My error message came to fruition instantly. `man libudev` says "on failure, `NULL` is returned", but doesn't mentions what the reasons for failure are.

I'm wondering if it's `udev_new` that's failing:

```c
if (!udev) {
  fprintf(stderr, "udev_new returned NULL\n");
  exit(EXIT_FAILURE);
}
```

No, it's our other friend:

```
udev_monitor_new_from_netlink returned NULL
[[Command exited with 1]]
```

I solved it out of sheer luck: I changed the value passed to `name` to
`"udev"`, and now it's working just fine. The takeaway: the `name` param is the
name of a known [Netlink][netlink]. A Netlink is similar to a domain socket and
is used for IPC. It looks like there's one for `"udev"`, and you need to
specify that.

Now I'm looking into what I can do with `udev_monitor`. The manual only
includes entries for functions (such as `udev_new`) but not for structs. Maybe
the monitor is meant to be passed around to other functions. How can I find
those? My copy of [The Linux Programming Interface][tlpi] by Michael Kerrisk
only introduces `udev` at a high level but doesn't go into detail. I guess it's
time to read examples on the Internet, unfortunately. This makes me miss the
Rust ecosystem. Documentation generated by `rustdoc` would make
cross-referencing searches very easy.

One practical solution I've found (besides reading existing work on the Internet) is to let my shell help me: if I type `udev_monitor_` and ask for an autocompletion with tab, I get this:

```sh
 ~/Development/usairb => man udev_monitor_
udev_monitor_enable_receiving                    udev_monitor_filter_update                       udev_monitor_receive_device
udev_monitor_filter_add_match_subsystem_devtype  udev_monitor_get_fd                              udev_monitor_ref
udev_monitor_filter_add_match_tag                udev_monitor_get_udev                            udev_monitor_set_receive_buffer_size
udev_monitor_filter_remove                       udev_monitor_new_from_netlink                    udev_monitor_unref
```

That's good enough. I can also use [LunarVim][lvim]'s LSP hints. Two functions
look interesting: `udev_monitor_receive_device` and
`udev_monitor_enable_receiving`. The latter sounds like a prerequisite, but I'm
going to go without it at first and see what happens.

```c
struct udev_device *device = udev_monitor_receive_device(udev_monitor);
printf("Prints if `udev_monitor_receive_device` is not blocking.");
```

It printed. I added a call to `udev_monitor_enable_receiving`, but the behavior
didn't change. Looks like my assumption that it's blocking isn't holding up!

After some sleuthing in `systemd` source code, I found [this comment][libudev-monitor-nonblock] that confirms that the call is non-blocking, and suggests two possible paths forward:

- use a variant of `poll()` on the file descriptor returned by `udev_monitor_get_fd`, or
- switch the file descriptor into blocking mode.

I don't know how to do either of these things, so it's time for some more reading.

[TLPI][tlpi] to the rescue! I can use `fcntl` to modify flags on the file descriptor returned by `udev_monitor_get_fd`.

Got it to work thanks to some bitwise-fu from [a StackOverflow answer][so-make-fd-blocking]:

```c
// udev_monitor_receive_device is non-blocking. To make it blocking,
// get the monitor file descriptor and unset its O_NONBLOCK flag.
int udev_monitor_fd = udev_monitor_get_fd(udev_monitor);
int udev_monitor_fd_flags = fcntl(udev_monitor_fd, F_GETFL);
fcntl(udev_monitor_fd, F_SETFL, udev_monitor_fd_flags & ~O_NONBLOCK);
```

I wrote a busy loop:

```c
while (1) {
  printf("Listening for new devices...\n");

  struct udev_device *device = udev_monitor_receive_device(udev_monitor);

  if (device) {
    printf("Found a device!\n");
  }
}
```

Aside: I added `-r` to my `watchexec` command so that it restarts the process
every time, otherwise the busy loop never exits.

The behavior is not quite what I expected: my program logs `"Found a device!"`
but it seems really excited about it and does it many times over on every
connection. Additionally, it does it when I disconnect a device. It feels like
a race condition.

To find out, I added a `sleep(1);` right after `printf("Found a device!\n");`.
To my surprise, my program still "finds the device" a bunch of times. Without
`sleep(1);`, it prints roughly the same amount of times.

After reading the documentation a bit more, I found some new clarifications:

- The existence of `udev_device_get_parent` pointed out the fact that what I'm
  seeing is a tree of devices, and not a single device. To find the root, I
  looked for a device with no parent, and this returned the expected: only one
  device. The root device always belongs to the `bdi` subsystem.
- The return value of `udev_device_get_action` may contain values like `"add"`,
  `"remove"`, `"bind"` or `"unbind"`, among others. I should probably start
  calling my variables `event` instead of `device`.

If I run `udevadm monitor --udev`, I get exactly the same output (well, in a different format):

```sh
 ~/Development/usairb => udevadm monitor -u
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing

UDEV  [8858.969668] add      /devices/virtual/workqueue/scsi_tmf_0 (workqueue)
UDEV  [8858.971981] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6 (usb)
UDEV  [8858.973386] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0 (usb)
UDEV  [8858.974132] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0 (scsi)
UDEV  [8858.974950] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/scsi_host/host0 (scsi_host)
UDEV  [8858.975633] bind     /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0 (usb)
UDEV  [8858.978815] bind     /devices/pci0000:00/0000:00:14.0/usb2/2-6 (usb)
UDEV  [8860.074537] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0 (scsi)
UDEV  [8860.075845] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0 (scsi)
UDEV  [8860.075899] add      /devices/virtual/bdi/8:0 (bdi)
UDEV  [8860.077712] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0/scsi_device/0:0:0:0 (scsi_device)
UDEV  [8860.078456] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0/bsg/0:0:0:0 (bsg)
UDEV  [8860.078522] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0/scsi_disk/0:0:0:0 (scsi_disk)
UDEV  [8860.131761] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0/block/sda (block)
UDEV  [8860.244986] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0/block/sda/sda2 (block)
UDEV  [8860.258575] add      /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0/block/sda/sda1 (block)
UDEV  [8860.259622] bind     /devices/pci0000:00/0000:00:14.0/usb2/2-6/2-6:1.0/host0/target0:0:0/0:0:0:0 (scsi)
```

Looks like new devices are added for different things: the partitions in my USB
flash drive, some accessors for protocols I don't understand (SCSI), etc.

Since they all point to the same thing in `/sys/devices`, I think I can just
filter for events for which `udev_device_get_devtype` returns `"usb_device"`.

```c
const char *device_type = udev_device_get_devtype(device);
if (!device_type || strcmp(device_type, "usb_device") != 0) {
  continue;
}

printf("Found a USB device:\n");
// ...
```

Now, I get only two events for each physical action: `add` and `bind` (in that
order) when plugging in the device and `remove` and `unbind` when unplugging
it. For now, I'm only going to care about `add` and `remove`, and we'll see
about this later!

Currently, the output looks like this when I plug a USB flash drive in and out:

```
Found a USB device:
   Node: /dev/bus/usb/002/026
   Subsystem: usb
   Devtype: usb_device
   Action: add
Found a USB device:
   Node: /dev/bus/usb/002/026
   Subsystem: usb
   Devtype: usb_device
   Action: remove
```

When I do the same with my USB hub, I get two distinct devices:

```
Found a USB device:
   Node: /dev/bus/usb/001/026
   Subsystem: usb
   Devtype: usb_device
   Action: add
Found a USB device:
   Node: /dev/bus/usb/004/015
   Subsystem: usb
   Devtype: usb_device
   Action: add
Found a USB device:
   Node: /dev/bus/usb/001/026
   Subsystem: usb
   Devtype: usb_device
   Action: remove
Found a USB device:
   Node: /dev/bus/usb/004/015
   Subsystem: usb
   Devtype: usb_device
   Action: remove
```

This is because, like we saw earlier, my hub registers once as a USB 2.0 device
and once again as a USB 3.0 device.

I'm calling this one a success! Next time I get a chance to work on this, I'll
be working on binding these devices automatically using `libusbip`. Thanks for
reading!

[tlpi]: https://man7.org/tlpi/
[netlink]: https://man7.org/linux/man-pages/man7/netlink.7.html
[watchexec]: https://crates.io/crates/watchexec-cli
[man-libudev]: https://www.freedesktop.org/software/systemd/man/libudev.html
[libudev-sysfs-tutorial]: http://cholla.mmto.org/computers/usb/OLD/tutorial_usbloger.html
[libudev-monitor-nonblock]: https://github.com/systemd/systemd/blob/be1eae4fad5562da5cb784c121981206d1b77254/src/libudev/libudev-monitor.c#L222-L240
[so-make-fd-blocking]: https://stackoverflow.com/questions/914463/how-to-make-a-file-descriptor-blocking
[lvim]: https://www.lunarvim.org/
