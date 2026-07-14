# Changelog

Notable changes to the public OPPO A1K LK port are recorded here.

## [v0.2.0] - 2026-07-14

Second hardware-tested source release focused on fastboot diagnostics, safer
RAM boot, deterministic physical keys, and an on-device fastboot interface.

### Added

- Full-screen fastboot UI with `START`, `RESTART BOOTLOADER`, `RECOVERY MODE`,
  and `POWER OFF` actions, plus live partition and transfer progress.
- Build, boot-mode, boot-reason, watchdog, secure-state, panel, DRAM, battery,
  boot-stage, and LK-copy diagnostic `getvar` values.
- Read-only SHA-256 comparison of the complete `lk` and `lk2` partitions.
- Strict `fastboot boot` validation for header versions, component sizes,
  address ranges, downloaded length, overflow, and reserved-memory overlap.
- Persistent stage codes from LK entry through kernel handoff, with recovery
  of the previous code after a watchdog reset.
- Exact RTC UI SOC display from the MT6357 `RTC_AL_MTH` fuel-gauge field.

### Changed

- Fixed physical CPH1923 key codes are used without parsing KPD assignments
  from DTBO: `VOL-` enters fastboot, double `VOL-` opens Boot Menu, `VOL+`
  enters recovery or selects the next UI action, and `POWER` confirms it.
- Fastboot fetch now uses bounded 64-bit ranges and an MTK-QMU-safe chunk size
  for `expdb`, `boot`, `recovery`, `dtbo`, `system`, and `vendor`.
- Low-battery checks are limited to long `system`/`vendor` flash and fetch
  operations, preserving access to smaller repair partitions.
- Fastboot fetch logging is reduced while the on-screen UI retains useful
  operation, partition, percentage, elapsed-time, and transfer-rate data.

### Security

- `fastboot flashing lock` is not registered for this target.
- The legacy MediaTek relock path rejects direct calls.
- Fastboot verifies the persistent lock state at startup and before commands,
  restoring the unlocked development state if an unexpected relock is seen.

### Commands

- Added diagnostic variables including `lk-build`, `lk-git-revision`,
  `boot-mode`, `boot-reason`, `boot-stage`, `last-boot-stage`, `wdt-status`,
  `lock-state`, `sboot-state`, `panel`, `dram-size`, `expdb-size`,
  `battery-level`, and the split LK/LK2 SHA-256 values.
- Documented `fastboot boot`, `fastboot fetch`, staged `expdb` upload, reboot
  commands, physical keys, and every new variable in
  [`docs/FASTBOOT.md`](docs/FASTBOOT.md).

### Distribution

- The release remains source-only and contains no flashable LK image, device
  dump, extracted DTB/DTBO, signing material, or device identifier.

## [v0.1.0] - 2026-07-11

Initial hardware-tested source release for the OPPO A1K CPH1923 (project
18540, MT6765/MT6762 family).

### Added

- OPPO A1K target and project configuration based on `svoboda18/lk@37ec8bc`.
- Persistent PL, LK, ATF, and kernel diagnostics in the 20 MiB `expdb`
  partition.
- Stock-compatible normal, recovery, fastboot, META, Factory, Advanced META,
  and ATE boot-mode handling.
- Boot image, DTB, and DTBO loading fixes, including normal and recovery RAM
  boot through `fastboot boot`.
- ILI9881C panel initialization, backlight control, logo rendering, and a
  stock-compatible display handoff to Linux.
- Read-only fastboot `fetch` for `expdb`, `boot`, `recovery`, `dtbo`, `system`,
  and `vendor`, including 64-bit offsets and the MTK QMU boundary workaround.
- Barcode-derived device serial number and watchdog-assisted early-boot
  diagnostics.
- English and Russian implementation notes, provenance records, and a
  mixed-license redistribution audit.

### Distribution

- The release contains source documentation and one reproducible patch only.
- No flashable LK image, firmware dump, extracted DTB/DTBO, private signing
  key, or device identifier is distributed.

[v0.1.0]: https://github.com/calan152/oppo-a1k-lk/releases/tag/v0.1.0
[v0.2.0]: https://github.com/calan152/oppo-a1k-lk/releases/tag/v0.2.0
