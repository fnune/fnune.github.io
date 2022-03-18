---
layout: post
title: 'Binding devices with libusbip - usairb devlog #2'
comments: true
categories: devlog usairb
date: 2022-03-18 14:56:00 +0200
excerpt: 'Entry #2 of the usairb development log (USB/IP and embedded Linux)'
---

{% include usairb.html %}

Second development entry. On the [first day][devlog-1], I implemented a simple
busy loop that prints to the console whenever a USB device connects or
disconnects, using a `udev` monitor. Today, I'll implement binding those
devices using [USB/IP][usbip], and try to attach to them from a client
device.

A **spoiler** and a word of warning: this didn't go so well and I ended up
giving up on `libusbip` and reverting the changes described in this post,
opting instead for calling the `usbip` CLI directly from my program using pipes
(`popen`). The next entry will be about that. Nevertheless, I learned a little
bit more about sockets, file descriptors and `libudev` as part of this attempt.
Stick around if you'd like to read that!

## Introduction

From the [USB/IP site][usbip]:

> USB/IP Project aims to develop a general USB device sharing system over IP
> network. To share USB devices between computers with their full
> functionality, USB/IP encapsulates "USB I/O messages" into TCP/IP payloads
> and transmits them between computers.

It looks like the [Linux kernel source][usbip-readme] does not include a
library for USB/IP. I'm resorting to this [community `libusbip`][libusbip] for
now.

After spending some time reading its source code, it looks like I have a plan:

- Get a device vendor ID and product ID from each connecting device from my
  `udev` monitor.
  - Handling two physical devices with the same vendor ID and product ID is out
    of scope for now.
- Figure out how to set up a socket in order to initialize USB/IP.
- Use `libusbip_open_device_with_vid_pid` to get a `libusbip_device` like in
  the [example][libusbip-example].
- Confirm that opening and configuring the device actually means the same as
  going `usbip bind` on the command line. I believe because `libusbip` has the
  same interface for the client and the server, it uses the names `open` and
  `close` instead of `bind`, `unbind` (the server terms), and `attach`,
  `detach` (the client terms).

## Getting a product ID and a vendor ID from a `udev_device`

The `udev_device_get_property_value` function seems like a good choice. USB/IP
and `udev` nomenclature differ a little: vendor IDs are called the same but
it's product ID in USB/IP and model ID in `udev`.

To find a list of available keys, I ran `udevadm monitor --udev --env`, and I
get something like this:

```
ID_VENDOR=GenesysLogic
ID_VENDOR_ENC=GenesysLogic
ID_VENDOR_ID=05e3
ID_MODEL=USB2.1_Hub
ID_MODEL_ENC=USB2.1\x20Hub
ID_MODEL_ID=0610
ID_REVISION=9304
ID_SERIAL=GenesysLogic_USB2.1_Hub
```

With this in mind:

```c
printf("%s %s vid:%s pid:%s\n", udev_device_get_action(device),
       udev_device_get_devnode(device),
       udev_device_get_property_value(device, "ID_VENDOR_ID"),
       udev_device_get_property_value(device, "ID_MODEL_ID"));
```

Looking good:

```
add /dev/bus/usb/004/013 vid:090c pid:1000
remove /dev/bus/usb/004/013 vid:(null) pid:(null)
```

Except that when I remove the device, I no longer get a `vid` or a `pid`. I
think that might just be fine for now. I'm not even sure if I need to handle
removals, so let's keep going.

Aside: your computer has a file with a database of known vendor and model IDs
alongside their names. You can find it in `/usr/share/hwdata/usb.ids`. I found
my flash drive `090c:1000`:

```sh
 ~ => grep -A6 '^090c' /usr/share/hwdata/usb.ids
090c  Silicon Motion, Inc. - Taiwan (formerly Feiya Technology Corp.)
        0371  Silicon Motion SM371 Camera
        0373  Silicon Motion Camera
        037a  Silicon Motion Camera
        037b  Silicon Motion Camera
        037c  300k Pixel Camera
        1000  Flash Drive
```

## Initializing USB/IP and learning about sockets

Since this is my first contact with sockets at this level, I'm going to record
my own introduction here.

After a first read of some `man` pages I've managed to gather some interesting
information:

- I can create a socket using the `socket(2)` syscall. The kernel needs to know
  what type of socket I want to create. The [USB/IP `README`][usbip-readme]
  mentions that it works on TCP port 3240, so going by `man socket` I need to
  pass:
  - `AF_INET` as the protocol family,
  - `SOCK_STREAM` (I suppose this is TCP as opposed to `SOCK_DGRAM` which
    should be UDP),
  - `0` for the protocol, since for most combinations of protocol family and
    socket type there is only one available protocol, the value `0`, which
    stands for "do the right thing for me". Let's hope it delivers.
- TCP sockets provide a two-way binding. The `accept(2)` syscall takes a
  connection-based socket (i.e. not `SOCK_DGRAM`/UDP) and gives you a file
  descriptor for a new socket, meant for clients to talk to.
  - Helpfully, it also warns that the original socket needs to have gone
    through some other syscalls first: `bind(2)` and `listen(2)`.

Aside: to get `man socket` I had to install `man-pages` and run `mandb`,
otherwise I was getting Perl documentation, or `No manual entry for socket in section 2` if I passed a section.

After some time, I've got this:

