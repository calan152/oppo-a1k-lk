# Fastboot Safety And Recovery

This target adds fail-closed safeguards around the operations most likely to
leave the device unbootable. They are target-specific and do not turn an
experimental bootloader into a general-purpose recovery tool.

## Critical Flash Verification

The `lk`, `lk2`, `boot`, `recovery`, and `dtbo` partitions use this completion
sequence:

1. Validate the target and downloaded image.
2. Calculate SHA-256 over the downloaded bytes.
3. Write the image.
4. Flush the eMMC cache, when it is enabled.
5. Read the complete written image back from offset zero.
6. Calculate SHA-256 over the read-back bytes and compare it exactly.
7. Report `OKAY` and 100 percent only after the comparison succeeds.

Sparse images are rejected for these critical partitions. The legacy MTK
ultra-flash path is also disabled for them so it cannot bypass read-back
verification. Aggregate bootloader images cannot update multiple critical
partitions in one command.

An `lk` or `lk2` write additionally requires at least 3700 mV. External power
is recommended and reported to the host, but it is not treated as a substitute
for the voltage check. Flash each LK copy with a separate command and test one
known-good copy before replacing the other.

If verification fails, fastboot reports:

```text
FLASH VERIFY FAILED
Partition: <name>
Device must not be rebooted
```

This path has been compiled and statically checked. A deliberate destructive
hash-mismatch test has not been performed on hardware.

## Relock Guard

The target does not register `flashing lock` and the legacy relock handler is
compiled to reject direct calls. It also queries the persistent lock state at
fastboot entry, before every USB command, and before every Fastboot UI action.

If the query fails or the state is not unlocked, the session becomes
fail-closed. Boot, reboot, poweroff, flash, erase, and all UI exit actions are
blocked. Read-only diagnostics remain available, along with the RAM download
transport and the explicit unlock flow. The screen shows:

```text
LOCKED - run fastboot flashing unlock
```

The guard never repairs `seccfg` automatically. The only intended state write
is the existing, explicit `fastboot flashing unlock` flow, including its normal
confirmation and data-wipe behavior. After a successful unlock, the guard
re-queries persistent state before allowing the device to leave fastboot.

On this target, `fastboot flashing unlock` opens the colored Fastboot UI
confirmation screen instead of the legacy text console. `CANCEL - KEEP LOCKED`
is selected by default. `VOL+` changes the selection and `POWER` confirms it,
so an unlock requires two deliberate physical-key actions. The orange header
pulses while LK waits, but no timeout or automatic selection is used.

The persistent state is queried before that screen is opened. If the device is
already unlocked, fastboot reports `already unlocked` and returns success
without showing a false locked warning, rewriting state, or wiping userdata.
If the state cannot be read, the command fails without making changes. Amber
is used for the destructive selection; red remains reserved for real errors.

There is deliberately no `fastboot flash unlock` alias: standard fastboot
parses that form as a request to write a partition named `unlock`.

## Rescue Profile

Build the restricted profile with:

```bash
make -j"$(nproc)" oppo6762_18540 NO_SIGN=yes OPPO_RESCUE_FASTBOOT=yes
```

Confirm it with:

```bash
fastboot getvar rescue-fastboot
```

The rescue command policy permits:

- `getvar` and bounded `fetch` diagnostics;
- `download` as the transport required by `boot` and `flash`;
- RAM-only `boot`;
- separate flashes of `boot`, `recovery`, `dtbo`, or `lk2`;
- `reboot` and `poweroff` while the relock guard is clear;
- explicit `flashing unlock` when the relock guard is active.

All other commands are rejected centrally, including erase, ultra-flash,
`seccfg`, preloader, NVRAM/NVDATA, protect partitions, and `lk` writes. The
normal build reports `rescue-fastboot: no`; the restricted build reports
`rescue-fastboot: yes`.

## Fastboot Boot Validation

The RAM boot path validates Android boot images before copying components:

- supported header version and page size;
- exact downloaded length and overflow-safe aligned sizes;
- kernel, ramdisk, recovery-DTBO, and DTB size limits;
- expected kernel, ramdisk, tags, and DTB addresses;
- no unsupported second stage;
- no overlap with LK, framebuffer, ATF, reserved memory, or scratch bounds.

The pure layout validator is shared with host tests:

```bash
make -C tests/bootimg_validator clean test
```

The suite covers bad magic, zero page size, oversized and truncated
components, invalid target addresses, unsupported second stage, and an invalid
DTB address without requiring a phone.

## Which LK Copy Booted?

The inspected MT6765 preloader source and the stock OPPO preloader binary use
the same policy. They read GPT active bits for `lk` and `lk2`, load `lk2` only
when the `lk` bit is zero and the `lk2` bit is nonzero, and load `lk` in every
other combination. The stock binary contains the matching messages `lk active
= ...`, `Loading LK2 Partition...`, and `Loading LK Partition...`.

The complete `boot_arg_t`, ATAG union, DRAM handoff buffer, and final
`bldr_jump(..., bootarg, size)` path were inspected. They contain boot
reason/mode, memory, logging, security, battery, and related platform data, but
no loaded-LK partition field or SRAM marker. The stock preloader strings also
contain no source/slot/copy handoff message.

Fastboot therefore exposes evidence separately:

```text
getvar:lk-active-bit
getvar:lk2-active-bit
getvar:lk-selection-policy
getvar:running-build-in
getvar:lk-source-evidence
getvar:preloader-bootarg-format
getvar:preloader-bootarg-addr
getvar:preloader-bootarg-size
getvar:preloader-bootarg-units
getvar:preloader-unknown-tags
```

`lk-selection-policy` predicts what the inspected preloader code would choose.
`running-build-in` searches both partition images for the exact, per-build
`LK_VER_TAG` embedded in the currently executing LK. A unique match is useful
evidence; `both`, `neither`, identical copies, or a GPT conflict remain
explicitly ambiguous. The boot-tag scan also reports any tag outside the known
MT6765 set instead of silently treating it as a source marker.

For that reason `getvar:active-lk-partition` intentionally remains `unknown`.
Do not advertise automatic fallback until a controlled hardware experiment
boots distinguishable `lk` and `lk2` images and independently records the
preloader's selection log. Deliberately corrupting either copy is not part of
the automated test suite.
