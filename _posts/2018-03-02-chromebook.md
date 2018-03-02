---
title: HP Chromebook 13 G1 Review – A Year Later
---

(Actually more than a year, but I digress before even having started...)

I still own the Chromebook. I no longer run ChromeOS, but have switched to
[GalliumOS](https://galliumos.org), a XUbuntu derivate with a number of driver
adaptations. It's also a little leaner than ChromeOS and doesn't need any
chroot shenanigans, so I can run a fresh kernel instead of the dusty one that
Google ships with ChromeOS.

The downside is that I still have to press <kbd>Ctrl</kbd> + <kbd>L</kbd> at
bootup and avoid pressing <kbd>SPACE</kbd> before the machine boots, otherwise
the machine might try to overwrite my system with a clean ChromeOS install (no
idea if that would work, and I have too much data on the machine to want to
try). I hear that the bootup sequence can be altered by removing the write
protect screw and flashing the BIOS with a so-called "servo" (or at least
setting some BIOS flags with a script). Unfortunately, five of the eleven
screws are behind the rubber feet, which are glued to the device and impossible
to remove without damaging them, so doing this is out of the question unless I
can somehow source replacement rubber feet.

Otherwise, I'm still happy with the Chromebook despite a few snags having
emerged in day-to-day use: First, headphones don't work automatically (but
there is a simple command to make them work, and I hear people on the GalliumOS
side are working on a fix), and the internal mic doesn't work at all (again,
probably just a matter of time). I worked around the latter problem by buying a
cheap USB mic, but it's still a nuisance. One of the rubber foots went loose,
so I glued it back on with some shoe glue. Not great, but no big deal either.
Otherwise I love the sleek look and the comfortable weight.

The display is still the best thing about the device: The resolution is great
and the image is bright enough to read in daylight, even though the screen is
a glare-type. I recently compared with a coworkers recently bought notebook and
the difference in brightness is striking. The downside (compared to my old
large IdeaPad) is only 13.3" screen estate, and the fact that due to my HiDPI
settings everything on Full HD monitors will look comically large. Worth it.

I have mostly adjusted to the keyboard (with the exception of some function key
combos) and will now press <kbd>Caps-Lock</kbd> + <kbd>→</kbd> to go to the end
of a line (among other combos). I bought my wife an Acer Chromebook 14 to get
her the same keyboard layout so I no longer mistype when using her notebook.
Funny, but true. The touchpad is a tad finicky at times, but doesn't detract
from the overall very good experience.

The CPU has proven good enough for my usage, including a number of Rust builds.
Having more memory would certainly be good, but I haven't run into real
problems yet, so I'm fine with the 4GB I have. The disk is predictably 85% full
and I sometimes clear out projects I no longer use. I sometimes wish cargo
would garbage-collect libraries more aggressively.

Battery life hasn't measurably deteriorated, which is still one of the upsides,
despite the display taking more power than a Full HD one. Hats off to HP
engineers for packing so much power in such a tiny space.

Would I buy it again? The answer is a sound "yes". Sure, there are more capable
devices out there, but I cannot afford them anyway. Would I recommend it to a
friend? It depends. Perhaps if that friend is already a Linux user and happy to
work around the kinks in the general setup. Otherwise they're probably happier
with a Windows device anyway.
