---
layout: post
title: "Hacking Ikea lights"
date: 2025-06-29 10:00:00 -0000
tags: ikea pelarboj iskärna esphome 
---

Doing something other than the dreaded Kasa/Tapo dimmers for once! This one should be a piece of cake in comparison.

P.S. You can find the project files for this at the bottom of the page!

# The premise

I have bunch of RGB lights plugged into Home Assistant via esphome. I _also_ have a couple fun lights from Ikea. This one's Enrique, he's  an Ikea ISKÄRNA and my Quest stand:


![Enrique](/assets/images/2025-06-29-hacking-ikea-lights/enrique.jpg)

Let's crack him open and see if we can wire him into system!

# The guts

Here's what's inside of Enrique:


![The guts...](/assets/images/2025-06-29-hacking-ikea-lights/enrique_open.jpg)
![...and the brains](/assets/images/2025-06-29-hacking-ikea-lights/enrique_pcb.jpg)

That little `U1` IC is just marked `wdp80`, and googling it just pull sup other people with Ikea lights 😁 So I guess I'll just invest some extra time and reverse the whole PCB - maybe it cold be useful in the future. 

In good news, the other light I have (Ikea PELARBOJ, this pencil looking thing) has the exact same PCB. The U1 there is marked `F1572-1` - and that's a Microchip MCU that I can actually find a datasheet on.

![The pencil's guts](/assets/images/2025-06-29-hacking-ikea-lights/pencil-guts.jpg)

# Tracing the PCB

I took some photos of one of the PCBs and loaded them into KiCAD. A few hours later, I have this mess:

![Traced pcb](/assets/images/2025-06-29-hacking-ikea-lights/pcb_traced.png)
![Schematic](/assets/images/2025-06-29-hacking-ikea-lights/schematic.png)

I'll admit I don't get what some of this (Q4?) does, but all I need to know is the power and logical connections for the MCU anyway.

# The mod

So what I want to do is pretty simple:
1. Wire my own MCU to the 3v3 power line
2. Sever the logical connections from the original MCU - don't want that thing interfering
3. Wire them up to the new MCU.

Sounds easy enough! I'll start with power just to make sure my controller can power up correctly. I did measure the voltage at 5v (not 3v3), so I'll wire to the VBUS of my XIAO ESP32C6, not directly into the 3v3.

...and, well, looks like the power regulator is not powerful enough - the board just keeps bootcycling. I tried knocking R1 off the board in hopes that Q4 serves as some sort of current limiter - and that didn't help either. 

I assume the most power-hungry oart has to be the onboard radio (since I haven't really connected anything to the MCU yet). Removing the network part of the esphome config seems to confirm it - now the led flashes as expected. Adding `power_save_mode: HIGH` instead doesn't seem to help, and neither does `output_power: 8.5db`. Doing both _and_ disabling the blinking LED seems to do the trick though - I am finally able to get the logs wirelessly from the chip (sometimes). Welp, that's not good!

So I tried switching gears and powering the XIAO with an external power source for now to see if the 3v3 logic level outputs would even work to drive the MOSFETs. And they did! That's one piece of good news. Maybe I can just use a separate voltage regulator to power the mcu straight from the 24v rail - I have some lying around...

...aand they only support up to 18v on the input. Looking into the datasheet for the XC6203 regulator on the original PCB, it's not really well suited for such a large stepdown eiher: `IOUT ≦ Pd / (VIN-VOUT)` and `Pd = 300mW (1500mW PCB mounted)` only gives us about 79mA before it overheats. From my googling, it looks like the XIAO can spike up to 650mA with the radio active. Crap. I even tried slapping a heatsink on that little thing, and it didn't really change anything.

So I so need a separate, switching voltage regulator. I don't have a reasonably-sized one on hand, but this monstrocity will do for a test:

![Regulator](/assets/images/2025-06-29-hacking-ikea-lights/regulator.png)

In fact, both lamps even have enough space for me jam all of that inside! I guess that's a wrap then 😁

Here's the [KiCad project](/assets/project-files/ikea-lights.zip) for reference!
