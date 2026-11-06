# Log channel inventory

The problem this list addresses: with the mainline kernel running, there was no reliable way to read kernel or
init state without the phone display. Every channel below was tried or evaluated, and the result is stated.

| Channel | What was tried | Result | Status |
| display | early framebuffer text, colour bands and initcall marks | visible in the first second, then the panel stops refreshing; later marks cannot be read | limited but measured |
| pstore and ramoops | mainline pstore with the factory ramoops addresses, pmsg token, two boot designs | /sys/fs/pstore empty after forced panic; Android to Android control with a stock 5.15 kernel was also empty | dead until a working control exists |
| /dev/mem | mainline /dev/mem write to the ramoops tail and to the splash region | writes to system RAM are refused; the ramoops region could be written but the next boot did not show the token | inconclusive |
| misc partition writes from init | counter and journal written to page 0 and to last_parti from the initramfs | no marker was ever found from Android with dd; UFS does not probe from the initramfs by default | negative |
| UFS raw write | raw block writes with direct IO and fsync from init after loading the UFS modules | UFS probed once with 101 modules, but the marker was still absent | open |
| USB gadget | usb_1 and PHYs enabled in the DTS, serial gadget in the kernel config | /sys/class/udc never appeared; no USB enumeration from the mainline init | dead with the current configuration |
| kmsg | init writes to /dev/kmsg from the initramfs | no console or reader; only useful with a working serial or USB channel | not a channel on its own |
| bootloader logs | logfs, logdump, xbl_sc_logs, devinfo, last_parti partitions read from Android | 128 KB of XBL logs in xbl_sc_logs, but the failing boot was not among them | no |
| fastboot getvar | slot, retry count, boot reason, battery | no log content; retry counters did not expose the abort reason | limited |
| time to fastboot | fallback delay used as a signal | 12 to 630 s with the same material; retry count varies between 2 and 4 | not a metric |
| PSCI power off | init sleeps 15 s and calls RB_POWER_OFF | screen went black 10 to 15 s after the logo and stayed black in the first run; the control run with an init that pauses went black in the same way | negative: no binary signal |
| serial UART | physical access to pads | no public UART pads found for this model; level and pad position unverified; not attempted | not available |
| kexec | load the mainline kernel from the running Android kernel | the installed Android 5.15 kernel has CONFIG_KEXEC and CONFIG_KEXEC_FILE disabled | not available without rebuilding the Android kernel |

No binary signal was found. The power off test did not distinguish a powered off phone from a dead display
because the control run without the power off call showed the same black screen. With the display driver missing
and no serial or USB channel, the mainline boot could not be observed past the bootloader. That is the reason
the work was closed.
