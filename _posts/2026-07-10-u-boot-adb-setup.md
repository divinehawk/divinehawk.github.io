---
layout: post
title: "Custom U-Boot for Allwinner A33 Frames via FEL Mode"
date: 2026-07-10
tags: [embedded, u-boot, adb, frame, allwinner, fel]
---

## Overview

This post documents the custom U-Boot binary used in the [previous FEL mode guide]({{ site.baseurl }}/2026-07-08-pastigio-frame-adb-fel-mode.html) — the bootloader that exposes the frame's eMMC as a USB mass storage device, enabling ADB key injection without any manufacturer restrictions.

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
3. **The frame's eMMC appears on your host** as a USB mass storage device
4. **Mount and edit** the `/data/misc/adb/adb_keys` file to plant your ADB public key
5. **Power cycle normally** — no UPDATE pad shorting needed on reboot
6. **`adb devices` shows the frame as authorized** with no prompt

## Files

- U-Boot source: mainline U-Boot with the defconfig and device tree changes above
- Device tree: `arch/arm/dts/sun8i-a33-q8-tablet.dts` (modified)
- Defconfig: `configs/q8_a33_tablet_1024x600_defconfig` (modified)

## Why This Works Without Fastboot or Serial

- **FEL mode** is baked into the Allwinner A33 boot ROM and cannot be disabled by vendor firmware
- **USB mass storage mode** in U-Boot gives direct block-device access to the eMMC without needing to boot Android
- **No manufacturer restrictions** apply below the Android layer — you're talking directly to the SoC

## Next Steps

With the frame mounted as USB storage, you can:
- Mount the userdata partition and edit any file
- Plant the ADB key to enable debugging
- Extract and replace the recovery or boot images
- Follow the [ImmichFrame Frameo setup guide](https://immichframe.dev/docs/getting-started/apps#frameo) to sideload the app

