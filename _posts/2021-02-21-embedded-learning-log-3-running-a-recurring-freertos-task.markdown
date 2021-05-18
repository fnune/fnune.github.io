---
layout: post
title: 'Embedded learning log - Running a FreeRTOS recurring task'
comments: true
categories: [embedded-learning-log]
date: 2021-02-21 21:00:51 +0200
excerpt: 'Running a recurring task with ESP8266_RTOS_SDK/FreeRTOS and calculating the stack size needed by a C function'
---

{% include embedded-learning-log.html %}

> Browse the accompanying repository [at the commit in which I was done with this part](https://github.com/fnune/toastboard/tree/061238f3ccffd2f7f13eb0a4d0844fc31d5ad9c8).

Documentation on the ESP8266_RTOS_SDK does not feature any information about task creation. I guess this is because the underlying RTOS is FreeRTOS, so I'm going to try to find documentation for that.

[Here's the documentation](https://www.freertos.org/a00125.html).

Apparently I can create tasks with `xTaskCreate` or `xTaskCreateStatic`. If I want to use `xTaskCreateStatic`, I need to provide my own memory, so I need to pass more parameters, but I can allocate this memory at compile time. I'm going to go with the simpler `xTaskCreate`, which allocates from [the FreeRTOS heap](https://www.freertos.org/a00111.html).

The most important argument to `xTaskCreate` is the task function I want to run:

> Tasks are normally implemented as an infinite loop, and must never attempt to return or exit from their implementing function. Tasks can however delete themselves.

To create a task that executes periodically, I just need to make it an infinite loop that calls `vTaskDelay` in it or `vTaskDelayUntil`. I'm going for the simpler `vTaskDelay`.

Another interesting argument is `usStackDepth`. It's in words, and the docs say a word is 32 bits in an ESP8266EX. One choice would be to find out the minimum stack size required for a task in FreeRTOS. I found this interesting article: [GNU Static Stack Usage Analysis](https://mcuoneclipse.com/2015/08/21/gnu-static-stack-usage-analysis/) by [Erich Styger](https://mcuoneclipse.com/author/mcuoneclipse/). I can pass `-fstack-usage` to `gcc` and get a nice `.su` file that gives me the stack usage of each function in bytes. I can append to `CFLAGS` for specific components, so I did so in `esp8266_sdk/memfault/component.mk`, and then found this file `memfault/memfault_platform_http_client.su`:

```
memfault_platform_http_client.c:178:5:memfault_platform_http_client_post_data	192	static
```

That's it! 192 bytes divided by my word size of 4 is exactly 48. I'll use 48 as a stack size for my task and eventually test to see if I can get by with less (I doubt it).

After creating the task, I get an crash loop due to a stack overflow on my new `task_upload_memfault_data`. Is my stack size not big enough?

## Finding out the stack size needed by a function

I found this interesting article: [GNU Static Stack Usage Analysis](https://mcuoneclipse.com/2015/08/21/gnu-static-stack-usage-analysis/) by [Erich Styger](https://mcuoneclipse.com/author/mcuoneclipse/). I can pass `-fstack-usage` to `gcc` and get a nice `.su` file that gives me the stack usage of each function in bytes.

I found out I can pass append to `CFLAGS` for specific components, so I did so in `esp8266_sdk/memfault/component.mk`, and then found this file `memfault/memfault_platform_http_client.su`:

```
memfault_platform_http_client.c:178:5:memfault_platform_http_client_post_data	192	static
```

That's it! 192 bytes divided by my word size of 4 is exactly 48. I'll use 48 as a stack size for my task and eventually test to see if I can get by with less (I doubt it).

### The stack size I calculated wasn't enough

I started by trying to allocate a stack of 48 words for my `task_upload_memfault_data` function, but that crashed my device due to bad allocations. I bluntly tried to increase it gradually until it worked, and it did! At 4096 words. I was off by a lot.

Could this be due to the fact that I measured stack usage for `memfault_platform_http_client_post_data` and not for my task's function `task_upload_memfault_data`? Let's build with `-fstack-usage` again, this time scouting for my task function instead. The number reported for `memfault_platform_http_client_post_data` was 192 bytes.

The output contains this:

```
main.c:148:6:task_upload_memfault_data	16	static
```

Eew! 16 is a lot less than 192. It seems like this isn't going all the way down the call stack to calculate stack usage. Maybe I'm missing some other flag?

## Runtime analysis of the target stack size

I'm going to try to let the code run and analyze how much memory the task takes after the fact. I probably need to make the task run through all of its different branches and take the maximum.

I remember that allocation of tasks created via `xTaskCreate` is handled by FreeRTOS, and there's another way to create tasks that lets me control allocation: `xTaskCreateStatic`.

My plan is to:

1. Refactor my task to use `xTaskCreateStatic`.
2. Flash and run the code.
3. Figure out how to analyze the memory layout after the fact.
4. Read how much of the stack I allocated was actually used by the task.

To be able to access `xTaskCreateStatic` I had to change `configSUPPORT_STATIC_ALLOCATION` to `1` in `components/freertos/include/freertos/FreeRTOS.h`. Surprisingly, this setting wasn't available under `make menuconfig` -> Component configuration -> FreeRTOS.

I'm giving up on this approach and returning to static analysis because compiling with `configSUPPORT_STATIC_ALLOCATION` requires me to implement my own allocators/deallocators for FreeRTOS's idle task, and I'm... not up for the task.

## Static analysis of the target stack size

The reason why my 192 number was wrong is because `gcc` outputs `.su` files for each `.o` file it produces; it doesn't look into other output objects to determine stack usage in a recursive fashion.

I found this Perl script by Daniel Beer [`avstack.pl`](https://dlbeer.co.nz/oss/avstack.html) which reportedly builds this tree or call graph for you and then sums the results:

> The script reads all .su files, and disassembles all .o files, including relocation data. The disassemblies are parsed and used to construct a call graph. Multiple functions in different translation units with the same name don't cause problems, provided there are no global namespace collisions. Information will appear on any unresolvable or ambiguous references.

I've added the script and results to the repository in [`src/avstack.txt`](src/avstack.txt). Functions prefixed with `>` are supposed to have a summary of function calls in lines below them, but my run seems to have produced a flat stack where no function calls other functions in its body. It's weird, and I got a 196 for `memfault_platform_http_client_post_data`, which looks like it's my previous 192 plus 4. This 4 is passed as an argument to `avstack.pl` called `call_cost`.

So! I'm giving up on this method as well. My last attempt: using FreeRTOS's `uxTaskGetStackHighWaterMark`.

## Measuring the target stack size at runtime using `uxTaskGetStackHighWaterMark`

I've instrumented my task like this:

```c
void task_upload_memfault_data( void * pvParameters )
{
    // [...]
    uxHighWaterMark = uxTaskGetStackHighWaterMark(NULL);
    ESP_LOGD(TAG, "Water mark at task entry: %lu", uxHighWaterMark);

    while (true)
    {
        // [...]
        uxHighWaterMark = uxTaskGetStackHighWaterMark(NULL);
        ESP_LOGD(TAG, "Water mark at task exit: %lu", uxHighWaterMark);
    }
}
```

The output looks like this, with irrelevant sections edited out:

```
D (807) TOASTBOARD: Water mark at task entry: 3976

D (10832) TOASTBOARD: Water mark at task exit: 3140

D (29036) TOASTBOARD: Water mark at task exit: 580
D (39037) TOASTBOARD: Water mark at task exit: 580
D (49039) TOASTBOARD: Water mark at task exit: 580
D (59041) TOASTBOARD: Water mark at task exit: 580
D (69042) TOASTBOARD: Water mark at task exit: 580
D (79045) TOASTBOARD: Water mark at task exit: 580
```

So it's 3976 on entry, on its first exit it's 3140, and then it's in the infinite loop (it has a 10 second delay after each run) and stabilizes at 580.

The initial drop from 3976 to 3140 is explained by the documentation:

> Calling the function will have used some stack space, we would therefore now expect uxTaskGetStackHighWaterMark() to return a value lower than when it was called on entering the task.

In my case, "the function(s)" are `vTaskDelay` and `memfault_esp_port_http_client_post_data`.

The second drop from 3140 to the stable 580 is explained by the fact that `memfault_esp_port_http_client_post_data` returned early because there was no more data to post. Cool!

Two questions remain:

- Is the high water mark going to be higher than 3976 if there's more than one chunk to send to Memfault? I've seen it post several at a time. I guess the stack size is going to be the same if the code runs inside a loop.
- I need to try and allocate a more optimized stack size to see if that affects execution. My new size in words is 3976.

With 3976 it works just fine, but on a new run the high water mark is 3864. Why couldn't it be more than my allocated 3976? Do people pad these values? I'm sticking with my original 4096 for good measure.

### Finding out the size of a word manually in C

I remember in Rust this would be the size of `usize`. I guess in C I can go `sizeof(some_type)` to get this.

After some Googling, I found `size_t`, which varies depending on the address size of the processor, and is the type returned by `sizeof`. So let's try `sizeof(size_t)`.

```
ESP_LOGI(TAG, "Size of a word is %u bytes!", sizeof(size_t));
```

And then in `make monitor`:

```
I (532) TOASTBOARD: Size of a word is 4 bytes!
```

Four bytes are 32 bits. Thanks Toastboard!

Next I'll be [integrating the Memfault Firmware SDK]({% post_url 2021-02-21-embedded-learning-log-4-integrating-the-memfault-firmware-sdk %}).
