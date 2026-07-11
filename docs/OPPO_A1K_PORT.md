# OPPO A1K LK Porting Notes

This document records the useful engineering conclusions from bringing the
CPH1923 bootloader up from an MT6765 BSP. It is intentionally about mechanisms,
not a transcript of every experiment.

## Device Contract

| Item | Value |
| --- | --- |
| Product | OPPO A1K / CPH1923 |
| OPPO project | 18540 |
| LK platform | MT6765 BSP, MT6762 revised platform name |
| LK load address | `0x48000000` |
| LK reserved size | `0x00400000` |
| Scratch | `0x48b00000`, size `0x09000000` |
| Display | 720 x 1560, 3-lane DSI video mode |
| Panel | ILI9881C TXD/BOE ZAL1890 |
| LK partitions | `lk`, `lk2`, 1 MiB each |
| Persistent log | `expdb`, 20 MiB on the tested GPT |

## Method

The port was debugged by treating the stock loader as a behavioral reference:

1. Capture UART and `expdb` from stock and source-built LK.
2. Place compact markers around stage boundaries instead of adding large
   logging changes everywhere.
3. Compare memory reservations, final command lines, FDT properties, and
   display registers immediately before kernel handoff.
4. Change one subsystem, build, flash both LK slots, and capture the next log.
5. Keep a known-good recovery path and never modify the preloader during LK
   bring-up.

This approach was more reliable than copying isolated constants from a binary.
Many apparent kernel or Android faults were actually mismatched handoff data.

## Boot And Device Tree

The generic built-in `lk_main_dtb` path did not reproduce the stock boot flow.
This target enables the early loader:

```make
LK_MAIN_DTB_BUILT_IN := no
CFG_DTB_EARLY_LOADER_SUPPORT := yes
```

LK loads the DTB carried by boot/recovery, applies the selected entry from the
`dtbo` partition, and then performs target-specific final fixups.

The most important final fix is Android's hardware property. The BSP's revised
platform is MT6762, but stock Android expects MT6765:

```c
#ifdef OPPO_FORCE_ANDROID_HARDWARE_MT6765
value = "mt6765";
#endif
```

Without it, first-stage Android chooses the wrong init family. Symptoms include
missing mounts, `Permission denied` under `/data`, HWC restarts, and a native
reboot into recovery. Always inspect both the command line and
`/firmware/android` in the final FDT.

The panel handoff also aliases reset/pinctrl states into `/lcm`, because the
kernel driver expects them there while this board's overlay exposes the source
states through the display node.

## Kernel Command Line

The final command line is assembled from several sources:

- `/chosen/bootargs` in the selected DTB;
- boot image header command line;
- AVB output;
- dynamic LK values such as boot mode, serial number, timing, and boot reason.

Blindly appending parameters produced duplicates and stale META/Factory state.
The target therefore filters diagnostic-only tokens, deduplicates mode keys,
and rebuilds normal-boot parameters in stock order while preserving the chosen
bootargs layer.

Ramdisk handling follows the stock distinction: recovery/META ramdisk paths
may use `root=/dev/ram`; normal boot keeps the stock raw-system root selection.
Do not infer ramdisk policy from one old `expdb` record because the partition
can contain several boots.

## Logging

Early `_dputc()` happens before storage and the normal LK log store are ready.
Calling the full storage path there caused stalls. The working design is:

1. Buffer early characters in a small aligned RAM ring.
2. Initialize storage and the stock LK log store.
3. Replay the early buffer into the normal ring.
4. Append PL/LK and ATF data into the tail log area of `expdb`.

The target uses `OPPO_EXPDB_LOG_AREA_SIZE=0x1400000`, matching the 20 MiB
partition on the tested device. The code still obtains the actual partition
size and keeps block-aligned headers.

Useful stage markers are deliberately short:

```text
OPPO_EARLY_LOG_V1
[OPPO_EARLY] first _dputc
[FDT_ANDROID] hardware=mt6765
NORMAL_BOOT >>
[OPPO_DISP_HANDOFF]
[OPPO_LCM_HANDOFF]
```

## Display Handoff

The source-built LK initially left OVL and mutex state different from a stock
snapshot. Directly clearing OVL/MUTEX looked attractive but caused DSI
underruns and made the kernel display path less stable. A full quiesce sequence
also changed RDMA/DSI state away from stock.

The working configuration avoids late orange-state drawing, keeps the
framebuffer intact, and uses read-only handoff dumps. The dumps cover:

- framebuffer virtual/physical address, size, pages, and first pixels;
- DSI mode, size, packet/timing, PHY, and interrupt registers;
- OVL and RDMA source state;
- MMSYS route and mutex state;
- selected LCM and all important callbacks;
- LCM power/reset/backlight callback counters.

The lesson is to compare snapshots at the same stage. `OVL_EN=0` in stock and
`OVL_EN=1` in a custom log do not prove that stock explicitly disabled OVL;
they may result from different last-frame activity.

## LCM And Backlight

The panel driver was ported from the corresponding kernel LCM implementation
and adapted to LK's API. Important board details include:

- 720 x 1560 SYNC_PULSE video mode;
- three DSI lanes;
- 334 MHz PLL clock;
- 65 x 140 mm physical dimensions;
- OPPO round-corner buffers;
- MT6370 backlight path and stock-scale brightness values.

Missing `esd_check` callbacks in LK are not automatically fatal. More useful
evidence comes from reset/power order, DSI state, and what the Linux display
driver reports after handoff.

## Boot Modes

The recovery key opens the boot menu as it does in stock behavior; it is not a
direct recovery request. The extended menu writes both `g_boot_mode` and the
boot argument mode, then applies mode-specific command-line and init markers.

Supported entries:

- Normal boot
- Recovery
- Fastboot
- META
- Factory
- Advanced META
- ATE Factory

Fastboot recovery reboot writes both the RTC marker and the recovery command so
the request survives the next reset.

## AVB And Root Of Trust

This ATF accepts the stock OPPO three-packet Root-of-Trust sequence. A generic
six-call AVB20 sequence returned `0xffffffff` after the legacy payload even
though earlier packets succeeded. The port keeps verification code enabled but
uses the stock call count and payload order.

Unlocked and locked devices do not have the same root/verity command line.
Do not copy a locked stock `dm-verity` line into an unlocked bring-up build.
Likewise, changing verified-boot state can make existing file-based encryption
state unusable; formatting user data is destructive and must never be an
automatic debugging step.

## Memory Layout

Memory reservations must be compared as a set rather than one address at a
time. This target matches the stock high modem placement and reserves a 144 MiB
scratch window from `0x48b00000` to `0x51b00000`. The final mblock list must
still leave ATF, TEE, SSPM/SCP, framebuffer, log store, and boot arguments
non-overlapping.

## Watchdog Strategy

During bring-up, a kernel that hangs before persistent logging can look exactly
like a dead LK. Immediately before handoff, this target arms a reset-mode WDT
for 20 seconds. A successful kernel takes over watchdog management; a silent
early hang resets and leaves a new boot record for analysis.

This is a diagnostic guard, not a substitute for fixing the kernel.

## Fastboot Diagnostics

The target adds upload support for persistent logs and bounded partition reads:

```text
getvar:expdb-size
dump
fetch:<partition>:<offset>:<size>
oem stage-expdb
upload
```

With a host fastboot implementation that supports fetch:

```bash
fastboot fetch expdb expdb.bin
fastboot fetch boot boot.img
fastboot fetch recovery recovery.img
fastboot fetch dtbo dtbo.img
fastboot fetch system system.img
fastboot fetch vendor vendor.img
```

Reads are restricted to the published allowlist, bounded by the partition
size, and use 64-bit offsets for partitions larger than 4 GiB. The target
publishes `max-fetch-size=0x3c80`: this keeps both regular transfers and the
known GPT partition tails away from a MediaTek QMU 512-byte status-boundary
failure. Keep these facilities read-only.

Downloaded images passed to `fastboot boot` default to normal boot. Header
v1/v2 `recovery_dtbo_size`, an explicit recovery command, or a recovery marker
in the image command line selects recovery instead. The mere presence of a
ramdisk is not treated as a recovery marker.

## What Not To Commit

- `expdb` or other partition dumps;
- IMEI, serial numbers, MAC addresses, or factory records;
- OFP packages and proprietary firmware images;
- private signing keys, including `CUSTOM_RSA_D` and `AUTH_PARAM_D` material;
- stock-derived `main_dtb_header.bin` or other extracted DTB blobs;
- locally built LK binaries and experimental binary patches;
- recovery-side diagnostics containing user data.

## Known Boundaries

- The port is tested on one CPH1923 hardware family and one panel path.
- META/Factory availability still depends on matching Android vendor content.
- A different GPT, DRAM size, panel, or PMIC requires a new hardware audit.
- The included source is research-grade and retains verbose compile-time
  diagnostics that can be disabled after a stable production configuration is
  established.
