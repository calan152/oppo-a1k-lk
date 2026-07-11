# Redistribution Audit

This is a technical provenance review, not legal advice. It records what was
checked before public release of the OPPO A1K port on 2026-07-10.

## Clean Repository Result

The published patch repository excludes private signing-key files, extracted
DTB/DTBO data, `scripts/boot.img`, flashable images, partition dumps, build
output, and the inherited BSP's prebuilt MediaTek/toolchain libraries.

The patch deliberately omits deletions of historical key files so their former
contents cannot leak through patch context. Signing inputs stay outside version
control; the documented build uses `NO_SIGN=yes` unless a developer supplies
their own authorized development keys.

## Findings

| Category | Finding | Public-release treatment |
| --- | --- | --- |
| LK base | The LK base uses MIT while imported files carry additional notices. | Keep MIT at `LICENSES/MIT.txt`; use root `LICENSE.md` only as a mixed-license overview. |
| Kernel-derived LCM | The ILI9881C driver has an OPPO copyright header and was taken from the OPPO A1K GPL kernel-source lineage. | Retain headers, provide provenance, and publish the corresponding source. |
| Stock DTB | `main_dtb_header.bin` is a binary DTB/header derived from stock firmware. | Removed from Git tracking; each developer obtains it locally from firmware they may use. |
| Signing material | `certs/IMG_AUTH_KEY.ini` contains `CUSTOM_RSA_D`; `VERIFIED_BOOT_IMG_AUTH_KEY.ini` contains `AUTH_PARAM_D`. | Removed from Git tracking and ignored. They are treated as private key material. |
| Firmware and diagnostics | `expdb`, OFP packages, device identifiers, partition dumps, and local binaries can contain proprietary or personal data. | Ignored and prohibited from releases. |
| Prebuilt tools/libraries | The inherited BSP includes prebuilt toolchain and vendor artifacts with varying notices. | Preserve notices; do not claim ownership or relicensing rights. |

## Remaining Risk

Removing files from the current tree does not erase them from already-published
Git history, forks, clones, caches, or releases. A full public remediation
requires making the current repository private or deleting it, then publishing
a new clean patch-only repository with fresh history.

## Recommended Public Release Model

1. Keep the complete hardware research checkout private.
2. Publish a clean repository containing documentation, target configuration,
   source patches, and GPL-compliant kernel-derived source only.
3. Require users to supply their own legally obtained DTB and signing material.
4. Attach no flashable LK binary unless its entire corresponding source and all
   redistribution rights have been audited.
5. Preserve upstream copyright notices and link to the relevant source
   repositories and commits.

## Pre-Release Checklist

- [ ] No file contains an IMEI, serial number, MAC address, token, password,
      private key, or real device log.
- [ ] No raw firmware, OFP package, boot image, DTB, logo, or partition dump is
      added without confirmed redistribution permission.
- [ ] Every imported GPL file retains its header and a source/provenance link.
- [ ] The README says that no single license applies to the entire tree.
- [ ] A qualified lawyer has reviewed the intended distribution if the project
      will be widely published or sold.
