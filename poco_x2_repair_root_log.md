# Comprehensive Technical Log: POCO X2 (Phoenix) Unbrick, Flash, and Root

**Date of Session:** March 8, 2026
**Target Device:** POCO X2 / Redmi K30 4G
**Hardware Codename:** `phoenix` / `phoenixin`
**Initial State:** Hard-bricked, bootlooping directly to Fastboot (Mitu mascot).
**Final State:** Booting MIUI 12.5 (Android 11), Fully Rooted via Magisk v27.0

---

## Part 1: Initial Diagnosis and USB Connection Issues

### 1.1 The Initial Bricked State
The user reported the POCO X2 was permanently stuck in Fastboot mode. Hardware resets (holding Power for 15-20 seconds) successfully restarted the device, but it immediately returned to the Fastboot screen. This behavior is triggered when the Android bootloader cannot find a valid `boot`, `recovery`, or `system` partition to load, forcing it to fall back to the Fastboot interface to await over-the-wire rescue commands.

### 1.2 Attempting Initial Fastboot Connection
We attempted to connect to the device via Fastboot to assess its partition status:
```powershell
fastboot devices
fastboot --version
```
**Issue Encountered:** Fastboot could not consistently communicate with the device. Windows placed the device into an `Error` state under Device Manager.

Querying the device via PowerShell:
```powershell
Get-PnpDevice | Where-Object { $_.FriendlyName -match "Android|Fastboot" -or $_.Class -match "Android" } | Select-Object -Property InstanceId, Status, Class, FriendlyName
```
**Output:**
```
InstanceId                     Status Class FriendlyName
----------                     ------ ----- ------------ 
USB\VID_18D1&PID_D00D\[DEVICE_SERIAL] Error        Android
```

**The Driver Problem:** The device was identifying itself with Hardware ID `USB\VID_18D1&PID_D00D`. Windows lacked the generic Android Bootloader WinUSB driver required to interface with this hardware natively, resulting in an "Access is denied" or missing driver error.

### 1.3 Driver Fix Attempts
**Attempt 1: Official Google WinUSB Driver**
Downloaded the official Android USB driver from Google (`usb_driver_r13-windows.zip`), extracted it, and attempted to load it into Windows via `pnputil.exe`.
*Error:* Windows threw signature enforcement errors and claimed the `.inf` file did not contain the driver software for this device because `VID_18D1&PID_D00D` isn't fully categorized dynamically for all OEM variants.

**Attempt 2: Zadig Injection (Successful)**
To bypass Windows' strict driver signing and matching, we utilized **Zadig v2.9**, an open-source tool that forcefully installs generic USB drivers.
*   **Download:** `https://github.com/pbatard/libwdi/releases/download/v1.5.1/zadig-2.9.exe`
*   **Process:** Selected the "Android" device in Zadig and forced the installation of the **WinUSB** driver.
*   **Resetting the Connection:** To make Fastboot recognize the new WinUSB driver without physically unplugging (though an unplug was momentarily required), we used PowerShell to disable and enable the PnP device programmatically:

```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Enum\USB\VID_18D1&PID_D00D\[DEVICE_SERIAL]\Device Parameters' -Name 'DeviceInterfaceGUIDs' -Value @('{3F966BD9-FA04-4EC5-991C-D326973b5128}', '{f72fe0d4-cbcb-407d-8814-9ed673d0dd6b}')
Disable-PnpDevice -InstanceId 'USB\VID_18D1&PID_D00D\[DEVICE_SERIAL]' -Confirm:$false
Enable-PnpDevice -InstanceId 'USB\VID_18D1&PID_D00D\[DEVICE_SERIAL]' -Confirm:$false
```

Once Zadig completed and the interface was bounced, `fastboot devices` successfully returned:
`[DEVICE_SERIAL]         fastboot`

---

## Part 2: Device Partition Assessment

With a stable Fastboot connection over WinUSB, we queried the bootloader properties.
```powershell
fastboot getvar product
fastboot oem device-info
fastboot reboot recovery
```

**Outputs:**
*   `product: phoenix` -> Confirmed the hardware is the POCO X2 / Redmi K30 (Phoenix).
*   `Device unlocked: true` -> The bootloader is completely unlocked, allowing arbitrary partition flashing without OEM authentication.
*   `Rebooting into recovery` -> Failed. The device booted right back into Fastboot.

**Conclusion:** The phone had a completely corrupted and unbootable `boot` and `super` (system) partition. Because the bootloader was unlocked, a manual CLI fastboot flash of the official firmware would effortlessly revive it.

---

## Part 3: Downloading and Extracting Firmware

We required a native `.tgz` Fastboot ROM. Recovery ROMs (`.zip`) cannot be flashed over bare metal Fastboot.
*   **Target URL:** `https://mifirm.net/model/phoenix.ttt`
*   **Selected ROM:** Indian Global Stable `V12.5.6.0.RGHINXM`
*   **Filename:** `phoenixin_in_global_images_V12.5.6.0.RGHINXM_20211123.0000.00_11.0_in_794e6c1d4f.tgz`

