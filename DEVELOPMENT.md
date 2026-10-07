# Egret2MAME --- DEVELOPMENT.md

> **CURRENT DEVELOPMENT CHECKPOINT --- 7 October 2026**
>
> **Version:** Beta 0.815\
> **Status:** TECHNICALLY COMPLETE / RELEASE CANDIDATE\
> **Target:** Windows 11 + official standalone MAME 0.289\
> **Controller:** TAITO EGRET II mini Paddle & Trackball Controller ---
> VID `0AE4`, PID `0701`
>
> Beta 0.815 replaces the previous custom test-signed kernel-driver
> architecture with a substantially simpler release path based on the
> official signed USBPcap driver plus a user-mode Egret2MAME
> application. The final candidate has been installed and uninstalled
> successfully on a second Windows 11 PC with normal Windows security
> settings, including Secure Boot enabled and Test Signing disabled.
>
> **Freeze rule:** The working Beta 0.815 program and installer should
> not be changed for the public release unless a concrete defect is
> found. The application/installer is functionally frozen. Documentation
> has now been expanded with freshly verified MAME 0.289 spinner and
> trackball setup examples. > Release packaging now includes the corresponding USBPcap 1.5.4.0
> upstream source archive and redistribution notice under `licenses/`.
> The final compact GUI has also passed UHD / 250% Windows scaling.
> Remaining administrative release work is the final release hash/package
> freeze and any final license-text review before public visibility.

------------------------------------------------------------------------

## 1. Scope of Beta 0.815

This document deliberately describes **only the development path that
led to Beta 0.815**.

The older v0.4711 Beta 1 DEV branch remains preserved as a known-good
historical fallback, but its custom `EgretFilter` /
`EgretVirtualButtons` driver architecture is **not part of Beta 0.815**.

The objective of this development cycle was:

1.  keep the already proven EGRET controller functionality;
2.  remove the requirement for our own test-signed kernel drivers;
3.  run on a normal Windows 11 installation with **Secure Boot ON**;
4.  run with **Test Signing OFF**;
5.  require no manual driver-development preparation from the user;
6.  retain trackball, spinner and all five physical buttons;
7.  provide one normal Egret2MAME installer;
8.  install the required signed USB capture component automatically;
9.  uninstall cleanly again.

The result is Beta 0.815.

------------------------------------------------------------------------

## 2. Final Beta 0.815 architecture

``` text
TAITO EGRET II mini Paddle & Trackball Controller
                    |
                    | USB
                    | VID 0AE4 / PID 0701
                    v
             Windows USB/HID stack
                    |
          +---------+---------+
          |                   |
          |                   v
          |               USBPcap
          |        (official signed driver)
          |                   |
          |                   v
          |              Egret2MAME
          |          direct capture/parser
          |                   |
          |          +--------+--------+
          |          |                 |
          v          v                 v
     native mouse  extra mouse      keyboard
       movement    correction       SendInput
          |          |                 |
          +----------+--------+--------+
                            |
                            v
                           MAME
```

The physical controller remains attached to the normal Microsoft
HID/mouse stack. Beta 0.815 does **not** replace the physical device
with a custom Egret driver.

USBPcap is used to observe the raw USB interrupt traffic. Egret2MAME
identifies the correct USBPcap root, determines the controller's current
USB device address, decodes the controller report and uses the decoded
values for the additional processing required by MAME.

The controller's native Windows mouse path remains useful for the
physical trackball/spinner behavior. Egret2MAME adds the required
scaling/correction and translates the five unusual buttons into normal
keyboard input.

------------------------------------------------------------------------

## 3. Why the previous driver path was replaced

The starting point for this cycle was the completed v0.4711 Beta 1 DEV
build.

That version proved that the complete controller could be made to work,
but it depended on our own kernel components and development/test
signing. For an experimental development system this was acceptable; for
a public Windows release it was not the desired end state.

The release requirement became:

``` text
Normal Windows 11
Secure Boot ON
Test Signing OFF
No manually installed Egret development certificate
No custom test-signed Egret kernel driver
One Egret2MAME Setup
```

Instead of trying to production-sign and distribute the experimental
Egret kernel drivers, the new cycle returned to USBPcap and investigated
whether the entire required input path could be handled in user mode.

That became the central change of Beta 0.815.

------------------------------------------------------------------------

## 4. Raw Input investigation

Before committing to USBPcap as a runtime dependency, Windows-native
input paths were checked again.

Raw Input successfully exposed:

-   trackball movement;
-   spinner movement as mouse wheel input.

