# Egret2MAME

**Egret2MAME** is an independent Windows utility for using the **TAITO
EGRET II mini Paddle & Trackball Controller** with standalone MAME on
Windows.

The project was created because the controller's trackball and spinner
are exposed to Windows as normal mouse input, while its five physical
buttons use unusual HID usages that are not directly useful in MAME.

Egret2MAME bridges that gap without requiring custom Egret kernel
drivers.

## Current release

**Beta 0.815**

Reference environment:

-   Windows 11 x64
-   TAITO EGRET II mini Paddle & Trackball Controller
-   VID `0AE4`, PID `0701`
-   official standalone **MAME 0.289**
-   MAME keyboard input provider: **DInput**
-   MAME mouse input provider: game-dependent; the tested Arkanoid and
    Centipede settings are documented in `MAME_SETUP.md`

Other MAME versions, frontends and emulator configurations may work, but
are not guaranteed by this release.

## What Beta 0.815 does

Egret2MAME:

-   detects the EGRET II mini Paddle & Trackball Controller;
-   reads its USB interrupt reports through USBPcap;
-   decodes Trackball X/Y, Spinner and all five buttons;
-   keeps native Windows mouse movement for the trackball;
-   can multiply trackball movement from 1x to 6x;
-   provides spinner scaling from 0.25x to 1.00x;
-   converts the five controller buttons to normal Windows keyboard
    input for MAME;
-   provides live controller, button, trackball and spinner feedback;
-   stores user speed settings.

No Egret-specific test-signed kernel driver, ViGEm or virtual HID device
is required by Beta 0.815.

## Button mapping

  EGRET control   Keyboard output
  --------------- -----------------
  SELECT          `5`
  START           `1`
  MENU            `SPACE`
  FIRE (L)        `LCTRL`
  FIRE (R)        `LALT`

For games with one action button, such as the tested Arkanoid and
Centipede configurations, `P1 Button 1` may be assigned to `LCTRL`,
`LALT`, or both according to personal preference.

## Trackball and spinner

Trackball speed can be selected from **1x to 6x**. The application
default is **3x**.

Spinner speed can be selected from **0.25x, 0.50x, 0.75x or 1.00x**. The
application default is **0.50x**.

Saved settings override these defaults.

Game-specific MAME analog values are recommendations only and can be
adjusted to personal preference. See `MAME_SETUP.md` for the tested
Arkanoid and Centipede examples.

## Installation

1.  Run the Egret2MAME Beta 0.815 setup.
2.  Accept the Windows UAC prompt.
3.  USBPcap is installed automatically if it is not already present.
4.  Reboot Windows when requested.
5.  Connect the EGRET II mini Paddle & Trackball Controller.
6.  Start Egret2MAME.
7.  Start MAME and configure the required game inputs.

**Secure Boot is supported but not required.**

**Windows Test Signing is not required.**

Egret2MAME runs elevated because direct access to the USBPcap capture
interface requires administrator access in the tested configuration.

## Uninstallation

If the Egret2MAME installer installed USBPcap, the uninstaller also
removes that USBPcap installation. If USBPcap was already installed
before Egret2MAME, it is left installed.

## MAME configuration

Beta 0.815 was developed and tested primarily with official standalone
**MAME 0.289**.

`MAME_SETUP.md` contains illustrated, tested reference setups for:

-   **Arkanoid (World, older)** --- Spinner / Dial
-   **Centipede (revision 4)** --- Trackball

The same basic setup method should work with other MAME spinner/dial and
trackball games, although analog sensitivity, reverse and
increment/decrement values may need game-specific adjustment.

## Third-party software

Beta 0.815 uses the official **USBPcap** driver as its runtime capture
component.

See `THIRD_PARTY_NOTICES.md` for credits and licensing information.

The repository also contains the USBPcap 1.5.4.0 compliance/source package under `licenses/`. It includes the complete corresponding upstream USBPcap 1.5.4.0 source archive used for this release and an accompanying notice. Egret2MAME's own source code is not part of the initial Beta 0.815 binary release.

## Development history

The project went through several approaches before reaching Beta 0.815,
including Raw Input experiments, direct HID access, virtual controller
output and custom driver experiments.

The final Beta 0.815 architecture uses Windows input, direct USBPcap
capture and normal keyboard/mouse injection.

See `DEVELOPMENT.md` for the detailed technical history.

## Status

Beta 0.815 has been tested on a clean Windows 11 PC with:

-   installation: PASS
-   reboot: PASS
-   controller detection: PASS
-   multiple USB ports: PASS
-   trackball: PASS
-   spinner: PASS
-   all five buttons: PASS
-   saved settings: PASS
-   uninstallation: PASS
-   Secure Boot enabled: PASS
-   Test Signing disabled: PASS
-   MAME 0.289 Arkanoid spinner setup: PASS
-   MAME 0.289 Centipede trackball setup: PASS
-   Windows 150% GUI scaling: PASS
-   UHD / Windows 250% GUI scaling: PASS

## Credits

Project / implementation / testing: **WiGGi**

Development assistance: **OpenAI ChatGPT**

See `CREDITS.md` and `THIRD_PARTY_NOTICES.md`.

## Disclaimer

Egret2MAME is an unofficial, independent community project.

It is not affiliated with, endorsed by, sponsored by or supported by
TAITO Corporation, MAMEDev, Microsoft, the Wireshark Foundation or the
USBPcap project.

TAITO and EGRET II mini are referenced only to identify compatible
hardware. MAME is referenced only to identify the emulator used as the
primary development and test target.