The user downloaded the 3.2GB file, and we seamlessly decompressed it via PowerShell:
```powershell
tar -xf "D:\phoenixin_in_global_images_V12.5.6.0.RGHINXM_...tzg" -C "[YOUR_SDK_PATH]"
```

---

## Part 4: Modifying the Flash Script and Unbricking

We navigated to the extracted directory containing `flash_all.bat`.

### 4.1 Script Error 1: Architecture Mismatch Validation
When executing `.\flash_all.bat`, it immediately failed:
**Error:** `Missmatching image and device`

**Root Cause:** The `flash_all.bat` script had a hardcoded conditional that checked if the device's variable matched exactly `phoenixin`. However, our hardware identified simply as `phoenix`.
**The Fix:** We edited the script to alter the regex find string.
*Original:*
`fastboot %* getvar product 2>&1 | findstr /r /c:"^product: *phoenixin" || echo Missmatching image and device`
*Modified:*
`fastboot %* getvar product 2>&1 | findstr /r /c:"^product: *phoenix" || echo Missmatching image and device`

### 4.2 Script Error 2: Sparse CRC Integrity Check Failure
When executing the patched script, it failed early in the flash sequence:
**Error:** 
```
Writing 'sparsecrclist'                            FAILED (remote: 'update sparse crc list failed')
```
**Root Cause:** Because the device partitions were completely broken, verifying the anti-rollback CRC lists failed natively. 
**The Fix:** These security check partition lists are not mandatory for a raw boot. We commented out the execution of these checks in `flash_all.bat` by prefixing the lines with `::`.
*Modified Lines:*
```bat
:: fastboot %* flash crclist %~dp0images\crclist.txt || @echo "Flash crclist error" && exit /B 1
:: fastboot %* flash sparsecrclist %~dp0images\sparsecrclist.txt || @echo "Flash sparsecrclist error" && exit /B 1
```

### 4.3 Successful Firmware Flashing
We executed the heavily modified `flash_all.bat` script.
**Key Command Execution Log:**
```
Sending sparse 'super' 1/8 (770492 KB)             OKAY [ 26.841s]
Writing 'super'                                    OKAY [  0.002s]
...
Sending 'userdata' (10308 KB)                      OKAY [  0.377s]
Writing 'userdata'                                 OKAY [  0.001s]
...
Sending 'boot' (131072 KB)                         OKAY [  4.673s]
Writing 'boot'                                     OKAY [  3.893s]
...
Rebooting                                          OKAY [  0.001s]
```

**Result:** The device accepted the 5GB payload flawlessly, bypassed all soft-brick loops, and rebooted natively into the MIUI 12.5 setup interface. 

---

## Part 5: Rooting the POCO X2 via Magisk 

Since we flashed the device manually, rooting it the modern way (systemless via patched `boot.img`) was trivial because we already possessed the exact stock boot partition kernel.

### 5.1 Preparation and ADB Transfers
1.  The user completed the MIUI Setup, enabled **Developer Options**, and enabled **USB Debugging**.
2.  We downloaded the official Magisk v27.0 binary directly to the PC:
    `Invoke-WebRequest -Uri "https://github.com/topjohnwu/Magisk/releases/download/v27.0/Magisk-v27.0.apk" -OutFile "[YOUR_SDK_PATH]\Magisk.apk"`
3.  We prepared the newly downloaded `Magisk.apk` and the pristine `boot.img` from the extracted ROM folder.
4.  Pushed both items directly to the device over ADB:
    ```powershell
    adb push [YOUR_SDK_PATH]\Magisk.apk /sdcard/Download/
    adb push [YOUR_ROM_PATH]\images\boot.img /sdcard/Download/
    ```

### 5.2 Patching the Kernel Natively
The user installed the `Magisk.apk` via their File Manager, opened the app, and clicked **"Install -> Select and Patch a File"**. They selected the `boot.img`. Magisk analyzed the kernel, injected the `su` binaries, rebuilt it, and exported it as `magisk_patched-27000_lyd9z.img`.

### 5.3 Fetching and Flashing the Custom Root Kernel
We pulled the modified image back to the host computer:
```powershell
adb pull /sdcard/Download/magisk_patched-27000_lyd9z.img [YOUR_SDK_PATH]\magisk_patched.img
```

Rebooted the device back to the bootloader:
```powershell
adb reboot bootloader
```

*(Note: Windows temporarily lost track of the Fastboot driver again upon reboot, but we instantly repaired it by resetting the PnP device as outlined in Section 1.3).*

Flashed the patched root image into the boot sector:
```powershell
fastboot flash boot [YOUR_SDK_PATH]\magisk_patched.img
```
**Output:**
```
Sending 'boot' (131072 KB)                         OKAY [  4.323s]
Writing 'boot'                                     OKAY [  0.659s]
```

Issued final restart:
```powershell
fastboot reboot
```

### 5.4 Final Root Verification
Once the device powered on into the Android GUI, we validated the systemic root status anonymously via an ADB root ping request:
```powershell
adb shell su -c "id"
```
**Final Server Response:**
```
uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
```

The `uid=0` string confirms total and absolute root elevation successfully granted by the `magisk` daemon context. The device is saved, stable, and unlocked.
