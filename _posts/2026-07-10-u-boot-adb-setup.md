---
layout: post
title: "Custom U-Boot for Allwinner A33 Frames via FEL Mode"
date: 2026-07-10
tags: [embedded, u-boot, adb, frame, allwinner, fel]
---

## Overview

This post documents the custom U-Boot binary used in the [previous FEL mode guide]({{ site.baseurl }}/2026-07-08-pastigio-frame-adb-fel-mode.html) — the bootloader that exposes the frame's eMMC as a block device.

## What This U-Boot Does

When booted via FEL mode, this custom U-Boot:
- Initializes the Allwinner A33 SoC and DRAM
- Boots directly into USB mass storage mode (`ums 0 mmc 2`)
- Exposes the frame's internal eMMC as a plain block device on the host
- Requires **no serial cable, no fastboot, no manual interaction**

## Building It

The binary is built from mainline U-Boot, starting from the `q8_a33_tablet_1024x600_defconfig` and adapted for the Pastigio frame's hardware:

### Device Tree Changes

The stock Q8 tablet device tree needed updates to match the actual board:

```dts
&mmc2 {
    pinctrl-names = "default";
    pinctrl-0 = <&mmc2_8bit_pins>;
    bus-width = <8>;
    non-removable;
    status = "okay";
};

&usb_otg {
    dr_mode = "peripheral";
    status = "okay";
};
```

### U-Boot Defconfig

```
CONFIG_ARM=y
CONFIG_ARCH_SUNXI=y
CONFIG_DEFAULT_DEVICE_TREE="sun8i-a33-q8-tablet"
CONFIG_DRAM_CLK=456
CONFIG_SPL=y
CONFIG_MACH_SUN8I_A33=y
CONFIG_DRAM_ZQ=15291
CONFIG_SUNXI_NO_PMIC=y
CONFIG_VIDEO_LCD_MODE="x:1024,y:600,depth:18,pclk_khz:51000,le:159,ri:160,up:22,lo:12,hs:1,vs:1,sync:3,vmode:0"
CONFIG_VIDEO_LCD_DCLK_PHASE=0
CONFIG_VIDEO_LCD_POWER="PH7"
CONFIG_VIDEO_LCD_BL_EN="PH6"
CONFIG_VIDEO_LCD_BL_PWM="PH0"
# CONFIG_SYS_MALLOC_CLEAR_ON_INIT is not set
CONFIG_CONS_INDEX=5
CONFIG_CMD_USB_MASS_STORAGE=y
CONFIG_USB_GADGET=y
CONFIG_USB_GADGET_DOWNLOAD=y
CONFIG_USB_MUSB_GADGET=y
CONFIG_BOOTDELAY=0
CONFIG_BOOTCOMMAND="ums 0 mmc 2"
```

Key points:
- `CONFIG_SUNXI_NO_PMIC=y` — no AXP PMIC on this board
- `CONFIG_BOOTDELAY=0` and `CONFIG_BOOTCOMMAND="ums 0 mmc 2"` — boots straight into USB mass storage with zero delay
- `CONFIG_CONS_INDEX=5` — serial output goes to the correct UART (inherited from Q8 config, happens to work for this board too)

### Build Command

```bash
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- q8_a33_tablet_1024x600_defconfig
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)
```

## How to Use It

See the [previous post]({{ site.baseurl }}/2026-07-08-pastigio-frame-adb-fel-mode.html) for the full procedure, but in brief:

1. **Short the UPDATE pads** on the MS33-V2.0 board while powering on to force FEL mode
2. **Boot this U-Boot via FEL** — no flashing required, it runs entirely in SRAM/DRAM:
   ```bash
   $ sunxi-fel -v uboot u-boot-sunxi-with-spl.bin
   ```
3. **The frame's eMMC appears on your host** as a USB mass storage device (e.g., `/dev/sdb`)

## Mounting the Userdata Partition and Adding ADB Keys

Once the eMMC is exposed as USB mass storage, you can mount the Android userdata partition and plant your ADB public key:

### Identify the Partition

List the connected USB mass storage device:

```bash
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   1 29.1G  0 disk
├─sdb1   8:17   1  256M  0 part
├─sdb2   8:18   1  512M  0 part
└─sdb3   8:19   1 28.3G  0 part
```

The userdata partition is typically the largest one (in this case, `sdb3`).

### Mount the Partition

Create a mount point and mount the userdata partition:

```bash
$ sudo mkdir -p /mnt/frame_userdata
$ sudo mount /dev/sdb3 /mnt/frame_userdata
```

### Plant Your ADB Public Key

First, generate an ADB keypair if you don't have one:

```bash
$ mkdir -p ~/.android
$ adb keygen ~/.android/adbkey
```

Navigate to the ADB keys directory and create the authorized keys file:

```bash
$ sudo mkdir -p /mnt/frame_userdata/misc/adb
$ sudo cp ~/.android/adbkey.pub /mnt/frame_userdata/misc/adb/adb_keys
$ sudo chown 1000:1000 /mnt/frame_userdata/misc/adb/adb_keys
$ sudo chmod 0644 /mnt/frame_userdata/misc/adb/adb_keys
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

## Files

- U-Boot source: mainline U-Boot with the defconfig and device tree changes above
- Device tree: `arch/arm/dts/sun8i-a33-q8-tablet.dts` (modified)
- Defconfig: `configs/q8_a33_tablet_1024x600_defconfig` (modified)

## Why This Works Without Fastboot or Serial

- **FEL mode** is baked into the Allwinner A33 boot ROM and cannot be disabled by vendor firmware
- **USB mass storage mode** in U-Boot gives direct block-device access to the eMMC without needing to boot Android
- **No manufacturer restrictions** apply below the Android layer — you're talking directly to the SoC

## Next Steps

With ADB now enabled, you can follow the [ImmichFrame Frameo setup guide](https://immichframe.dev/docs/getting-started/apps#frameo) to sideload the app or continue with any other development workflows.
