---
layout: post
title: "Building and Deploying Custom U-Boot for Allwinner A33 Frames"
date: 2026-07-10
tags: [embedded, u-boot, adb, frame, allwinner, fel]
---

## Overview

This post documents the custom U-Boot binary used in the [previous FEL mode guide]({{ site.baseurl }}/2026-07-08-pastigio-frame-adb-fel-mode.html) — the bootloader that exposes the frame's eMMC as a USB mass storage device.

## What This U-Boot Does

When booted via FEL mode, this custom U-Boot:
- Initializes the Allwinner A33 SoC and DRAM
- Boots directly into USB mass storage mode (`ums 0 mmc 2`)
- Exposes the frame's internal eMMC as a plain block device on the host
- Requires **no serial cable, no fastboot, no manual interaction**

## How to Use It

See the [previous post]({{ site.baseurl }}/2026-07-08-pastigio-frame-adb-fel-mode.html) for the full procedure, but in brief:

1. **Short the UPDATE pads** on the MS33-V2.0 board while powering on to force FEL mode
2. **Boot this U-Boot via FEL** — no flashing required, it runs entirely in SRAM/DRAM:
   ```bash
   $ sunxi-fel -v uboot u-boot-sunxi-with-spl.bin
   ```
3. **The frame's eMMC appears on your host** as a USB mass storage device

## Mounting the Userdata Partition and Adding ADB Keys

Once the eMMC is exposed as USB mass storage, you can mount the Android userdata partition and plant your ADB public key:

### Mount the Partition

Based on the partition layout from the previous post, the userdata partition is typically `/dev/sda1`. Create a mount point and mount it:

```bash
$ sudo mkdir -p /mnt/frame_userdata
$ sudo mount /dev/sda1 /mnt/frame_userdata
```

### Plant Your ADB Public Key

First, generate an ADB keypair if you don't have one:

```bash
$ mkdir -p ~/.android
$ adb keygen ~/.android/adbkey
```

Create the ADB keys directory and copy your public key:

```bash
$ sudo mkdir -p /mnt/frame_userdata/misc/adb
$ sudo cp ~/.android/adbkey.pub /mnt/frame_userdata/misc/adb/adb_keys
```

### Unmount and Reboot

Safely unmount the partition:

```bash
$ sudo umount /mnt/frame_userdata
```

**Power cycle the frame normally** — no UPDATE pad shorting needed on reboot. The frame will boot Android and automatically trust your ADB public key.

### Verify ADB Connection

After the frame boots, verify that `adb devices` shows the frame as authorized:

```bash
$ adb devices
List of attached devices
<frame-serial>    device
```

No prompt or fingerprint prompt should appear.

## Binary Details

This prebuilt U-Boot binary is based on mainline U-Boot configured for the Allwinner A33 with these key settings:

- **Boot command**: `ums 0 mmc 2` — boots straight into USB mass storage mode with zero delay
- **No PMIC**: Configured for boards without the AXP PMIC
- **USB gadget**: Supports USB mass storage gadget mode for block device access
- **Device tree**: Customized for the MS33-V2.0 board to properly initialize the eMMC and USB OTG controller

## Why This Works Without Fastboot or Serial

- **FEL mode** is baked into the Allwinner A33 boot ROM and cannot be disabled by vendor firmware
- **USB mass storage mode** in U-Boot gives direct block-device access to the eMMC without needing to boot Android
- **No manufacturer restrictions** apply below the Android layer — you're talking directly to the SoC

## Next Steps

With ADB now enabled, you can follow the [ImmichFrame Frameo setup guide](https://immichframe.dev/docs/getting-started/apps#frameo) to sideload the app or continue with any other development workflows.
