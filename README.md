# RedMagic 9 Pro Global OS11 Root via EDL / init_boot_b

This repository documents a successful root method for the **RedMagic 9 Pro Global / NX769J** running **RedMagic OS11 / Android 16**.

The key finding: fastboot could detect `init_boot_b`, but refused to write it with `FAILED (remote: 'unknown command')`. The patched `init_boot_b` image was flashed successfully through Qualcomm EDL/firehose instead.

> Tested on one RedMagic 9 Pro Global device only. Use at your own risk.

## Device Tested

| Item | Value |
| --- | --- |
| Device | RedMagic 9 Pro Global |
| Product | `NX769J` |
| Toolbox profile | `PQ83A01` |
| OS | RedMagic OS11 / Android 16 |
| Active slot | `_b` |
| Bootloader | Unlocked |
| Patched partition | `init_boot_b` |

## Required Tools

The tools below were used from ZTE Family Toolbox `1.1.5-test`:

- Qualcomm HS-USB QDLoader 9008 driver
- `QSaharaServer.exe`
- `fh_loader.exe`
- `ptanalyzer.exe`
- `magiskboot.exe`
- ADB/Fastboot
- Magisk `28.1`
- Firehose for toolbox profile `PQ83A01`

This repo intentionally does **not** redistribute proprietary binaries, firehose files, Magisk APKs, or Qualcomm tools. Bring your own trusted toolkit files.

## Safety Warnings

- Check your active slot before flashing:

```bat
adb shell getprop ro.boot.slot_suffix
```

- Do not blindly copy sector numbers unless your GPT matches.
- Flashing the wrong slot, LUN, sector, or image can bootloop or brick your phone.
- Keep backups of original `init_boot_a` and `init_boot_b`.

## Windows Driver Note

If Device Manager shows Qualcomm 9008 with **Code 52**, Windows rejected the driver signature.

Temporarily boot Windows with driver signature enforcement disabled:

```text
Settings > System > Recovery > Advanced startup > Restart now
Troubleshoot > Advanced options > Startup Settings > Restart
Press 7 or F7: Disable driver signature enforcement
```

Then reconnect the phone in EDL/9008 and confirm the Qualcomm QDLoader device status is OK.

## Method Overview

1. Confirm active slot.
2. Enter EDL/9008.
3. Send `PQ83A01` firehose.
4. Read GPT without `--skip_config`.
5. Locate `init_boot_a` and `init_boot_b`.
6. Extract both original images.
7. Patch the active-slot `init_boot` with Magisk.
8. Attempt fastboot flash.
9. If fastboot fails with `unknown command`, reboot to EDL.
10. Flash patched `init_boot_b` through EDL/firehose.
11. Reboot and verify root.

## Commands

### Check active slot

```bat
adb shell getprop ro.boot.slot_suffix
```

Tested result:

```text
_b
```

### Send firehose

Replace `COMx` with your actual Qualcomm 9008 COM port.

```bat
QSaharaServer.exe -p \\.\COMx -s 13:bin\res\PQ83A01\firehose
```

Expected success:

```text
File transferred successfully
Sahara protocol completed
```

### Read GPT

Important: reading GPT with `--skip_config` produced invalid GPT data on this device. Reading without `--skip_config` worked.

Found on UFS LUN 4:

```text
init_boot_a: LUN 4, start sector 340102, sectors 2048
init_boot_b: LUN 4, start sector 683226, sectors 2048
```

Each image is 8 MB:

```text
2048 sectors * 4096 bytes = 8388608 bytes
```

### Extract init_boot_a and init_boot_b

Use [`xml/read_init_boot_ab.xml`](xml/read_init_boot_ab.xml):

```bat
fh_loader.exe --port=\\.\COMx --memoryname=ufs --sendxml=xml\read_init_boot_ab.xml --convertprogram2read --mainoutputdir=backup --noprompt
```

Verify with:

```bat
magiskboot.exe unpack backup\init_boot_b_OS11_from_phone.img
```

Expected image characteristics:

```text
HEADER_VER      [4]
KERNEL_SZ       [0]
PAGESIZE        [4096]
RAMDISK_FMT     [lz4_legacy]
VBMETA
```

### Patch with Magisk

Copy the original active-slot image to the phone and patch it:

```text
Magisk > Install > Select and Patch a File
```

Example patched output:

```text
magisk_patched-30700_e6C0i.img
```

Your filename will probably be different.

### Fastboot failure observed

Fastboot saw the device and the bootloader was unlocked:

```bat
fastboot getvar unlocked
```

Result:

```text
unlocked: yes
```

But flashing failed:

```bat
fastboot flash init_boot_b magisk_patched-30700_e6C0i.img
```

Result:

```text
Sending 'init_boot_b' (8192 KB)                    OKAY
Writing 'init_boot_b'                              FAILED (remote: 'unknown command')
fastboot: error: Command failed
```

This also failed:

```bat
fastboot --slot=b flash init_boot magisk_patched-30700_e6C0i.img
```

### Reboot to EDL from Android

```bat
adb reboot edl
```

### Flash patched init_boot_b via EDL

Use [`xml/write_init_boot_b_patched.xml`](xml/write_init_boot_b_patched.xml).

Make sure the patched image is in your `backup` folder or adjust `--search_path`.

```bat
fh_loader.exe --port=\\.\COMx --memoryname=ufs --sendxml=xml\write_init_boot_b_patched.xml --search_path=backup --mainoutputdir=tmp --noprompt
```

Expected success:

```text
{<program> (8.00 MB) 2048 sectors needed at location 683226 on LUN 4}
==================== {SUCCESS} ========================
{All Finished Successfully}
```

### Reboot from firehose

Use [`xml/power_reset.xml`](xml/power_reset.xml):

```bat
fh_loader.exe --port=\\.\COMx --memoryname=ufs --sendxml=xml\power_reset.xml --mainoutputdir=tmp --noprompt
```

## Root Verification

After Android boots:

```bat
adb shell getprop ro.boot.slot_suffix
adb shell su -c id
```

Successful result:

```text
_b
uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
```

## Credits

Thanks to the RedMagic/ZTE community and the existing XDA guides that documented the original RedMagic 9 Pro bootloader/root workflow. This repo only adds the OS11-specific finding from this successful test: EDL/firehose writing worked when fastboot writing failed.
