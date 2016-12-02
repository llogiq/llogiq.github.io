---
title: HP Chromebook 13 – First impressions
---

After my previous IdeaPad broke down slowly, and attempts to repair it remained fruitless, I resolved to get a new
notebook. It should be smaller and lighter, have at least a Full HD screen, a keyboard that's pleasant to type on and
not too loud, and a CPU that's roughly on par with my previous Core i5-4200U. I planned on spending in the 500€ range.

After some consideration I decided to extend that range by 130€ and get myself an HP Chromebook 13 G1. First, I'm a
sucker for HiDPI displays, the keyboard, though quite sparse on keys, is really good for the price range and the Core
m3-6Y30 CPU has made itself a name powering the Apple MacBook, being sufficient for most applications – and in
benchmarks showed to be on par, at times even slightly faster than the older Core i5. Not too shabby for a CPU that
runs in a 4.5W power envelope.

That doesn't mean there aren't downsides apart from the aforementioned dearth of keys (for example, there's no Home/End
nor PgUp/PgDn buttons, no Del key nor any Lock keys (the key in the usual CapsLock position is redefined as Fn or
"search" key). Pressing the search key in conjunction with the cursor keys will emit `Home` / `End` / `PgUp` / `PgDn`
presses, which crouton will also do if you install the `keyboard` target, so it's not that bad. Also at least, there's
Escape. Take that, Apple!

Disk space is severely restricted at 32GB (of which roughly 6GB are claimed by the system, the rest is for personal
use), but as I don't store too many 4K movies on internal memory, I'll be fine. Ports are also quite restricted, so I
got a Dell travel dock giving me HDMI, VGA and Ethernet – otherwise two USB-C, one USB-A (3.0), one headphone jack and
one microSD card reader are all I get. The dock should allow me to present slides on my next talks and also connect to
Ethernet. I scarcely had any use for rotating media, so the lack of DVD drive doesn't concern me at all.

First, the ChromeOS sets the display geometry to 1600x900, half of the actual resolution. This is a bit surprising, but
perhaps Chrome does not do too well with High DPI displays? I *can* set the resolution to native, but that makes the
fonts unreadable and the icons too small to click. Given that Google has renewed the Pixel line of Chromebooks, they'll
probably do something for those great displays sooner or later. I've yet to install a dev version of ChromeOS, maybe
it's better with that.

**Update**: I've been reliably informed that the resolution setting just changes the size of the UI controls. Looking
at the screen really closely, this appears to be the case. Sorry, the setting is weird, and I didn't wear my glasses
when I first wtote this.

Regardless the surprise, I'm using the Linux part of the equation to 99% anyway. Setting the xDPI value in the XFCE
settings editor to 180 brought me acceptable font and icon sizes. However, those apply to external displays, too, which
means fonts on HD displays will look comically large. I also needed to add `-Dswt.autoscale=175` to my Eclipse Neon's
`eclipse.ini`. There's some room for improvement here, but on average, it works out quite well, and with 3200×1800
pixels, fonts look simply great.

Bringing the Chromebook into a corporate network proved to be a bit of a hassle – I couldn't get the EAP-TTLS / CHAPv2
tunneling working, so I'm on Ethernet for now. This is something that works out of the box on both Linux and Android,
so I'm a bit puzzled why it fails to run here. Googling the error was fruitless so far. Perhaps some ChromeOS luminary
could step in to help.

**The Verdict**

OMG THIS IS THE BEST PIECE OF TECH I'VE EVER HAD!!! Or, if you prefer the low-key version, if you are a developer
outside the Apple filter bubble and don't have too much data to store on disk, the HP Chromebook 13 G1 is a *very*
compelling offer. Sure, it may be expensive for a Chromebook, but compared to other systems capable of running Linux
with roughly similar specs, it's a steal.
