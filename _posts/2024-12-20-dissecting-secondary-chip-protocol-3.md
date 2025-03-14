---
layout: post
title: "Dissecting the secondary chip protocol - pt.3"
date: 2024-12-20 10:00:00 -0000
tags: kasa tapo ks225
---

Part 2 [here]({% link _posts/2024-11-12-dissecting-secondary-chip-protocol-2.md %}).

# Let's analyze some data!

In the last post, I was finally able to get the data from some operations on the Kasa KS225 dimmer. Time to try and figure out what they mean!

# Extracting the messages
It looks like the communication is structured with some kind of constant message header followed by the message itself. The header is 0x5a5a00 for messages TO the secondary chip, and 5a5a01 for the messages FROM it.

The main chip sends out the following message about once a second: 0x000401000000371a. This could be some kind of ping or heartbeat response, but the response doesn't seem to be constant. Here's the list of once-per-second messages received from the secondary chip:
* 00060100001c**110025f5**
* 00060100001c**0f0085fc**
* 00060100001c**0e0015fd**
* 00060100001c**0d00e5fd**
* 00060100001c**0c0075fc**
* 00060100001c**0b0045fe**
* 00060100001c**090025ff**
* 00060100001c**0800b5fe**
* 00060100001c**070045fb**
* 00060100001c**0600d5fa**
* 00060100001c**0600d5fa**

So maybe this actually conveys some data? The first two bytes seem to follow a descending pattern, and some bits seem to stay the same between the messages... 🤷

Here's the request/response messages used when turning the dimmer on:

Request:

0b000401188da71aae
0b0004f70104c1efab
0b000401188da71aae
0b000401170722da38
0b00040115183f2351
0b00040115183f2351

Response:

0b000601180d27000086a7
0b00067701044100009152
0b000601180d27000086a7
0b00060117072200005ee0
0b00060115183f00004c0e

Note that these were recorded in different sessions, so this is not really a response to the exact same request

Here's turning the brightness up

Request:
0b0004f7800044a438

0b000405117156acff
0b0004f70105c0bf6b

Response:
0b00060115183f00004c0e

0b00060511715600008833
0b0006770105400000ad02

And, finally, turning the dimmer off:

Request:
0b0004f7800044a438
0b00040115cceb7c0e
0b0004f70100c5eca8
0b00040115cceb7c0e
0b0004011734113f6c
0b000401186f45f366
0b000401195f74e7e2
0b0004011a50781217


0b0004011b200936a3
0b0004011bbc955fca
0b0004011bbd940f

Response:
0b0006770000440000607e
0b000601154c6b0000ac5f
0b00067701004500006012
0b000601154c6b0000ac5f
0b0006011734110000151f
0b000601186f450000e019
0b000601195f740000ef7a
0b0006011a50780000f8fd
0b0006011b20090000e38a
0b0006011b3c150000b54c

0b0006011b3d140000891c

...and idk what any of this means :c

Part 4 [here]({% link _posts/2025-03-10-dissecting-secondary-chip-protocol-4.md %}).