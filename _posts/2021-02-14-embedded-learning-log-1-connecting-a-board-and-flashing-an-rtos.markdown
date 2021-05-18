---
layout: post
title: 'Embedded learning log - Connecting a board and flashing an RTOS'
comments: true
categories: [embedded-learning-log]
date: 2021-02-14 19:22:51 +0200
excerpt: 'I learn how to connect an ESP8266 board to my computer and flash the ESP8266_RTOS_SDK onto it.'
---

{% include embedded-learning-log.html %}

> Browse the accompanying repository [at the first commit](https://github.com/fnune/toastboard/tree/f237fa412d0d07d87962aa1f41e6281efff51f30) or [at the commit in which I managed to run the `hello_world` project](https://github.com/fnune/toastboard/tree/3a6f273e5a9df8ae388962348052cf6833d9812c).

I'm a web developer, so this is all new to me. I hope this log turns out to be an interesting read for experienced embedded systems developers and a way to help newbies like me feel less lonely in their adventure.

## Installing the ESP8266_RTOS_SDK toolchain

[Here's the documentation](https://docs.espressif.com/projects/esp8266-rtos-sdk/en/latest/get-started/linux-setup.html).

I'm surprised that I need to download tarfiles.

- I want tooling that helps me achieve reproducible builds.
- The Espressif docs encourage me to install things in `~/esp` and then add them to my path. What if I lose access to this version of the toolchain and I want to run this in the future?
- There seems to be no unifying tool, I guess the ecosystem for each board is a different world.

---

A first `ls` in the binaries offered by the toolchain gives me this:

```bash
 ~/Development/toastboard => ls ~/.esp/xtensa-lx106-elf/bin
xtensa-lx106-elf-addr2line  xtensa-lx106-elf-c++filt       xtensa-lx106-elf-gcc         xtensa-lx106-elf-gcov       xtensa-lx106-elf-ld       xtensa-lx106-elf-ranlib
xtensa-lx106-elf-ar         xtensa-lx106-elf-cpp           xtensa-lx106-elf-gcc-8.4.0   xtensa-lx106-elf-gcov-dump  xtensa-lx106-elf-ld.bfd   xtensa-lx106-elf-readelf
xtensa-lx106-elf-as         xtensa-lx106-elf-ct-ng.config  xtensa-lx106-elf-gcc-ar      xtensa-lx106-elf-gcov-tool  xtensa-lx106-elf-nm       xtensa-lx106-elf-size
xtensa-lx106-elf-c++        xtensa-lx106-elf-elfedit       xtensa-lx106-elf-gcc-nm      xtensa-lx106-elf-gdb        xtensa-lx106-elf-objcopy  xtensa-lx106-elf-strings
xtensa-lx106-elf-cc         xtensa-lx106-elf-g++           xtensa-lx106-elf-gcc-ranlib  xtensa-lx106-elf-gprof      xtensa-lx106-elf-objdump  xtensa-lx106-elf-strip
```

I can only guess what some of those things are for. I haven't read the full documentation yet. Here are my **guesses**:

- `xtensa-lx106` is the architecture, just like my computer's is `x86_64`?
- `xtensa-lx106-elf` specifies the ELF format these utilities deal with. I know about ELF files, but I'm not sure if they can be in different formats or not, or which part is different.
- I assume `*-gdb` is an `xtensa-lx106`-specific build of GDB. It would be interesting to see if there's a "default" version of GDB.
- `*-gprof` is for profiling.
- `*-objdump` is to show me the content of a built executable in a more human-readable way?
- `*-strip`, `*-strings`, `*-ranlib` all sound like libs I can use in my programs.
- `*-gcov` is a test coverage tool?
- `*-gcc` is the compiler! I'm pretty sure about that one! Is `*-cc` just an alias?
- `*-strip` is for removing or renaming symbols?

Some of these tools are already available in my `PATH` but not specifically for `xtensa-lx106-elf`, I guess because I've installed `build-essential`: `gcc`, `strip`, `objdump`, `objcopy`... what do these binaries work on? `x86_64` Linux?

---

The "supported" IDE is Eclipse, but the docs start with this:

> We suggest building a project from the command line first, to get a feel for how that process works. You also need to use the command line to configure your ESP8266_RTOS_SDK project (via `make menuconfig`), this is not currently supported inside Eclipse.

That means I'll get introduced to interacting with the project via command line, so that's good.

## Downloading ESP8266_RTOS_SDK

[Here's the documentation](https://docs.espressif.com/projects/esp8266-rtos-sdk/en/latest/get-started/index.html#get-esp8266-rtos-sdk).

Once again, the documentation suggests I just dump the whole RTOS into `~/esp`. Should I be using submodules instead? Maybe for the toolchain, too. The next developer that works on this is inevitably going to clone `master` at a different commit. Let's just follow along for now.

> The toolchain programs access ESP8266_RTOS_SDK using IDF_PATH environment variable. This variable should be set up on your PC, otherwise projects will not build. Setting may be done manually, each time PC is restarted. Another option is to set up it permanently by defining IDF_PATH in user profile.

I suppose this means I need `export IDF_PATH=~/esp/ESP8266_RTOS_SDK` in my `.zshrc`.

[Next up](https://docs.espressif.com/projects/esp8266-rtos-sdk/en/latest/get-started/index.html#install-the-required-python-packages), I'm installing some Python packages globally. I think if I wanted to make these builds reproducible I'd have to go through a lot more steps than those described in the documentation. Or maybe these tools just don't change so much?

## Running the `hello_world` example project

At first, I was surprised that `make menuconfig` is what I'm supposed to run here, because that target is not in the `Makefile`, but these are its contents:

```
PROJECT_NAME := hello-world

include $(IDF_PATH)/make/project.mk
```

I didn't know you can include `Makefile`s from other `Makefile`s.

### python vs python2 vs python3

Running `make menuconfig` fails because it expects `python` in `PATH`, but I have `python3`. I also installed the required libraries using `python3`, so I'm going to install `python` (which in Ubuntu means 2.7) and reinstall the dependencies using `python`. I think on Arch `python` points to Python 3.

After doing that, I get:

```
pkg_resources cannot be imported probably because the pip package is not installed and/or using a legacy Python interpreter. Please refer to the Get Started section of the ESP-IDF Programming Guide for setting up the required packages.
```

Apparently `pip` points to `python3-pip` in my system, so now I have to find out whether this works after installing `python2-pip` (I'm guessing that's the name) or if I do need everything on Python 3 and I need to solve the `python` vs `python3` name problem in some other way.

I ended up using `alias python=python3` and running `sudo apt install python-is-python3` because the SDK depends on `/usr/bin/env python`, making my alias useless. I guess they should have just fixed that to `/usr/bin/env python3`.

### Back in business: finding my device

The docs ask me to do this:

> You are almost there. To be able to proceed further, connect ESP8266 board to PC, check under what serial port the board is visible and verify if serial communication works. Note the port number, as it will be required in the next step.

As if I knew how to do that! Time to DuckDuckGo.

It seems like the Internet has settled on this command:

```bash
~/Development/toastboard [2] => sudo dmesg | grep tty
[    0.183219] printk: console [tty0] enabled
[    0.784336] 0000:00:16.3: ttyS4 at I/O 0x3060 (irq = 19, base_baud = 115200) is a 16550A
[  594.954203] usb 1-1: cp210x converter now attached to ttyUSB0
[  679.763000] cp210x ttyUSB0: cp210x converter now disconnected from ttyUSB0
```

At first, I thought my device was connected to `/dev/ttyS4` because `sudo dmesg | grep tty` showed it alongside a baud rate, but perhaps that's the default for the port? As it turns out, my device is on `/dev/ttyUSB0`.

Part one done: my device is on `/dev/ttyUSB0`.

### Communicating over serial

As for verifying that serial communication works: I've read mentions of `minicom`, so I'm going to try it out. After some configuration, my chip starts to send me a dot character every second, which is the serial output of a previous Arduino program I wrote, which I had already flashed with the Arduino IDE. It's working! Here's the command I ran:

```bash
sudo minicom -b 115200 -o -D /dev/ttyUSB0
```

I run `make menuconfig`. The serial port is set to `/dev/ttyUSB0`. The generated configuration looks really specific to my laptop, so I'm going to put it in `.gitignore`. It seems like other developers would have to just run `make menuconfig` on their machines, too.

The output of `make flash` looked promising until I got a `Permission denied: /dev/ttyUSB0` error. I suppose I can run `sudo make flash`, but that seems a bit excessive.

```
~/Development/toastboard/hello_world [2] => ls -l /dev/ttyUSB0
crw-rw---- 1 root dialout 188, 0 Feb 14 21:09 /dev/ttyUSB0
```

If I add myself to the `dialout` group, I should have access to my device. Thanks SO.

```
sudo usermod -a -G dialout $USER
```

After a reboot (surprisingly, logging out and then logging back in didn't work), I could flash the RTOS onto the device. It's now blinking its LED very infrequently.

The serial communication output is interesting:

```
rnn||bll
```

It seems to then print some more `rnn||bll` lines, then clears itself and restarts when the LED blinks.

### Baud rates

Reading the code in [`hello_world_main.c`](hello_world/main/hello_world_main.c) I can't find much correlation between what I get through the serial port and the code. Perhaps only that there's a loop that resets the device every ten seconds, and that's when the LED blinks:

```c
for (int i = 10; i >= 0; i--) {
    printf("Restarting in %d seconds...\n", i);
    vTaskDelay(1000 / portTICK_PERIOD_MS);
}
```

I need to read into the `printf` function. Maybe there's a decoding problem, or the baud rate is wrong. Time to continue reading the docs!

A `make monitor` target is included. Running it produces serial output at the same rate as the `rnn||bll` before but this time it's completely readable, and matches what's in the source code (go figure). I'm wondering how I could produce the same results using `minicom`. There's a `74880` number in the output of `make monitor` that looks suspiciously like a baud rate. I was connecting with `115200`.

> In a digitally modulated signal or a line code, symbol rate, also known as baud rate and modulation rate, is the number of symbol changes, waveform changes, or signaling events across the transmission medium per unit of time ([Wikipedia](https://en.wikipedia.org/wiki/Symbol_rate)).

I thought the baud rate was the problem, but running `minicom -b 74880 -o -D /dev/ttyUSB0` still gave me the `rnn||bll` output.

Apparently, you can specify baud rates like this: `make flash ESPBAUD=9600`, so I'm trying to see if the `minicom` output changes with this. Flashing at `9600` is so slow! Some minutes later, I run `minicom -b 9600 -o -D /dev/ttyUSB0`, and I get slightly different results, but still gibberish. More or less this:

```
ɏɏɏɏ
ɏɏɏɏɏɏɏɏɏɏɏɏ
GGq4O.
```

Time to read the source code to figure out what's up. The output of `make monitor` includes some interesting information:

- The script that runs is `idf_monitor`, and its source code is in `$IDF_PATH/tools/idf_monitor.py`.
- I can modify that file and change the output of `make monitor`.
- This helped me find out that the default baud rate is `74880`.

Running `screen /dev/ttyUSB0 74880` produces gibberish, too:

```
GGq4O.���ɏ�ɏQ�0�~?�4�!�S{�O�:9�O�:9�COAa�$\���S��4�MS��F�����.P�r|Vz4E^V8���M�0�zx��#A�AVZ��#�pRZR�rRRZ�E��&��,CORRV��'K�EEZG�Ň�EERGG���MK�EERG��'K�EEZG��#�G�����Gq���4O.�ɏ�ɏ��ɏ�ɏ��ɏ�ɏ��ɏ�ɏ��ɏ�ɏQ�0�~?�4�!�S{�O�:9�O�:9�COAa�$L���S��4�MS��F�����.P�r|Vz4E^V8���M�0�zx��#A�AVZ��#�pRZR�rRRZ�E��&��,CORRV��'K�EEZG�Ň�EERGG���MK�EER
```

Rebuilding with baud rate 115200 makes `screen /dev/ttyUSB0 115200` produce output more similar to the original output from `minicom`:

```
GGq4O.���ɏ�ɏQ�0�~?�4�!�S{�O�:9�O�:9�COAa�$\���S��4�MS��F�����.P�r|Vz4E^V8���M�0�zx��#A�AVZ��#�pRZR�rRRZ�E��&��,CORRV��'K�EEZG�Ň�EERGG���MK�EERG��'K�EEZG��#�G�����Gq���4O.�ɏ�ɏ��ɏ�ɏ��ɏ�ɏ��ɏ�ɏ��ɏ�ɏQ�0�~?�4�!�S{�O�:9�O�:9�COAa�$L���S��4�MS��F�����.P�r|Vz4E^V8���M�0�zx��#A�AVZ��#�pRZR�rRRZ�E��&��,CORRV��'K�EEZG�Ň�EERGG���MK�EER��r�b�nn�lnnl��r�nn�||�bbl��r�b�nn�lnn��r�nn�||�bbl��r�b�nn�lnn��r�nn�||�bbl��r�b�nn�lnn��r�nn�||�bbl��r�b�nn�lnnl�r�nn�||�bbl��r�b�nn�lnn�r�nn�||�bbl
```

At this point I'm going to give up and just use `make monitor`, and figure this out later with the help of someone more experienced.

---

As it turns out, there are "standard" baud rates, blessed by most tooling around, and it seems like `screen` and `minicom` aren't able to read at 74880bps even if configured to do so. I'm still not sure why flashing with `ESPBAUD=SOMETHING_ELSE` doesn't then let me connect at speed `SOMETHING_ELSE`, so it seems like I'll have to rely on `make monitor` for my connection.

With `make monitor`, the `Hello, world!` message shows up. Success! In [the next post]({% post_url 2021-02-20-embedded-learning-log-2-dmesg-tty-and-minicom %}) I'll be trying to figure out why that `sudo dmesg | grep tty` pipe made sense.
