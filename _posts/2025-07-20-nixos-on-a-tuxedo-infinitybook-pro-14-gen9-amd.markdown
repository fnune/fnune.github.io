---
layout: post
title: 'NixOS on a TUXEDO InfinityBook Pro 14 Gen9 AMD laptop'
comments: true
categories: [hardware]
date: 2025-07-20 16:47:00 +0200
excerpt: 'Setting up NixOS on a TUXEDO InfinityBook Pro 14 Gen9 AMD laptop, including driver configuration, power management, and troubleshooting common issues.'
---

> I link to TUXEDO a lot in this blog post. These links are not affiliate links, and this post is not sponsored (duh).

[TUXEDO Computers][tuxedo] is a laptop manufacturer based in Augsburg, Germany, who are special to Linux users because they're one of the few manufacturers developing specifically for Linux.

They ship with their own [TUXEDO OS][tuxedo-os], which comes with KDE Plasma by default. For my new job, I actually wanted a Framework laptop with an AMD chip, but they were temporarily out of stock so I opted for a TUXEDO laptop, which would have been my second choice. I picked [their TUXEDO InfinityBook Pro 14 Gen9 AMD model][infinitybook].

In this blog post, I'll go over how I set up my NixOS computer to work on this TUXEDO laptop. NixOS is [**not among the supported operating systems**][tuxedo-supported-os], so I was expecting having to do some digging.

## The result

Here are the changes you'd need to make. I'd recommend reading the rest of the post to understand some trade-offs.

```nix
{
  pkgs,
  config,
  ...
}: {
  boot = {
    # Motorcomm YT6801 LAN drivers
    extraModulePackages = with config.boot.kernelPackages; [yt6801];
    kernelParams = [
      "acpi.ec_no_wakeup=1" # Fixes ACPI wakeup issues
      "amdgpu.dcdebugmask=0x10" # Fixes Wayland slowdowns/freezes
    ];
  };

  # Requires a system module from this flake:
  # https://github.com/sund3RRR/tuxedo-nixos?tab=readme-ov-file#option-2-nix-flake
  hardware = {
    tuxedo-drivers.enable = true;
    tuxedo-control-center.enable = true;
  };

  # Let TUXEDO Control Center handle CPU frequencies
  services.power-profiles-daemon.enable = false;
}
```

Include these changes in your `/etc/nixos/configuration.nix` in some way. In my case, I use [a system flake][system-flake] that includes [a laptop-specific configuration module][configuration.melian.nix].

## Drivers and the TUXEDO Control Center

```nix
# Requires a system module from this flake:
# https://github.com/sund3RRR/tuxedo-nixos?tab=readme-ov-file#option-2-nix-flake
hardware = {
  tuxedo-drivers.enable = true;
  tuxedo-control-center.enable = true;
};
```

The NixOS Wiki has [an article on TUXEDO devices][nixos-wiki-tuxedo]. It points at the fact that, as of July 2025, there is no package in `nixpkgs` that can install the official TUXEDO Control Center. It looks like the reasons include implementation attempts including [thousands of generated Nix code to declare the dependencies of the Angular frontend][thousands], and the frontend requiring Electron 11, which was [released in 2020][electron-11-release] and is deemed ancient. It also points at a community flake that has been abandoned, and that makes this module available outside of `nixpkgs`.

The other option suggested by the Wiki is to use [`tuxedo-rs`][tuxedo-rs], which exists because "TCC and its tccd service rely on Node.js which makes it slow, memory hungry and hard to package". I agree in principle, but it lacks certain features that I found useful in the TUXEDO Control Center:

- A system tray applet that lets me manually select a power profile (`tuxedo-rs` has a small GTK application but no system tray applet)
- An option to select which profile should activate when my laptop is plugged to electricity or working off of battery

So, even though I don't love going with a community flake for such a critical feature of my laptop, I went for the TUXEDO Control Center option, and I found this flake which is alive: [`sund3RRR/tuxedo-nixos`][sund3rrr]. Thanks `sund3RRR`!

## Getting Ethernet to work

Although I'd enabled `hardware.tuxedo-drivers`, my Ethernet port was still not working. After some digging, I found that I needed to enable a driver for it:

```nix
boot = {
  # Motorcomm YT6801 LAN drivers
  extraModulePackages = with config.boot.kernelPackages; [yt6801];
};
```

