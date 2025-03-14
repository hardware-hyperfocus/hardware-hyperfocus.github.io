---
layout: post
title: "Dissecting the secondary chip protocol - pt.2"
date: 2024-11-12 10:00:00 -0000
tags: kasa tapo ks225
---

Part 1 [here]({% link _posts/2024-10-20-dissecting-secondary-chip-protocol.md %}).

# Moving forward

After the spectacular failure of my last attempt, I needed to think of a better way to sniff the inter-chip-comminication while actually connected to the mains. 

The best thing I could think of would be to eliminate any kind of wired connection - so I'm thinking of something like this:
![concept illustration](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/concept.png)

Both RX lines on the external MCU can be then wirelessly logged somewhere. This would be pretty easy to do with another MCU flashed with esphome for example, since it supports uart logging in a pretty simple way.

# But...

I've looked into my little stash of mcu boards, and here's what I'm working with here:
![MCU stash](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/mcu-stash.jpg)

I have no ESPs or Pi pico W's left! There's not much, and the only wireless boards here are the Ardino Nano RP2040 Connect and the Nano 33 IOT. Neither are supported by esphome, so we'll need to use something more custom I guess...

*While writing this, I suddenly realized that I also ahve a few SBCs that could be used as well! It would be very versatile if it worked, but... I've had tons of issues with UART on Raspberry Pi before, even just enabling it can be tricky, so let's stick to the Arduinos for now*

# Let's figure it out!

All right, so what I can do is just install the Arduino IDE and whip out a quick sketch to dump Serial data somewhere over WiFi. Right? Maybe there's even an existing project I could use?

