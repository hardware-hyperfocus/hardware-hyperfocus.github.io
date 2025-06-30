---
layout: post
title: "Dissecting the secondary chip protocol - pt.4"
date: 2025-03-10 10:00:00 -0000
tags: kasa tapo ks225 esphome
---

Part 3 [here]({% link _posts/2024-12-20-dissecting-secondary-chip-protocol-3.md %}).

# Aaand we're back

It's 1:30 AM on Monday, it's been three months, so let's take another crack at it.

What was the problem last time? I think _maybe_ it was that the data I was static. Just looking at those couple of logs - what's the fun in that? So this time let's get some more interactivity going here.

# A different logger

Since my last attempt I got some more stuff to play with, including some XIAO ESP32-C6 modules. So one thing I could do for more interactivity is switch to one of those, and set up the UART logging through `esphome` this time. Hopefully, that will give me more data to work with!

So, for the next four hours or so I've been playing with that and landed on a config like this:

```
esphome:
  name: logger
  friendly_name: logger

esp32:
  board: esp32-c6-devkitc-1
  variant: ESP32C6
  flash_size: 4MB
  framework:
    type: esp-idf
    version: 5.3.1
    sdkconfig_options:
      CONFIG_ESPTOOLPY_FLASHSIZE_8MB: y
      CONFIG_BT_BLE_50_FEATURES_SUPPORTED: y
      CONFIG_BT_BLE_42_FEATURES_SUPPORTED: y
    platform_version: 6.9.0

external_components:
  - source: github://PhracturedBlue/c6_adc

# Enable logging
logger:

ota:
  - platform: esphome
    password: "xxx"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

captive_portal:

uart:
  - id: uart_one
    tx_pin: GPIO17
    rx_pin: GPIO19
    baud_rate: 115200
    debug:
      dummy_receiver: true
      after:
        timeout: 100ms
      sequence:
        - lambda: UARTDebug::log_hex(uart::UART_DIRECTION_TX, bytes, ':');
  - id: uart_two
    # tx_pin: GPIO20
    rx_pin: GPIO20
    baud_rate: 115200
    debug:
      dummy_receiver: true
      after:
        timeout: 100ms
      sequence:
        - lambda: UARTDebug::log_hex(uart::UART_DIRECTION_RX, bytes, ':');
```

Things I've tried but didn't work out:
* Single-wire tx/rx UARTs - esphome doesn't seem to have a way to set that up
* Connecting RX or TX one at a time and doing a "replay" of a command - the replay didn't seem to do anything. I guess since the actual MCU is still connected there's some inteference going on there.

Things that worked out great:
* Logging both directions of communication - much more convenient!
* Better connection - I was getting tired of the finicky "wires-in-the-air" setup I had before. Thankfully, I found a spot inside of the dimmer where the XIAO just fits! So now I have the dimmer fully assembled (no more exposed live leads, yay!), and I can still connect to the xiao wirelessly.

![xiao on the inside of the dimmer!](/assets/images/2025-03-10-dissecting-secondary-chip-protocol-4/xiao-back.jpg)
![connections here](/assets/images/2025-03-10-dissecting-secondary-chip-protocol-4/xiao-front.jpg)

...and now it's alsmost 6AM so...

# We're back on the next night!

I've collected some logs and re-confirmed what we already knew:
* There's MCU-specific prefixes to each message: "5A:5A:01" for the "main" MCU and "5A:5A:00" for the "secondary" one
* There's some ping-style messages going back and forth every once in a while

Adding the prefixes as the logging separator will hopefully make it clearer:
```
      after:
        delimiter: [0x5A, 0x5A, 0x01]
```

...and it does indeed help! Now the commands are logged in the correct order, and each command is separated into a separate log. Now I can clearly see that:
* Each message is the same length - 11 bytes for "main", 9 for "secondary" (after the prefix)
* The "ping" happens every 1-2s and seems to include current time (?) in the main-side message

All of that allows us to idolate separate commands and start tracing the ones that go around at specific operations (e.g. on/off, setting brightness). Some patterns start to emerge (e.g. command prefixes), but a lot of the data still seems pretty weird. Here's an example:

```
On sequence:
// on??
  >>> 0B:00:06:01:18:0D:27:00:00:86:A7
  <<< 0B:00:04:01:18:8D:A7:1A:AE

// save dimming level?
  >>> 0B:00:06:77:01 : 07:42:00:00:D5:A2 - 100%
                       04:41:00:00:91:52 - 40%
  <<< 0B:00:04:F7:01 : 07:C2:1E:EB - 100%
                       04:C1:EF:AB - 40%
// same same
  >>> 0B:00:06:01:18:0D:27:00:00:86:A7
  <<< 0B:00:04:01:18:8D:A7:1A:AE

Off sequence:
//on??
  >>> 0B:00:06:01:10:5B:79:00:00:1D:36
  <<< 0B:00:04:01:10:DB:F9:80:91

// save dimming level?
  >>> 0B:00:06:77:01 : 00:45:00:00:60:12 - 0%
  <<< 0B:00:04:F7:01 : 00:C5:EC:A8 - 0%

//same same
  >>> 0B:00:06:01:10:5B:79:00:00:1D:36
  <<< 0B:00:04:01:10:DB:F9:80:91
```

There's a command that repeats in a lot of different operations. I'm assuming saves (or maybe just sets?) the brightness level. Here's the values used with it (seem to be stable between tries, reboots etc):

```  
  >>> 0B:00:06:77: 00 - off  :  04:41:00:00:91:52 - 40% on
                   01 - on      05:40:00:00:AD:02 - 60% on
                                06:43:00:00:E9:F2 - 80% on
                                07:42:00:00:D5:A2 - 100% on

                                00:45:00:00:60:12 - 0% (on?)

                                04:40:00:00:91:3E - 40% off
                                05:41:00:00:AD:6E - 60% off
                                06:42:00:00:E9:9E - 80% off
                                
  <<< 0B:00:04:F7: 00 - off  :  04:C1:EF:AB - 40%  on
                   01 - on      05:C0:BF:6B - 60%  on
                                06:C3:4E:2B - 80%  on
                                07:C2:1E:EB - 100% on
                                
                                00:C5:EC:A8 - 0% (on?)

                                04:C0:EF:3B - 40%  on
                                05:C1:BF:FB - 60%  off
                                06:C2:4E:BB - 80%  off
```

I can see the pattern of the first byte rising with percents - and being the same whether we are on or off - but the rest is magic to me so far.
It does make me think though that the brighness levels I can set with the dimmer buttons is fixed to just a few specific values, so maybe it would be beneficial to provision the dimmer and set specific values throught the app or Home Assistant. So next time we'll be doing that I guess!

# Getting more granular logs

It's a couple days later, and I've provisioned the dimmer in the tapo app now. Let's see what logs we get with the much more granular brightness support. I'll start with setting the brightness to each percentage value between like 45 and 51, while the dimmer is on. Here's what I get (kept only the bit that seemed relevant, the values in between send the same thing):

```
// 30%
[06:56:42][D][uart_debug:114]: >>> 0B:00:06:77:01 : 02:47:00:00:18:B2
[06:56:42][D][uart_debug:114]: <<< 0B:00:04:F7:01 : 02:C7:4D:28

// 33%
[06:53:58][D][uart_debug:114]: >>> 0B:00:06:77:01 : 03:46:00:00:24:E2
[06:53:58][D][uart_debug:114]: <<< 0B:00:04:F7:01 : 03:C6:1D:E8

// 51%
[06:38:10][D][uart_debug:114]: >>> 0B:00:06:77:01 : 04:41:00:00:91:52
[06:38:10][D][uart_debug:114]: <<< 0B:00:04:F7:01 : 04:C1:EF:AB

//68%
[07:01:00][D][uart_debug:114]: >>> 0B:00:06:77:01 : 05:40:00:00:AD:02
[07:01:00][D][uart_debug:114]: <<< 0B:00:04:F7:01 : 05:C0:BF:6B
```

So the granularity here is not great, and it _just so happens_ to match up perfectly with the LEDs at the top of the dimmer changing states 😁 I guess that's what this command actually does then. The first "data" byte even matches the number of LEDs that are lit up! Still no idea frat the rest is though, maybe some kind of checksum/parity stuff?