However, the five unusual EGRET buttons were not available in the
required form. No useful separate `RIM_TYPEHID` input path was found
that solved the button problem.

Result:

``` text
Raw Input — Trackball              PASS
Raw Input — Spinner                PASS
Raw Input — Five EGRET buttons     FAIL
Raw Input as complete solution     REJECTED
```

Raw Input therefore could not replace the complete controller reader.

------------------------------------------------------------------------

## 5. Direct HID investigation

A direct user-mode HID reader was also tested again.

Windows exposes the EGRET controller primarily as a mouse HID
collection. The relevant collection is owned by the Windows input stack.

Attempting to open it directly resulted in:

``` text
Win32 Error 5
Access Denied
```

The reported HID input size was six bytes, but direct application access
to the live mouse collection was not available.

Result:

``` text
Direct HID open/read               FAIL
Reason                             Windows owns mouse collection
```

This confirmed that a normal direct HID reader was not a viable release
solution.

------------------------------------------------------------------------

## 6. USBPcap direct-access breakthrough

USBPcap had previously been useful as a development/reverse-engineering
tool. The important question for Beta 0.815 was whether Egret2MAME
itself could communicate with USBPcap directly, without requiring
Wireshark or `USBPcapCMD` as a runtime application.

The answer was yes.

After installing USBPcap and rebooting, the USBPcap kernel service was
active and a native test program successfully opened USBPcap control
devices directly:

``` text
\\.\USBPcap1
\\.\USBPcap2
...
```

This established the new release direction:

``` text
Egret2MAME
    |
    +--> direct CreateFile access to USBPcap
    |
    +--> receive USB packet stream
    |
    +--> locate EGRET traffic
    |
    +--> decode reports
```

No Wireshark runtime dependency is required.

No USBPcap command-line capture program is required.

------------------------------------------------------------------------

## 7. Automatic USBPcap root selection

Hard-coding a value such as `\\.\USBPcap5` would only work on the
development PC and was therefore unacceptable.

Beta 0.815 development added automatic root matching.

The EGRET device is first located through Windows Plug and Play using:

``` text
VID = 0AE4
PID = 0701
```

The PnP parent chain is then walked upward until the USB root hub is
found.

Example observed root:

``` text
USB\ROOT_HUB30\9&2902E077&0&0
```

Each USBPcap control device can be queried with the USBPcap hub-symlink
IOCTL:

``` cpp
CTL_CODE(FILE_DEVICE_UNKNOWN, 0x803, METHOD_BUFFERED, FILE_ANY_ACCESS)
```

After normalizing the returned root-hub names, Egret2MAME can match the
physical root containing the EGRET controller to the corresponding
`\\.\USBPcapN` capture device.

Result:

``` text
Find EGRET by VID/PID               PASS
Walk PnP parent chain               PASS
Find physical USB root              PASS
Map root to USBPcapN                PASS
Hard-coded USBPcap number needed    NO
```

This was later validated by moving the controller between multiple USB
ports on the clean test PC.

------------------------------------------------------------------------

## 8. USBPcap packet framing

The direct capture reader was built around the actual USBPcap stream
format.

The capture begins with a 24-byte PCAP global header. Each captured
packet then contains a 16-byte PCAP record header followed by the
USBPcap packet header.

The relevant fields used by the EGRET reader were identified as:

``` text
USBPcap device address     offset 19
Endpoint                   offset 21
Transfer type              offset 22
Data length                offset 23
Payload                    header length (normally 27)
```

The EGRET input traffic of interest is interrupt-IN traffic on:

``` text
Endpoint       0x81
Transfer       1
Data length    5
```

This allowed Egret2MAME to isolate candidate controller reports directly
from the USB capture stream.

------------------------------------------------------------------------

## 9. Actual EGRET report format

The development work confirmed that the useful controller payload is
five bytes:

``` text
Byte 0   Buttons Low
Byte 1   Buttons High
Byte 2   Trackball X
Byte 3   Trackball Y
Byte 4   Spinner
```

Trackball X, trackball Y and spinner are signed 8-bit relative values.

The controller reports at approximately:

``` text
8 ms interval
~125 Hz
```

This five-byte raw payload is the basis of the Beta 0.815 input decoder.

------------------------------------------------------------------------

## 10. Five-button decoding

All five physical buttons were mapped from the raw report.

``` text
Physical control   HID usage   Raw report

Pink SELECT        0x17        Byte 0 = 0x02
Blue START         0x18        Byte 0 = 0x04
White MENU         0x1A        Byte 0 = 0x10
FIRE LEFT          0x1D        Byte 0 = 0x80
FIRE RIGHT         0x1E        Byte 1 = 0x01
```

