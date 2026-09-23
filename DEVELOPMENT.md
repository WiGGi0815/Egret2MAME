# Egret2MAME -- Development History

## How it started

I bought the **TAITO EGRET II mini Paddle & Trackball Controller**
hoping that, sooner or later, someone would create a proper Windows/MAME
driver or compatibility solution for it.

That never really happened. When I looked into using the controller
properly with MAME on Windows, the answer was basically: **it doesn't
work**.

So I decided to find out *why* --- and, with the help of ChatGPT,
started building my own solution.

What was supposed to be a small controller fix quickly turned into USB
packet captures, HID descriptors, RawInput experiments, virtual
controllers, driver development, MAME input-provider testing and a
surprising amount of reverse engineering.

**Egret2MAME was born.**

The goal became simple:

> Plug in the original EGRET II mini Paddle & Trackball Controller,
> start Egret2MAME, start MAME and play.

The project is intended as an independent community project. It is not
developed, endorsed, sponsored or supported by TAITO.

------------------------------------------------------------------------

## 1. Understanding the controller

The first task was finding out what the controller actually sends.

``` text
Vendor ID:   0x0AE4
Product ID:  0x0701
Product:     TAITO USB Paddle & Trackball Controller
```

Windows exposes the device primarily through a mouse HID collection.
Initial HID inspection revealed unusual button usages `0x17`, `0x18`,
`0x1A`, `0x1D` and `0x1E`. Unlike ordinary mouse buttons, these were not
simply appearing in MAME as conventional mouse buttons.

The strange situation was therefore: the trackball worked through
Windows, the buttons were not exposed to MAME in the useful form we
needed, and the spinner behaved differently again.

**What was the controller actually sending over USB?**

## 2. Attempt #1 -- Direct Windows HID access

The obvious solution was to open the HID device directly. The controller
reports a Generic Desktop / Mouse top-level collection and Windows
reported a six-byte HID input report.

Direct reads failed with:

``` text
Win32 Error 5
Access Denied
```

Windows owns the mouse collection exclusively, preventing the simple
user-mode reader we wanted.

``` text
Direct HID reader
STATUS: FAILED
REASON: Windows exclusive access to the mouse HID collection
```

## 3. HID Analyzer experiments

A small analyzer was built around `VID_0AE4` / `PID_0701`. The
descriptor showed button usages in Usage Page `0x09`, range
`0x16 ... 0x1E`, plus:

``` text
0x30 = X
0x31 = Y
0x38 = Wheel
```

This strongly suggested X/Y = trackball and Wheel = spinner/paddle. The
descriptor helped, but direct reading remained blocked, so we needed to
look below the normal HID layer.

## 4. Attempt #2 -- USBPcap

USBPcap was used to observe the real interrupt traffic. After decoding
the capture header, the actual controller payload turned out to be
**five bytes**, despite Windows reporting a six-byte HID input report:

``` text
Byte 0   Buttons Low
Byte 1   Buttons High
Byte 2   Trackball X
Byte 3   Trackball Y
Byte 4   Spinner
```

X, Y and Spinner are signed 8-bit relative deltas. Reports arrive at
roughly 8 ms / 125 Hz through interrupt-IN endpoint `0x81`.

This was the first major breakthrough.

## 5. Identifying every physical button

Pressing each control individually while watching the raw reports
produced the final mapping:

``` text
Pink Select  HID 0x17   Byte 0 = 0x02
Blue Start   HID 0x18   Byte 0 = 0x04
White Menu   HID 0x1A   Byte 0 = 0x10
Fire Left    HID 0x1D   Byte 0 = 0x80
Fire Right   HID 0x1E   Byte 1 = 0x01
```

The two large Fire buttons are independent. Usages `0x16`, `0x19`,
`0x1B` and `0x1C` appear unused in our testing.

## 6. Egret2MAME v0.19 -- First reliable decoder

v0.19 was the first build that reliably displayed Buttons, Trackball X/Y
and Spinner in real time. The USB protocol was no longer the mystery.

The next question was harder: **how do we feed those inputs back into
MAME?**

## 7. Attempt #3 -- Keyboard SendInput

