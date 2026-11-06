# Experiment log

All runs were made on the same retail phone. Factory boot partitions were restored and verified with sha256
after every write. Fastboot commands were serialised. A run took between 3 and 10 minutes. The device was never
left in a state in which the bootloader could not be reached.

Evidence levels: measured means observed on the phone or read from a saved image; inferred means it follows
from measurements but was not isolated; unverified means there is no local evidence.

The complete machine-readable list is DATA/experiments.csv. The text below groups the runs and states what was
and was not concluded.

## Phase 1, boot image format and init location (2026-11-02)

A gzip compressed kernel produced a 2 to 3 second loop. An uncompressed Image started. A panic timeout placed
in the device tree had no effect because the kernel did not use that command line. An init placed in the boot
ramdisk was not executed; an init placed in init_boot was executed and started Android normally when handed
over. The Android kernel loads the UFS driver as a module, so an initramfs init cannot see block devices unless
it loads the module chain itself.

Watchdog work: disabling the Gunyah watchdog driver, an early initcall and a direct SCM call did not stop the 3
to 5 second reset seen in early mainline runs. Removing the SCM interconnect properties produced 6 to 10 resets
and then a hang.

## Phase 2, the vendor_boot device tree route (2026-11-05)

A custom vendor_boot with a 131,960 byte mainline device tree and no symbols returned to fastboot about 6
seconds after reboot. The same happened with a vendor_boot built with the factory header addresses. With the
factory four FDT tree and the factory bootconfig the phone stayed without USB enumeration for about 10.5
minutes and then returned to fastboot, so the bootloader did try to start the kernel with that tree.

Rebuilding the device tree with -@ and the labels qcom_tzlog and arch_timer produced a 171,683 byte tree with
555 symbols. The nine factory overlays reference 164 labels and 148 of them are still missing, so the result
did not change. A custom dtbo with one benign overlay targeting qcom_tzlog made the bootloader proceed to
boot: one run returned to fastboot at 28 seconds, another looped with the OnePlus logo for about 3 minutes and
40 seconds. At that time the kernel tree still contained self-inflicted reset instrumentation, so the later
behaviour of those runs says nothing about the kernel.

## Phase 3, the reference kernel with an appended device tree (2026-11-05 to 2026-11-06)

The port packaging script puts the kernel Image and the device tree back to back in boot.img without a ramdisk,
and leaves vendor_boot, dtbo and misc factory. With a custom init in init_boot this route produced the first
repeatable kernel execution signals. A kernel build preserved from 2026-11-05 produced a static green screen
for more than 218 seconds with an init that pauses, and repeated green screen loops with an init that reboots
after 120 seconds. The same route with a 15 second init returned to fastboot at 28 seconds.

A later rebuilt kernel hung early and was traced to two separate defects: a start_kernel call left inside a
comment in head.S and a splash memory region marked no-map while the early framebuffer used a physical
translation. After fixing both, the rebuilt kernel produced dose response timings with an init that sleeps a
given number of seconds and reboots: 15 seconds gave 52 and 71 seconds to fastboot, 30 seconds gave 150
seconds, 45 seconds gave 102 seconds. A 7.3-rc5 kernel gave about 280 seconds with 120 seconds and 143 seconds
with 15 seconds.

These timings are not a clean metric. The bootloader retries a failed boot between 2 and 4 times, and the
fallback time for the same material ranged from 12 to 630 seconds in earlier runs. A later counter written by
init to the last_parti partition or to kmsg was never found from Android, so the claim that user space was
reached is treated as unproven.

Two runs that were later corrected: a boot image with an appended device tree was reported as rejected, but
the trace shows it was flashed while a previously rejected vendor_boot was still on the device. The vendor_boot
built by boot-deploy on that session also did not have a factory header: its addresses were offset by
0x10000000, its bootconfig was empty and it had no AVB footer.

## Phase 4, observation channels (2026-11-05 to 2026-11-06)

The early framebuffer can show text, colour bands and initcall marks for about one second, then the panel stops
refreshing. Reading later marks from photographs was unreliable and was the source of the earlier claim that
the kernel stopped at late_initcall level 7. That claim was a display artefact.

pstore did not work for the mainline kernel with the factory ramoops parameters. A control with the Android
kernel itself (panic forced on Android, same kernel reads the area) also left /sys/fs/pstore empty, which points
to the platform area rather than to a pstore format mismatch.

A counter written from init to the misc partition and to kmsg was not found in later checks. The UFS did probe
once from an initramfs that loaded 101 modules and misc appeared as /dev/sda3, but the marker was still absent.

## Phase 5, pmOS boot-deploy packaging (2026-11-06)

A complete pmbootstrap build produced boot.img (52,432,896 bytes, kernel only), vendor_boot.img (11,735,040
bytes with the mainline device tree and the pmOS initramfs), initramfs and device tree. Flashing boot and
vendor_boot with the factory dtbo and factory init_boot returned to fastboot 44 seconds after reboot. This run
is confounded for three reasons listed above.

A second run used a boot image with the mainline kernel and device tree and a pmOS initramfs compressed with xz
in init_boot. It returned to fastboot 40 seconds after reboot, but the rejected vendor_boot was still flashed.

## Phase 6, power off as a binary signal (2026-11-07)