The two fire buttons are independent.

This removed the need for the old virtual HID button driver.

Result:

``` text
SELECT decode       PASS
START decode        PASS
MENU decode         PASS
FIRE LEFT decode    PASS
FIRE RIGHT decode   PASS
```

------------------------------------------------------------------------

## 11. Automatic USB device-address detection

The USB device address assigned by Windows cannot safely be hard-coded
because it can change between systems, ports or connection states.

The direct-input test core therefore added automatic device-address
discovery.

After the correct USB root has been selected, Egret2MAME observes
candidate interrupt reports matching the known EGRET characteristics:

``` text
Endpoint       0x81
Transfer       interrupt
Data length    5
Known report/button structure
```

Candidate device addresses are counted during a short detection window.
The most frequent valid candidate is selected, with a minimum-report
threshold used to reject insufficient matches.

The successful development implementation used approximately a
1.5-second observation period and required at least five candidate
reports.

This is intentionally constrained to the already uniquely matched USB
root. It is not a general-purpose USB topology enumerator; it is a
targeted EGRET identification method.

Result:

``` text
Automatic USB device address       PASS
Manual device-address setting      NOT REQUIRED
Reconnect / different USB ports    PASS in final clean-PC test
```

------------------------------------------------------------------------

## 12. Direct input core milestone

The standalone direct USBPcap test reached a major milestone with the
v0.10 test core.

It automatically performed:

1.  EGRET VID/PID discovery;
2.  PnP root-hub discovery;
3.  USBPcap root matching;
4.  direct USBPcap capture;
5.  USB device-address detection;
6.  endpoint/report filtering;
7.  trackball decoding;
8.  spinner decoding;
9.  five-button decoding;
10. idle-report suppression;
11. clean capture shutdown.

All physical controls were successfully observed.

``` text
USBPcap direct reader       PASS
Trackball raw data          PASS
Spinner raw data            PASS
Five button raw data        PASS
Automatic root selection    PASS
Automatic address select    PASS
```

This became the input foundation for Beta 0.815.

------------------------------------------------------------------------

## 13. Button output to MAME

Decoding the buttons was only half of the problem. MAME also had to see
five distinct, useful controls.

The intended standard keyboard mapping for Beta 0.815 became:

``` text
SELECT       -> 5
START        -> 1
MENU         -> SPACE
FIRE LEFT    -> LCTRL
FIRE RIGHT   -> LALT
```

### 13.1 Scan-code-only SendInput

A scan-code-only `SendInput` experiment did not produce the required
result.

``` text
STATUS: FAIL
```

### 13.2 Virtual-key-only SendInput

Sending normal virtual keys worked in ordinary Windows applications.

However, in MAME the inputs appeared as:

``` text
Scan000
```

The important discovery was that MAME could see the injected events, but
all five appeared with the same unusable scan identity.

``` text
Windows/editor test       PASS
MAME distinct buttons     FAIL
MAME result               Scan000 for all keys
```

### 13.3 Final SendInput solution

The successful v0.13 output test used both fields:

``` text
wVk   = desired virtual key
wScan = MapVirtualKeyW(wVk, MAPVK_VK_TO_VSC)
```

Crucially, `KEYEVENTF_SCANCODE` is **not** set.

Key release uses `KEYEVENTF_KEYUP`.

This preserved the working virtual-key injection path while also
providing a meaningful scan value that MAME/DInput could distinguish.

Result:

``` text
SELECT -> 5              PASS
START -> 1               PASS
MENU -> SPACE            PASS
FIRE LEFT -> LCTRL       PASS
FIRE RIGHT -> LALT       PASS
Distinct MAME inputs     PASS
```

This breakthrough removed the need for `EgretVirtualButtons`, Microsoft
VHF or a virtual Xbox controller in Beta 0.815.

------------------------------------------------------------------------

## 14. MAME input-provider conclusion

During testing, MAME input configuration temporarily produced
joystick-like/recentering behavior that initially looked like a
trackball problem.

The behavior remained even when the Egret test tool was not running,
proving that the problem was in the MAME configuration rather than the
new capture core.

Resetting the affected MAME configuration restored normal
mouse/trackball behavior.

For the Beta 0.815 architecture, the tested useful MAME path is:

``` text
DInput or Win32
```

RawInput is not the intended provider for the mouse-multiplication
method used here.

The public documentation should therefore include a separate MAME
configuration guide rather than attempting to force one universal MAME
configuration from Egret2MAME itself.

------------------------------------------------------------------------

## 15. Trackball output and speed multiplication