We converted the unusual buttons into normal keyboard events with
Windows `SendInput`. It worked in ordinary Windows applications, but not
as required with MAME's RawInput path.

Changing MAME's provider could alter that behavior, but one of the
project's goals had already become clear:

> Egret2MAME should adapt to MAME rather than requiring users to
> reconfigure MAME around Egret2MAME.

The keyboard approach was therefore abandoned for the buttons.

## 8. Attempt #4 -- Virtual Xbox controller

The next experiment used **ViGEmBus + Nefarius.ViGEm.Client** to create
a virtual Xbox 360 controller:

``` text
Pink Select  -> Xbox BACK
Blue Start   -> Xbox START
White Menu   -> Xbox X
Fire Left    -> Xbox A
Fire Right   -> Xbox B
```

MAME recognized it immediately. This became the first successful button
bridge without special MAME keyboard-provider configuration.

## 9. Egret2MAME v0.22b -- Buttons solved

v0.22b became the reference build for button behavior:

``` text
EGRET button -> USB report -> Egret2MAME -> Virtual X360 controller -> MAME
```

All five physical buttons could be assigned normally in MAME.

## 10. Attempt #5 -- Trackball through an Xbox analog stick

Because ViGEm worked for buttons, we tried the trackball as an Xbox
stick. Technically it worked; practically it was wrong.

A trackball reports **relative movement** (`+3 X`, `-2 Y`), whereas a
joystick reports an **absolute position**. Fast spins felt unnatural.

``` text
STATUS: REJECTED
REASON: Relative trackball motion does not map naturally to an absolute joystick axis
```

## 11. Attempt #6 -- Relative Windows mouse injection

Next, Egret2MAME kept the trackball relative and reinjected scaled mouse
movement. Multipliers of `1x` through `6x` were added.

This worked through ordinary Windows mouse handling, but MAME exposed an
important distinction: RawInput did not react to the injected/scaled
path in the same way as other providers.

## 12. Testing MAME input providers

We tested RawInput, DirectInput and Win32. DInput and Win32 responded to
the generated relative movement in our testing; RawInput behaved
differently.

That explained several apparently contradictory results. The behavior
depended partly on **which Windows input path MAME was reading**.

The primary target is current standalone MAME on Windows. RetroArch/MAME
2003 experiments are separate compatibility work and are not the main
development target.

## 13. Centipede testing

Centipede became a main trackball test. We experimented with MAME
sensitivity values around `100/100`, `150/100` and `200`, together with
Egret2MAME multipliers.

A combination that felt substantially better during testing was:

``` text
Egret2MAME Trackball: 3x
MAME X Sensitivity:   100
MAME Y Sensitivity:   100
```

These are development observations, not universal recommendations.
Current Egret2MAME default: **Trackball 3x**.

## 14. Spinner investigation

The purple spinner is byte five of the raw report. Positive movement
produced values such as `01 02 03`; the opposite direction produced
`FF FE FD`. It is a signed relative input corresponding to HID Wheel
usage `0x38`.

Arkanoid became a primary test. Full speed felt too sensitive, so
scaling was added:

``` text
0.25x
0.50x
0.75x
1.00x
```

`0.50x` and `0.75x` felt considerably more plausible during testing.
Current default: **Spinner 0.50x**.

## 15. Attempt #7 -- Building our own Windows driver

We investigated removing external capture/remapping dependencies with a
KMDF driver using Microsoft's Virtual HID Framework (VHF):

``` text
Physical EGRET -> our driver/reader -> Egret2MAME -> virtual HID -> MAME
```

The driver project compiled, but Windows driver security and Secure Boot
made deployment a much larger problem. A public custom driver needs a
proper production-signing/distribution strategy.

The route was **not technically abandoned**; it was postponed because of
installation and signing complexity.

## 16. Why USBPcap remained during development

USBPcap already provided reliable access to the raw traffic while
Windows continued to use the physical controller. It therefore remained
the practical development backend.

The next usability objective was automatic discovery.

## 17. Egret2MAME v0.28 -- Automatic controller detection

v0.28 scans `USBPcap1` through `USBPcap12` and discovers the appropriate
device/endpoint automatically, so moving the controller to another USB
port does not require manually editing capture settings.

