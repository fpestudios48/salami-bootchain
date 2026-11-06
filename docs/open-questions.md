# Method corrections and open questions

## Corrections to earlier conclusions

Two conclusions from 2026-11-06 were found to be unsupported when the flash trace was reconstructed:

1. The claim that the bootloader only accepts a factory vendor_boot was based on a run where the image was not
   a factory-header image. Its header used addresses offset by 0x10000000, it had a 0 byte bootconfig and no AVB
   footer, and its device tree had no __symbols__. It was flashed with the factory dtbo, which is the
   configuration already known to fail because the nine overlays cannot resolve. That run does not test the
   signature.
2. The claim that a boot image with an appended device tree is rejected was based on a run where the rejected
   vendor_boot from the previous test was still flashed on the device. The variable was not isolated.

A third correction concerns timing. Time to fastboot was used as a dose response signal in earlier notes. It is
not a metric: with the same material the fallback happened between 12 and 630 s, and the number of bootloader
retries (between 2 and 4) was never controlled.

## Open questions

1. Which exact boot image, kernel build, vendor_boot, dtbo and init_boot does the public port maintainer use on
   this device, and what are their sha256 hashes? The port is published at
   https://github.com/Ynkcc/salami-mainline and its packaging script produces a boot image with the device tree
   appended; the device wiki lists booting as working but publishes no installation steps or images.
2. Which firmware and bootloader version was the device on when that configuration booted? The device here is on
   Android 16 with a bootloader that applies nine dtbo overlays from a table with board id 0x10008.
3. Does the vendor_boot signature get checked when vbmeta verification is disabled? A controlled test would be a
   factory-header vendor_boot with a valid factory DTB and no AVB footer, everything else factory.
4. Why does an empty dtbo also fail at about 6 s? If reproducible, a minimum of one matching overlay may be
   required by the bootloader before it applies the table.
5. Does a boot image with an appended device tree work when vendor_boot and dtbo are fully factory? The run that
   would answer this was confounded and has not been repeated.
6. Is the licence of the public port scripts compatible with reuse of their image layout in a derived
   measurement setup? The licence of scripts/boot-img.sh in the public port was not verified.

## What would settle the main question

A single clean run with everything factory except the boot image and init_boot: factory vendor_boot, factory
dtbo, factory vbmeta, a mainline kernel with its device tree appended to boot.img, and an init in init_boot
that turns the phone off after a fixed delay. A black screen that stays black, with the cable disconnected,
would mean the kernel started PID 1. A screen that stays on or falls back to the logo means the run is
indistinguishable from a kernel that died early. This test was run on 2026-11-07 and the power off control
failed: both runs went black. The work was closed at that point. The questions remain open for anyone with the
port artefacts.
