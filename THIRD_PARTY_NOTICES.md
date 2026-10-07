# Third-Party Notices

Egret2MAME Beta 0.815 relies on or was developed with the following
third-party software and platform technologies.

## USBPcap

**Role:** Required runtime capture component\
**Project:** USBPcap --- USB packet capture for Windows\
**Author / copyright:** Tomasz Moń and contributors\
**Source:** https://github.com/desowin/usbpcap

Egret2MAME Beta 0.815 uses the official USBPcap capture driver and
distributes the official USBPcap installer rather than a modified
Egret-specific USBPcap build.

The upstream USBPcap project states:

-   `USBPcapDriver` --- GNU General Public License version 2 (GPL-2.0)
-   `USBPcapCMD` --- BSD 2-Clause License

Egret2MAME requires the USBPcap driver at runtime. It does **not**
require USBPcapCMD or Wireshark at runtime.

For Beta 0.815, the repository's `licenses/` area contains the complete
corresponding upstream USBPcap **1.5.4.0** source archive together with a
USBPcap redistribution/source notice. The source archive is the source
corresponding to the USBPcap version bundled by the Egret2MAME installer.

The USBPcap material remains third-party software. Egret2MAME does not
claim ownership of USBPcap.

## Inno Setup

**Role:** Build/installer tool only\
**Authors:** Jordan Russell and Martijn Laan\
**Project:** https://jrsoftware.org/isinfo.php

The Egret2MAME Windows installer is built with Inno Setup 7. End users
do not need Inno Setup.

## Wireshark

**Role:** Development/research tool only\
**Project:** https://www.wireshark.org/

Wireshark was used during development to inspect and validate USB
traffic. It is not bundled with Egret2MAME and is not required at
runtime.

Wireshark is distributed under the GNU General Public License version 2
or later.

## Microsoft Windows / .NET Framework / Windows Forms

**Role:** Target platform and application framework

Egret2MAME Beta 0.815 is a Windows desktop application using Microsoft
Windows APIs and .NET Framework / Windows Forms.

Microsoft, Windows and .NET are trademarks of Microsoft Corporation.
Their mention does not imply endorsement.

## MAME

**Role:** Primary target emulator; not bundled\
**Reference version:** official standalone MAME 0.289\
**Project:** https://www.mamedev.org/

MAME is not included with Egret2MAME. Egret2MAME is an independent
compatibility utility and is not part of or endorsed by the MAME
project.

MAME is a registered trademark of Gregory Ember.

## TAITO / EGRET II mini

The supported controller is the TAITO EGRET II mini Paddle & Trackball
Controller.

Egret2MAME is an unofficial independent project and is not affiliated
with or endorsed by TAITO Corporation.

## Runtime dependency summary

  Component                     Beta 0.815 status
  ----------------------------- ----------------------------------
  USBPcap Driver                Required / installed when needed
  USBPcapCMD                    Not required
  Wireshark                     Not required
  Inno Setup                    Build tool only
  .NET Framework / WinForms     Windows platform
  MAME                          Target application / not bundled
  Egret custom kernel drivers   Not used
  ViGEm                         Not used
  Microsoft VHF                 Not used

## Release package note

Beta 0.815 redistributes the official USBPcap 1.5.4.0 runtime installer
unchanged. The corresponding upstream USBPcap 1.5.4.0 source archive and
an explanatory notice are provided under `licenses/`.

The initial Beta 0.815 release does **not** publish the Egret2MAME
application source code. USBPcap remains a separate third-party runtime
component.

This notice is an attribution summary and does not replace the applicable
upstream license terms.
