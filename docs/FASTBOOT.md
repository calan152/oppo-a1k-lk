# Fastboot interface

This document describes the fastboot extensions provided by the OPPO A1K
CPH1923 LK port. The examples use the Android platform-tools `fastboot`
client. Commands that write storage still require an already-unlocked device.

## Physical keys

The CPH1923 build uses fixed physical hardware inputs instead of reading the
key assignment from the KPD node in DTBO:

| Input | LK hardware code | Power-on behavior |
| --- | ---: | --- |
| `VOL-` | matrix key `1` | Enter fastboot after one press or a hold. |
| double `VOL-` | matrix key `1` | Open the extended Boot Menu. The second press must arrive within the LK detection window. |
| `VOL+` | PMIC home/reset input `17` | Enter recovery. |
| `POWER` | PMIC power input `8` | Confirm the selected on-screen action. |

Inside the fastboot UI, `VOL+` selects the next action and `POWER` confirms it.
The available actions are:

| Action | Result |
| --- | --- |
| `START` | Boot Android from the normal `boot` partition. |
| `RESTART BOOTLOADER` | Reboot back into LK fastboot. |
| `RECOVERY MODE` | Boot the installed recovery image. |
| `POWER OFF` | Shut the device down. |

This mapping is deliberately independent of DTBO. It prevents a different or
damaged overlay from swapping fastboot and recovery keys. The numeric values
are specific to the CPH1923 board and must not be copied to another device
without checking its matrix and PMIC wiring.

## Diagnostic variables

Read one variable with:

```bash
fastboot getvar lk-build
```

Read every published variable with:

```bash
fastboot getvar all
```

The port adds these variables:

| Variable | Description |
| --- | --- |
| `lk-build` | Build identity generated from the project name and `git describe`, for example `oppo6762_18540-7b0387f-dirty`. |
| `lk-git-revision` | Short Git commit used by the build system. |
| `boot-mode` | Numeric LK boot mode plus a readable name. |
| `boot-reason` | Preloader/LK boot reason plus a readable name. |
| `boot-stage` | Current LK stage, such as `0x40 (dtb-ready)`. |
| `last-boot-stage` | Stage marker recovered from the watchdog non-reset register after the previous reset. |
| `wdt-status` | Raw MediaTek watchdog status register. |
| `lock-state` | Persistent lock state reported by the security layer. |
| `sboot-state` | Secure-boot state reported by the security layer. |
| `panel` | LCM selected by LK. |
| `dram-size` | Detected physical DRAM size in hexadecimal bytes. |
| `expdb-size` | Actual size of the `expdb` partition. |
| `active-lk-partition` | Reserved for preloader slot reporting. It currently returns `unknown` because this preloader does not pass a trustworthy source-slot identifier. |
| `lk-active-bit`, `lk2-active-bit` | GPT active attributes read for the two LK partitions. |
| `lk-selection-policy` | Partition predicted by the inspected preloader policy; this is not proof of the copy that actually ran. |
| `running-build-in` | Partition images containing the exact `LK_VER_TAG` of the executing build: `lk`, `lk2`, `both`, or `neither`. |
| `lk-source-evidence` | Summary of agreement or conflict between build-tag evidence and GPT policy. |
| `preloader-bootarg-format` | Parsed preloader handoff format (`atag`, `legacy`, or invalid/unknown). |
| `preloader-bootarg-addr`, `preloader-bootarg-size` | Address and size of the handoff block supplied to LK. |
| `preloader-bootarg-units` | Number of valid known ATAG units scanned by LK. |
| `preloader-unknown-tags` | Number of tags outside the known MT6765 set. Unknown tags are reported, not assumed to identify an LK copy. |
| `battery-voltage` | PMIC battery voltage in millivolts. |
| `battery-level` | Last exact Android UI SOC stored by the MT6357 fuel-gauge driver in `RTC_AL_MTH`; no voltage interpolation is used. |
| `battery-soc-ok` | Indicates that the RTC UI-SOC reader is enabled in this target. |
| `lk-size`, `lk2-size` | Detected sizes of the two LK partitions. |
| `lk-sha256-0`, `lk-sha256-1` | First and second halves of the SHA-256 digest of `lk`. Join the two values to obtain the full 64-character digest. |
| `lk2-sha256-0`, `lk2-sha256-1` | First and second halves of the SHA-256 digest of `lk2`. |
| `lk-copies-match` | `yes` when the complete `lk` and `lk2` partition images have identical sizes and SHA-256 digests. |
| `rescue-fastboot` | `yes` when the restricted `OPPO_RESCUE_FASTBOOT` build profile is active. |
| `max-fetch-size` | Maximum payload accepted by one fetch transaction. The non-packet-aligned value avoids the MTK QMU tail failure. |
| `fetch-partitions` | Comma-separated allowlist of readable partitions. |

The hash variables are split because one fastboot response is limited to 64
bytes. All LK copy checks are read-only. The stock preloader selects `lk2`
only when the `lk` active bit is zero and the `lk2` bit is nonzero, but the
handoff structure has no trustworthy loaded-copy field. Consequently
`active-lk-partition` stays `unknown`; `lk-selection-policy` and
`running-build-in` are deliberately exposed as separate evidence.

