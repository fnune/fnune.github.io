---
layout: post
title: 'Embedded learning log - A foray into Linux specifics: dmesg, tty and playing with minicom'
comments: true
categories: [embedded-learning-log]
date: 2021-02-20 20:22:51 +0200
excerpt: 'In a previous post in this log I used `sudo dmesg | grep tty` to find the serial port to which my board was connected. But what does `dmesg` do and why am I looking for `tty`?'
---

{% include embedded-learning-log.html %}

In a previous post in this log I used `sudo dmesg | grep tty` to find the serial port to which my board was connected. But what does `dmesg` do and why am I looking for `tty`?

Here's what I know about `dmesg`: it prints logs from the kernel. And here's all my loose bits of unchecked knowledge about `tty`:

- It stands for TeleTypewriter.
- In Linux, I can go to a different `tty` by using `CTRL+ALT+F#`.
- Some of the `#`s are used by specific processes. In my case, `CTRL+ALT+F1` takes me to GDM (which shows my log-in screen), and `CTRL+ALT+F2` takes me back to my current Xorg session.
- I've used `CTRL+ALT+F#` with numbers other than 1 and 2 to get me out of problems with me Xorg session.
- I suppose that means processes are tied to `tty`s, or `tty`s are somehow pointers to processes.

Time to strengthen this knowledge.

```bash
man tty
```

> `tty` - print the file name of the terminal connected to standard input

```bash
~/Development/toastboard/hello_world => tty
/dev/pts/3
```

I have three terminals open, and that one was number 3 apparently.

A TTY is transitively [a software emulation of a TeleTypewriter](https://www.howtogeek.com/428174/what-is-a-tty-on-linux-and-how-to-use-the-tty-command/), or a pseudoterminal. In Linux, there's the concept of a pseudoterminal master and pseudoterminal slaves.

> The file /dev/ptmx is a character file with major number 5 and minor number 2, usually of mode 0666 and owner.group of root.root. It is used to create a pseudoterminal master and slave pair. When a process opens /dev/ptmx, it gets a file descriptor for a pseudoterminal master (PTM), and a pseudoterminal slave (PTS) device is created in the /dev/pts directory. Each file descriptor obtained by opening /dev/ptmx is an independent PTM with its own associated PTS, whose path can be found by passing the descriptor to ptsname(3).

I suppose this is what terminal emulators such as Kitty do when they open: they get assigned a file under `/dev/pts`. It must also be what `tmux` does when I open a new pane or window.

So now I know! `sudo dmesg` prints kernel logs and we try to find matches for `tty`, which should give us logs for when devices connected. When devices connect, Linux will create files for them under `/dev`, and these show up as `/dev/tty*`, in my case `/dev/ttyUSB0` because my device is the first on the USB serial port adapter.

### Playing with `minicom`

On a first attempt to verify that serial communication to my development board works, I did this:

```
 ~/Development/toastboard/hello_world => minicom -p /dev/ttyS4
minicom: argument to -p must be a pty
```

A `pty` is a pseudoterminal. I guess if I open a new terminal, it'll be assigned `/dev/pts/4` and then I can communicate to it? I'm curious to try it out.

Open a new terminal and then on terminal `/dev/pts/3`:

```
minicom -p /dev/pts/4
```

Type some stuff... and it appears on my newest terminal! Cool! Trying to type on `/dev/pts/4` produces the same characters on `/dev/pts/3`, but some get dropped. I'm curious to find out why, but let's get back to [the main course]({% post_url 2021-02-21-embedded-learning-log-3-running-a-recurring-freertos-task %}).
