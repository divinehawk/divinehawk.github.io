---
layout: post
title: "Quick U-Boot Setup & Enable ADB"
date: 2026-07-10
tags: [embedded, u-boot, adb, frame]
---

## Overview

I've built and tested a u-boot binary with all necessary options enabled for quick frame access and ADB debugging. This post walks through downloading and flashing it to get up and running fast.

## Download Pre-built U-Boot

The pre-built binary is ready to use:

📥 **[Download u-boot-sunxi-with-spl.bin]({{ site.baseurl }}/assets/u-boot-sunxi-with-spl.bin)**

**File details:**
- File size: ~571 KB
- Built with full feature set enabled
- Includes SPL (secondary program loader) combined with u-boot

## Prerequisites

Before flashing, ensure you have:

- USB-to-serial adapter (TTL UART, 3.3V)
- `fastboot` tool installed locally
- USB cable for device connection
- Device in bootloader/fastboot mode

## Installation Steps

### 1. Connect the Serial Interface

Connect your USB-to-serial adapter to the device's UART pins:
- TX → RX (device)
- RX → TX (device)
- GND → GND

### 2. Flash U-Boot

Once in fastboot mode, flash the binary:

```bash
fastboot flash bootloader u-boot-sunxi-with-spl.bin
```

### 3. Reboot the Device

```bash
fastboot reboot
```

## Enable ADB

After the device boots with the new u-boot:

1. Access the device's developer settings
2. Enable **USB Debugging** (Android Debug Bridge)
3. Connect via ADB:

```bash
adb connect <device-ip>
```

Or over USB directly:

```bash
adb devices
```

## Troubleshooting

**Device not appearing in fastboot:**
- Check USB cable and serial adapter connections
- Verify device is in bootloader mode
- Try: `fastboot devices`

**ADB connection fails:**
- Ensure USB Debugging is enabled on the device
- Check network connectivity for IP-based connections
- Try: `adb kill-server && adb start-server`

## What's Included in This Build

This u-boot binary was compiled with:
- USB support enabled
- Networking stack enabled
- FastBoot protocol support
- SPL (embedded loader) for faster boot

## Next Steps

With ADB enabled, you can now:
- Debug via `adb shell`
- Push/pull files with `adb push` and `adb pull`
- Install APKs directly
- Access device logs with `adb logcat`

Have questions or run into issues? Feel free to reach out!
