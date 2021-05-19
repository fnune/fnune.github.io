---
layout: post
title: 'Embedded learning log - Integrating the Memfault Firmware SDK'
comments: true
categories: [embedded-learning-log]
date: 2021-02-21 21:00:51 +0200
excerpt: 'I take a stab at integrating my the product my company produces with my development board project.'
---

{% include embedded-learning-log.html %}

> **Note**: I make many mistakes during this integration. It's not an example of how one should go about it, but just a log of my first attempt as a newcomer to embedded development. If you're looking for a guide, [the documentation](https://docs.memfault.com/docs/embedded/esp8266-rtos-sdk-guide/) is a perfect place to start.

I'm starting by getting some application code in place. It's just going to be [an HTTP server with simple routes](https://github.com/fnune/toastboard/commit/6671fbf094254d0afd4aef5a65e8cd3eb9b12798) adapted from the ESP8266_RTOS_SDK examples. Once I'm done, I'll try to integrate the Memfault SDK on top.

After [following the basic steps in the documentation](https://docs.memfault.com/docs/embedded/esp8266-rtos-sdk-guide/#integration-steps) (here's the [related commit](https://github.com/fnune/toastboard/commit/edbfc2c9b19160f6e74d2c1db1245671cc0f643c)), I ran into this while compiling:

```
In file included from /home/fausto/Development/toastboard/src/components/memfault_port/memfault-firmware-sdk/components/include/memfault/components.h:20,
                 from /home/fausto/Development/toastboard/src/main/main.c:17:
/home/fausto/Development/toastboard/src/components/memfault_port/memfault-firmware-sdk/components/include/memfault/config.h:30:39: fatal error: memfault_platform_config.h: No such file or directory
 #define MEMFAULT_PLATFORM_CONFIG_FILE "memfault_platform_config.h"
                                       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

I later found this from the `CHANGES.md` file in the Memfault SDK:

```md
- You must create the file `memfault_platform_config.h` and add it to your
  include path. This file can be used in place of compiler defines to tune the
  SDK configurations settings.
```

After solving that by creating `src/main/include/memfault_platform_config.h`, there was another missing header file:

```
/home/fausto/Development/toastboard/src/components/memfault_port/memfault-firmware-sdk/ports/esp8266_sdk/memfault/esp_reboot_tracking.c:14:10: fatal error: internal/esp_system_internal.h: No such file or directory
 #include "internal/esp_system_internal.h"
          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

I got a bunch of compiler errors because I was on the `master` branch of ESP8266_RTOS_SDK and the Memfault integration supports only v3.3. The changes from `master` to v3.3 are significant in the HTTP server example, so I'm [updating my application code](https://github.com/fnune/toastboard/commit/6ae1b36f236fec02dce7c4918ca0d14bab31e9fd).
The next compiler error was a missing header file `esp_console.h` expected in `memfault_cli.c`. I could solve it by adding this to `src/sdkconfig`:

```
CONFIG_USING_ESP_CONSOLE=y
```

Et voila! It has built! Time to get it flashed onto the board.

### Reboot loop after integrating the SDK

There are some useful logs:

```
I (492) system_api: Base MAC address is not set, read default base MAC address from EFUSE
Guru Meditation Error: Core  0 panic'ed (LoadProhibited). Exception was unhandled.
```

And:

```
E (463) mflt: Coredumps enabled but no storage partition found!
```

I'm not sure which one is causing the reboots, so I'm going to fix the MAC address problem first. It should be a matter of using a different method that doesn't get the "base" MAC address but the one from EFUSE. For context, I'm trying to use the MAC address as a device identifier to send to the Memfault cloud.

---

I gave up trying to fix the MAC address problem because no matter what I picked, it would crash. I'll name this device `"the-one-and-only"` instead so that I can skip the problem of fetching the MAC address.

We're left with:

```
E (463) mflt: Coredumps enabled but no storage partition found!
```

It's a bit weird because I'm still not at the step where I can [add coredumps integration](https://docs.memfault.com/docs/embedded/coredumps/).

Either way: I can print the partition table of the device with `make partition_table`:

```
# Espressif ESP32 Partition Table
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  24K,
phy_init, data, phy,     0xf000,  4K,
factory,  app,  factory, 0x10000, 960K,
```

[My chip](https://www.amazon.de/-/en/gp/product/B08BTXCZC1/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1) supposedly has 4MB of storage, and that's roughly 1MB in the partition table, so I guess I could flash something else onto it. For now, I'm going to go with this.

Now I need to:

1. Identify which of those is the storage partition.
2. Figure out how to tell Memfault to use that.

The [ESP8266_RTOS_SDK documentation on partition tables](https://docs.espressif.com/projects/esp8266-rtos-sdk/en/release-v3.3/api-guides/partition-tables.html?highlight=storage) says NVS stands for non-volatile storage. I guess that's what we want. That means I want the partition starting at `0x9000` that's 24K in size.

I'm going to look for a way to do (2) in `make menuconfig`.

The Memfault-specific configuration has an option "Enable saving a coredump to flash storage (OTA slot)". The other choice is "User defined coredump storage region". I guess I could pick the second, but there's a default partition table that includes an OTA slot from the SDK, so I'm going to use that instead of the NVS partition.

The configuration changed like so:

```diff
- CONFIG_PARTITION_TABLE_SINGLE_APP=y
+ # CONFIG_PARTITION_TABLE_SINGLE_APP is not set
- # CONFIG_PARTITION_TABLE_TWO_OTA is not set
+ CONFIG_PARTITION_TABLE_TWO_OTA=y
```

I'm going to add that to the defaults file because it seems critical to the project. I also changed how I'm getting my WiFi SSID and password. Time to flash it again!

### The reboot loop continues

But now it's different! There are no more errors from Memfault, and I get a reassuring message, but the device still reboots in a loop:

```
I (499) mflt: Coredumps will be saved at 0x110000 (983040B)
```

This means my new partition table is being used correctly.

To begin debugging, I removed the `memfault_platform_boot()` call. Now, the device works correctly and does what it's supposed to do (not much). I can look into my Memfault-related files to find the culprit.

After some time trying to figure this out, I'm going to give up and hardcode the hardware version instead of trying to get it from `chip_info`, since I'm suspicious of my C.

Now, the device is stable! I just don't know C.

### Sending data

I need to add `memfault_esp_port_http_client_post_data()` somewhere.

My device exposes an HTTP server, so I decided to create a `GET /upload` route that will upload anything saved up to Memfault.

I open `192.168.0.24:80/upload` on a browser and...

```
D (30030) mflt: Posting Memfault Data
D (30339) mflt: Posting Memfault Data Complete!
I (30342) mflt: Result: 0
```

My device now exists on Memfault!

```
the-one-and-only	1.0.0+4bb166	ESP8266EX_CH340C	a minute ago
```

### The ESP-IDF console

The Memfault docs mention a `crash` CLI command. I'm not sure what this is. I tried to find some information about this, thinking it's related to the `CONFIG_USING_ESP_CONSOLE=y` flag I passed in for compilation. The ESP8266 documentation doesn't mention the `console` component or the `crash` CLI command.

ESP-IDF documentation mentions this:

> ESP-IDF provides console component, which includes building blocks needed to develop an interactive console over serial port.

I guess I'm supposed to build the interactivity into a new target? Maybe I'm supposed to test with the `console` example provided with the RTOS.

I'm blocked at this step because the next steps depend on this: the crash data I could get from the `crash` CLI would be uploaded with the `memfault` CLI, and then the web application would warn me there's a missing symbols file for the data, and I could then upload the symbols file and get issues from it.

I guess I need to find a way around the problem if I can't find out what this `crash` CLI command is.

### Trying to force a crash by flashing buggy code

Let's try adding an infinite loop in a `GET` request handler under `/crash`.

```
esp_err_t crash_get_handler(httpd_req_t *req)
{
    while (true) {
      // ...
    };

    const char* resp_str = (const char*) req->user_ctx;
    httpd_resp_send(req, resp_str, strlen(resp_str));

    return ESP_OK;
}

httpd_uri_t crash = {
    .uri       = "/crash",
    .method    = HTTP_GET,
    .handler   = crash_get_handler,
    .user_ctx  = "The application should crash before this shows up"
};
```

Calling `/crash` left my browser loading for about 30 (!) seconds before the device crashed. Here's the full output of the crash, from `make monitor`:

```
Saving Memfault Coredump!
Erasing Coredump Storage: 0x0 983040
coredump erase complete
Task watchdog got triggered.

Guru Meditation Error: Core  0 panic'ed (IllegalInstruction). Exception was unhandled.
Core 0 register dump:
PC      : 0x4026f3d4  PS      : 0x00000030  A0      : 0x40247d9c  A1      : 0x3fff57c0
0x4026f3d4: crash_get_handler at /home/fausto/Development/toastboard/src/main/main.c:54

0x40247d9c: httpd_uri at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_uri.c:218

A2      : 0x40107cec  A3      : 0x4026f3d4  A4      : 0x00000002  A5      : 0x00000004
0x4026f3d4: crash_get_handler at /home/fausto/Development/toastboard/src/main/main.c:54

A6      : 0x00000073  A7      : 0x00000001  A8      : 0x00000001  A9      : 0x00000068
A10     : 0x00000068  A11     : 0x80808080  A12     : 0x40107c9c  A13     : 0x40107cec
A14     : 0x40107c9c  A15     : 0x40107cec  SAR     : 0x00000018  EXCCAUSE: 0x00000000

Backtrace: 0x4026f3d4:0x3fff57c0 0x40247d9c:0x3fff57c0 0x4024853b:0x3fff57e0 0x402485ed:0x3fff5860 0x402475df:0x3fff5880 0x40246f58:0x3fff5890 0x40246fc4:0x3fff58b0
0x4026f3d4: crash_get_handler at /home/fausto/Development/toastboard/src/main/main.c:54

0x40247d9c: httpd_uri at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_uri.c:218

0x4024853b: httpd_parse_req at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_parse.c:527

0x402485ed: httpd_req_new at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_parse.c:594

0x402475df: httpd_sess_process at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_sess.c:327

0x40246f58: httpd_server at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_main.c:113

0x40246fc4: httpd_thread at /home/fausto/.esp/ESP8266_RTOS_SDK/components/esp_http_server/src/httpd_main.c:113

@R%dPMT%w@Y)Q
OT5V'I
T       )dE     5Y
NZA     I^NTR
@\Y!NZA I^{YP1g@\@Y
[)!E   pL@LYYX)IIPKVqRA I       YЛ     pL@PWK   V       ZQHYYV[)!E     pL@Y^ZYY
                         1eʉE  xO@Y     J5)aEEAJA%Y     J)Q
AEaE   xL@Y     J-I
%ʉE    xO@YZAFPYM)!E   `M@YI9V5ԻT)dVEEAJAH%Y=eZ)R
                                                 E
                                                  E    h%Y-JP-G

                                                               E       h%Y1iGQP)PNP-E  h%Y--HPPG
-AKRMYV@Q                                                                                       YAJAH^-G-HPP-YAJAH^w\PRZANPYEEAJAHq
         E     h)YVYP1DT
                        PTg
                           TWp)ʉE      hM)YVY1DT
                                                P%=DT
                                                     T5қ
                                                        @)ʉE   h)YVYRP#PR
