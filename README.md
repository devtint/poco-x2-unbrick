# POCO X2 (Phoenix) - Unbrick and Root Guide

This repository contains the comprehensive technical log for rescuing a hard-bricked POCO X2 (Redmi K30 4G) stuck in a Fastboot loop, manually restoring the official firmware, and immediately installing a systemless root via Magisk.

## Motivation & Overview
The device was trapped in Fastboot due to corrupted `boot` and `super` partitions. The operating system could not be initialized, causing an automatic hardware fallback to Fastboot mode. 

This guide documents the exact process taken to:
1. Bypass strict Windows driver mismatch errors (and forcefully inject WinUSB using Zadig/PowerShell).
2. Modify the official MIUI flashing scripts (`flash_all.bat`) to bypass OEM hardware identifier restrictions.
3. Flash the complete official Android firmware natively over the Fastboot CLI.
4. Extract the pristine kernel (`boot.img`), patch it directly on the device, and flash it to achieve permanent root.

## Contents
* [`poco_x2_repair_root_log.md`](poco_x2_repair_root_log.md): The highly detailed, step-by-step master log covering the entire diagnosis, flashing, and rooting session.
* [`flash_all_patched.bat`](flash_all_patched.bat): The exact modified flashing script used to bypass the Xiaomi hardware mismatch error (`phoenix` vs `phoenixin`) and the anti-rollback sparse CRC errors.

## Notes & Privacy
All sensitive paths (like local Windows directories) and the device's unique Fastboot Hardware Serial Number have been masked as `[YOUR_SDK_PATH]`, `[YOUR_ROM_PATH]`, and `[DEVICE_SERIAL]`.

## Disclaimer
These logs are provided as an educational benchmark and demonstration of Android CLI debugging and partition management. Proceed at your own risk if applying these methods to your own hardware.
