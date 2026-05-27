# Redrix EC Firmware

Custom Chromium OS Embedded Controller firmware for the HP Elite Dragonfly Chromebook (redrix).

## Hardware Platform

- **Device**: HP Elite Dragonfly Chromebook (redrix)
- **EC Chip**: Nuvoton NPCX9M3F
- **EC Flash**: 512KB (internal flash, independent from the main 32MB SPI BIOS flash)

## EC Flash Layout and Dual-Firmware Mechanism

```
Offset        Size    Content
0x00000      256KB    RO region (hardware write-protected, locked at factory)
0x40000      256KB    RW region (freely writable, used for normal operation)
```

EC boot sequence:

1. NPCX boot ROM reads the flash header and copies the RO image from flash to SRAM
2. EC boots from RO
3. The AP issues `reboot_ec RW`; RO code calls the `download_from_flash()` ROM API
4. NPCX copies the RW image from flash to SRAM and jumps to it (sysjump)

If RW is corrupted or fails verification, the EC falls back to RO. RO serves as the factory safety net, while RW is the primary operational firmware.

## EC Issues and Fixes

This branch is based on mainline chrome-ec with the following fixes for redrix when running non-ChromeOS systems:

### Keyboard Backlight Lost After Sysjump

#### Symptom

After flashing new RW firmware (`ectool reboot_ec RW` or system reboot), the keyboard backlight stops working:

```
$ sudo ectool pwmsetkblight 100
Keyboard backlight set.
$ sudo ectool pwmgetkblight
Keyboard backlight disabled.
```

`pwmsetkblight` reports success but the hardware does not respond.

#### Root Cause

The keyboard backlight driver struct `kblight` is a static global variable (`common/keyboard_backlight.c`):

```c
static struct kblight_conf kblight;  // BSS section, initially kblight.drv = NULL
```

The driver registration function `keyboard_backlight_init()` is bound to `HOOK_CHIPSET_STARTUP`:

```c
DECLARE_HOOK(HOOK_CHIPSET_STARTUP, keyboard_backlight_init, HOOK_PRIO_DEFAULT);
```

`HOOK_CHIPSET_STARTUP` only fires during the chipset transition from S5 (power-off) → S4 → S3, which is the AP cold-boot power sequence. After a sysjump, the chipset is already in S0 (running), so the hook will not fire again.

Timing diagram:

```
RO firmware (factory)
  Cold boot → G3→S5→S4→S3
    → HOOK_CHIPSET_STARTUP fires
    → kblight_register() → kblight.drv points to PWM driver ✅

  AP boots → ectool reboot_ec RW
    → sysjump to RW
───────────────────────────────────────────────────
RW firmware (newly flashed)
  sysjump, BSS cleared → kblight.drv = NULL
  chipset already in S0
    → HOOK_CHIPSET_STARTUP won't fire ❌
    → kblight.drv = NULL permanently

  AP issues pwmsetkblight 100:
    → kblight_set(100) → variable set successfully ✔
    → kblight_enable(1) → deferred call scheduled ✔
    → kblight_enable_deferred runs:
      → if (!kblight.drv) return;  ← NULL, silently skipped
    → PWM hardware is never touched ❌
```

`kblight_set()` and `kblight_enable()` only set in-memory variables and schedule deferred calls — they do not touch hardware. The actual PWM operation happens in the deferred function, but it checks whether `kblight.drv` is NULL and silently returns if so. This is why `ectool pwmsetkblight 100` reports success while the hardware remains untouched.

#### Fix

Added a sysjump re-registration path in `common/keyboard_backlight.c`. The new function is registered on `HOOK_INIT` (which also fires on sysjump), checks `system_jumped_to_this_image()` to detect sysjump, and only reassigns `kblight.drv` without reinitializing PWM hardware (hardware registers are preserved across soft jumps).

- **Cold boot**: `HOOK_CHIPSET_STARTUP` → normal init; `HOOK_INIT` also fires but the sysjump check fails, so it returns immediately.
- **Sysjump**: `HOOK_CHIPSET_STARTUP` won't fire; `HOOK_INIT` → re-registers the driver.