5P@H)IJ)!E     hN)YVRP%=DT
                          Tg
)IJ)!E h                    P!)IJ)!E   h)YVYPRPT5eHP@
        %YYP-^1qQHT-TRYI (411) system_api: Base MAC address is not set, read default base MAC address from EFUSE
```

It looks like the system gave me the backtrace and the exact location of my crash: the body of the `crash_get_handler`. What a surprise.

On the Memfault application nothing's showing up yet. I guess it's because my device doesn't upload data unless I tell it to, using the `GET /upload` route.

After I upload the data in the device, I get this in the issues view:

```
Unprocessed traces exist due to missing symbol file(s):

Upload 1.0.0+5a05af (toastboard-firmware)
```

Cool! Uploading that now. I'm such a noob that I don't even know what I'm supposed to upload. I think it's `build/toastboard.elf`, though.

Let's try crashing the same away again to test issue deduplication. While my device takes some time to decide to crash through the infinite loop, I'm thinking of two questions:

1. Why does it take so long for it to crash? I'd have guessed the timeout is shorter. Although it's an infinite loop, shouldn't it still be cancelable by some interrupt or whatever mechanism the RTOS has?
2. Why is it showing up as a `Hard Fault` instead of a `Watchdog` issue? Wasn't the device restarted by the watchdog?

Looking around in `make menuconfig`, I found the answer to (1): `Task Watchdog timeout period (seconds) (26.2144s)` under `ESP8266-specific`. I was pretty close with my 30 seconds guess!

I think I can answer (2): the issue is a `Hard Fault`. The reboot event should be have the watchdog as a cause. However, it shows up as `Unspecified`. I guess I have something else to integrate.

Issue deduplication seems to be working fine. Here's what the Queue Status widget shows:

```
a few seconds ago	Attached to Issue #38075
26 minutes ago	Attached to Issue #38075
```

Both were attached to issue `38075`.

### Integrating reboot reasons

[Here's the documentation I'm following](https://docs.memfault.com/docs/embedded/reboot-reason-tracking).

I guess my reboot is showing up as having an `Unspecified` reason because I haven't completed integration. Let's continue with that.

The guide mentions I should `#include "memfault/core/reboot_tracking.h"`, but I confirmed that's already included with `"memfault/components.h"`.