The physical EGRET trackball remains visible to Windows as native
relative mouse movement.

Egret2MAME therefore does not need to replace the base mouse movement.
Instead, higher speed settings add extra relative movement based on the
raw EGRET delta:

``` text
extra X = (speed - 1) * raw X
extra Y = (speed - 1) * raw Y
```

The native physical movement plus the injected extra movement produces
the effective multiplier.

Beta 0.815 provides trackball speed settings:

``` text
1x
2x
3x
4x
5x
6x
```

Default:

``` text
3x
```

Result:

``` text
Native trackball movement       PASS
Trackball multiplication        PASS
MAME operation                  PASS
```

Game-specific maximum movement speed remains controlled by the game
itself; reaching Centipede's maximum shooter speed is not an Egret2MAME
defect.

------------------------------------------------------------------------

## 16. Spinner output and scaling

The physical spinner is exposed through the Windows mouse-wheel path.

Beta 0.815 retains the established correction method for spinner speeds
below the native 1.0 rate.

Available settings:

``` text
0.25x
0.50x
0.75x
1.00x
```

Default:

``` text
0.50x
```

The application maintains an accumulator and injects compensating
opposite wheel movement where required so that the effective output is
reduced relative to the physical 1.0x input.

Result:

``` text
Spinner input          PASS
Spinner scaling        PASS
MAME operation         PASS
```

------------------------------------------------------------------------

## 17. Beta 0.815 GUI

The working direct-input/output core was merged with the proven visual
design of the earlier Egret2MAME GUI.

The final Beta 0.815 interface retains:

-   controller illustration;
-   live controller activity display;
-   five aligned live button indicators;
-   trackball live display/scope;
-   trackball speed selector;
-   spinner speed selector;
-   MAME provider information;
-   Save Settings;
-   About;
-   fixed MAME button-mapping reference.

The final mapping shown in the GUI is:

``` text
MAME BUTTON MAPPING

SELECT      5
START       1
MENU        SPACE
FIRE (L)    LCTRL
FIRE (R)    LALT
```

During cleanup, development-only UI elements no longer needed by the
release candidate were removed, including the visible report counter and
the old button-alignment controls.

The MAME provider area was reduced to fit the final layout.

Result:

``` text
Controller graphic          PASS
Live controller display     PASS
Five live button lights     PASS
Trackball display           PASS
Speed controls              PASS
Mapping reference           PASS
Final layout                PASS
```

------------------------------------------------------------------------

## 18. Preventing the GUI from reacting to injected controller keys

An integration issue appeared after keyboard button output was added.

The physical MENU button maps to `SPACE`. If a normal GUI button such as
About held keyboard focus, the injected Space event could activate that
GUI button.

Likewise, old numeric GUI hotkeys were undesirable because SELECT and
START intentionally generate `5` and `1`.

The GUI was therefore adjusted so that its buttons do not retain normal
tab focus in a way that allows controller-generated keyboard events to
activate the application itself. Obsolete speed hotkey behavior was also
removed/neutralized for the final workflow.

Result:

``` text
MENU activates About accidentally     FIXED
Controller keys alter own GUI         FIXED
Five buttons in MAME                  PASS
```

------------------------------------------------------------------------

## 19. Settings persistence

Beta 0.815 stores the user's speed settings in:

``` text
%LOCALAPPDATA%\Egret2MAME\settings.ini
```

Stored values:

``` text
TrackballSpeed=<1..6>
SpinnerQuarter=<1..4>
```

No runtime log file is required for normal use.

Result:

``` text
Save Settings       PASS
Reload settings     PASS
```

------------------------------------------------------------------------

## 20. Administrator requirement

During integration testing, direct USBPcap access behaved differently
depending on process elevation.

Observed:

``` text
Started elevated / admin     USBPcap access PASS
Started normally             USBPcap access unavailable
```

The final application manifest therefore uses:

``` text
requireAdministrator
```

Windows displays the normal UAC elevation prompt when Egret2MAME starts.

This is considered acceptable for Beta 0.815 and is substantially
preferable to requiring Windows Test Mode or disabling Secure Boot.

------------------------------------------------------------------------

## 21. Components no longer required by Beta 0.815

The new architecture eliminates several components from the active
release path.

Beta 0.815 does **not** require:

``` text
EgretFilter.sys
EgretVirtualButtons.sys
EgretPhysicalInstall.exe
EgretPhysicalRollback.exe
EgretVirtualDevice.exe
Microsoft VHF virtual button device
ViGEm virtual Xbox output
Windows Test Signing mode
Secure Boot disabled
Egret development test certificate
Wireshark at runtime
USBPcapCMD at runtime
```