Commit: `keyboard_backlight: re-register driver after sysjump`

### MKBP Host Event Delivery (Side Volume Buttons Not Working)

#### Symptom

The side buttons (volume up/down/power) are registered in the Linux input subsystem:

```
$ evtest /dev/input/event7
Input device name: "cros_ec_buttons"
  Event code 114 (KEY_VOLUMEDOWN)
  Event code 115 (KEY_VOLUMEUP)
  Event code 116 (KEY_POWER)
```

But pressing the buttons produces no events in `evtest`. Regular `ectool` commands (`ectool version`, `ectool flashread`, etc.) work fine.

#### Root Cause

The Chrome EC has two independent hardware paths for notifying the AP:

| Path | Mechanism | Hardware Signal |
|------|----------|----------------|
| **GPIO interrupt** | EC pulls `GPIO_EC_PCH_INT_ODL` low | Dedicated EC→PCH GPIO pin |
| **Host event + SCI** | EC triggers SCI via eSPI virtual wire | eSPI virtual wire (SERIRQ) |

These paths are completely independent and handled by different AP hardware blocks.

The GPIO interrupt path does not work on custom Linux (non-ChromeOS) because the ACPI GPIO interrupt mapping is incomplete — the AP's GPIO controller driver does not correctly register the interrupt. As a result, the kernel's `cros_ec` driver never calls `EC_CMD_GET_NEXT_EVENT` to consume MKBP events. This is confirmed by the EC boot log:

```
[4.097459 already in S0]              ← power_chipset_init confirms AP is in S0
[4.151074 mkbp switches: 1]           ← MKBP event generated
[6.152177 MKBP: The AP is failing to respond despite being powered on.]
          ↑ EC explicitly knows the AP is in S0, yet the AP still fails to consume events
```

(The keyboard continues to work because it uses the 8042 protocol (eSPI PS/2 compatibility) and does not depend on MKBP interrupts.)

#### Fix

Instead of the GPIO interrupt path, enable the **host event + SCI** path as a fallback. SCI (System Control Interrupt) is delivered over the eSPI virtual wire, a separate hardware channel independent of the GPIO bus. Two changes are required:

**Fix 1: `common/mkbp_event.c` — always send host event even in S0**

The original code only set `EC_HOST_EVENT_MKBP` when the AP was in suspend state, since on ChromeOS this event is not in the SCI mask during S0. The fix always sets the host event, attempting SCI delivery in S0 as well.

**Fix 2: `board/redrix/board.c` — configure the SCI mask**

The NPCX chip checks the SCI mask to decide whether to actually generate an SCI pulse. The default SCI mask is 0 (BSS initialization), so `EC_HOST_EVENT_MKBP` must be explicitly added. Additionally, the SCI mask is cleared during every S3→S0 transition by `lpc_s3_resume_clear_masks()`, so it must be restored via the `HOOK_CHIPSET_RESUME` hook.

Post-fix event path:

```
Physical button press
    → EC GPIO interrupt → mkbp_fifo_add()
    → activate_mkbp_with_events()
    → host_set_single_event(EC_HOST_EVENT_MKBP)  ← New: also in S0
    → NPCX eSPI SCI virtual wire
    → AP PCH receives SCI → ACPI SCI handler
    → cros_ec driver → EC_CMD_GET_NEXT_EVENT
    → cros_ec_buttons → input subsystem → evtest ✅
```

Commits:
- `redrix/board: enable SCI delivery for MKBP host events`
- `mkbp_event: send host event in S0 as fallback notification`

### USB PD AP Mode Entry (USB4/Thunderbolt Auto-Negotiation)

#### Problem

When `CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY` is enabled, the EC does not autonomously enter any alt mode (DP/TBT/USB4). Instead, it waits for the AP to direct mode entry via host commands (`ectool typeccontrol`). This is standard ChromeOS behavior — the AP decides whether to enter DP or USB4 based on user settings and display requirements.

On non-ChromeOS systems, there is no AP-side component to issue these host commands, so the EC remains stuck in USB3 mode and never negotiates USB4/Thunderbolt.

