# OnePlus 11 (salami) boot chain measurements for mainline Linux

This repository records what a real OnePlus 11 (CPH2449, salami, Snapdragon 8 Gen 2) does when it is asked
to boot a mainline Linux kernel. The measurements were made between 2026-11-01 and 2026-11-07. The work was
closed on 2026-11-07 without reproducing a verifiable mainline boot; the measurements are the result.

What it contains:
- the measured partition layout and boot flow of the device;
- the payload configurations that were tried, with image hashes where they were kept, the observed
  behaviour, reboot counts and times to fastboot;
- the log channels that were tried and why they did not work;
- corrections to earlier conclusions that were later found to be confounded;
- open questions for the maintainer of the public mainline port.

What it does not contain:
- proprietary firmware, dumps, keys or personal data;
- a claim that mainline Linux cannot boot on this device;
- a claim that the public port does not work.

Evidence levels used in the text: measured means observed on the phone or read from the saved images;
inferred means it follows from the measurements but was not isolated in a controlled run; unverified means
there is no local evidence, usually an external claim.
Files:
docs/bootchain.md
docs/experiments.md
docs/log-channels.md
docs/open-questions.md
DATA/experiments.csv

Licence: MIT. See LICENSE and LICENSE-data.
