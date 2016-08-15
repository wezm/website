+++
type = "blog"
author = "PulkoMandy"
title = "Contract weekly report #52 - On to beta1!"
date = "2014-11-14T08:52:27.000Z"
tags = ["contract work", "release"]
+++

Hi,

This week I continued work on moving Beta1 forward, fixing some important and less important bugs. To make things clear about what to expect in the upcoming weeks, I will spend more time on Beta1 tasks, but I'll also continue working on WebKit. However, my work there will focus on fixing bugs, rather than adding new features.
<!--break-->
So, this week I continued reviewing the beta1 and R1 tickets. There were still some low hanging fruits that I solved last friday and during the weekend. I fixed the docbook toolchain we still use to build the Haiku Interface Guidelines. This was broken for a long time because it uses an old copy of libxml in the Haiku tree. What I did is merging a patch there to make it work with a current zlib, so things work again. We should probably get rid of the whole thing and require people willing to build the documentation to install docbook on their system - which is easy now, even on Haiku as I did packages for that. It would be nice to see some activity on updating the HIG again. It's getting old and we have changed our rules a bit over time to accomodate for larger and higher resolution displays, and modernize the UI a bit. We have also drifted away from BeOS in some subtle aspects. And the HIG could be made more easily browsable and styled to the Haiku theme like the user guide or API docs are. For the record an HTML version is online here: http://api.haiku-os.org/HIG/

I updated the Mesa package to fix a BeOS compatibility issue. libGL was not linked to libbe, and when running old BeOS apps you would often get an error that 'be_bold_font' could not be found. Puckipedia did all thehard work on investigating the problem during BeGeistert, all I had to do was fix and rebuild the Mesa package. It is for example possible to run Pixel on Haiku now.

I also fixed various other small issues, such as a localization problem in AboutSystem, getting the "enable on startup" checkbox in Notifications to work, improving the selection/zoom algorithm in Mandelbrot to be easier to use, fixed several drawing glitches in the MAcDecorator, fixed the build of the bluetooth stack, updated VL-Gothic fonts to the current version.

