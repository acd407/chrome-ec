# Primus EC Firmware

**English** | [简体中文](README.zh-CN.md)

Custom Chromium OS Embedded Controller firmware for Google Primus (Chromebook, brya baseboard), intended to run under **non-ChromeOS systems** (self-built Linux).

## Hardware Platform

- **Device**: Google Primus (brya baseboard, Alder Lake-P)
- **EC Chip**: Nuvoton NPCX993F
- **USB-PD**: RT1715 TCPC (`CONFIG_USB_PD_TCPM_RT1715`)
- **Retimer**: Burnside Bridge (`bb_retimer`)

## Branch Notes

`primus-rw` is based on redrix upstream `0654d51ba4`. That base was chosen because it does **not** contain the destructive commit `393bea15b8` ("brya: Enable CONFIG_SVDM_RSP_DFP_ONLY"), which removed the whole brya TBT/USB4 SVDM responder from `baseboard/brya/usb_pd_policy.c` (186 lines). With it, the peer sees this machine as having no UFP modal, no USB4 capability and no Intel SVID.

Commits on this branch, relative to the base:

| Commit | Description |
|---|---|
| `e1f9578dc4` | **Primus-specific**: enable autonomous USB4/TBT alt-mode entry |
| `cc288a3984` | Generic: add debug output for bad commands (cherry-picked from redrix-rw) |
| `054fd9c9a4` | Generic: npcx rom_chip, suppress GCC `-Warray-bounds` |
| `ec64ab774e` | Generic: ecst, suppress GCC 14 false positives |
| `6b9b2143ad` | Generic: keyboard_backlight, re-register driver after sysjump |

## EC Issues and Fixes

### USB PD AP Mode Entry (USB4/Thunderbolt Auto-Negotiation)

#### Problem

When `CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY` is enabled, the EC does not autonomously enter any alt mode (DP/TBT/USB4). Instead it waits for the AP to direct mode entry via host commands (`ectool typeccontrol`). This is standard ChromeOS behavior — the AP decides whether to enter DP or USB4 based on user settings and display requirements.

On non-ChromeOS systems there is no AP-side component to issue these host commands, so the EC remains stuck in USB3 mode and never negotiates USB4/Thunderbolt:

- `ectool inventory` shows `42: Host-controlled Type-C mode entry`
- `boltctl` never lists a peer device
- `/sys/class/typec/` has no USB4/Thunderbolt entries

#### Fix

Clear the option (`primus/board: enable autonomous USB4/TBT alt-mode entry`). The EC now enters the best mutually-supported mode (USB4 > TBT > DP) without waiting for AP direction, and `EC_FEATURE_TYPEC_REQUIRE_AP_MODE_ENTRY` (bit 42) disappears.

Also add `CONFIG_USB_PD_DATA_RESET_MSG`, which is mandatory for USB4 negotiation. Redrix has always set it; primus did not.

#### Known Limitation: host-to-host Still Needs a Kernel Patch

The fix above makes the EC side negotiate autonomously, which is enough for **host-to-peripheral** links (a USB4/TBT dock or eGPU). It is *not* enough for **host-to-host**: PD completes, but the Thunderbolt link still never trains and `usb4_port*/link` stays `none`.

The remaining gap is in the kernel. `cros_ec_typec.c` registers the Thunderbolt compatibility alternate mode only when `ap_driven_altmode` is set, i.e. when the EC reports `EC_FEATURE_TYPEC_REQUIRE_AP_MODE_ENTRY`. With an autonomous EC that condition is false, so `port_altmode[CROS_EC_ALTMODE_TBT]` stays `NULL` and `cros_typec_enable_tbt()` programs the SoC Type-C mux into SAFE_MODE.

The kernel-side fix (register unconditionally with `mode_selection = true`, leaving the USB4/DP paths untouched) is packaged as a DKMS module:

<https://github.com/acd407/cros-ec-typec-dkms>

> Note: do not be misled by the name `CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY`. The kernel's `cros_ec_typec` driver is **not** the missing AP — it does implement `EC_CMD_TYPEC_CONTROL`, but with an autonomous EC it deadlocks on the `-EPERM` check in `typec_altmode_enter()` (the EC waits for the kernel to enter the mode while the kernel waits for the EC to report the port alt mode active). ChromeOS breaks that cycle with the typecd daemon. Restoring this option would also break the automatic USB4 negotiation for an eGPU dock, because the kernel never sends `Enter_USB` on its own. **Keep the EC autonomous and patch the kernel.**

### Keyboard Backlight Lost After Sysjump

When the EC sysjumps from RO to RW, its BSS is zeroed and `kblight.drv` becomes NULL. The keyboard backlight driver registers itself from `HOOK_CHIPSET_STARTUP`, which only fires on a cold AP boot (S5→S4→S3); after a sysjump the chipset is already in S0, so the hook never runs.

The fix (`6b9b2143ad`, same as redrix) adds a re-registration path from `HOOK_INIT` for the post-sysjump case.

### GCC 14+ Build Warnings

Modern GCC (14 and later) reports the following as `-Werror`-level false positives; both are suppressed in `ec64ab774e` and `054fd9c9a4`:

- `util/ecst.c`: intentional patterns misjudged
- `chip/npcx/rom_chip.c`: ROM API table past the end of an array (legal in practice)

## Building

An arm-none-eabi cross-compilation toolchain is required.

```bash
make CROSS_COMPILE=arm-none-eabi- BOARD=primus
```

Build artifacts:

| File | Size | Description |
|---|---|---|
| `build/primus/ec.bin` | 512KB | Full RO + RW image |
| `build/primus/RO/ec.RO.flat` | ~252KB | RO firmware only |
| `build/primus/RW/ec.RW.bin` | ~243KB | RW firmware only (flash this) |

> The build may end with `CHECK_ALLOWED ... env: "vpython3": No such file or directory`. That warning is unrelated; the firmware images have already been produced.

## Flashing the EC Firmware

### Safe Flashing Procedure (RW only, RO untouched)

```bash
# 1. Back up the current full EC firmware
sudo ectool flashread 0 524288 ec_backup.bin

# 2. Erase the RW region
sudo ectool flasherase 0x40000 262144

# 3. Write the new RW firmware
sudo ectool flashwrite 0x40000 build/primus/RW/ec.RW.bin

# 4. Read back and verify
SZ=$(stat -c%s build/primus/RW/ec.RW.bin)
sudo ectool flashread 0x40000 $SZ check.bin
sha256sum build/primus/RW/ec.RW.bin check.bin   # must match

# 5. Reboot the EC into RW
sudo ectool reboot_ec cold
```

**Warning**: `reboot_ec` triggers a **full system reboot** (the EC drives the AP power sequence), so a remote SSH session drops for roughly 60–90 seconds. When operating remotely, defer it with `nohup`:

```bash
nohup bash -c 'sleep 3; ~/tmpsudo ~/ectool reboot_ec cold' > ~/reboot.log 2>&1 &
```

### Verifying the Flash

```bash
sudo ectool version
# Firmware copy: RW      <- key line
# RW version: primus_v0.0.xxxx-xxxxxxxxxx
```

## Troubleshooting

### EC falls back to RO after flashing

A failed RW image verification causes a fallback. Check the `Firmware copy` field in `ectool version` — if it reads `RO`, the RW image did not verify and must be re-flashed.

### RO region protection

The RO region is hardware write-protected and is never touched by this procedure. Do **not** attempt `flasherase 0`.

## License

EC firmware code is copyright ChromiumOS Authors and follows its original license (see `LICENSE`). Modifications in this branch follow the same license.
