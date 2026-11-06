# Measured boot chain of the OnePlus 11 (salami)

## Device

OnePlus 11 CPH2449, codename salami, Snapdragon 8 Gen 2 (SM8550), Android 16 build BP4A.251205.006,
bootloader unlocked, slot A, vbmeta_a flags 3 (AVB verification disabled). All measurements were made on one
physical phone between 2026-11-01 and 2026-11-07. Backups of the factory boot partitions were taken before the
first write and verified with sha256.

## Partition layout (measured)

| Partition | Size | Content |
| boot_a | 192 MiB | Android kernel 45,636,096 B, ramdisk 0 |
| vendor_boot_a | 192 MiB | factory DTB 2,122,773 B (four concatenated FDTs, 1,575 symbols) and vendor_ramdisk 17,436,690 B |
| init_boot_a | 8 MiB | generic ramdisk 2,927,215 B, first stage init |
| dtbo_a | 24 MiB | nine OnePlus overlays, board id 0x10008 |
| vbmeta_a | 64 KiB | flags 3, verification disabled |

Android boots with the factory vendor_boot and the nine dtbo overlays applied by the bootloader. The init_boot
init is executed: replacing it with a test init prevents Android from booting.

## Observed behaviour with mainline payloads

| Payload condition | Observation | Evidence |
| vendor_boot with mainline DTB 131,960 B (no __symbols__), factory dtbo | return to fastboot about 6 s after reboot | measured |
| same DTB rebuilt with -@ and labels qcom_tzlog and arch_timer (171,683 B, 555 symbols), factory dtbo | return to fastboot about 6 s | measured |
| vendor_boot with the symbols DTB and a custom dtbo with one benign overlay (4,332 B, target qcom_tzlog) | bootloader proceeds to boot the kernel; one run returned to fastboot at 28 s, another looped with the logo for about 3m40 | measured |
| vendor_boot with the factory DTB, factory bootconfig and cmdline, factory boot | no USB enumeration for about 10.5 min, then fastboot | measured, several variables changed at once |
| boot.img with Image and DTB appended (58,896,384 B), factory vendor_boot, dtbo and misc, custom init in init_boot | kernel executes; static green screen for more than 218 s with an init that pauses; dose response timings in docs/experiments.md | measured |
| vendor_boot from boot-deploy with mainline DTB 131,960 B and pmOS initramfs, factory dtbo, factory init_boot | return to fastboot about 44 s | confounded: header not factory (addresses +0x10000000, bootconfig 0 B, no AVB footer), DTB without symbols, factory dtbo |
| boot.img with appended DTB and pmOS initramfs in init_boot | return to fastboot about 40 s | confounded: the rejected vendor_boot was still flashed |
| init_boot with test init, factory boot and factory vendor_boot | Android does not boot; the init_boot init is the one that runs | measured |

## Why the dtbo matters

The bootloader applies the nine overlays in dtbo_a to the DTB found in vendor_boot before starting the kernel.
The overlays reference 164 labels. The factory DTB has 1,575 symbols and resolves all of them. A mainline DTB
built from the public port (131,960 B) has no __symbols__; a rebuilt one with labels and -@ (171,683 B) still
resolves only 16 of the 164. With the factory dtbo the bootloader does not start the kernel and returns to
fastboot about 6 s after reboot. With a dtbo that contains one benign overlay targeting qcom_tzlog, the
bootloader proceeds to boot, which is the only configuration where a mainline kernel was started through the
vendor_boot path.

An empty dtbo also returned to fastboot in about 6 s in one run. If that is reproducible, the labels alone do
not explain the rejection. A second condition could be that dtbo must contain at least one overlay that matches
the board. That hypothesis is unverified.

## The route that executed a mainline kernel

The public port packaging script creates a boot image with the kernel and the device tree concatenated, without
ramdisk, and keeps the factory vendor_boot, dtbo and misc partitions. A custom init placed in init_boot is
executed after the kernel starts. This configuration produced repeated kernel execution signals, described in
docs/experiments.md. It has not been shown to reach user space with a reliable channel: an init that pauses and
a dead kernel can both leave a static screen, and the time to fastboot depends on how many times the bootloader
retries, which was not controlled.

## Custom ROM control

An installed custom Android ROM (AlphaDroid 16) boots with a vendor_boot whose device tree is byte for byte
identical to the factory one; only the vendor_ramdisk differs (419 bytes smaller) and the kernel in boot_a is a
different Android 5.15 build. The published installation instructions for LineageOS 24.0 and /e/OS for this
device also replace boot, dtbo, init_boot, vbmeta and vendor_boot with their own images, but their kernels are
Android 5.15 vendor kernels and their device trees come from the same Android source. These cases show that the
bootloader accepts another complete Android payload. They do not show that it accepts a mainline device tree,
because none of them contains one. Whether the vendor_boot signature is checked is not known: vbmeta has
verification disabled, and no controlled test of a vendor_boot without an AVB footer has been made with
everything else factory.

## What is not concluded

The signature requirement often mentioned for this device is not measured. The only measured condition is that
the dtbo overlays must resolve against the DTB supplied in vendor_boot, plus the header and payload format
conditions listed above. How far those conditions go is open.

## Status

The work was closed on 2026-11-07 without reproducing a verifiable mainline boot. No tested configuration could
be observed past the bootloader with a reliable channel: the display dies about one second after the kernel
takes over, the power off control gave the same black screen with and without the power off call, and the time
to fastboot depends on an uncontrolled bootloader retry count. The measured boot chain behaviour, the experiment
log and the corrections to earlier conclusions are the result of this work.
