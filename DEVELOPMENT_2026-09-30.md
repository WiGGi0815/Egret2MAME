# Egret2MAME -- Development History

> **CURRENT DEVELOPMENT CHECKPOINT --- 30 September 2026, 23:13 CEST**
>
> **Version line:** v0.4711 Beta 1 DEV\
> **Status:** One-click installation is working on the Windows 11
> notebook and the complete controller path has been function-tested.\
> **Known remaining blocker:** uninstall cleanup is not yet fully clean.
> Do **not** modify the proven installation/driver path while fixing
> uninstall.
>
> A ZIP backup of the current working installer/source state has been
> preserved as the reference checkpoint.

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

## 23. Clean-PC goal

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

# Current development architecture --- 30 September 2026

The USBPcap/ViGEm architecture described earlier in this document is
historical development work. The current v0.4711 Beta 1 DEV path uses
the project's own physical and virtual driver components.

``` text
TAITO EGRET II mini Paddle & Trackball Controller
                    |
                    | USB  VID_0AE4 / PID_0701
                    v
             Windows USB/HID stack
                    |
                    v
              EgretFilter
        (physical kernel driver)
                    |
                    v
              Egret2MAME
                    |
          +---------+---------+
          |                   |
          v                   v
 Trackball / Spinner      Five Buttons
          |                   |
          |                   v
          |          EgretVirtualButtons
          |          + Microsoft VHF
          |                   |
          +---------+---------+
                    |
                    v
                   MAME
```

### Current installation components

-   `EgretFilter` --- physical EGRET driver/filter package.
-   `EgretPhysicalInstall.exe` **v2** --- native physical-device
    installer using `INSTALLFLAG_FORCE`.
-   `EgretPhysicalRollback.exe` --- guarded rollback to Microsoft's HID
    driver.
-   `EgretVirtualButtons` --- virtual five-button HID kernel driver.
-   `EgretVirtualDevice.exe` --- creates/binds the virtual ROOT device.
-   `03_Install_Drivers.ps1` --- validates packages, invokes the
    physical installer, creates the virtual device and verifies the
    resulting state.
-   Inno Setup installer --- packages the application, drivers and
    helper programs into the current Beta 1 DEV installer.

The application packaged in Beta 1 remains the proven
`Egret2MAME_v035.exe`; the v0.4711 designation describes the current
installer/driver-development package.

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

------------------------------------------------------------------------

## 24 September 2026 --- v0.35 DEV installer: uninstall validation and test-mode-off check

### Tested environment and scope

This section records observations on the development Windows 11 PC,
**not** a successful test on a fresh third-party PC. The final candidate
is still a development/test-signed build, not a portable consumer
release. The `README_FINAL_CANDIDATE.txt` included in the original
source snapshot predates the completed tests and describes the uninstall
as untested; the observations below supersede that historical status
statement.

### Uninstall guard and rollback

The installed Inno Setup uninstaller invokes `09_Uninstall_Guard.ps1`.
Its guard calls the native `EgretPhysicalRollback.exe --uninstall-all`,
verifies physical EGRET instances have returned to Microsoft's
`input.inf`, removes the virtual device, and blocks uninstallation if
these checks fail. It deliberately does not delete OEM driver packages
from the DriverStore.

During testing, the first guard attempt could not locate `pnputil.exe`
from 32-bit PowerShell. Using `System32` alone was also insufficient
because of WOW64 redirection. The guard was updated to resolve
`Sysnative\pnputil.exe` when running as a 32-bit process on 64-bit
Windows. After rebuilding and reinstalling the candidate, the
uninstaller displayed its successful-removal message.

A separate **read-only** post-uninstall check reported:

``` text
PHYSICAL USB\VID_0AE4&PID_0701\C&7596C39&0&1 | INF=input.inf | provider=Microsoft | present=yes
PHYSICAL USB\VID_0AE4&PID_0701\C&7596C39&0&2 | INF=input.inf | provider=Microsoft | present=no
PREVIEW ONLY: 2 EGRET instances. No changes made.
```

The command querying `Get-PnpDevice` for `ROOT\HIDCLASS\*` returned no
entries. Thus, on this development machine, the physical devices were
back on the Microsoft INF and the queried virtual ROOT/HIDCLASS device
was absent after uninstall. The OEM packages may still remain staged in
DriverStore by design.

### Reinstall attempt with Windows test signing disabled

After disabling Windows test signing and uninstalling the prior
installation, the developer started the same v0.35 DEV setup again. The
application window opened, but the controller path did **not** work. The
screenshot showed:

-   `Waiting for EgretFilter...` and `Driver: \\.\EgretFilter`;
-   LIVE panel: `Virtual Controller Active`,
    `Input Driver Microsoft VHF`, `Input Backend EgretFilter`,
    `Reports: 0`;