The detection is not tied to one individual controller serial number.
Another controller of the same supported model should therefore be
discoverable through the same device/report characteristics.

## 18. Administrator privilege problem

One GUI build accidentally lost elevation in its startup script and
showed `Reports: 0` / controller not found. The decoder was fine;
USBPcap lacked the required privileges. Restoring elevation fixed it.

This is exactly the sort of failed detail worth documenting: it can save
another developer hours of debugging the wrong component.

## 19. GUI development

**v0.29:** first proper GUI.

**v0.30:** added a controller image and live visualization. An early
build hit a `NullReferenceException` because of obsolete GUI references.

**v0.30a:** fixed the crash and restored live visualization.

## 20. v0.31 -- Polishing the controller display

Live visualization was added for Select, Start, Menu, Fire Left, Fire
Right, Trackball and Spinner.

The five button overlays initially did not align perfectly with the
image, so a Button Alignment system was added for pixel nudging and the
corrected positions were later baked into the program.

## 21. v0.31e -- Button alignment

All five live button indicators were aligned with their physical
positions without altering the known-good input backend.

A useful development rule emerged:

> Do not break working controller input just to improve the GUI.

## 22. v0.31f -- Trackball Live Scope

The small live indicator on the yellow trackball was retained, while the
previously unused Trackball panel became a larger X/Y direction/scope
display.

At this point the development build has working USB auto-detection,
five-button decoding, virtual X360 button bridging, trackball and
spinner handling/scaling, and live GUI visualization.

## 23. Reference / Tested MAME Version

Egret2MAME was developed and tested primarily with the **official
standalone MAME 0.289 for Windows**, using MAME's current built-in user
interface and input system.

This is the project's reference configuration.

Older MAME releases, RetroArch cores, MAME 2003, third-party frontends
and other emulator/input configurations may also work, but they are
**not officially tested or guaranteed** unless explicitly documented by
the project.

Input handling can differ significantly between versions, cores and
frontends, particularly for analog controls such as the **trackball and
spinner/dial**. During development, for example, RetroArch with the MAME
2003 core behaved differently when configuring Arkanoid's spinner/dial
input.

That does not necessarily mean such configurations cannot work. It means
they are outside the currently tested support scope and users may need
to determine the appropriate emulator/input settings themselves.

When reporting an Egret2MAME issue, reproducing it with the official
standalone **MAME 0.289 Windows build** is therefore the preferred
reference test.

------------------------------------------------------------------------

## 24. Clean-PC goal

The current prototype proves the concept, but USBPcap and ViGEmBus are
still development/runtime dependencies.

The public-release target is:

> **Fresh Windows 11 + original controller -\> start Egret2MAME -\>
> start MAME -\> play.**

Ideally users should not need to hunt down drivers, install analysis
software, understand USB capture devices or manually edit technical
configuration.

Several additional PCs and a notebook are planned as clean-system test
machines before a public 1.0.

------------------------------------------------------------------------

# Current development architecture

``` text
             TAITO EGRET II mini
          Paddle & Trackball Controller
                       |
                       | USB
                       v
                Windows HID stack
                       |
              +--------+--------+
              |                 |
              v                 v
        Normal Windows       USBPcap
          mouse path            |
                                v
                           Egret2MAME
                                |
                 +--------------+--------------+
                 |                             |
                 v                             v
          Trackball/Spinner             Five Buttons
           mouse handling                    |
                                             v
                                      ViGEm virtual
                                     Xbox 360 controller
                                             |
                 +--------------+--------------+
                                v
                               MAME
```

This is the development architecture, not a promise that the same
dependencies will remain in the final public release.

