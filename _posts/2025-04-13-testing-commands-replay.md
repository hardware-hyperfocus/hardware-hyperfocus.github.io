---
layout: post
title: "Testing commands replay"
date: 2025-04-13 10:00:00 -0000
tags: kasa tapo ks225 esphome
---

Previous post in the series [here]({% link _posts/2025-03-10-dissecting-secondary-chip-protocol-4.md %}).

# Aaand we're back (again...)

You know the drill: it's been way too long again, so I'm returning to this project for a bit. This time, hopefully, I should be able to build and upload a test firmware to try and replay some of the commands that I've sniffed before.

I already had flashing the chip figred out a long time ago, but coming back to it was jarring. I spent quite a bit of time to make it flash properly again, and even then - it was just bootlooping for some reason. Hours later, after trying everything I could, I think I finally know the answer - it's a power issue. It's still bootloping when powered by the FTDI, but it works just fine when assembled!

That made me realise a big problem for this kind of incremental flashing; libretiny doesn't support OTA for the RTL8720CF/CM. I dreaded the idea of having to implement it myself, but thankfully I don't have to! There's already [a great PR](https://github.com/libretiny-eu/libretiny/pull/302) from [@prokoma on Github](https://github.com/prokoma)  to enable it. Worked like a charm for me!

I do (of course!) immediately run into another issue: the UART won't work. I'm getting `Component uart is marked FAILED` in the log - with no additional info whatsoever. It seems to work fine if I use UART1 (pins 15/16), but, of course, that's not what the secondary chip is connected to. It's time to look into the code then!

# Looking into the code

One thing I've noticed in the logs is that the UART gets created with `type: software`. That may suggest that esphome fails to iniialize the hardware serial for these pins. Browsing the libretiny repo, it looks like the relevant pin definitions are in `boards/variants/generic-rtl8720cf-2mb-992k.h`. Notably (and as [this Github issue notes as well](https://github.com/esphome/issues/issues/6909)), there's multiple pins defined here for `SERIAL0` and `SERIAL1`.

Looking at `cores/realtek-ambz2/arduino/libraries/Serial/Serial.cpp`, I don't really see any support for multiple possible pins defined for the different hardware UARTs. It looks like, instead, the Serial instances are just created statically with the singular defined pins (e.g. `PIN_SERIAL0_RX, PIN_SERIAL0_TX`). So maybe just renaming the macros for the pins I want to use could serve as a quick fix for my particular use case?

I've forked @prokoma's repo and did [a quick test of that approach](https://github.com/hardware-hyperfocus/libretiny/tree/test) - and it worked! I have a working UART in esphome now!!

# New data

So now we're ready to try replaying commands, right? Not so fast. I mean, yeah, I can send commmands all right, but they don't do anything. In fact, the secondary chip just keeps spamming the same byte sequence over and over, and it's something I haven't seen before:
```
5A:5A:02:08:00:06:43:4D:53:02:00:00:74:42
```

Notably, even the prefix is now different. We've seen `5A:5A:00` and `5A:5A:01`, but what the hell is `5A:5A:02`?? I went on a sidequest here and tried to look for an existing messaging protocol that would be using `5a5a` as a separator. And, well... it's most of them I think. `5a5a` or `a5a5` are just alternating ones and zeros, so they are convenient to use as a separator for UART I guess.

My hunch is that this could be some kind of a boot sequence/auth thing, so let's test that theory! I've re-flashed the original firmware dump and tried resetting the dimmer while the logger MCU is active. Sure enough, we do get the 02 thing:

```
[10:01:19][D][uart_debug:114]: >>> 00:0A:00:04:01:00:00:00:37:1A:00
[10:01:22][D][uart_debug:114]: <<< 01:0A:00:06:01:00:00:1C:00:00:75:F9:00:FE:5A:5A
[10:01:22][D][uart_debug:114]: <<< 02:08:00:06:43:4D:53:02:00:00:74:42:FE:5A:5A
[10:01:22][D][uart_debug:114]: <<< 02:08:00:06:43:4D:53:02:00:00:74:42:5A:5A
[10:01:22][D][uart_debug:114]: >>> 0F:00:00:FE:FE:5A:5A
[10:01:22][D][uart_debug:114]: <<< 01:08:00:06:43:4D:53:02:00:00:7B:B2:5A:5A
[10:01:22][D][uart_debug:114]: >>> 00:08:00:01:00:F0:27:5A:5A
[10:01:22][D][uart_debug:114]: <<< 01:0B:00:06:72:19:2C:74:00:00:68:6B:5A:5A
[10:01:22][D][uart_debug:114]: >>> 00:0B:00:04:F2:19:AC:74:53:95:5A:5A
[10:01:22][D][uart_debug:114]: <<< 01:0B:00:06:01:19:2D:06:00:00:8C:C1:5A:5A
[10:01:22][D][uart_debug:114]: >>> 00:0B:00:04:01:19:AD:86:02:26:5A:5A
[10:01:24][D][uart_debug:114]: >>> 00:0B:00:04:F7:00:07:C3:1E:7B:5A:5A
[10:01:24][D][uart_debug:114]: <<< 01:0B:00:06:77:00:07:43:00:00:D5:CE:5A:5A
[10:01:26][D][uart_debug:114]: <<< 01:0A:00:06:01:00:00:1C:03:00:85:F9:5A:5A
[10:01:26][D][uart_debug:114]: >>> 00:0A:00:04:01:00:00:00:37:1A:5A:5A
[10:01:27][D][uart_debug:114]: <<< 01:0A:00:06:01:00:00:1C:02:00:15:F8:5A:5A
[10:01:27][D][uart_debug:114]: >>> 00:0A:00:04:01:00:00:00:37:1A:5A:5A


```

...aand yeah, looks like I was confusing the direction of the communication this entire time. So what I though was requests was in fact responses and vice versa. Damn.

That explains a lot though! After some more back-and-forth and testing, I was finally able to actually do something! The following command did - as expected - light up the dimming indicator at the 40% state:
```
00:0B:00:04:F7:01:04:C1:EF:AB:5A:5A
```

Wooo 🥳 

...and, apparently, there goes my motivation (writing this two months later 😥). See you in the [next post]({% link _posts/2025-06-08-building-the-protocol.md})!