During the weekend and some evenings as well (so this isn't counted in my contract time) I helped miqlas get Homeworld running on Haiku. This appears to be a rather nice game from 1999 which is now open source. There were two problems, one was a broken libsdl_x86 package that led to the game rendering in a 32x48 pixels area in a corner of its window, and the other being a compilation flag used in Homeworld leading to structure padding problems and a crash.

Finally, I did some embedded development this week with an STM32 development board, and this led to the upload of an STM32flash package, as well as adding some scripting support to SerialConnect so it can be used in collaboration with stm32flash. Nothing really useful to the general public here.

So, that was a busy week-end. Unfortunately the week wasn't as productive, but here is a list of things I spent my contract time on this week:
<ul>
<li>Terminal now accepts the % character as a part of URLs when using the command+click feature. This was annoying me a lot when trying to access the Trac URLs I got in my mailbox (I'm reading mails with mutt in Terminal). While I was working on Terminal, I also reviewed the way settings work there, and moved all the persistent settings to the settings window. There are still some available for quick-access in the settings menu, but they are not saved to disk anymore. the settings window also now has the standard Defaults/Revert buttons instead of the previous MS-Windows-like OK/Cancel. Finally, Terminal will now make sure its windows open inside the screen, which wouldn't work before if you had changed resolution to a lower one. I also made a small cosmetic fix to the teminal tab bar.</li>
<li>The network preferences received some fixes as well. "/dev/net/" is removed from the interface names to make things less verbose, and switching betweeh the Interfaces and Services tabs now works as expected. There are still some known bugs with this preferences panel and I'll get back to it.</li>
<li>I improved the PackageInstaller (this is the app to install BeOS PKG files) to handle some more relocated directories. We rewrite scripts used by PKG files on the fly to make them put things in the correct non-packaged directories. Some cases were missed previously and are now handled, so more packages should install.</li>
<li>I fixed a small issue in the bfs tools where they would crash if given a path that can't be resolved to a device, for example when running "bfs_info /". No harm was done but now you get a better error message in that case.</li>
<li>OpenSSH was updated to a more recent version. The main change is this fixes the script that generates ssh keys on first boot.</li>DiskProbe user interface was rewritten to use the Layout Kit. This fixed a bug, and introduced another UI glitch, which I'll fix later today.</li>

And finally for the last 3 days I started work on some long-standing issues in our CD code. We have several tickets reporting that ripping audio CDs will often crash the system in many unpredictable ways. There seem to be a problem in the CDDA file system. I tried to have a look at this, but I discovered more issues than I solved.

The first audio CD I inserted immediately triggered a KDL. After enabling some tracing options in the CDDA and CD session code I found this to be because there was an invalid session description on the CD. This seems to be done on purpose as a copy protection scheme. Haiku would think there is a partition starting at sector -1 on the CD, which by itself is fine, except that trying to read sector -1 will always fail. And it did, when Haiku started to identify the content of that partition.

So, I fixed this first issue and I could finally read my CD. But the bug reports were right, after copying some CDs I could esily trigger KDLs in various apparently random places. There is probably a memory corruption problem happening here, with the CDDA code accessing memory it's not supposed to, and changing things in memory, leading to crashes in unrelated parts of the system. This means it's hard to see exactly what happened from the kernel debugger.

So, I did a quick code review of the CDDA code. I fixed issues pointed by coverity, but these were unlikely to have any harmful consequences. I also found a case where CDDA would overrun a buffer, which I fixed. But I still could trigger the panics after that, so what I fixed was not the thing causing all our problems.

While investigating problems in the kernel debugger I made use of the dis (disassemble) command. This was introduced in 2008 and allows to disassemble the code in memory to look at it in a more readable form. However it did not do any symol lookup, and would print addresses as hex values. You'd then have to use the ls (lookup symbol) command to find what's at the address. This works but gets boring very quickly. So I updated our copy of udis86 - the library we use for disassembling code - , and used the new feature in the current version to perform symbol lookup directly. Now the dis command shows the symbol directly, making the assembler code more readable and easier to correlate to the corresponding C or C++ source code. It's easier to keep track of things, at least.

Next I tried enabling the guarded heap. The guarded heap is an "extreme" debugging tool, it will allocate memory so any out of bounds access is immediately detected and triggers a panic. This is similar to the allocator used for userland in libroot_debug. However, the fact that it runs in kernel land needs a different implementation. Well, the guarded heap code has not been used for a while, so first of all I had to get it building again. I then trying booting Haiku with it, and I was greeted with an "Out of memory!" panic. The guarded heap is implemented in a way that wastes a lot of memory (it allocates a whole 4K page even for the smallest allocations (1 or even 0 bytes) and also wastes a lot of addres space (each of the allocated pages is followed by a 4K memory hole, which allows to catch the out of bounds accesses). It used to be possible to boot haiku in those conditions, but the package manager needs more kernel memory, so this doesn't work anymore. There is a way out, however: I will try this with a 64-bit system, where the address space is much bigger and shouldn't run out as fast. I hope this will boot far enough to test the CDDA code with the guarded heap. If that also fails, I will have to get CDDA running in userland_fs so I can test it in userland side with libroot_debug, without having to use the guarded heap. And even after all this, I don't know if I will be able to catch the issue, because the corruption may as well happen on stack memory, rather than on the heap. Well, running with the Guarded heap may uncover issues in other parts of the kernel code anyway, so it's always worth a try.

While testing this I also discovered more problems. Some of my audio CDs have an extra data track, with bonus materials (video clips, for example). The ISO9660 filesystem will manage to mount those, but then any attempt to access the files (just doing ls or opening the volume in Tracker) will hit KDL again. It turns out our handling of this format isn't quite right yet, and there is a mixup in the handling of sector numbers. The ISO9660 filesystem uses offsets relative to the start of the disk, but we interpret them relative to the start of the session. We don't run into problems with standard single-session discs, but in multi-session ones we try to read sectors far after the end of the disc, which of course fails.

I wanted to do more testing of mixed audio/data CDs, and I tried to read some old game CDs which had the data track first, followed by the audio ones. While we have some code to handle this, I couldn't get it to work, and only the audio tracks are visible in Haiku. I haven't researched that part deeper, yet, but it's now also on my list of things to fix for proper CD handling.

So, these investigations in the CD code didnt lead to a fix yet, and uncovered some new problems. That's still progress, but it doesn't really get our Beta1 any closer.

Plans for next week include investigating the remaining problems in MediaConverter, so it doesn't truncate the end of audio files when converting them, and maybe trying to get it playing well with the ffmpeg ogg encoder again so we can enable that in the release. This has been broken for a long while and it's even hard to tell when exactly it stopped working. But it would be great to get it fixed, still.