#### Fix

Undefine this config option (`redrix/board: undef CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY`). The EC now enters the best mutually-supported mode (USB4 > TBT > DP) without waiting for AP direction.

## Building

An arm-none-eabi cross-compilation toolchain is required.

```bash
make CROSS_COMPILE=arm-none-eabi- BOARD=redrix
```

Build artifacts:

| File | Size | Description |
|---|---|---|
| `build/redrix/ec.bin` | 512KB | Full RO + RW image |
| `build/redrix/RO/ec.RO.flat` | ~252KB | RO firmware only |
| `build/redrix/RW/ec.RW.bin` | ~243KB | RW firmware only (flash this) |

### Adding a Custom Version Tag

Edit the `build_info` string in `common/version.c` to include a tag like `LOCAL_BUILD`:

```c
const char build_info[] =
    VERSION " " CROS_FWID32 " " DATE " " BUILDER " LOCAL_BUILD";
```

After rebuilding, the Build info line in `ectool version` will show the tag, making it easy to distinguish custom firmware from official builds.

## Flashing the EC Firmware

### Check Flash Info

```bash
sudo ectool flashinfo
```

Typical output:
```
FlashSize 524288
WriteSize 1
EraseSize 65536
ProtectSize 65536
WriteIdealSize 240
Flags 0x0
```

Total flash size is 512KB (524288 bytes). The lower 256KB is RO, and the upper 256KB is RW. We flash the RW region.

### Safe Flashing Procedure (RW only, RO untouched)

```bash
# 1. Back up the current full EC firmware
sudo ectool flashread 0 524288 ec_backup.bin

# 2. Erase the RW region
sudo ectool flasherase 0x40000 262144

# 3. Write the new RW firmware
sudo ectool flashwrite 0x40000 build/redrix/RW/ec.RW.bin

# 4. Reboot EC to RW (or reboot the whole system)
sudo ectool reboot_ec RW
```

### Verifying the Flash

```bash
# Read back and compare
sudo ectool flashread 0x40000 262144 rw_readback.bin
sha256sum build/redrix/RW/ec.RW.bin rw_readback.bin

# Confirm the new firmware is running
sudo ectool version
# Firmware copy: RW    ← Key line
# Build info: ... LOCAL_BUILD  ← Custom tag
```

## Disable EC Software Sync in BIOS

If you are using [mrchromebox.tech](https://mrchromebox.tech/) coreboot/edk2 firmware to replace the stock ChromeOS firmware (which most users do), you must disable EC Software Sync in the BIOS to prevent the AP firmware from overwriting the EC RW partition at boot.

### Steps

1. Reboot and press `ESC` to enter the BIOS setup screen
2. Locate the **EC Software Sync** option and disable it
3. Save and exit

![BIOS EC Software Sync setting](docs/images/BIOS_EC_Software_Sync.jpg)

If this option is left enabled, the AP firmware will write its embedded EC RW image into the EC at every boot, overwriting your custom firmware.

## FAQ

### EC Falls Back to RO After Flashing

- Try a full system reboot (`sudo reboot`) so the EC goes through a complete power sequence
- Ensure the RW region is erased before writing (`flasherase` + `flashwrite`)
- If building outside the ChromeOS SDK, `CROS_FWID_MISSING` in the fwid field is normal

### RO Region Protection

The RO region has hardware write protection (controlled by a motherboard GPIO level). Even if flashwrite starts from offset 0, writes to the RO region are rejected by hardware. There is usually no need to modify RO.

## Version Strings

Example `ectool version` output:

```
RO version:    redrix_v2.0.26378-d5dba7885d
RO cros fwid:  redrix_14505.831.0
RW version:    redrix_v2.0.27860-0f0590ceec
RW cros fwid:  redrix_16238.2+tbt5
Firmware copy: RW
Build info:    ... DATE BUILDER [LOCAL_BUILD]
```

The version number is generated by `util/getversion.sh`: `git describe` finds the nearest tag, the commit count is the number of commits from the tag to HEAD, and a `+` suffix is appended for uncommitted changes (dirty marker).
