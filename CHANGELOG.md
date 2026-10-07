# Changelog

## Beta 0.815 --- Initial public beta

### Added

-   Direct EGRET II mini Paddle & Trackball Controller input through
    USBPcap.
-   Automatic controller detection using VID `0AE4` / PID `0701`.
-   Automatic USBPcap root-hub matching.
-   Automatic detection of the controller's current USB device address.
-   Decoding of the controller's five-byte interrupt report.
-   Trackball X/Y input.
-   Spinner input.
-   All five physical buttons.
-   MAME-oriented keyboard output:
    -   SELECT → `5`
    -   START → `1`
    -   MENU → `SPACE`
    -   FIRE (L) → `LCTRL`
    -   FIRE (R) → `LALT`
-   Trackball speed control from 1x to 6x; default **3x**.
-   Spinner speed control from 0.25x to 1.00x; default **0.50x**.
-   Live controller/button/movement display.
-   Persistent speed settings.
-   Compact GUI validated with Windows 150% display scaling.
-   `by WiGGi` author credit in the application.
-   Notices/About information in the application.
-   Windows installer with automatic USBPcap installation when required.
-   Ownership-aware uninstall behavior so a pre-existing USBPcap
    installation is preserved.
-   Illustrated `MAME_SETUP.md` with tested MAME 0.289 reference setups
    for Arkanoid spinner/dial input and Centipede trackball input.

### MAME reference configuration

The documented MAME 0.289 setup now includes:

-   Arkanoid: `Dial Device Assignment = mouse`
-   Arkanoid: `Keyboard Input Provider = dinput`
-   Arkanoid: `Dial Analog = Mouse Scroll V`
-   Arkanoid recommended analog starting values: speed `15`, reverse
    `On`, sensitivity `1`
-   Centipede: `Trackball Device Assignment = mouse`
-   Centipede: `Keyboard Input Provider = dinput`
-   Centipede: `Mouse Input Provider = dinput`
-   Centipede: Trackball X/Y mapped to Mouse X/Y axes
-   Centipede recommended analog starting values: X/Y speed `10`, X
    reverse `On`, Y reverse `Off`, X/Y sensitivity `50`
-   P1 Button 1 may use `LCTRL`, `LALT`, or both

Analog values are recommendations only and may be adjusted to personal
preference.

### Architecture

Beta 0.815 no longer requires the earlier Egret-specific test-signed
kernel drivers, virtual HID button driver or ViGEm experiments.

Final architecture:

`EGRET controller → USBPcap → Egret2MAME → Windows keyboard/mouse input → MAME`

### Validated

Clean Windows 11 validation passed for installation, reboot, controller
detection, multiple USB ports, trackball, spinner, five buttons and
uninstallation.

Secure Boot is supported but not required. Windows Test Signing is not
required.

The final GUI layout was also validated at 150% Windows display scaling.

Arkanoid spinner and Centipede trackball configuration were freshly
verified in official standalone MAME 0.289.

### Reference target

Official standalone **MAME 0.289 for Windows**.