The reference kernel build, with the device tree appended to boot.img and the factory vendor_boot, dtbo and
misc, was paired with an init in init_boot that sleeps 15 seconds and calls RB_POWER_OFF. The boot image is
58,859,520 bytes with sha256 c413f7231c2618820a226f06ac6e960d48baae26f5ebc969e6809ba2b96ee7fa. It contains
the preserved kernel build number 64 (58,720,256 bytes, sha256
e1410ec16a1145f643e349bb185ec901485037c0d2fdbd91c62e932bf6c07918) and the progreso device tree (131,903 bytes,
sha256 f3400f9caf6f9596948169e520a71fb08e98e917022ae77114723cb05ceba611) appended after it. The two init_boot
images are 8,388,608 bytes each: sha256 c1996a3cbba9e100885c60780315e46d4aa23639a5a91533866dbb861fede263 for
the power off init and 55530bd939075f166e33f7eb0ea66de1e47e0b798807bc6552f48f875344df43 for the pause init. The cable was
disconnected about 6 seconds after the logo. The screen went black 10 to 15 seconds after the logo and stayed
black for more than 10 minutes, with no logo and no charging screen. The phone was then brought to fastboot
with the volume and power keys. This is consistent with a real power off, but the control run with an init that
pauses instead of powering off is needed to rule out a display that simply went black.

The control run used the same boot image and an init that sleeps 15 seconds and then pauses forever. The screen
went black in the same way, between 10 and 15 seconds after the logo, and stayed black. The two runs are
indistinguishable: the black screen is the display dying, not evidence that the power off call ran. The power
off signal therefore does not work without a display driver.

As pre-registered, this result closed the objective of booting pmOS on this device. No further boot runs were
made. The negative result and the measurements above are the outcome.

## Table of runs

| run | date | result |
| r01 | 2026-11-02 | gzip kernel looped every 2 to 3 s; uncompressed kernel started |
| r02 | 2026-11-02 | panic timeout ignored; init in boot ramdisk not executed |
| r03 | 2026-11-02 | pmOS kernel config hung; salami config ran calibration about 30 s once |
| r04 | 2026-11-02 | watchdog disabled in three places; 3 to 5 s reset persisted |
| r05 | 2026-11-02 | SCM without interconnect properties; 6 to 10 resets then hang |
| r06 | 2026-11-02 | fallback times 284, 642 and 103 s; not usable as a metric |
| r07 | 2026-11-02 | /dev/fb0 with devtmpfs produced a green screen |
| r08 | 2026-11-02 | serial gadget did not enumerate in 40 s |
| r09 | 2026-11-02 | cold boot ran; warm reboot looped every 3 to 5 s |
| r10 | 2026-11-05 | custom vendor_boot with mainline DTB and factory dtbo returned to fastboot |
| r11 | 2026-11-05 | boot-deploy boot and vendor_boot returned to fastboot at 6 s |
| r12 | 2026-11-05 | board id 0x10009 variant returned to fastboot at 6 s |
| r13 | 2026-11-05 | factory DTB vendor_boot stayed without USB about 10.5 min then fastboot |
| r14 | 2026-11-05 | symbols DTB with factory dtbo still failed; 148 of 164 labels missing |
| r15 | 2026-11-05 | benign dtbo made the bootloader proceed; later kernel behaviour invalidated by resets |
| r16 | 2026-11-05 | appended DTB route, fastboot at 46 to 75 s, misc empty |
| r17 | 2026-11-05 | debug initramfs with factory DTB, fastboot at 18 s, misc empty |
| r18 | 2026-11-05 | debug init in init_boot with factory kernel, fastboot at 12 s, misc empty |
| r19 | 2026-11-06 | appended DTB kernel, static green screen more than 218 s |
| r20 | 2026-11-06 | init sleeps 15 s and reboots; no reboot observed |
| r21 | 2026-11-06 | old kernel loops, new kernel static; source difference not isolated |
| r22 | 2026-11-06 | rc5 dose response 280 s and 143 s; metric confounded |
| r23 | 2026-11-06 | fixed kernel dose response 52, 71, 150 and 102 s; retries 2 to 4 |
| r24 | 2026-11-06 | regulator and initcall blacklist route, fastboot at 302 s, retired |
| r25 | 2026-11-06 | init counter never found in last_parti or kmsg |
| r26 | 2026-11-06 | probe init in init_boot with factory kernel prevents Android from booting |
| r27 | 2026-11-06 | boot-deploy packaging returned to fastboot at 44 s; confounded |
| r28 | 2026-11-06 | appended DTB plus pmOS initramfs returned to fastboot at 40 s; confounded |
| r29 | 2026-11-05 | simple framebuffer tree produced no text; fastboot at 12 s |
| r30 | 2026-11-05 | init powers off at 3 s; no power off, fastboot at 21 s |
| r31 | 2026-11-05 | factory DTB control, no USB more than 80 s, logo loop, fastboot at 84 s |
| r32 | 2026-11-06 | pstore token init; no power off; fastboot at 34 s |
| r33 | 2026-11-07 | power off at 15 s: screen black 10 to 15 s after logo, black more than 10 min |
| r34 | 2026-11-07 | control with pause init: also black, indistinguishable from r33 |