The previous v0.4711 Beta 1 DEV package remains preserved separately as
historical fallback and development evidence.

It must not be confused with the Beta 0.815 release architecture.

------------------------------------------------------------------------

## 22. USBPcap runtime dependency

Beta 0.815 does require USBPcap.

The installer currently packages:

``` text
USBPcap 1.5.4.0
```

The development build process verifies the installer with SHA-256:

``` text
87A7EDF9BBBCF07B5F4373D9A192A6770D2FF3ADD7AA1E276E82E38582CCB622
```

USBPcap is installed silently by the Egret2MAME setup when it is not
already present.

A reboot is required after the driver installation before the capture
path can be used reliably.

From the user's perspective the intended process is:

``` text
Run Egret2MAME Setup
        |
        v
USBPcap installed automatically if required
        |
        v
Reboot
        |
        v
Run Egret2MAME
```

The user does not need to separately download or configure USBPcap.

USBPcap licensing/redistribution notices and credits must be included in
the public release documentation.

------------------------------------------------------------------------

## 23. Installer and release packaging

The Beta 0.815 release installer is built with **Inno Setup 7**.

The public installer packages the x64 Egret2MAME application, the
required application artwork/resources and the official USBPcap 1.5.4.0
installer into a single setup package.

The build/release process:

1.  builds the x64 Egret2MAME application;
2.  embeds the administrator/UAC manifest;
3.  verifies the bundled USBPcap installer;
4.  creates the standalone Egret2MAME setup package.

Internal development-machine paths, temporary build locations and local
working filenames are deliberately not part of the public development
documentation because they are not relevant to installing, using or
understanding Egret2MAME.

The final installer creates the normal Egret2MAME installation and
shortcuts and installs USBPcap only when required.

------------------------------------------------------------------------

## 24. USBPcap ownership problem during uninstall

A release-quality uninstaller must not remove software that was already
present before Egret2MAME was installed.

The required rule is:

``` text
USBPcap already existed before Egret2MAME
    -> leave USBPcap installed

USBPcap was installed by Egret2MAME
    -> remove USBPcap with Egret2MAME
```

### 24.1 First ownership implementation --- file marker

The first installer attempted to record ownership using a marker file
inside the Egret2MAME application directory.

During testing, Egret2MAME itself uninstalled but USBPcap remained
installed.

Changing the order so that USBPcap was requested for removal before
Egret2MAME did not solve the problem.

### 24.2 Isolating the USBPcap uninstaller

The USBPcap silent uninstaller was then tested directly from an elevated
PowerShell:

``` powershell
& "C:\Program Files\USBPcap\Uninstall.exe" /S
```

USBPcap disappeared correctly from Windows Installed Apps.

Result:

``` text
USBPcap silent uninstall itself      PASS
Egret ownership/uninstall trigger    FAIL
```

This isolated the problem to Egret2MAME's ownership tracking rather than
USBPcap.

### 24.3 Final solution --- registry ownership

The application-directory marker was removed.

The installer now records ownership persistently in:

``` text
HKLM\SOFTWARE\Egret2MAME
```

with:

``` text
USBPcapOwned = 1
```

but **only when USBPcap was absent before Egret2MAME Setup began**.

On uninstall:

1.  the ownership value is read;
2.  if `USBPcapOwned=1`, the USBPcap silent uninstaller is executed
    first;
3.  Egret2MAME waits for that operation;
4.  the ownership value is removed;
5.  the normal Egret2MAME uninstall continues.

If USBPcap was present before installation, Egret2MAME does not claim
ownership and does not remove it.

Final result:

``` text
Install USBPcap automatically             PASS
Remember USBPcap ownership                PASS
Preserve pre-existing USBPcap             BY DESIGN
Remove Egret-installed USBPcap            PASS
Remove Egret2MAME                          PASS
Clean uninstall                            PASS
```

------------------------------------------------------------------------

## 25. Final clean-PC validation

After development-machine testing was complete, Beta 0.815 was tested on
another Windows PC as the release validation target.

Important conditions:

``` text
Windows 11
Secure Boot ON
Test Signing OFF
No requirement for custom Egret test drivers
```

The installer completed successfully.

After the required reboot, Egret2MAME ran correctly.

The controller was also tested on **multiple USB ports**. Automatic
discovery continued to work; no USBPcap device number or USB address had
to be configured manually.

The uninstaller was then tested and successfully removed the Egret2MAME
installation and the USBPcap installation owned by Egret2MAME.