Let's look at the other command that goes across when chagning the brightness - would make sense if that's the actual dimmer level?

```
// 1%
[07:17:21][D][uart_debug:114]: >>> 0B:00:06:01 : 1B:3C:15:00:00:B5:4C
[07:17:21][D][uart_debug:114]: <<< 0B:00:04:01 : 1B:BC:95:5F:CA

// off from 1%
[07:18:26][D][uart_debug:114]: >>> 0B:00:06:01 : 1B:3D:14:00:00:89:1C
[07:18:27][D][uart_debug:114]: <<< 0B:00:04:01 : 1B:BD:94:0F:0A

// 1% when calibrated as high as it goes
[07:26:23][D][uart_debug:114]: >>> 0B:00:06:01 : 0A:34:0C:00:00:11:63
[07:26:23][D][uart_debug:114]: <<< 0B:00:04:01 : 0A:34:0C:30:3C

// off from 100% when 1% is calibrated as high as it goes
[07:27:36][D][uart_debug:114]: >>> 0B:00:06:01 : 0A:35:0D:00:00:2D:33
[07:27:36][D][uart_debug:114]: <<< 0B:00:04:01 : 0A:35:0D:60:FC

// 50%
[07:09:33][D][uart_debug:114]: >>> 0B:00:06:01 : 15:45:62:00:00:32:8C
[07:09:33][D][uart_debug:114]: <<< 0B:00:04:01 : 15:45:62:4A:A8

// 51%
[07:09:54][D][uart_debug:114]: >>> 0B:00:06:01 : 15:18:3F:00:00:4C:0E
[07:09:54][D][uart_debug:114]: <<< 0B:00:04:01 : 15:18:3F:23:51

// 52%
[07:12:35][D][uart_debug:114]: >>> 0B:00:06:01 : 14:61:47:00:00:C9:AA
[07:12:35][D][uart_debug:114]: <<< 0B:00:04:01 : 14:E1:C7:31:43

// 53%
[07:13:05][D][uart_debug:114]: >>> 0B:00:06:01 : 14:2A:0C:00:00:3B:CD
[07:13:05][D][uart_debug:114]: <<< 0B:00:04:01 : 14:AA:8C:F6:35

// 100% when calibrated as low as it goes
[07:26:55][D][uart_debug:114]: >>> 0B:00:06:01 : 07:1C:29:00:00:BB:56
[07:26:55][D][uart_debug:114]: <<< 0B:00:04:01 : 07:1C:29:28:72


// 100%
[07:14:59][D][uart_debug:114]: >>> 0B:00:06:01 : 01:74:47:00:00:06:A3
[07:14:59][D][uart_debug:114]: <<< 0B:00:04:01 : 01:F4:C7:65:5C

// off from 100% (there's a bunch of extra commands happening in a row here, probably dimming down to 0?)
[07:21:02][D][uart_debug:114]: >>> 0B:00:06:01 : 1B:3D:14:00:00:89:1C
[07:21:02][D][uart_debug:114]: <<< 0B:00:04:01 : 1B:BD:94:0F:0A


```

So this is promising! I can still only really see the pattern in the first two data bytes, but that seems to just be the "reverse" of brightness (attenuation?), but in the range of 01:74 (100% brightness) to 1B:3D(0% brightness). When calibrating the dimmer to a different setting, the values in commands still reflect the actual real-world brightness, not the percentage, which makes a lot of sense!

One weird thig though is that there doesn't seem to be a command that would torn the dimmer off. Even then the minimum value is calibrated really high, turning off the light only dims to the calibrated level and doesn't seem to do anything else. However, in real life this of course does turn the light off, not just dims it. So I see two possibilities here:
* Maybe the on/off info is sent around in the "ping" command (0A:00:06:01:00:00)
* Or, maybe the main MCU triggers that itself, through its own GPIO

The second option doesn't _seem_ to be true, since I was able to trace all the contacts between the PCBs to the secondary chip, and there doesn't _seem_ to be a communication channel between the two other than the UART we're looking at. So back to the "ping" thing it is?