# Things that did NOT work

  -----------------------------------------------------------------------
  Approach                Result                  Why
  ----------------------- ----------------------- -----------------------
  Direct HID reading      Failed                  Windows owns the mouse
                                                  HID collection

  Keyboard SendInput +    Failed for our goal     Injected input is not
  MAME RawInput                                   equivalent to normal
                                                  RawInput

  Trackball -\> X360      Rejected                Relative movement does
  stick                                           not map naturally to an
                                                  absolute joystick axis

  Mouse multiplication +  Limited/unsuitable      RawInput follows a
  RawInput                                        different input path

  Mouse multiplication +  Worked in testing       MAME receives generated
  DInput/Win32                                    relative movement
                                                  through these paths

  ViGEm virtual           Worked                  MAME recognizes the
  controller for buttons                          virtual X360 controller

  USBPcap raw capture     Worked                  Gives access to the raw
                                                  five-byte reports

  Custom KMDF/VHF driver  Experimental            Technically promising;
                                                  signing/deployment
                                                  remains a major issue
  -----------------------------------------------------------------------

A failed experiment is not wasted work if it prevents the next developer
from spending a night discovering the same limitation.

# Third-party components and research

Egret2MAME was developed specifically for this project, but it does
**not** exist in isolation.

Development benefited from or currently uses:

-   **USBPcap** -- raw USB capture during development/current prototype.
-   **ViGEmBus / Nefarius.ViGEm.Client** -- virtual Xbox 360 controller
    for the current button bridge.
-   **Microsoft Windows APIs / KMDF / VHF** -- input handling and
    experimental driver work.
-   **Community HID research** -- useful early clues about unusual
    controller usages, subsequently tested against the physical
    controller and raw reports.

Before public distribution, all third-party licenses, notices,
redistribution requirements and credits must be reviewed. Controller
artwork used during development must also be reviewed; a neutral
original illustration may replace it if necessary.

# Project philosophy

Egret2MAME is intended to remain free.

The goal is not to replace MAME's input system. The goal is to make this
unusual but excellent original controller behave on a Windows MAME
system as naturally as its owner expected when buying it.

This project started because I bought a controller and waited for
somebody to make it work.

Eventually the answer seemed to be:

> **It can't be done.**

So we started investigating.

A few HID reports, USB captures, failed experiments and many test builds
later, the answer became:

> **Apparently it can.**

And perhaps the finished project can save the next EGRET owner from
having to start at byte zero.

---

## September 23, 2026 — Replacing USBPcap with our own USB lower filter

The next major development goal was to remove USBPcap from the runtime path. Direct HID access had already failed because Windows owns the EGRET's mouse collection exclusively, and Raw Input exposed the trackball but not the spinner and unusual button usages. The important question therefore became: **where can we observe the complete five-byte interrupt report without breaking the normal Windows HID stack?**

### The wrong layer: HID collection upper filter

The first KMDF experiments attached a filter around the HID collection. The drivers could be built and test-signed, but installing them caused the normal EGRET trackball to stop working. Even a deliberately minimal pass-through version reproduced the problem.

That was useful evidence: the project was filtering at the wrong layer.

Rather than continuing with trial-and-error builds, the actual Windows device stack was inspected. The physical USB device reported `HidUsb` as its service. Before the new filter work, the relevant stack showed USBPcap below `HidUsb`. Since USBPcap was already known to see all five raw bytes, this established the correct architectural direction: a **device-specific USB lower filter on the physical `USB\VID_0AE4&PID_0701` device**.

### v0.10 — transparent USB lower-filter checkpoint

EgretFilter v0.10 was intentionally minimal. It attached as a lower filter to the physical EGRET USB device and did not intercept or modify USB requests.

The resulting stack was:

```text
HidUsb
EgretFilter
USBHUB3
```

The normal Windows trackball and spinner continued to work. This was the first proof that EgretFilter could live at the correct layer without disrupting the controller.

### v0.11 — URB completion checkpoint

v0.11 added handling for `IOCTL_INTERNAL_USB_SUBMIT_URB` and installed a completion routine, while deliberately avoiding payload inspection or modification.

After reboot the driver was active as `oem96.inf`. The stack showed:

```text
HidUsb
EgretFilter
USBPcap
USBHUB3
```

Trackball and spinner still worked normally. This proved that the filter could observe completion of USB requests while preserving the standard HID path.

### v0.12 — complete five-byte report tap

v0.12 added the part the project had been working toward: inspection of successfully completed bulk/interrupt IN URBs. On matching five-byte transfers, EgretFilter copies the returned bytes into its own protected buffer. It never modifies the original USB transfer buffer.

The copied report retains the already established format:

```text
[Buttons Low] [Buttons High] [Trackball X] [Trackball Y] [Spinner]
```

A small read-only control interface (`\\.\EgretFilter`) and console viewer were added for verification.

The test build was signed with the project's development test certificate and installed as `oem97.inf`. Windows reported the driver started successfully. The active stack then showed:

```text
HidUsb
EgretFilter
USBHUB3
```

Notably, USBPcap was no longer present in that device stack.

The normal Windows trackball and spinner still worked. More importantly, the EgretFilter viewer produced reports for **every physical button, trackball movement and spinner movement**. This independently confirmed that the new filter sees the complete EGRET input stream.

This was the decisive USBPcap-replacement milestone.

### v0.32 — Egret2MAME switches to EgretFilter

With the v0.12 report path proven, the known-good v0.31f GUI was used as the base for Egret2MAME v0.32. Only the input backend was intentionally replaced.

The old runtime path:

```text
EGRET -> USBPcap -> Egret2MAME -> ViGEm / mouse injection -> MAME
```

became:

```text
EGRET -> EgretFilter v0.12 -> Egret2MAME v0.32 -> ViGEm / mouse injection -> MAME
```

The following known-good behavior from v0.31f was preserved:

- five-button X360 mapping through ViGEm
- trackball multiplier, default **3x**
- spinner scaling, default **0.50x**
- controller GUI and live button display
- trackball live scope
- button-alignment controls

The USBPcap capture process and USBPcap root/device auto-detection were removed from the application backend. v0.32 reads the latest five-byte report directly from `\\.\EgretFilter`.

### MAME 0.289 verification

v0.32 was then tested with the project's reference emulator, **official standalone MAME 0.289 for Windows**.

Result:

- controller detected through EgretFilter: **PASS**
- all five buttons: **PASS**
- trackball: **PASS**
- 3x trackball behavior: **PASS**
- spinner: **PASS**
- 0.50x spinner behavior: **PASS**
- operation in MAME 0.289: **PASS**

The GUI reported:

```text
CONTROLLER CONNECTED
EgretFilter v0.12
Direct 5-byte report backend
```

At this point **Egret2MAME no longer requires USBPcap to read the controller**.

USBPcap and Wireshark remain installed on the development machine temporarily as comparison/recovery tools, but they are no longer part of the v0.32 runtime input path.

### Current known-good checkpoint

The current development reference is:

- **EgretFilter v0.12** — complete read-only five-byte USB report tap
- **Egret2MAME v0.32** — EgretFilter backend
- **MAME 0.289 standalone Windows** — verified reference emulator
- Trackball default: **3x**
- Spinner default: **0.50x**
- ViGEm still used for the virtual X360 button output

v0.32 is intentionally being treated as a frozen known-good checkpoint before the next dependency is changed.

### Development signing and Secure Boot

The current EgretFilter build is a development driver signed with a local test certificate. Driver development was performed with Secure Boot disabled and Windows test signing enabled.

After the successful v0.32 test, the development machine was returned toward normal operation:

```text
testsigning No
```

Secure Boot can then be restored in UEFI using the normal Windows UEFI mode. The development certificate and test-signed driver are **not** the intended public distribution mechanism.

A public release must use a production-compatible Windows driver-signing path so that users can run with normal Secure Boot enabled and without Windows test mode.

### What remains before 1.0

The largest remaining external runtime dependency is **ViGEm**, currently used to present the five EGRET buttons to MAME as a virtual Xbox 360 controller. The next development phase will investigate replacing that dependency while preserving the known-good v0.32 behavior.

The intended 1.0 experience remains:

```text
Fresh Windows 11
        |
Original TAITO EGRET II mini Paddle & Trackball Controller
        |
Install / start Egret2MAME
        |
Start official MAME
        |
Works
```

No separate Wireshark installation. No separate USBPcap installation. No Windows test mode. No requirement to disable Secure Boot.

The project has therefore moved from proving that the unusual controller can be decoded to proving that its complete data path can be handled by an Egret2MAME-specific Windows driver.

The original challenge was:

> "It can't be done."

The current development build answers:

> "Apparently it can — without USBPcap, too."