This closes the principal release blocker that existed in the previous
development branch.

------------------------------------------------------------------------

## 26. MAME scope

The reference emulator for this project remains:

``` text
Official standalone MAME 0.289 for Windows
```

Beta 0.815 has been designed around the normal MAME Windows input path,
with DInput/Win32 being the intended provider path for the mouse
multiplication behavior.

Individual MAME games still require appropriate control assignment and
may have their own analog sensitivity/speed behavior.

A separate MAME setup guide should document the recommended clean
configuration and mappings.

Other MAME versions, frontends or RetroArch may work, but they are not
automatically part of the Beta 0.815 compatibility guarantee unless
separately tested.

------------------------------------------------------------------------

## 27. Final Beta 0.815 button mapping

``` text
EGRET CONTROL     MAME/KEYBOARD OUTPUT

Pink SELECT       5
Blue START        1
White MENU        SPACE
FIRE LEFT         Left Ctrl
FIRE RIGHT        Left Alt
```

Trackball and spinner remain relative pointing-device controls rather
than joystick axes.

------------------------------------------------------------------------

## 28. Final security/deployment result

The most important release objective of this development cycle was
achieved.

Beta 0.815 no longer asks the user to weaken normal Windows
driver-security settings for Egret2MAME's own code.

Final state:

``` text
Secure Boot                         SUPPORTED / NOT REQUIRED
Windows Test Signing               OFF
Custom Egret kernel driver         NOT USED
Custom Egret driver signing        NOT REQUIRED
Development certificate            NOT REQUIRED
Official signed USBPcap driver      USED
UAC elevation for Egret2MAME        REQUIRED
```

This is the architecture intended for public distribution.

------------------------------------------------------------------------

## 29. Development conclusions

The Beta 0.815 cycle resolved three separate problems that had
previously been coupled together:

### Input acquisition

Direct HID could not read the Windows-owned mouse collection, and Raw
Input did not expose the five buttons in the required form.

Direct USBPcap access solved raw input acquisition.

### Button output

Virtual HID/VHF and virtual-controller approaches were no longer
necessary after the successful `SendInput` combination of a normal
virtual key plus a real mapped scan value.

### Deployment

Removing the project's own kernel drivers eliminated the
production-signing obstacle. Bundling the already signed USBPcap
component provided a practical normal-Windows installation path.

The resulting release architecture is smaller and easier to deploy than
the previous development-driver branch.

------------------------------------------------------------------------

## 30. Known limitations / intentional design choices

Beta 0.815 intentionally has the following characteristics:

-   Egret2MAME requests administrator elevation because direct USBPcap
    access requires it in the tested environment.
-   USBPcap 1.5.4.0 is a runtime dependency.
-   A reboot is required after USBPcap installation.
-   The USB device-address detector is a targeted heuristic operating
    after the correct physical USB root has already been identified.
-   Trackball speed multiplication is intended for MAME DInput/Win32
    rather than RawInput.
-   MAME game configuration remains game/user specific.
-   Beta 0.815 targets the TAITO controller with VID `0AE4`, PID `0701`;
    it is not intended as a generic USB controller translator.

None of these items blocked the completed clean-PC validation.

------------------------------------------------------------------------

## 31. Third-party components, credits and acknowledgements

Egret2MAME is an independent community project. It is not developed,
endorsed, sponsored or supported by TAITO.

The project relies on, was built with, or benefited from the following
third-party software and platform technologies.

### USBPcap --- runtime component

**USBPcap** is the principal third-party runtime component used by Beta
0.815.

USBPcap is developed by **Tomasz Moń** and contributors and provides the
Windows USB packet-capture driver that makes the Beta 0.815 direct input
architecture possible.

Egret2MAME distributes the official USBPcap 1.5.4.0 installer rather
than a modified Egret-specific build.

The USBPcap project documents its licensing as:

``` text
USBPcapDriver   GPLv2
USBPcapCMD      BSD 2-Clause
```

Beta 0.815 depends on the USBPcap capture driver. Egret2MAME does not
require USBPcapCMD or Wireshark at runtime.

The public release package must retain/provide the applicable USBPcap
license and copyright notices in accordance with the USBPcap
distribution terms.

Project: https://github.com/desowin/usbpcap

### Inno Setup --- build/installer tool

The Egret2MAME Windows setup package is produced with **Inno Setup 7**,
the Windows installation builder by **Jordan Russell and Martijn Laan**.

Inno Setup is a build-time tool. End users do not need to install Inno
Setup in order to use Egret2MAME.

Project: https://jrsoftware.org/isinfo.php

