# Changelog

Notable changes to the public OPPO A1K LK port are recorded here.

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
