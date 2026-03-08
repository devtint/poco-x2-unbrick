# The Magic Guide: Rooting POCO X2 (Phoenix) with APatch

This guide provides the absolute fastest, cleanest, and most stable method for gaining kernel-level root access on the POCO X2 (Legacy 4.14 Kernel) using **APatch**, while completely preserving your userdata, accounts, and official MIUI firmware.

*Forget custom kernels. Forget TWRP. Forget bootloops.*

---

## 🛠 Prerequisites

1.  **Unlocked Bootloader:** Your device must already have an unlocked bootloader.
2.  **Stock `boot.img`:** You must extract the pristine `boot.img` from the exact same official MIUI Fastboot ROM version currently installed on your phone.
3.  **ADB & Fastboot:** Installed on your PC.

---

## 🪄 The Process (Step-by-Step)

### Step 1: Prepare the Files
1.  Download the latest [APatch Manager APK](https://github.com/bmax121/APatch/releases) from GitHub.
2.  Transfer both the `APatch.apk` and your stock `boot.img` to your phone's `/sdcard/Download/` folder.

### Step 2: Patching the Kernel Natively
1.  Install and open the **APatch** application on your phone.
2.  Tap on the **Patch** button (Do *not* click Embed KPM).
3.  Select your stock `boot.img` from the Downloads folder.
4.  **Important:** Enter a **SuperKey**. This acts as your root password for the system. Remember it!
5.  Tap **Start**.
6.  APatch will heavily modify the kernel binary and generate a custom `apatch_patched_xxxx.img` file in your Downloads folder.

### Step 3: Fastboot Flashing (The Magic)
1.  Transfer the newly generated `apatch_patched_xxxx.img` from your phone back to your PC.
2.  Reboot your POCO X2 into Fastboot mode (Volume Down + Power).
3.  Connect to your PC and verify connection:
    ```bash
    fastboot devices
    ```
4.  Flash the modified APatch kernel directly over the top of your stock boot sector:
    ```bash
    fastboot flash boot apatch_patched_xxxx.img
    ```
5.  Reboot the phone into Android:
    ```bash
    fastboot reboot
    ```
    *(Note: This terminal command only alters the `boot` engine partition. Your `userdata` partition remains 100% insulated and untouched.)*

### Step 4: Final Activation
1.  Once booted to the Android home screen, open the **APatch** app.
2.  Navigate to settings/authorization and input the **SuperKey** you created in Step 2.
3.  The status should instantly turn **Blue (Working)** indicating successful kernel-level root hooks!
4.  Tap **Install** under the **AndroidPatch** section to activate backend support for standard Magisk/Root modules.
5.  Reboot your phone one final time.

You are now 100% rooted with hardware-level stealth and absolute stock stability!
