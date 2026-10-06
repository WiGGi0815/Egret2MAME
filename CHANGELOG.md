# Changelog

## Beta 0.815 — Initial public beta

### Added

- Direct EGRET II mini Paddle & Trackball Controller input through USBPcap.
- Automatic controller detection using VID `0AE4` / PID `0701`.
- Automatic USBPcap root-hub matching.
- Automatic detection of the controller's current USB device address.
- Decoding of the controller's five-byte interrupt report.
- Trackball X/Y input.
- Spinner input.
- All five physical buttons.
- MAME-oriented keyboard output:
  - SELECT → `5`
  - START → `1`
  - MENU → `SPACE`
  - FIRE (L) → `LCTRL`
  - FIRE (R) → `LALT`
- Trackball speed control from 1x to 6x.
- Spinner speed control from 0.25x to 1.00x.
- Live controller/button/movement display.
- Persistent speed settings.
- Windows installer with automatic USBPcap installation when required.
- Ownership-aware uninstall behavior so a pre-existing USBPcap installation is preserved.

### Architecture

Beta 0.815 no longer requires the earlier Egret-specific test-signed kernel drivers, virtual HID button driver or ViGEm experiments.

Final architecture:

`EGRET controller → USBPcap → Egret2MAME → Windows keyboard/mouse input → MAME`

### Validated

Clean Windows 11 validation passed for installation, reboot, controller detection, multiple USB ports, trackball, spinner, five buttons and uninstallation.

Secure Boot is supported but not required. Windows Test Signing is not required.

### Reference target

Official standalone **MAME 0.289 for Windows** using DInput or Win32 input.