This is, in fact, mentioned in [the TUXEDO FAQ for this laptop][infinitybook-faq], but at this point I had not read this resource yet.

## Fixing laptop wake-ups while shut

While shut, the laptop would wake up and spin the fans periodically. This was very annoying. To solve this one, I messaged their support. They were very welcoming to my question even though it was related to an unsupported OS, and advised that I add these two kernel parameters:

```nix
kernelParams = [
  "acpi.ec_no_wakeup=1" # Fixes ACPI wakeup issues
  "amdgpu.dcdebugmask=0x10" # Fixes Wayland slowdowns/freezes
];
```

I have not tested the second one, but rebooting after adding `acpi.ec_no_wakeup=1` proved that it solved the random wake-ups.

The `acpi.ec_no_wakeup=1` param [was also mentioned in their FAQ][infinitybook-faq]. Oops…

## PowerDevil vs. TUXEDO Control Center

The addition of TUXEDO Control Center left me confused as to which widget to use to manage CPU frequencies and governors. KDE Plasma's PowerDevil provides integration, such as a slider under the system tray battery icon, and (as of recently) the battery icon changes to indicate which power profile is in use.

Changing the profile using TUXEDO Control Center does not reflect in a change in the KDE Plasma widget, so ~Claude wrote~ I wrote a script to monitor different system properties as I changed the TUXEDO Control Center and KDE Plasma options:

```sh
#!/usr/bin/env bash
while true; do
  freq=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq 2>/dev/null || echo "0")
  min_freq=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq 2>/dev/null || echo "0")
  max_freq=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq 2>/dev/null || echo "0")
  gov=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null || echo "N/A")
  epp=$(cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference 2>/dev/null || echo "N/A")
  boost=$(cat /sys/devices/system/cpu/cpufreq/boost 2>/dev/null || echo "N/A")

  printf "%s | %-10s | %-9s | %-12s | %-14s | %s\n" \
    "$(date +%H:%M:%S)" "$gov" "$((freq / 1000))" "$((min_freq / 1000))-$((max_freq / 1000))" "${epp:0:14}" "$boost"
  sleep 1
done
```

Here's a log of what was happening. I've redacted away a lot of lines so that you can see just the changes:

```
 1: TCC set to TUXEDO Defaults
 2: Plasma widget set to power saver
 3: 15:27:37 | powersave  | 1559  | 400-3801  | balance_perfor | 1
 4:
 5: TCC changed from TUXEDO Defaults to "Default" (I believe no change)
 6: 15:27:55 | powersave  | 2452  | 400-3801  | balance_perfor | 1
 7:
 8: TCC changed from "Default" to "Cool and breezy"
 9: 15:28:03 | powersave  | 1459  | 400-1900  | balance_perfor | 1
10:
11: TCC changed from "Cool and breezy" to "Powersaver extreme"
12: 15:28:08 | powersave  | 548   | 400-400   | balance_perfor | 1
13:
14: TCC changed from "Powersaver extreme" to "TUXEDO Defaults"
15: 15:28:25 | powersave  | 1297  | 400-3801  | balance_perfor | 1
16:
17: Plasma widget from power saver to balanced
18: 15:28:30 | powersave  | 1098  | 1098-3801 | balance_power  | 1
19: 15:28:35 | powersave  | 1540  | 400-5137  | balance_perfor | 1
20:
21: Plasma widget from balanced to performance
22: 15:28:44 | performance | 1098 | 1098-5137 | performance    | 1
23: 15:28:52 | powersave   | 1331 | 400-5137  | balance_perfor | 1
```

The interesting bits here are that both are apparently ultimately modifying the minimum and maximum CPU frequencies. They probably do this through different means, and I'm not an expert here, but I believe what's happening is that KDE Plasma is talking to `power-profiles-daemon` and setting a set of preferences (governor to `powersave`, energy performance preference (EPP) to `balance_performance`), and that results in something lower-level updating the minimum and maximum CPU frequencies.

You'll notice that in the two last changes, where I'm using the KDE Plasma widget, I included two log lines per change. It looks like TUXEDO Control Center is periodically monitoring the value of these settings and correcting them to what it wants:

- Governor set to `performance`: corrected to `powersave` after a few seconds
- EPP set to `performance`: corrected to `balance_performance` after a few seconds

