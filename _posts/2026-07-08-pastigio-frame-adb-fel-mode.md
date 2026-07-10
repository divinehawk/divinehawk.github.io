# Enabling ADB on a Pastigio Frameo Photo Frame via Allwinner FEL Mode

I bought a [Pastigio 10.1" Frameo digital picture frame](https://www.amazon.com/dp/B0D41ZMYB2) (model **ZC002**) with one goal: run [ImmichFrame](https://github.com/immichFrame/ImmichFrame) on it instead of the stock Frameo cloud app, so photos would come straight from my self-hosted Immich instance instead of a third-party sync service. That meant I needed `adb`.

This is the story of how a "we've disabled this for your safety" support ticket turned into a teardown, an Allwinner FEL session, and a hand-planted ADB key.

## The dead end: a toggle that does nothing

Frameo's on-device settings menu actually has a "Enable ADB debugging" option. Flip it, and normally you'd expect the familiar Android "Allow USB debugging?" dialog with the RSA key fingerprint to pop up the moment you run `adb devices` from a connected host. On this frame, nothing happens. The toggle flips, `adbd` presumably starts, but no authorization prompt ever renders — `adb devices` just sits there, or shows the device as `unauthorized` forever with no dialog to accept.

I opened a support ticket. The reply, verbatim:

> "We've thoroughly discussed your request with the manufacturer. They indicated that since the ADB debugging interface is subject to strict system-level permission controls, enabling it could potentially cause irreversible damage to the system programs. Therefore, this setting is currently unavailable. We appreciate your understanding."

That's not really an explanation — ADB debugging doesn't cause "irreversible damage" any more than any other Android feature — but it did confirm this wasn't a bug I could fix from the settings menu. So I opened the case.

## What's actually inside

Popping the back off the frame revealed a board silkscreened **MS33-V2.0**, built around an **Allwinner A33** SoC — a quad-core Cortex-A7 part that Allwinner has shipped in a huge number of budget tablets and, apparently, photo frames.

More interestingly, the board has an unpopulated set of pads clearly labeled **UPDATE**, footprinted for a tactile switch that was never soldered on. On Allwinner designs this is almost always the FEL/recovery strap: a GPIO sampled by the boot ROM at power-on that, when pulled to the active state, forces the SoC to skip its normal boot media (eMMC/NAND/SD) and drop into **FEL mode** instead.

There's also an RX/TX header on the board — 3.3V logic levels, 115200 baud, the usual Allwinner debug UART setup. Hooking up a USB-to-serial adapter and powering the frame on normally, though, got nothing readable: either silence or garbage that didn't resolve at any of the usual rates. Whatever console the stock firmware uses, it isn't coming out on those pins during a normal boot.

![MS33-V2.0 board, showing the Allwinner R6383BA (A33) SoC, the unpopulated UPDATE button footprint, and the RX/TX serial header](ms33-v2-board.jpg)
*The MS33-V2.0 board out of the frame — Allwinner SoC dead center, `UPDATE` footprint and `RX`/`TX` pads on the left edge, WiFi/BT module bottom-left.*

## FEL mode: the backdoor Allwinner can't disable

FEL (Firmware Loader) is a first-stage USB recovery protocol baked into the boot ROM (BROM) of every Allwinner SoC — it's mask-programmed into silicon, so no amount of vendor software lockdown on top of it can remove it. When a chip enters FEL, it enumerates over USB with a fixed identity:

```
Bus 001 Device 004: ID 1f3a:efe8 Allwinner Technology sunxi SoC OTG connector in FEL/flashing mode
```

At this stage there's no OS, no filesystem, no Android permission model — just the BROM waiting for commands to read/write SRAM and DRAM and to hand off execution. The standard tool for talking to it is [`sunxi-fel`](https://github.com/linux-sunxi/sunxi-tools), part of the `sunxi-tools` package from the linux-sunxi project.

## Forcing FEL and talking to the chip

With the UPDATE pads shorted (a pair of tweezers works fine — no soldering needed since the footprint is exposed) while applying power, the frame came up silent with a blank screen and enumerated as `1f3a:efe8` on the host. From there:

```bash
$ sunxi-fel version
AWUSBFEX soc=00186300(A33) ver=0001 44 08 scratchpad=00007e00 00000000 00000000
```

That confirms the A33 identification and that the BROM is alive and listening.

## Booting a custom U-Boot with USB Mass Storage

FEL by itself only gives you raw memory read/write and the ability to execute code — it doesn't understand partitions or filesystems. To actually get at the frame's internal storage, I cloned [mainline U-Boot](https://github.com/u-boot/u-boot) and built a `sun8i-a33` target with USB Mass Storage (UMS) gadget support enabled.

The stock `sun8i-a33-q8-tablet` device tree needed two changes: enabling the eMMC controller (the frame's storage lives on `mmc2`, wired for 8-bit bus width and non-removable, unlike the Q8 tablets this DT normally targets), and forcing the USB OTG controller into pure peripheral mode so it comes up as a gadget instead of trying to negotiate host/device role over OTG ID pin detection:

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

For the defconfig, I started from `configs/q8_a33_tablet_1024x600_defconfig` — mainline U-Boot already has this for the generic Q8-format A33 tablets, and the panel timings and GPIO wiring (`PH7`/`PH6`/`PH0` for LCD power, backlight enable, and PWM) matched this board closely enough to use as a base. Rather than generate `.config` first and hand-edit that, I edited the defconfig file itself — it's the thing worth keeping around afterward, and `.config` just gets regenerated from it. Stock, it looked like this:

```
CONFIG_ARM=y
CONFIG_ARCH_SUNXI=y
CONFIG_DEFAULT_DEVICE_TREE="sun8i-a33-q8-tablet"
CONFIG_DRAM_CLK=456
CONFIG_SPL=y
CONFIG_MACH_SUN8I_A33=y
CONFIG_DRAM_ZQ=15291
CONFIG_AXP_GPIO=y
CONFIG_VIDEO_LCD_MODE="x:1024,y:600,depth:18,pclk_khz:51000,le:159,ri:160,up:22,lo:12,hs:1,vs:1,sync:3,vmode:0"
CONFIG_VIDEO_LCD_DCLK_PHASE=0
CONFIG_VIDEO_LCD_POWER="PH7"
CONFIG_VIDEO_LCD_BL_EN="PH6"
CONFIG_VIDEO_LCD_BL_PWM="PH0"
# CONFIG_SYS_MALLOC_CLEAR_ON_INIT is not set
CONFIG_REGULATOR_AXP_DRIVEVBUS=y
CONFIG_REGULATOR_AXP_USB_POWER=y
CONFIG_AXP_DLDO1_VOLT=3300
CONFIG_CONS_INDEX=5
CONFIG_USB_MUSB_HOST=y
```

Two things about it didn't fit this board. First, it assumes an AXP-series PMIC handling power sequencing and VBUS/USB regulator control (`CONFIG_AXP_GPIO`, the `REGULATOR_AXP_*` options, `CONFIG_AXP_DLDO1_VOLT`) — the frame doesn't wire one up the same way, so those all had to come out in favor of `CONFIG_SUNXI_NO_PMIC=y`. Second, `CONFIG_USB_MUSB_HOST=y` puts the USB controller in host mode, which is backwards from what I needed — that's for the tablet to read a USB drive plugged into *it*, not for it to appear as a drive to a PC. Swapping that for `CONFIG_USB_MUSB_GADGET=y` plus the gadget/mass-storage options, and adding an auto-executing `BOOTCOMMAND` so U-Boot drops straight into UMS mode with zero manual console interaction, got the defconfig to its final state:

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

`BOOTDELAY=0` plus `BOOTCOMMAND="ums 0 mmc 2"` means there's no U-Boot prompt to babysit — it boots straight into exposing `mmc2` (the eMMC) as a USB mass-storage device the moment SPL hands off to it. With the defconfig edited, generating `.config` from it and building was two commands — building from an x86_64 host for the A33's 32-bit ARM core meant setting `ARCH` and `CROSS_COMPILE` on both:

```bash
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- q8_a33_tablet_1024x600_defconfig
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)
```

and injected the resulting image straight into SRAM/DRAM over FEL, with no write to the frame's actual storage yet:

```bash
$ sunxi-fel -v uboot u-boot-sunxi-with-spl.bin
```

If the DTS and defconfig are right, this is the whole show: SPL brings up DRAM, U-Boot loads, `BOOTCOMMAND` fires automatically, and the eMMC shows up as a plain block device on the host — no drivers, no Android, no bootloader lock to fight.

One side effect of all this: `CONFIG_CONS_INDEX=5` carried over untouched from the base Q8 defconfig, and it turned out to be exactly right for this board too — the moment this custom U-Boot started running, the RX/TX header that stayed dead silent on a normal boot lit up with a clean 115200 8N1 log. Whatever UART the stock firmware's boot chain uses, it isn't this one, or it isn't enabled at all in the production build. Either way, the debug console only exists on this board once you're already running your own bootloader.

## Mounting the frame's storage on my PC

Back on the host, the frame now shows up as an ordinary disk. The full partition table:

```
Disk /dev/sda: 28.91 GiB, 31037849600 bytes, 60620800 sectors
Disk model: UMS disk 0
Disklabel type: dos

Device      Boot    Start      End  Sectors  Size Id Type
/dev/sda1   *     4268032 60719103 56451072 26.9G  b W95 FAT32
/dev/sda2         73728    139263    65536   32M  6 FAT16
/dev/sda3         1       4128768  4128768    2G  5 Extended
/dev/sda5         139264   172031    32768   16M 83 Linux
/dev/sda6         172032   204799    32768   16M 83 Linux
/dev/sda7         204800  2301951  2097152    1G 83 Linux
/dev/sda8        2301952  2334719    32768   16M 83 Linux
/dev/sda9        2334720  2400255    65536   32M 83 Linux
/dev/sda10       2400256  3973119  1572864  768M 83 Linux
/dev/sda11       3973120  4005887    32768   16M 83 Linux
/dev/sda12       4005888  4038655    32768   16M 83 Linux
/dev/sda13       4038656  4039679     1024  512K 83 Linux
/dev/sda14       4039680  4071423    31744 15.5M 83 Linux
/dev/sda15       4071424  4235263   163840   80M 83 Linux
/dev/sda16       4235264  4268031    32768   16M 83 Linux
```

Twelve small `Linux`-type (id `83`) partitions and one tiny FAT16 partition sit behind the extended container — none of those needed touching for this. The one that mattered is `/dev/sda1`: a single 26.9G **FAT32** partition, bootable-flagged, that turns out to be where this device keeps everything userspace-writable — including, as it turns out, `/data/misc/adb`:

```bash
$ sudo mount /dev/sda1 /mnt/frame
```

A stock recovery log later confirmed exactly what each of those partitions is for. Its kernel command line includes a `partitions=` argument mapping every one of them by name to an `mmcblk0pN` device:

```
partitions=bootloader@mmcblk0p2:env@mmcblk0p5:boot@mmcblk0p6:system@mmcblk0p7:misc@mmcblk0p8:recovery@mmcblk0p9:cache@mmcblk0p10:metadata@mmcblk0p11:private@mmcblk0p12:frp@mmcblk0p13:empty@mmcblk0p14:alog@mmcblk0p15:media_data@mmcblk0p16:UDISK@mmcblk0p1
```

Matched up against the sizes from `fdisk`, it's a completely conventional Allwinner/Android layout — `bootloader`, `env`, `boot`, `system` (the 1G partition), `misc`, `recovery`, `cache` (768M, presumably where the frame caches synced photos/video), `metadata`, `private`, `frp` (factory reset protection, all of 512K), an unused `empty` slot, `alog` for persistent logs, and `media_data`. What's conspicuously *not* in that list is anything called `data`. The role a stock Android `/data` partition would normally fill is instead just folded into `UDISK@mmcblk0p1` — the same 26.9G FAT32 volume the product page advertises for dragging photos onto over USB-C. There's no separate userdata partition at all; `/data/misc/adb` lives on the general-purpose FAT32 storage volume because, on this device, that volume *is* `/data`.

## Planting the ADB key by hand

Android's ADB authorization dialog isn't magic — it's backed by a plain text file. When you accept the "Allow USB debugging?" prompt, `adbd` appends the connecting host's public key (the contents of `~/.android/adbkey.pub` on the host) as a line in:

```
/data/misc/adb/adb_keys
```

On every subsequent connection, `adbd` checks incoming keys against that file and skips the dialog entirely if there's a match. Since the frame's launcher apparently never implements or shows that dialog at all, the toggle in Frameo's settings was starting `adbd` but leaving `adb_keys` permanently empty — so no key could ever be authorized through the UI, no matter how long you waited.

With the userdata partition mounted, the fix was direct — write the host key straight into the file `adbd` already checks:

```bash
$ mkdir -p /mnt/frame/misc/adb
$ cat ~/.android/adbkey.pub >> /mnt/frame/misc/adb/adb_keys
```

No `chmod`/`chown` here — the partition turned out to be **FAT32**, which doesn't store per-file Unix ownership or permission bits at all, so those calls are meaningless on this filesystem (and `chown` just fails outright). Whatever UID/GID `adbd` sees on this file once Android boots normally comes from fixed `uid=`/`gid=` mount options in the device's `fstab`, not from anything stored on the medium itself — so there was nothing to set on the host side, and it worked regardless.

Then, cleanly unmount before pulling power — this is a raw block device, not a live filesystem, so no lazy writes should be left pending:

```bash
$ sudo umount /mnt/frame
```

## The payoff

Power cycled the frame normally (no more shorting the UPDATE pads — this boots from eMMC as usual), plugged it into the host over its regular USB-C port, and:

```bash
$ adb devices
List of devices attached
<serial>    device
```

`device`, not `unauthorized` — no prompt ever appeared, because none needed to. The key was already in the trusted list before Android's ADB stack even started.

## What's next

With a working, authorized `adb` connection, the actual goal — sideloading ImmichFrame and pointing it at my self-hosted Immich server instead of relying on Frameo's cloud sharing — is just a normal `adb install` away. I'll cover that setup in a follow-up post.

## If you're trying this yourself

A few honest caveats:

- You're writing to a raw partition from outside Android's normal write path. Dump the partition (or at least the parts you're touching) before you write anything, so you have something to restore from if a permission or SELinux context assumption turns out wrong on your particular firmware build.
- The UPDATE-pad-as-FEL-strap trick is common on Allwinner designs but not guaranteed — confirm your SoC and strap behavior before assuming it maps the same way.
- This process only works because you have physical access to hardware you own. It's not a remote exploit and shouldn't be treated as one.

Manufacturer software policies can lock a menu toggle. They can't touch what's burned into the SoC's boot ROM.