Just to be sure, I've recorded a couple more logs of turning the dimmer on and off. The only commands that I could see were indeed:
```
0A:00:06:01 -> 0A:00:04:01 - "ping"
0B:00:06:77 -> 0B:00:04:F7 - indicator LEDs
0B:00:06:01 -> 0B:00:04:01 - dimmer brightness set
```

Here's the "ping" commands that happened in one of those instances. Looking into a bunch of logs, I still can't find any pattern in the data here - except for the response always being the same.

```
[08:06:20][D][uart_debug:114]: >>> 0A:00:06:01:00:00:1B:40:00:74:79
[08:06:20][D][uart_debug:114]: <<< 0A:00:04:01:00:00:00:37:1A
[08:06:22][D][uart_debug:114]: >>> 0A:00:06:01:00:00:1B:41:00:E4:78
[08:06:22][D][uart_debug:114]: <<< 0A:00:04:01:00:00:00:37:1A
[08:06:23][D][uart_debug:114]: >>> 0A:00:06:01:00:00:1B:41:00:E4:78
[08:06:23][D][uart_debug:114]: <<< 0A:00:04:01:00:00:00:37:1A

//Dimmer on here

[08:06:25][D][uart_debug:114]: >>> 0A:00:06:01:00:00:1B:3D:00:24:58
[08:06:25][D][uart_debug:114]: <<< 0A:00:04:01:00:00:00:37:1A

//Dimmer off here

```

To get some more data, I've tried collecting a really long log, doing a bunch of stuff to the dimmer. I've then removed all of the commands that are "known" by now, and here's what remained:

```
[08:18:05][D][uart_debug:114]: >>> 0B:00:06:72:0A:34:7F:00:00:C9:99
[08:18:05][D][uart_debug:114]: <<< 0B:00:04:F2:0A:34:FF:31:4F
[08:18:06][D][uart_debug:114]: >>> 0B:00:06:72:0A:34:7F:00:00:C9:99
[08:18:06][D][uart_debug:114]: <<< 0B:00:04:F2:0A:34:FF:31:4F
[08:18:09][D][uart_debug:114]: >>> 0B:00:06:71:0A:34:7C:00:00:FA:69
[08:18:10][D][uart_debug:114]: <<< 0B:00:04:F1:0A:34:FC:74:0F
```

After re-trying everything I've done while monitoring the logs, it looks like these happen when calibrating the dimmer: 0B:00:06:71 is sent when changing the min level, and 0B:00:06:72 when changing the max. So I think we can ignore these for now and come back to them later.

I've also addressed a random thought I had and added a load to the dimmer just in case it changes anything. Welp, it didn't.

Another thing that could in theory work here is - maybe? - we set the "min" and "max" calibration values, and then anything below that is treated as "off", anything above is "all the way on"? Let's look closer at those calibration comands:

```
// set min brightness:
// min
[08:48:03][D][uart_debug:114]: >>> 0B:00:06:71 : 1B:7A:23:00:00:C3:B2
[08:48:03][D][uart_debug:114]: <<< 0B:00:04:F1 : 1B:7A:A3:E9:2A

//one step from min
[08:48:27][D][uart_debug:114]: >>> 0B:00:06:71 : 1A:34:6C:00:00:FC:A9
[08:48:27][D][uart_debug:114]: <<< 0B:00:04:F1 : 1A:B4:6C:1D:6F

//higher
[08:48:49][D][uart_debug:114]: >>> 0B:00:06:71 : 19:2C:77:00:00:5B:9B
[08:50:51][D][uart_debug:114]: <<< 0B:00:04:F1 : 19:AC:77:16:D5

// set max brightness:
// any value here sends the same thing
[08:46:48][D][uart_debug:114]: >>> 0B:00:06:72 : 1A:34:6F:00:00:CF:59
[08:46:48][D][uart_debug:114]: <<< 0B:00:04:F2 : 1A:B4:6F:58:2F
```

These also seem stable with time, restarts etc. And I think I have my confirmation here! After the min brightness is set, turning the lights off sets the brightness to the same value that was set as the minimum! (At least looking at the first data byte, still no idea if the rest carry any significance).

Seems like it will soon be time to try sending those commands from the chip iself!