### Microsoft Windows / .NET Framework / Windows Forms

Egret2MAME Beta 0.815 is a Windows desktop application built using
Microsoft Windows APIs and the Microsoft .NET Framework / Windows Forms
platform.

Windows 11 includes a supported .NET Framework 4.x runtime, so the
current Egret2MAME package does not need to present .NET Framework as a
separate bundled third-party application.

Microsoft, Windows, .NET and related names are trademarks of Microsoft
Corporation. Their mention describes the platform used by Egret2MAME and
does not imply Microsoft endorsement.

### Wireshark --- development/research tool only

**Wireshark** was used during development while investigating and
validating USB traffic.

Wireshark is **not** a Beta 0.815 runtime dependency and is not required
to install or use Egret2MAME.

### USBPcapCMD --- development/reference tool only

USBPcap's command-line capture utility was useful during the earlier
investigation of the controller and USBPcap capture format.

The finished Beta 0.815 application communicates with the USBPcap
capture devices directly and therefore does not require USBPcapCMD at
runtime.

### MAME --- target emulator, not bundled

Egret2MAME was developed and tested primarily for the official
standalone Windows build of **MAME 0.289**.

MAME is not included in the Egret2MAME package. Egret2MAME is an
independent compatibility utility and is not an official MAME project.

### TAITO / EGRET II mini

The hardware target is the **TAITO EGRET II mini Paddle & Trackball
Controller**.

TAITO and EGRET II mini are referenced only to identify compatible
hardware. Egret2MAME is an unofficial independent project and is not
affiliated with or endorsed by TAITO Corporation.

### Development assistance

Development, investigation, debugging, documentation and iterative code
work were carried out by the Egret2MAME project with assistance from
**OpenAI ChatGPT**.

The final human project/release credit wording can be adjusted before
publication.

### Runtime dependency summary

``` text
COMPONENT                    BETA 0.815 STATUS

USBPcap Driver               REQUIRED / bundled installer
USBPcapCMD                   NOT REQUIRED
Wireshark                    NOT REQUIRED
Inno Setup                   BUILD TOOL ONLY
.NET Framework / WinForms    WINDOWS PLATFORM
MAME                         TARGET APPLICATION / NOT BUNDLED
Egret custom kernel drivers  NOT USED
ViGEm                        NOT USED
Microsoft VHF                NOT USED
```

------------------------------------------------------------------------

## 32. Release freeze

The Beta 0.815 executable and installer have now passed the intended
technical validation.

From this checkpoint onward, the working program/installer should be
treated as frozen unless a reproducible defect is found.

Remaining pre-publication work is documentation/release work:

``` text
Release notes                         TODO
Credits                               TODO
USBPcap license/redistribution text   TODO (include with release)
Final DEVELOPMENT.md review           IN PROGRESS
MAME setup guide                      TODO
Final release hashes                  TODO
Archive/freeze final package          TODO
```

No functional rewrite is planned before publication.

------------------------------------------------------------------------

## Current status at end of 5 October 2026

``` text
USBPcap direct access                     PASS
Automatic EGRET VID/PID discovery         PASS
Automatic USB root discovery              PASS
Automatic USBPcap root mapping            PASS
Automatic USB device-address detection    PASS

Raw 5-byte EGRET report decode            PASS
Trackball decode                          PASS
Spinner decode                            PASS
SELECT decode                             PASS
START decode                              PASS
MENU decode                               PASS
FIRE LEFT decode                          PASS
FIRE RIGHT decode                         PASS

Trackball output/scaling                  PASS
Spinner output/scaling                    PASS
Five distinct keyboard outputs            PASS
MAME DInput/Win32 button distinction      PASS

SELECT -> 5                               PASS
START -> 1                                PASS
MENU -> SPACE                             PASS
FIRE LEFT -> LCTRL                        PASS
FIRE RIGHT -> LALT                        PASS

Controller GUI                            PASS
Live controller display                   PASS
Live button indicators                    PASS
Settings persistence                      PASS
GUI self-trigger prevention               PASS

Administrator/UAC startup                 PASS
Secure Boot ON operation                  PASS
Secure Boot required                      NO
Test Signing OFF operation                PASS
Custom Egret kernel drivers eliminated    PASS

One-click installer                       PASS
Automatic USBPcap installation            PASS
Required reboot flow                      PASS
USBPcap ownership tracking                PASS
Egret2MAME uninstall                      PASS
Owned USBPcap uninstall                   PASS

Second Windows 11 PC installation         PASS
Multiple USB-port test                    PASS
Second-PC runtime test                    PASS
Second-PC uninstall test                  PASS

Beta 0.815 technical release candidate    PASS

Release notes                             TODO
Credits / third-party notices             DRAFTED
USBPcap license identified (Driver GPLv2)  PASS
MAME configuration guide                  TODO
Final release hashes/package freeze       TODO
```