## Read-only partition fetch

The preferred way to retrieve an allowed partition is:

```bash
fastboot fetch expdb expdb.bin
fastboot fetch boot boot.img
fastboot fetch recovery recovery.img
fastboot fetch dtbo dtbo.img
fastboot fetch system system.img
fastboot fetch vendor vendor.img
```

The allowlist is intentionally limited to `expdb`, `boot`, `recovery`, `dtbo`,
`system`, and `vendor`. LK validates every 64-bit offset and length against the
real GPT partition size. The host splits large partitions into transactions
using `max-fetch-size`, so fetching `system` can take several minutes and
produce many progress lines.

Low-battery policy blocks `system` and `vendor` fetch because these very large
reads can run for a long time. Rescue-sized reads such as `expdb`, `boot`,
`recovery`, and `dtbo` remain available. External power is still strongly
recommended.

Legacy expdb retrieval is also available:

```bash
fastboot oem stage-expdb
fastboot get_staged expdb.bin
```

`fastboot oem dump-expdb` is an alias for the staging command. The raw protocol
also registers `dump` and `upload`; `fastboot fetch` is preferred because it
does not require the complete 20 MiB partition to be staged at once.

## RAM boot

Boot an image without flashing it:

```bash
fastboot boot boot.img
fastboot boot recovery.img
```

LK classifies a recovery image from its recovery DTBO or recovery command-line
markers. A normal image is not treated as recovery merely because it contains
a ramdisk.

Before copying any component, LK checks:

- Android boot header magic and supported header version;
- page size and all aligned size calculations for overflow;
- complete image size against the amount downloaded by USB;
- maximum kernel, ramdisk, recovery-DTBO, and DTB sizes;
- expected kernel, ramdisk, and DTB load addresses;
- overlap with LK, framebuffer, ATF, TEE, reserved mblocks, and scratch memory;
- unsupported second-stage images.

Validation failures are returned to fastboot as readable errors instead of
jumping through a malformed image.

## Reboot commands

The relevant host commands are:

```bash
fastboot reboot
fastboot reboot bootloader
fastboot reboot recovery
fastboot reboot fastboot
```

Recovery reboot writes both the RTC recovery marker and the recovery request
used by the next LK boot.

## Flashing safeguards

The CPH1923 target does not register `fastboot flashing lock`. Its legacy
MediaTek relock handler also rejects direct calls, and fastboot verifies the
persistent lock state at startup and before every command. If the state is
locked or cannot be read, LK remains in fastboot and blocks boot, reboot,
poweroff, flash, erase, and on-screen exit actions. It does not modify
`seccfg`. Read-only diagnostics, download transport, and the explicit
`fastboot flashing unlock` flow remain available.

`fastboot flashing unlock` uses a colored confirmation screen with a pulsing
orange header. `CANCEL - KEEP LOCKED` is selected by default; `VOL+` changes
the selection and `POWER` confirms it. There is no timeout or automatic
unlock. The normal platform unlock path still performs its expected userdata
wipe after deliberate confirmation. The destructive selection uses amber,
while red is reserved for actual failures. If the device is already unlocked,
the command returns `already unlocked` without showing a false locked screen,
rewriting state, or wiping data. An unreadable lock state fails without making
changes.

Writes to `lk`, `lk2`, `boot`, `recovery`, and `dtbo` use this completion
sequence: validate, hash the download, write, flush eMMC, read the complete
image back, hash it again, compare, and only then report `OKAY` and 100
percent. If verification fails, LK displays `FLASH VERIFY FAILED` and warns
that the device must not be rebooted. `lk` and `lk2` additionally require at
least 3700 mV and must be flashed with separate host commands.

The low-battery write check is retained for the large `system` and `vendor`
partitions. Smaller recovery paths including `expdb`, `boot`, `recovery`, and
`dtbo` are not blocked by that long-operation policy. This is intended for
repair access, not as a claim that flashing without stable external power is
safe.

## Restricted rescue build

Build the reduced repair interface with:

```bash
make -j"$(nproc)" oppo6762_18540 NO_SIGN=yes OPPO_RESCUE_FASTBOOT=yes
fastboot getvar rescue-fastboot
```

It permits diagnostics, bounded fetch, RAM boot, separate writes to `boot`,
`recovery`, `dtbo`, or `lk2`, reboot/poweroff when unlocked, and the explicit
unlock flow. It centrally rejects erase, ultra-flash, `seccfg`, preloader,
NVRAM/NVDATA, protect partitions, `lk`, and every command outside that
allowlist. See [`FASTBOOT_SAFETY.md`](FASTBOOT_SAFETY.md) for the exact policy
and host-side boot-image test command.

## On-screen progress

The full-screen fastboot UI remains visible while USB commands run. It shows
the operation, partition, percentage, elapsed time, transfer rate, build ID,
panel, DRAM, watchdog status, voltage, and RTC battery percentage. Key actions
are ignored while a USB command is actively writing or reading storage.