There are two things I need to figure out from my chip and use in the initialization code:

1. A pointer to the reboot reason register.
2. Additional information about the reboot reason, mapped to a `MfltResetReason`.

As for (2), the ESP8266_RTOS_SDK documentation points to a function [`esp_reset_reason`](https://docs.espressif.com/projects/esp8266-rtos-sdk/en/release-v3.3/api-reference/system/system.html?highlight=reset#_CPPv416esp_reset_reasonv) that returns a value of type [`esp_reset_reason_t`](https://docs.espressif.com/projects/esp8266-rtos-sdk/en/release-v3.3/api-reference/system/system.html?highlight=reset#_CPPv418esp_reset_reason_t). I need to figure out how to map from `esp_reset_reason_t` to `MfltResetReason`.

As for (1), I can't find information in the documentation indicating how to access that register. There's only the functions described above. Looking at  the RTOS source code, I found this: `RTC_RESET_HW_CAUSE_REG`. I'm wondering if that's defined. To find that out, I'm going to try to compile with that and `kMfltRebootReason_Unknown` for now.

```
error: 'RTC_RESET_HW_CAUSE_REG' undeclared
```

Nope! However, I found that `RTC_RESET_HW_CAUSE_REG` points to a register `RTC_STATE1`, which I can access if I `#include "esp8266/rtc_register.h"`. This feels really hacky, but the code now compiles!

Reboot reasons still show up as `Unspecified` and `Unexpected Reset`. Time to continue with the integration: mapping `esp_reset_reason_t` to `MfltResetReason` and passing it to the Memfault SDK.

I built the mapper and passed it to `memfault_reboot_tracking_boot`, and everything compiles just fine, but the reboots in the app are still `Unspecified`. There's more integration steps to take, so I probably need to continue.

I needed to dedicate some storage space for events and pass it to Memfault. Once that was done, I tortured the little thing once more with the `GET /crash` endpoint, and voila! There's a reason for my reboot on the Memfault application: `Software Watchdog`.

This is a bit weird because the logs from `make monitor` show this:

```
I (503) mflt: ESP Reset Cause 0x7
I (509) mflt: Reset Causes:
I (514) mflt:  Hardware Watchdog
```

Maybe I got the mappings wrong!

```
/* Reset (software or hardware) due to interrupt watchdog. */
case ESP_RST_INT_WDT: return kMfltRebootReason_HardwareWatchdog;
/* Reset due to task watchdog. */
case ESP_RST_TASK_WDT: return kMfltRebootReason_SoftwareWatchdog;
/* Reset due to other watchdogs. */
case ESP_RST_WDT: return kMfltRebootReason_SoftwareWatchdog;
```

Reset cause `0x7` corresponds to "other watchdogs", so I'm going to map that to `HardwareWatchdog` instead, since that's what the logs say. The `Hardware Watchdog` log from the Memfault SDK was there long before I added my mapping, so that suggests to me that they knew the reset reason all along. Why did I have to implement the mapping, then?

---

As it turns out, it's because I didn't read the port included in the SDK well enough (duh). It includes [an implementation of `memfault_reboot_reason_get`](https://github.com/memfault/memfault-firmware-sdk/blob/master/ports/esp8266_sdk/memfault/esp_reboot_tracking.c) which I should have used from the start.

### Sending data periodically

I simply used the code from [my previous investigation]({% post_url 2021-02-21-embedded-learning-log-3-running-a-recurring-freertos-task %}) on how to create a recurring task on FreeRTOS and called `memfault_esp_port_http_client_post_data` from there. [Here's how it ended up looking](https://github.com/fnune/toastboard/blob/061238f3ccffd2f7f13eb0a4d0844fc31d5ad9c8/src/main/main.c#L148).

So! My device is sending data to the cloud. I'd call this a success for now.
