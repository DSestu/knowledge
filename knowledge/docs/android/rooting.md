# Rooting Android

## Important before rooting

> **!! READ THE WHOLE ARTICLE BEFORE DOING ANYTHING.!!**

> I lost all my data because I didn't made a backup of my OS img.

You need to dump your current OS image before rooting.

Using magisk, then upload the img to your computer. (/data is maybe overkill)

This image can be extracted using payload-dumper-go if you have a payload.bin file.

This will allow you to restore your phone images if something goes wrong.

> Bricking your phone and struggling to download its actual OS image is your worst case scenario. Oneplus discontinued hosting OxygenOS images, so you are very likely to need to factory reset the phone and loosing data. From my experience, I couldn't find a boot.img that actually let me boot properly...

The only boot.img that worked for me was the one extracted from my own phone using magisk.

## Rooting (Oneplus 11 Pro example)

> Assuming you have OEM unlocked. This manipulation has to wipe the phone. But it's one time (or every time it is locked back, but you can choose to never lock it again, just don't lock it, it's manual).

You need platform-tools on your computer for adb and fastboot.

You will need to install TWRP custom recovery first.

Then you can install Magisk from TWRP.

You need to find the right version of TWRP, and flash the new recovery on your phone.

Go to fastboot mode (Power off the phone, then hold Volume Up + Power button).

or `adb reboot bootloader` if phone booted in os with usb debugging.

### Fastboot device not showing on windows

You need to make a something: device manager > other devices > android (with a like yellow triangle) > right click > update driver > browse my computer > let me pick from a list > select android device > android bootloader interface.

Then you should see your device with `fastboot devices`.

You need to find the right TWRP image for your phone model.
<https://twrp.me/Devices/>

OP11Pro not on official site, but available here:
<https://unofficialtwrp.com/unofficial-twrp-3-7-0-root-oneplus-11/>

However, it can't decrypt my data partition in the recovery.

For me, using twrp version for the OP7Pro worked fine:
<https://eu.dl.twrp.me/guacamole/>

However, the TWRP is in chinese by default. More on that after the flashing.

Get the current slot (a or b):

```
fastboot getvar current-slot
```

Flash the recovery to the current slot (replace X with a or b):

```
fastboot flash recovery_X twrp.img
```

Reboot in recovery:

```
fastboot reboot recovery
```

This will boot your phone into TWRP recovery, but this is chinese.

You have to change the language to english in the settings.

2nd column, 3rd row is settings. The last tab is language, here you can select english.

## Backup your current OS from TWRP (HEAVILY RECOMMENDED!!)

You have to backup your images and get them on your computer before proceeding. In this case, you will be able to restore those images by flashing them back using fastboot.

Main menu > Backup

Then reboot on main OS, usb transfer the files to your computer:

`/TWRP/BACKUPS/[your device id]/[date folder]/*`

The files are `.emmc.win` files, which are basically raw images of your partitions. If you want to flash them back, you need to rename them to `.img` and flash them using fastboot.

for example:

```
mv boot.emmc.win boot.img
fastboot flash boot boot.img
```

Select the partitions you want to backup (boot, system, vendor, data at least). I recommend everything. However, `Data` and `Super` are related to your full OS installation and can be very big. It's up to you, I personally exclude those because my primary use of these backups is to unbrick the phone if something goes wrong, not storing every setting that I have.

> Notes in case of brick.

* Either you flash the images directly from the computer using fastboot.
* Or, you can still access to your storage in TWRP, and so u transfer the backup from your computer to your phone using MTP, then use the restore function of TWRP.
* VBmeta can be the cause of the bootloop if the signature verification is enabled. You can flash a vbmeta image with disabled verification if needed:

```
fastboot --disable-verification flash vbmeta vbmeta.emmc.win
```

It solves some bootloop issues for me.

## Backup done, now install Magisk

Get the magisk apk from the OFFICIAL GITHUB REPO (60k stars, be careful to not download fake versions with malware), from the releases section.

<https://github.com/topjohnwu/Magisk/>

<https://github.com/topjohnwu/Magisk/releases>

Install the apk from your main OS.

Magisk will need to patch the boot.img, so you need to get it from your backup.

Copy the `boot.emmc.win` file from your TWRP backup to your phone storage or computer, rename it to `boot.img`, do the same for `init_boot.img`.

Then use the magisk app to patch the boot.img: select the install button, then "Select and Patch a File", then select the `boot.img` file.

Read the logs, it will create a new image called `magisk_patched-***.img` in the download folder.

Move it to the root of your phone. Rename it with the date (to not forget which one it is). Also rename the boot.img to something like boot_original.img with the date to not forget which one is the original.

> DO NOT OVERLOOK BACKUP STEPS. THEY ARE CRUCIAL. Once it's too late, you can't go back to fix things, you will loose photos and things forever!

Uninstall the magisk app.

**Finally**, reboot into bootloader:

```
adb reboot bootloader
fastboot --disable-verification flash vbmeta_a vbmeta.emmc.win
fastboot --disable-verification flash vbmeta_b vbmeta.emmc.win
fastboot flash boot_a magisk_patched-***.img
fastboot flash boot_b magisk_patched-***.img
fastboot flash init_boot_a magisk_patched-***.img
fastboot flash init_boot_b magisk_patched-***.img
```

Reboot in system. A empty magisk app should be present. Launch it, it will install the full app, now you have root access!

## Disable OxygenOS Updates

The issue now is that OxygenOS updates will overwrite the boot.img and remove root access.

You need to disable the OnePlus update app.

Needs developper mode enabled.

In developper settings, disable the automatic system updates toggle.

If you want to update, use the OxygenUpdater app, but you will need to extract, backup, and patch the new boot.img again with magisk.

# Recommended apps

* Greenify: to hibernate apps in background and save (a lot of) battery.

* AdAway: to block ads system wide.

* Termux: terminal emulator with a lot of packages available.

* F-Droid: alternative app store with only free and open source apps.