```c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>

int BACKLOG_LENGTH = 1; // Not sure if I need more.

int usairb_connect(int port) {
  int socket_fd = socket(AF_INET, SOCK_STREAM, 0);
  if (socket_fd < 0) {
    fprintf(stderr, "socket returned error code %i\n", socket_fd);
    exit(EXIT_FAILURE);
  }

  struct sockaddr_in socket_address_in = {
      .sin_family = AF_INET,
      .sin_port = htons(port),
      .sin_addr = {.s_addr = INADDR_ANY},
  };
  int socket_len = sizeof(socket_address_in);
  struct sockaddr *socket_address = (struct sockaddr *)&socket_address_in;

  int bind_result = bind(socket_fd, socket_address, socket_len);
  if (bind_result != 0) {
    fprintf(stderr, "bind returned error code %i\n", bind_result);
    exit(EXIT_FAILURE);
  }

  int listen_result = listen(socket_fd, BACKLOG_LENGTH);
  if (listen_result != 0) {
    fprintf(stderr, "listen returned error code %i\n", listen_result);
    exit(EXIT_FAILURE);
  }

  return socket_fd;
}
```

What took me longest was figuring out what I need to pass to `bind`, and
understanding the cast from `sockaddr_in` to `sockaddr` (if you haven't noticed
yet, I'm new to C):

```c
struct sockaddr *socket_address = (struct sockaddr *)&socket_address_in;
```

Generally, I'm also starting to notice that my style of naming variables does
not go well with the style of the kernel API. Maybe one day I'll budge and
start calling a `sockaddr` by its name.

Accepting connections on the socket seems easier:

```c
int usairb_accept(int socket_fd) {
  int client_socket_fd = accept(socket_fd, NULL, NULL);
  if (client_socket_fd < 0) {
    fprintf(stderr, "accept returned error code %i\n", client_socket_fd);
    exit(EXIT_FAILURE);
  }
  return client_socket_fd;
}
```

`man accept` mentions setting `errno` when the return value is `-1`. I haven't
been doing that, and other syscalls I've been using likely report errors via
`errno` as well.

I sprinkled this after each call that may set `errno`:

```c
int listen_result = listen(socket_fd, BACKLOG_LENGTH);
if (errno) {
  fprintf(stderr, "listen produced errno %i\n", errno);
  exit(EXIT_FAILURE);
}
```

Right now, my program runs but produces no output and seems to hang. Even if I
add a `printf` right at the beginning of `main`, it still won't print. I'm not
sure why that happens, but the hang is certainly related to some of the network
syscalls.

Reading `man` pages for `socket`, `bind`, `listen` and `accept`, I found that
`accept` is a blocking call by default. At face value, this makes sense because
we want to wait for connections to listen to:

> If the listen queue is empty of connection requests and O_NONBLOCK is not set
> on the file descriptor for the socket, accept() shall block until a
> connection is present. If the listen() queue is empty of connection requests
> and O_NONBLOCK is set on the file descriptor for the socket, accept()
> shall fail and set errno to [EAGAIN] or [EWOULDBLOCK].

If I do this in `usairb_accept`:

```c
int socket_fd_flags = fcntl(socket_fd, F_GETFL);
fcntl(socket_fd, F_SETFL, socket_fd_flags | O_NONBLOCK);
```

Then the program does not hang. However, as vaticinated by `man accept`, I get
this:

```c
 ~/Development/usairb [127] => ./target/usairb
listen produced errno 11
```

Running `strace ./target/usairb` produces some clearer output:

```
socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(3240), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 1)                            = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
accept(3, NULL, NULL)                   = -1 EAGAIN (Resource temporarily unavailable)
write(2, "accept produced errno 11\n", 25accept produced errno 11
) = 25
exit_group(1)                           = ?
+++ exited with 1 +++
```

So the `listen` queue is empty of connection requests and `O_NONBLOCK` is set
on the file descriptor for the socket, so I'm getting `[EAGAIN]`. I need to
find a way to call `accept` only if there's something in the connection request
queue.

After some sleuthing, I found this [comment on StackOverflow][so-comment]
suggesting a way: I can use `select` to check for listening connections and
then call `accept` only if there's a request to serve. On further inspection,
this is exactly what `man accept` suggests as well:

```
APPLICATION USAGE
       When  a  connection is available, select() indicates that the file
       descriptor for the socket is ready for reading.
```

I also found a great [article by Julia Evans][julia-evans-select] introducing
this topic. It recommends `epoll`, but I'm going to go vanilla for learning
purposes and use `select`.

[julia-evans-select]: https://jvns.ca/blog/2017/06/03/async-io-on-linux--select--poll--and-epoll/
[so-comment]: https://stackoverflow.com/questions/12861956/is-it-possible-and-safe-to-make-an-accepting-socket-non-blocking#comment17408534_12862015

## Getting stuck

After getting something hacked together using `select` and getting the program
to compile and run without hanging, I still get no effect that's visible from
the `usbip` tool (e.g. running `usbip list -r localhost` does not return
anything). I'm starting to lose hope in `libusbip`. After all, it hasn't been
updated in ten years. I'm going to try a different path: simply calling the
`usbip` executable from my program. There'll be something to learn by doing it
that way, too.

My consolation is that I found out about vendor and model IDs and how to get
them with `libudev`. I also had a first contact with sockets in C and got a
tiny bit more experience working with file descriptors.

On the next post I'll be trying to execute the `usbip` CLI from my program and
act on its output. See you then!

[libusbip-example]: https://github.com/forensix/libusbip/blob/master/examples/idev_cmd.c#L114
[libusbip]: https://github.com/forensix/libusbip
[usbip]: http://usbip.sourceforge.net
[usbip-readme]: https://github.com/torvalds/linux/blob/master/tools/usb/usbip/README
[devlog-1]: /devlog/usairb/2022/02/05/listening-to-devices-with-libudev-usairb-devlog-1/
