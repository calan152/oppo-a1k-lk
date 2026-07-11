# Third-Party Notices And Provenance

This repository is a research port assembled from multiple codebases. It is
not offered under one blanket license. Every file keeps its original copyright
notice and license terms.

## Little Kernel Base

The MIT text supplied with the Little Kernel base is preserved at
[`LICENSES/MIT.txt`](LICENSES/MIT.txt). It applies only to material whose
copyright holder placed it under those terms; it does not override notices in
MediaTek, OPPO, Linux, GNU, or other third-party files. The root
[`LICENSE.md`](LICENSE.md) is a licensing overview, not a blanket grant.

## OPPO A1K Kernel-Derived Panel Driver

`dev/lcm/ili9881c_hd_dsi_vdo_txd_boe_zal1890/`

The ILI9881C panel implementation was adapted for LK from the OPPO A1K kernel
source lineage. The source reference is:

```text
https://github.com/Frostleaft07/kernel_source_OP486C
forked from: oppo-source/A1k-9.0-kernel-source
path: drivers/misc/mediatek/lcm/ili9881c_hd_dsi_vdo_txd_boe_zal1890/
```

The upstream file retains its OPPO copyright header. The kernel-source tree
contains `COPYING` and is distributed as GPLv2 source. Keep the source header,
keep this notice, document subsequent modifications, and distribute the
corresponding source when distributing a binary containing this derivative.

## MediaTek And Android-Derived Code

The repository includes components bearing MediaTek and Android copyright
notices. Their presence does not imply that OPPO or MediaTek granted a new
license through this repository. Preserve each source-file notice and obtain
separate permission where a file does not state an open-source license.

## Toolchain And Build Utilities

The `gcc/` directory and several build utilities are third-party toolchain
artifacts. Their accompanying notices, including `gcc/NOTICE`, must remain
with any redistribution. This project does not claim ownership of them.

## Intentionally Excluded Inputs

The following are local-only and must not be committed or attached to releases:

- `main_dtb_header.bin` and any DTB extracted from stock firmware;
- `certs/IMG_AUTH_KEY.ini` and `certs/VERIFIED_BOOT_IMG_AUTH_KEY.ini`;
- private RSA exponents, signing keys, certificate bundles, and unlock tokens;
- OPPO firmware packages, partition dumps, `expdb` logs, and device records.

The public build supports `NO_SIGN=yes`; producing a signed image requires
keys that the builder is entitled to use.

## No Trademark Or Affiliation

OPPO, MediaTek, Android, and Little Kernel are used only to describe
compatibility and provenance. This is an independent research project and is
not endorsed by any of those parties.