-   error dialog: `Microsoft VHF controller could not be started.` /
    `Das System kann die angegebene Datei nicht finden`.

The `Active` status is not proof of a functioning virtual controller: it
contradicts the explicit start error and zero reports. The screenshot
alone does not establish which individual driver-installation step
failed or the exact underlying cause. Test-signed kernel drivers are not
a supported normal-mode distribution path. The application opening must
not be described as successful end-to-end installation.

**Result:** v0.35 installer/uninstaller was validated on the configured
development PC with test signing; a subsequent test with test signing
off showed that the application opens but the driver/controller chain
fails. No successful fresh-PC or Secure-Boot-on test has been recorded.

### Remaining work before a public release

1.  Replace the first-install WDK/DevCon dependency with a supported,
    self-contained device-installation path.
2.  Arrange appropriate production driver signing and validate loading
    under normal Windows security settings, including Secure Boot.
3.  Test installation, operation, failure handling, and uninstall on a
    clean Windows 11 PC; capture driver-install logs and device status.
4.  Fix the GUI's misleading `Virtual Controller Active` indicator when
    VHF startup fails; report distinct states for installed, started,
    connected, and receiving reports.
5.  Preserve the validated v0.35 DEV candidate and signed driver backups
    unchanged while developing the portable release.

**Current status:** validated **DEV** candidate on the development
machine; **not** a general-purpose release installer.

------------------------------------------------------------------------

# 30 September 2026 --- v0.4711 Beta 1 DEV: working one-click install on notebook

## Major result

The current Beta 1 DEV installer was successfully tested on the Windows
11 notebook.

After installation:

``` text
EgretFilter          STATE: RUNNING
EgretVirtualButtons  STATE: RUNNING
```

Egret2MAME was then started normally and the complete controller path
worked:

``` text
Trackball  PASS
Spinner    PASS
Buttons    PASS
```

This is the first recorded notebook test in which the current installer
performs the driver setup and the controller works end-to-end without
manually completing the driver installation afterward.

## Physical-driver installation problem and solution

Windows initially continued selecting Microsoft's `input.inf` for the
physical controller even though the EgretFilter package was present in
DriverStore.

Observed state before the fix:

``` text
Device:       USB\VID_0AE4&PID_0701
Driver INF:   input.inf
Service:      HidUsb
LowerFilters: empty
```

`pnputil /add-driver ... /install` staged the EgretFilter package but
did not replace the higher-ranked Microsoft HID driver. Earlier
SetupAPI-based installation attempts also failed with:

``` text
E0000217
```

The working solution is the native helper:

``` text
EgretPhysicalInstall.exe v2
```

It installs the physical driver with `INSTALLFLAG_FORCE`. Manual
notebook testing produced:

``` text
Egret2MAME physical driver installer v2
Hardware ID: USB\VID_0AE4&PID_0701
Installing EgretFilter with INSTALLFLAG_FORCE...
SUCCESS: EgretFilter installed.
REBOOT_REQUIRED: NO
```

After this, `EgretFilter` was running successfully.

**Important:** This helper is now part of the known-good installation
path. Do not replace or redesign it casually.

## Virtual device

`EgretVirtualDevice.exe` successfully creates and binds the virtual
device:

``` text
ROOT\EGRET2MAME_VBUTTONS
```

The resulting virtual HID device appears as:

``` text
ROOT\HIDCLASS\0000
Egret2MAME Virtual Five-Button HID (prototype)
```

and the kernel service reaches:

``` text
EgretVirtualButtons  STATE: RUNNING
```

The buttons function correctly through the resulting virtual-device
path.

## Driver package validation

`EgretFilter.inf` was checked with the Windows Driver Kit
`infverif.exe`:

``` text
ExitCode: 0
```

`Inf2Cat.exe` completed with:

``` text
Signability test complete.
Errors:   None
Warnings: None
Catalog generation complete.
```

The resulting `EgretFilter.cat` was signed with the development
certificate:

``` text
Subject:    CN=Egret2MAME Development Test
Thumbprint: 8D25A08F676769BB11728952CEEFB4DF0D239A82
Status:     Valid
```

This remains a development/test-signed configuration. It is not
documentation of a production-signed public driver.

## Installer-script failure discovered during Beta 1 test

An early Beta 1 build appeared to install without an Inno Setup error
but actually left:

``` text
Physical device: input.inf / HidUsb
Virtual device:  absent
EgretVirtualButtons: STOPPED
```

The cause was not the drivers. `03_Install_Drivers.ps1` had accidentally
been packaged with PowerShell here-string wrapper text:

``` text
@'
...
'@ | Set-Content ".\03_Install_Drivers.ps1" -Encoding UTF8
```

PowerShell therefore treated the script body as string data, returned
exit code `0`, and performed no installation.

The wrapper lines were removed and the script was reparsed:

``` text
========== SYNTAX OK ==========
```

The installer was rebuilt. The corrected build then installed
successfully on the notebook and the controller worked completely.

This failure is important historical information: a successful setup UI
and process exit are not sufficient proof that the driver chain was
installed. Device/service state must be verified.

## Current known-good installation path

The validated flow is:

``` text
Inno Setup
   |
   v
03_Install_Drivers.ps1
   |
   +--> validate EgretFilter INF/SYS/CAT
   |
   +--> validate EgretVirtualButtons INF/SYS/CAT
   |
   +--> stage EgretFilter package
   |
   +--> EgretPhysicalInstall.exe v2
   |       INSTALLFLAG_FORCE
   |
   +--> verify physical driver/service
   |
   +--> stage EgretVirtualButtons
   |
   +--> EgretVirtualDevice.exe
   |
   +--> verify virtual HID device/service
   |
   v
Egret2MAME ready
```

The working installation behavior should now be treated as frozen while
uninstall cleanup is repaired.

## Uninstall test --- partially successful

Windows **Settings -\> Apps -\> Installed apps -\> Egret2MAME -\>
Uninstall** was tested on the notebook.

The safety-critical rollback worked.

After uninstall, the physical controller reported:

``` text
DEVPKEY_Device_DriverInfPath  input.inf
DEVPKEY_Device_Service        HidUsb
DEVPKEY_Device_LowerFilters   <empty>
```

The virtual `ROOT\HIDCLASS\0000` Egret2MAME device was no longer
present.

Therefore:

``` text
Physical rollback to Microsoft HID     PASS
Virtual device removal                 PASS
```

## Remaining uninstall cleanup defect

The uninstall is **not yet clean enough for release**.

Immediately after uninstall:

``` text
EgretFilter          RUNNING
EgretVirtualButtons  STOPPED
```

The application directory remained because these files were still
present:

``` text
C:\Program Files\Egret2MAME\Egret2MAME_v035.log
C:\Program Files\Egret2MAME\EgretFilter.sys
```

A reboot was performed to determine whether this was only a transient
loaded-driver condition.

After reboot the state was still:

``` text
EgretFilter          RUNNING
EgretVirtualButtons  STOPPED
```

and both files were still present.

Therefore this is a real uninstall-cleanup defect, not merely a
pending-reboot artifact.

## NEXT TASK --- do this first

Do **not** reopen the physical-installation problem. Installation is
currently proven working.

The next development task is exclusively to make uninstall cleanup
complete while preserving the already validated rollback behavior.

Required final uninstall state:

``` text
Physical EGRET:
  DriverInfPath = input.inf
  Service       = HidUsb
  LowerFilters  = empty

Virtual Egret2MAME device:
  absent

EgretFilter service:
  absent

EgretVirtualButtons service:
  absent

C:\Program Files\Egret2MAME:
  absent

EgretFilter.sys:
  absent

Egret2MAME_v035.log:
  absent
```

The uninstall logic should remove the two Egret driver services only
**after** the physical device has safely returned to Microsoft's driver
and the virtual device has been removed. If a loaded driver file cannot
be removed immediately, use a controlled delete-on-reboot mechanism
rather than weakening the rollback guard.

## Regression test required after uninstall fix

After changing the uninstall path, perform the complete cycle again on
the notebook:

``` text
1. Install Beta 1 DEV normally from the installer.
2. Confirm EgretFilter RUNNING.
3. Confirm EgretVirtualButtons RUNNING.
4. Start Egret2MAME.
5. Test trackball.
6. Test spinner.
7. Test all buttons.
8. Uninstall from Windows Installed Apps.
9. Verify physical device = input.inf / HidUsb / no LowerFilters.
10. Verify virtual device absent.
11. Verify both Egret services absent.
12. Verify program directory absent.
13. Reboot.
14. Repeat the uninstall-state verification.
```

Only after this complete cycle passes should the Beta 1 checkpoint be
considered fully clean.

## Preserved checkpoint

A ZIP archive of the current working installer/source state has been
supplied and should be treated as the rollback/reference snapshot for
this checkpoint.

**Do not overwrite or casually modify the preserved known-good
snapshot.**

## Current status at end of 30 September 2026

``` text
Physical driver installation      PASS
Virtual driver installation       PASS
One-click notebook installation   PASS
Egret2MAME startup                PASS
Trackball                         PASS
Spinner                           PASS
Five buttons                      PASS
Physical uninstall rollback       PASS
Virtual-device removal            PASS
Driver-service cleanup            FAIL / TODO
Residual-file cleanup             FAIL / TODO
Fresh third-PC test               TODO
Production driver signing         TODO
```

The key milestone for the day is real: **Egret2MAME v0.4711 Beta 1 DEV
installs and works end-to-end on the notebook.**

The next session starts with uninstall cleanup --- not with driver
installation.
