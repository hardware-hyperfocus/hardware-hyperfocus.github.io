---
layout: post
title: "Dissecting the secondary chip protocol - pt.1"
date: 2023-10-20 10:00:00 -0000
tags: kasa tapo ks225
---

# What?

Now I can flash stuff onto the chip, and things actually work! I can blink LEDs and stuff! 

The only thing that's missing is... well, the actual core functionality of a dimmer switch: switching and dimming the light. That's done by the secondary chip, so let's try to reverse-engineer the protocol to communicate to it.

I've re-flashed the stock firmware and confirmed that it works (or at least the networking, onboard leds etc all work).

# First attempt

So I've tried to just probe the UART pins with a wire and sniff on the connection! I've chosen the BAUD at random for now, so we're going with 960000.

![probing](/assets/images/2023-10-20-dissecting-secondary-chip-protocol/probing.jpg)


Aand we have a problem. It looks like the onl communication that happens is sending the same bytes every second, regardless of what I do with the switch and dimmer buttons. yikes.

![protocol sniff](/assets/images/2023-10-20-dissecting-secondary-chip-protocol/protocol-sniff.png)

But I do have a clue I think. The dimmer level LEDs also stay off when the PCB is not connected to the high-voltage part. So maybe there's some kind of status sensing that enables or disables the actual data transfer between the two?

## Please don't do this. Seriously, don't.
![janky mains setup 1](/assets/images/2023-10-20-dissecting-secondary-chip-protocol/jank-1.jpg)
![janky mains setup 2](/assets/images/2023-10-20-dissecting-secondary-chip-protocol/jank-2.jpg)

Off to electrocute myself then!

...all right, so I definitely burned the FTDI converter and maybe burned the MCU itself and the USB port on my monitor. Fun! 

P.S. After some time, the USB hub actually started working again (with the exeception of the port that the FTDI was plugged in). Hooray!

# Post-mortem

I've probed around the hardware to figure out what happened and... well... the **5v line is straight up referenced to the line contact** of the dimmer. Now, I didn't have the 5v line hooked up, since I wouldn't want to backfeed 5v back into the USB port... But since GND is now "mains - 5v", I don't think that was enough.

I'll have to think of something else next time!