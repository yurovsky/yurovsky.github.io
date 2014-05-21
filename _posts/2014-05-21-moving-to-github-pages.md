---
layout: post
title: Moving to github pages
---

I've tried a few 'blog' approaches over the years and haven't found what I was looking for, namely simple hosting 
with enough low-level editing capability to include code snippets and diagrams while having some kind of commenting system.

I first used Google's blogger service but the editor and syntax were terrible and nothing ever looked right.  I then
unfortunately tried to use [Google+](https://plus.google.com/u/0/+AndreyYurovsky/posts) as a blog and that definitely doesn't work -- its editor is awful and it's extremely
feature-limited, all you can really do is bold things.


That brings me to this, github's pages feature and [Jekyll](http://jekyllrb.com/) for static-content blogging.  Now I can include code snippets,
diagrams, links, and whatever, I can use disqus for comments, and I can control the look and feel while managing the whole
thing in git and editing with vi or online via github's built-in text editor.  I used [poole](http://getpoole.com/) to set things up so I didn't even have to deal with ruby stuff.  Awesome!

### GNU tools

I dug up a couple links to potentially useful posts on Google+, please check these out if you're working on MCU firmware
using GNU tools:

* [Using newlib-nano for MCU firmware](https://plus.google.com/102918008620512795495/posts/XUr9VBPFDn7)
* [Using ARM semihosting with GNU tools](https://plus.google.com/102918008620512795495/posts/5rupuziHKGC) ...this is where I foolishly tired using Google+ to write technical stuff.

### openocd improvements

I've been involved in a few openocd improvements over the past year or so,

* You can now use the STLinkv2 adapter to trace (via the SWO pin in SWD mode) on Cortex M3, M4, and M4F microcontrollers -- [Google+ post about it](https://plus.google.com/102918008620512795495/posts/8kev5pwiJPT) it's a simple implementation for now using port 0 and logging directly to a file.  I started wirting a simple parser for that file, it's called [swo-tracer](https://github.com/yurovsky/swo-tracer).
* openocd supports CMSIS-DAP debuggers and that includes Atmel's EDBG adapter which is built into their Xplained Pro development kits for Cortex-M microcontrollers -- [Google+ post about it](https://plus.google.com/102918008620512795495/posts/5JTehC7ngTq) ...I added support for the very nice low-power SAM4L Cortex-M4 MCU (scripts, Flash driver, etc.) as well as the low-cost new SAMD20 and SAMD21 MCUs and their respective development kits.  Give it a try!
* I made a minor change to openocd's JLink driver and now JLink-OB (that's onboard) adapters work in openocd -- [Google+ post about it](https://plus.google.com/102918008620512795495/posts/ZcbBGW1Kwpa) ...this lets you use the debugger included on various vendors' development kits.  The change is simply a different USB PID and different endpoint addresses, it turns out the onboard adapters are otherwise identical.
* I took a first stab at adding support scripts for TI's TMS570 Cortex-R4 MCU.  TI have made it impossible to write a proper Flash driver but at least there are now scripts for debugging in openocd, plus Seth Laforge from Google did awesome work on [making the TMS570's BE32 mode work properly](http://openocd.zylin.com/#/c/2064/) as it was a gray area in ARM's ABI and TI took some liberty with how it's implemented.

### Next

I have a few things planned with regard to openocd.  My TODO list includes:

* adding SWO trace support for Atmel's EDBG adapter.  This is actually quite easy but now that there are two trace-capable adapters (STLinkv2 and EDBG), I need to find a reasonable way to integrate general "tracing" functionality into openocd and hook them up to that.
* helping with the TMS570-related changes that are coming in.
* helping to test the SWD reorganization changes and other ARM stuff, including an attempt at supporting SWD on JLink adapters.