**Checkpoint:** Beta 0.815 has completed the technical development
cycle. The new release path works on normal Windows 11 without the
project's former test-signed kernel drivers. Preserve the working
application and installer unchanged while completing documentation,
credits, licensing and release packaging.

------------------------------------------------------------------------

## Current status at end of 7 October 2026

Beta 0.815 remains functionally frozen. No input/backend architecture
changes were required during the final documentation pass.

### Final application defaults

``` text
Trackball speed default             3x
Spinner speed default               0.50x
Saved settings                      Override defaults
SELECT                              5
START                               1
MENU                                SPACE
FIRE (L)                            LCTRL
FIRE (R)                            LALT
```

### GUI / release presentation

``` text
Compact GUI at Windows 150% scale   PASS
Save / Notices / About / Exit       PASS
Author display "by WiGGi"           PASS
Controller artwork                  PASS
No-CIBO joystick artwork            PASS
```

### MAME 0.289 reference setup

The public `MAME_SETUP.md` now contains illustrated, freshly tested
reference configurations for:

-   **Arkanoid (World, older)** --- spinner/dial
-   **Centipede (revision 4)** --- trackball

Arkanoid was verified with `Dial Device Assignment = mouse`,
`Keyboard Input Provider = dinput`, and `Dial Analog = Mouse Scroll V`.

Centipede was verified with `Trackball Device Assignment = mouse`,
`Keyboard Input Provider = dinput`, `Mouse Input Provider = dinput`,
`Trackball X Analog = Mouse X-Axis`, and
`Trackball Y Analog = Mouse Y-Axis`.

The documented analog values are recommendations only. Users may adjust
sensitivity, reverse and increment/decrement speed to suit the game and
personal preference. The same basic configuration method should also
work with other MAME spinner/dial and trackball games.

### PASS / FAIL / TODO

``` text
Beta 0.815 application              PASS
USBPcap direct input                PASS
Automatic controller detection      PASS
Automatic USBPcap root matching     PASS
Automatic USB address detection     PASS
Trackball                           PASS
Spinner                             PASS
Five physical buttons               PASS
Keyboard output for MAME            PASS
Settings persistence                PASS
Windows 150% GUI scaling            PASS
One-click installer                 PASS
Clean Windows 11 installation       PASS
Secure Boot ON                      PASS
Test Signing OFF                    PASS
Multiple USB ports                  PASS
Ownership-aware uninstall           PASS
Arkanoid spinner setup / MAME .289  PASS
Centipede trackball / MAME .289     PASS
Illustrated MAME_SETUP.md           PASS

Concrete functional defect           NONE KNOWN

Third-party source/compliance package PASS (USBPcap 1.5.4.0 source + notice)
Own-project LICENSE                   DEFERRED (Egret2MAME source not released)
Public Egret2MAME source cleanup      N/A for initial binary-only Beta
Release hashes / package freeze      TODO
```

**Freeze rule remains in effect:** do not change the working Beta 0.815
program or installer unless a concrete defect is found.


------------------------------------------------------------------------

## Current status at end of 7 October 2026

``` text
Beta 0.815 application                PASS
One-click installer                   PASS
Clean Windows 11 installation         PASS
Secure Boot enabled                   PASS
Windows Test Signing disabled         PASS
Controller detection                  PASS
Multiple USB ports                    PASS
Trackball                             PASS
Spinner                               PASS
Five buttons                          PASS
Saved settings                        PASS
Uninstallation                        PASS
MAME 0.289 Arkanoid setup             PASS
MAME 0.289 Centipede setup            PASS
Windows 150% GUI scaling              PASS
UHD / Windows 250% GUI scaling        PASS
Concrete functional defect            NONE KNOWN

USBPcap 1.5.4.0 source package        PREPARED
USBPcap redistribution notice         PREPARED
Egret2MAME source release             NOT PLANNED FOR INITIAL BETA
Own-project open-source LICENSE        DEFERRED
Final release SHA-256 / package freeze TODO
```

**Checkpoint:** Beta 0.815 is functionally frozen. The compact 75% GUI
layout has passed the UHD / 250% scaling test. The repository release
material now includes the corresponding USBPcap 1.5.4.0 upstream source
archive and redistribution notice under `licenses/`. Do not modify the
working application/installer unless a concrete defect is found.
