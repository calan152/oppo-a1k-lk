# OPPO A1K (CPH1923) LK port

This repository documents and distributes a source patch for the MediaTek LK
port developed for the OPPO A1K (project 18540, MT6765/MT6762 family).

The repository intentionally contains **no flashable images, device dumps,
private signing keys, extracted DTB/DTBO files, or MediaTek prebuilt
libraries**. It is a clean-history patch repository, not a firmware mirror.

## What was implemented

- OPPO A1K target and project configuration;
- UART and persistent `expdb` LK logging;
- stock-compatible boot-mode selection and recovery handling;
- boot image, DTB and DTBO loading fixes;
- device LCM/display initialization and kernel display handoff;
- RAM boot for downloaded normal and recovery images;
- read-only fastboot `fetch` support for `expdb`, `boot`, `recovery`, `dtbo`,
  `system`, and `vendor`;
- normal-boot command-line and ramdisk handling;
- watchdog and handoff diagnostics used to isolate boot failures.

The investigation history and implementation notes are in
[`docs/OPPO_A1K_PORT.md`](docs/OPPO_A1K_PORT.md). A Russian overview is in
[`docs/README_RU.md`](docs/README_RU.md).

## Apply the patch

The patch is based on commit `37ec8bc` of
[`svoboda18/lk`](https://github.com/svoboda18/lk).

```bash
git clone https://github.com/svoboda18/lk.git
cd lk
git checkout 37ec8bc
git apply /path/to/0001-oppo-a1k-cph1923-port.patch
make -j8 oppo6762_18540 NO_SIGN=yes
```

The patch expects device-specific DTB/signing inputs to be supplied locally.
Do not commit keys, factory images, partition dumps, IMEI/NVRAM data, or files
extracted from retail firmware.

## Licensing and provenance

This is a mixed-provenance port and no single blanket license applies to the
whole repository. The base LK code keeps its original MIT license and
copyright notices. The LCM adaptation is derived from the GPL-licensed OPPO
A1K kernel source published at
[`Frostleaft07/kernel_source_OP486C`](https://github.com/Frostleaft07/kernel_source_OP486C),
which in turn identifies OPPO source material.

See the root [`LICENSE.md`](LICENSE.md),
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md), and
[`docs/REDISTRIBUTION_AUDIT.md`](docs/REDISTRIBUTION_AUDIT.md) before
redistributing or adding files. The presence of source code here does not
grant rights to OPPO trademarks, retail firmware, proprietary MediaTek blobs,
or third-party signing material.

This repository is an independent community project and is not affiliated
with or endorsed by OPPO or MediaTek.

## Safety

Bootloader development can permanently make a device unbootable. Keep verified
partition backups and test on hardware you own. No warranty is provided.
