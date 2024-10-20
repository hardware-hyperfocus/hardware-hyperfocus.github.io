layout: post
title: "Figuring out kasa switches"
date: 2023-10-17 10:00:00 -0000
categories: kasa-tapo

# What?

Soo I have a few Kasa and Tapo smart dimmers. Thay are good and cheap, but what's missing for me is an OSS firmware with local-only fulfillment.

Let's try to get esphome on it!

# Starting the journey

After googling around, I wasn't able to find any info on the hardware in these, so let's open one and see for ourselves!

Ta-da-da-dam... that's a RTL8720CM! It's (mostly) supported in esphome thtough libretiny, so that's cool!

# How does it actually work?

Well idk, let's reverse it and figure out!

Took some pictures, cropped and stretched them to roughly fit the actual dimentions of the board. Though Google seems to think otherwise, KiCAD has a feature to insert images in the PCB editor now, so let's start from there!

I've added custom symbols/footprints for the two ICs and got to work tracing the PCBs from the photos. This went surprisingly smoothly, even if it took me a shit ton of time due to my lack of experience.

This gives us a look at what's going on! The main MCU, RTL8720CM, is doing most of the heavy lifting: connectivity, buttons, lights - you name it. It's connected to the other one, CMS8S58BD, via UART0, and that chip handles the control of the actual dimmer cirquitry (triacs?) and output to the dimming level LEDs. 


