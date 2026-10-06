# MAME Setup Guide

This guide describes the reference MAME setup used while developing Egret2MAME Beta 0.815.

## Reference version

The primary test target is the official standalone Windows build of **MAME 0.289**.

Other versions and frontends may work, but Beta 0.815 does not guarantee identical behavior with them.

## Recommended input provider

Use **DInput or Win32** for the relevant Windows input path.

The Beta 0.815 keyboard-button solution was validated with MAME distinguishing the generated keys correctly under these providers.

RawInput was not the reference configuration for the final button-output path.

## Button assignments

| Controller | MAME key |
|---|---|
| SELECT | `5` |
| START | `1` |
| MENU | `SPACE` |
| FIRE (L) | `LCTRL` |
| FIRE (R) | `LALT` |

## Trackball

The physical controller already appears to Windows as mouse-style relative input.

Egret2MAME keeps the native movement and, when a multiplier above 1x is selected, injects the additional relative movement required to reach the selected effective speed.

Available values: **1x, 2x, 3x, 4x, 5x, 6x**.

**1x is the recommended neutral starting point.**

Game-specific MAME sensitivity settings can still affect how a particular title feels.

## Spinner

Available values: **0.25x, 0.50x, 0.75x, 1.00x**.

## If trackball movement feels wrong

During development, an old/modified MAME configuration caused trackball movement to behave as if it were joystick-like and to recenter unnaturally.

If a game behaves incorrectly even when Egret2MAME is not running, test with clean MAME configuration files before assuming the controller bridge is at fault.

For the development test with Centipede, resetting the affected MAME configuration restored normal smooth mouse/trackball behavior.

Back up configuration files before deleting them if they contain mappings you want to keep.

Typical files to inspect include:

- `cfg/default.cfg`
- the individual game's file under `cfg/`

## Game behavior still matters

Egret2MAME changes the input delivered to MAME; it does not alter the original game's movement rules. A game can impose its own maximum movement speed even when the physical trackball is spun faster.

## Scope

This guide documents the configuration that worked during Egret2MAME development rather than claiming that every MAME version, frontend or input-provider combination behaves identically.