I also did not like that the maximum CPU frequency would be set to `3801` or `5137` depending on which KDE Plasma option was picked last. TUXEDO Control Center tries to update things back to TUXEDO Defaults (which has a max of `5137`) but only manages `3801`, and I believe this is because the selected KDE Plasma option was `powersave`.

After seeing all of these interactions (or interruptions) I decided to use the TUXEDO Control Center exclusively:

```nix
# Let TUXEDO Control Center handle CPU frequencies
services.power-profiles-daemon.enable = false;
```

I lose the KDE Plasma integration with the battery widget, but that's OK.

## Remaining annoyances

There are just a couple of remaining nit-picks.

On boot, I get some ACPI-related warnings à la:

```
[    0.305246] ACPI BIOS Error (bug): Failure creating named object [\_SB.PCI0.GPP6.WLAN._DSM], AE_ALREADY_EXISTS (20240827/dswload2-326)
```

These seem innocuous. I contacted TUXEDO to see what could be done, and they mentioned they'd [updated their FAQ to mention that these messages do not affect operation][faq-update].

The other thing is that when the laptop wakes from sleep, the fans spin at full speed for a couple of seconds. I have a hunch that this is default behavior, and wanted, but it would be lovely if the laptop were quieter when opening the lid.

## Thoughts

Altogether, NixOS has had me fight a lot more fights than I would have fought on another distribution. I've been learning a lot, but I begin to question whether the benefits are worth the effort. I have had trouble setting up, for example, [Vanta][vanta-setup] for work (a compliance monitoring agent), [Playwright][playwright-setup] for browser end-to-end tests, and I simply cannot run some software unless I make the effort of packaging it for Nix and NixOS, for example [using KWallet as a `pinentry` program][pinentry-kwallet-pr] or [installing a database client I'd like to try out][tableplus-issue].

With the [release of Debian 13 Trixie][debian-13] looming, I may try to move my setup to a Debian stable base and keep most of my Home Manager setup so that I can use Nix for anything that's not my basic system, while still enjoying the mainstream support provided by Debian. We'll see!

[configuration.melian.nix]: https://github.com/fnune/home.nix/blob/main/os/configuration.melian.nix
[electron-11-release]: https://www.electronjs.org/blog/electron-11-0
[faq-update]: https://www.tuxedocomputers.com/en/Infos/Help-Support/Frequently-asked-questions/ACPI-Error-Reports.tuxedo
[infinitybook-faq]: https://www.tuxedocomputers.com/en/FAQ-TUXEDO-InfinityBook-Pro-14-Gen9-AMD.tuxedo
[infinitybook]: https://www.tuxedocomputers.com/en/TUXEDO-InfinityBook-Pro-14-Gen9-AMD.tuxedo
[nixos-wiki-tuxedo]: https://nixos.wiki/wiki/TUXEDO_Devices
[powerdevil]: https://docs.kde.org/stable5/en/powerdevil/kcontrol/powerdevil/index.html
[ppd]: https://search.nixos.org/options?channel=25.05&show=services.power-profiles-daemon.enable&query=power-profiles-daemon
[sund3rrr]: https://github.com/sund3RRR/tuxedo-nixos
[system-flake]: https://github.com/fnune/home.nix/blob/main/flake.nix#L60
[thousands]: https://github.com/sund3RRR/tuxedo-nixos/blob/master/pkgs/tuxedo-control-center/node-packages.nix
[tuxedo-os]: https://www.tuxedocomputers.com/en/TUXEDO-OS_1.tuxedo
[tuxedo-rs]: https://github.com/AaronErhardt/tuxedo-rs
[tuxedo-supported-os]: https://www.tuxedocomputers.com/en/TUXEDO-Support-Guidelines.tuxedo
[tuxedo]: https://www.tuxedocomputers.com/
[pinentry-kwallet-pr]: https://github.com/NixOS/nixpkgs/pull/425755
[tableplus-issue]: https://github.com/TablePlus/TablePlus-Linux/issues/122#issuecomment-2499906970
[vanta-setup]: https://github.com/fnune/home.nix/blob/main/packages/vanta.nix
[playwright-setup]: https://github.com/search?q=repo%3Afnune%2Fhome.nix+playwright&type=commits
[debian-13]: https://www.debian.org/releases/trixie/releasenotes
