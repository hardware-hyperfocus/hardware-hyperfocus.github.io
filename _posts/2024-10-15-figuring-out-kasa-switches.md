---
layout: post
title: "Figuring out kasa switches"
date: 2024-10-15 10:00:00 -0000
tags: kasa tapo ks225
---

# What?

I really like the convenience of having smart dimmers installed throughout the house. Killing the lights as you leave or go to sleep with a quick voice command is noice.

Over the years (and moving between apartments) I got to try a bunch or cheap-to-midrange smart dimmers, and most had some big downsides. So far, the Kasa/Tapo dimmers have been the bast at tactile feel, connectivity stability, and dimming performace (read: not trying to trigger someone's epilepsy every time the dimmer is below 100%). They are pretty cheap to, as far as smart dimmers go!

One small problem though: I'd really like to flash some custom firmware on these.

# Why?

A few reasons:
* The firmware does not support dimmer callibration from third-party apps. You can't set it through Matter for models that support it either. It *must* be added to either the Kasa or tje Tapo app (and, conequently, TP-Link's cloud services) to do that.
* It's pretty hard (though not impossible) to block the dimmers' access to the Internet - they are known to stop responding to local commands, or at the very least start blicking the red LED, which is pretty annoying.
* One thing I don't like about these is the debounce on the main paddle. It just won't register faster clicks, so it you want your light to turn on or off - you need to press it slowly and with purposefully 😁
* I'm bored, apparently 🤷

# Starting the journey

After googling around, I wasn't able to find any info on the hardware in these, so let's just open one and see for ourselves! The usual technique of shoving an old plastic card into the gaps until it opens seems to work, though the clips are pretty stiff and hard to work apart:

![opening the device](/assets/images/2024-10-15-figuring-out-kasa-switches/opening-the-device.jpg)

There's two PCBs inside: one for the high-voltage components and one for the logic, wifi etc:

![opening the device](/assets/images/2024-10-15-figuring-out-kasa-switches/two-PCBs.jpg)

Looking closer at the logic board, it seems to be run by an RTL8720CM. Thankfully, it's (mostly) supported in `esphome` through `libretiny`!

# How does it actually work?

Well idk, let's reverse it and figure out!

Took some pictures, cropped and stretched them to roughly fit the actual dimentions of the board:


![pcb top](/assets/images/2024-10-15-figuring-out-kasa-switches/pcb-top.png)
![pcb bottom](/assets/images/2024-10-15-figuring-out-kasa-switches/pcb-bottom.png)

Though Google seems to think otherwise, KiCAD has a feature to insert images in the PCB editor now, so let's start from there!


I've added custom symbols/footprints for the two ICs and got to work tracing the PCBs from the photos. This went surprisingly smoothly, even if it took me a shit ton of time due to my lack of experience. I'm not aiming to trace everything or deduce actual values for the components on the board, jut try to figure out the bits that are important for my needs:

![pcb traced](/assets/images/2024-10-15-figuring-out-kasa-switches/pcb-traced.png)
![partial schematics](/assets/images/2024-10-15-figuring-out-kasa-switches/partial-schematics.png)

This gives us a look at what's going on! The main MCU, RTL8720CM, is doing most of the heavy lifting: connectivity, buttons, lights - you name it. It's connected to the other one, CMS8S58BD, via UART0, and that chip handles the control of the actual high-voltage dimmer cirquitry and output to the dimming level LEDs. We also have test points going straight to the important bits (i.e. UART1, GND, 5v and 3v3)!

Using an FT232RL serial dongle and lpchiptool, I was even able to dump the original firmware from the chip! Let's stop here for now then.