Well... Looks like the wifi chip on both of these is supported by the [WiFiNINA](https://docs.arduino.cc/libraries/wifinina/) library. It has some examples of HTTP servers (score!), and a UDP client/server (could also be a good idea), but if I'm going this route I'd like a web interface that keeps logging the serial data continuously.

For context, my primary background is in web dev, so I'm reasonably knowledgable in how to achieve that. I've looked into WebSocket implementations for Arduino, but they all seem to focus on a WebSockets *client*, not *server*. And I *really* don't want to implement the Websockets protocol myself: that seems like a huge hustle.

So maybe a better alternatives could be server-sent events? I've been looking for an excuse to use them for something for years, but for the things I do they never really fit well due to their unique downsides. But, they are *very* simple to implement in the server code!

Starting with the examples from WiFiNINA, I was able to throw together a simulated thing that just prints the same messsage over and over:

![first try loop](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/first-try-loop.png)
![first try handle client function](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/first-try-handle-client.png)
![receiving events](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/events.png)

And after some more poking around (and getting confused with the different frameworks that can be used with my Nano RP2040 Connect in Arduino), I can pipe UART into it!

![getting uart](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/with-uart.png)

Now, let's finally connect it to the switch and see what's up. I've soldered some wires to the signal contacts and reassembled the dimmer with them sticking out.

![soldered](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/soldered.jpg)
![assembled](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/assembled.jpg)

...aand I hooked up the wrong board. But once I connected the right one, it actually worked!

![first bytes](/assets/images/2024-11-12-dissecting-secondary-chip-protocol-2/first-bytes.png)

I'm not getting anythin on UART2, and I think the BAUD might not be right, but that's a start. I've tried switching the UART lines around, and was still only getting info on UART1. That's good news, since it means something's wrong with my code and not how I soldered the headers to the dimmer - I don't want to go poking around there again!

According to the RTL8720 datasheet the default UART BAUD is 115200, so let's try that one next. Over a few hours, I've updated the code to be able to update the baud on-the-fly (that was surprisingly hard, as the Arduino just kept crushing for seemingly no reason...). So I've tried all the standard bauds from the CMS8S5880 datasheet and still can't see anything to point to which one is right 🤷 Some rates seem more "stable" than others (i.e. periodic messages come with same content, not changing once in a while), but that's about it.

In that case, let's try to detect the actual baud! After a bit of searching, I found [a ready implementation](https://www.electronicsforu.com/electronics-projects/hardware-diy/uart-automatic-baud-rate-detector#:~:text=Once%20the%20arduino%20sketch%20is,one%20of%20the%20test%20results.) that I should be able to just plug into my sketch.

With a new button that sends a request for BAUD detection, we get about 111111 bps, which is pretty close to 115200! Seems like that's the BAUD to use after all.

# Actual dumps

With that in mind, here's the data I recorded after turning the dimmer on, increasing the brightness by one step and turning it off again:
```
UART1: 5a5a00
UART1: 000401000000371a
ping!
UART1: 5a5a00
UART1: 000401000000371a5a5a000b000401188da71aae5a5a000b0004f70104c1efab5a5a000b000401188da71aae5a5a000b000401170722da385a5a000b00040115183f23515a5a000b00040115183f23515a5a00
UART1: 000401000000371a
ping!
UART1: 5a5a00
UART1: 000401000000371a
ping!
UART1: 5a5a000b0004f7800044a4385a5a00
UART1: 000401000000371a5a5a000b000405117156acff5a5a000b0004f70105c0bf6b
ping!
UART1: 5a5a00
UART1: 000401000000371a
ping!
ping!
UART1: 5a5a00
UART1: 000401000000371a
ping!
UART1: 5a5a000b0004f7800044a4385a5a000b00040115cceb7c0e5a5a000b0004f70100c5eca85a5a000b00040115cceb7c0e5a5a000b0004011734113f6c5a5a000b000401186f45f3665a5a000b000401195f74e7e25a5a000b0004011a507812175a5a00
UART1: 000401000000371a5a5a000b0004011b200936a35a5a000b0004011bbc955fca5a5a000b0004011bbd940f
ping!
ping!
UART1: 5a5a00
UART1: 000401000000371a
```

And here's the same with the other header (green 😁):

```

UART1: 5a5a01
UART1: 00060100001c110025f5
ping!
ping!
UART1: 5a5a01
UART1: 00060100001c0f0085fc
ping!
UART1: 5a5a01
UART1: 00060100001c0e0015fd5a5a010b000601180d27000086a75a5a010b000677010441000091525a5a010b000601180d27000086a75a5a010b00060117072200005ee05a5a010b00060115183f00004c0e5a5a01
UART1: 00060100001c0d00e5fd5a5a010b00060115183f00004c0e
ping!
UART1: 5a5a01
UART1: 00060100001c0c0075fc
ping!
UART1: 5a5a010b000605117156000088335a5a010b0006770105400000ad025a5a01
UART1: 00060100001c0b0045fe
ping!
UART1: 5a5a01
UART1: 00060100001c090025ff
ping!
ping!
UART1: 5a5a01
UART1: 00060100001c0800b5fe5a5a010b0006770000440000607e5a5a010b000601154c6b0000ac5f5a5a010b000677010045000060125a5a010b000601154c6b0000ac5f5a5a010b0006011734110000151f5a5a010b000601186f450000e0195a5a010b000601195f740000ef7a5a5a010b0006011a50780000f8fd5a5a010b0006011b20090000e38a5a5a010b0006011b3c150000b54c5a5a01
UART1: 00060100001c070045fb5a5a010b0006011b3d140000891c
ping!
UART1: 5a5a01
UART1: 00060100001c0600d5fa
ping!
UART1: 5a5a010b0006770000440000607e5a5a01
UART1: 00060100001c0600d5fa
```

Thanks to the photos from before, I know that the yellow wire (first dump) goes to brown wire on the inside. So, the first dump is data going from the RTL to the CMS, and the second one is the reverse! Now we can finally start digging into these, but that's another post 😁

Part 3 [here]({% link _posts/2024-12-20-dissecting-secondary-chip-protocol-3.md %}).
