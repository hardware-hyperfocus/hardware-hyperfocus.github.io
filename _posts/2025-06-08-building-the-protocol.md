---
layout: post
title: "Building the protocol"
date: 2025-06-08 10:00:00 -0000
tags: kasa tapo ks225 esphome
---

Previous post in the series [here]({% link _posts/2025-04-13-testing-commands-replay.md %}).

# Let's goooo

The first thing I want to try is sending the commands I already know, but stripping the part that I don't get. Specifically, there's some bytes at the end of almost every command that I can't seems to see the use for. Let's see if substituting that for 00s still works!

```
this:         00:0B:00:04:F7:01:04:C1:EF:AB:5A:5A
becomes this: 00:0B:00:04:F7:01:04:00:00:00:5A:5A

```
...and that doesn't work! Ok, so those bytes actually do something. Signature? Parity? idk 🤷 It's probably fine as long as they stay the same across different switches - and I seem to have logs from two different switches that show that it does. So I can just keep replaying the exact values I have saved!

Time to try putting it together!

# esphome config
I've set up a "proxy" light component that would pass values to a Template output, where I can send the needed commands with  lambda. 

...aaand I don't have all the values for the different brightness levels. Back to the drawing (guessing?) board! A few hours with ChatGPT (had to forget my dignity for a second) didn't result in much. It does seem like byte 4 of the message refers to the lendth of the payload - and that's about all I could get from that.

So I went on a huge side-quest trying to disassemble/decompile the original firmware. The bruteforce approach of feeding the dump (or even the fw1 portion of the dump) to radar2 or ghidra didn't yield much, as all the references seem to broken when doing that. A few more hours went into trying to figure out if I can get a .elf file from the dump - in which case the disassembly should work smoothly. Looking into libretiny (which builds esphome for rtl flatforms) led me to the ltchiptool elf2bin command - and sure enough, running that for esphome's firmware.elf does produce the right .bin file. Let's see if the process is reversible.

The handler for that command can be found in ltchiptool's `ltchiptool/ltchiptool/soc/ambz2/binary.py`, it's the `elf2bin` method. I'm interested in the `out_ota1` image, which is written from `data[ota1_offset:ota1_end]`, `data` being the result of `flash.pack(hash_key=config.keys.hash_keys["part_table"])`. The `firmware` list is built from the `config.fw` thing, that is ultimately taken from the board's .json definition. Here's the relevant part:

```
"fw": [
    {
        "type": "FWHS_S",
        "sections": [
            {
                "name": "fwhs.sram",
                "type": "SRAM",
                "entry": "__ram_start_table_start__",
                "elf": [
                    ".ram.img.signature",
                    ".ram.func.table",
                    ".data",
                    ".ram.code_text",
                    ".ram.code_rodata"
                ]
            },
            {
                "name": "fwhs.psram",ls
                "type": "PSRAM",
                "entry": "__psram_start__",
                "elf": [
                    ".psram.data",
                    ".psram.code_text",
                    ".psram.code_rodata"
                ]
            }
        ]
    },
    {
        "type": "XIP",
        "sections": [
            {
                "name": "fwhs.xip_c",
                "entry": "XIP_RamImgSignature_s",
                "type": "XIP",
                "elf": [
                    ".xip.code_c"
                ]
            }
        ]
    },
    {
        "type": "XIP",
        "sections": [
            {
                "name": "fwhs.xip_p",
                "entry": "__xip_code_rodata_start__",
                "type": "XIP",
                "elf": [
                    ".xip.code_p"
                ]
            }
        ]
    }
]
```

Each section is built into a file named `f"raw.{section.name}.bin"` with the `_build_section` method. This is where it's getting impossible to keep all of the moving parts in my head 😅 Maybe I could run the command, come in with a debugger and just look at the resulting values directly?

Before I resort to that, let's reconstruct the format of the firmware image. We have some info in the AmebaZ2 application note, but it's a bit incomplete, so let's supplement it with the implementation in libretiny (I wonder where these guys got their info from! upd: looks like they decompile the manufacturer's build tools 🤯).

Here's what I could gather about the firmware image format then:

| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | OTA signature (32B) | Hash of 1st sub-image header
| 32               | Public key 0 - 5 (32B each) | key0 is for OTA signature, Header and FST, the rest are reserved (0xFF); only two keys for OTA!
| 0xE0 224 + ? * n      | Sub-image n | sub-images repeat until the end of the firmware image


Sub-image format:
| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | Sub-image header (96B) | 
| 96               | FST (sub-image security table) (96B) | 
| 192 + ? * n      | Sub-image section n | sections repeat, aligned to 0x20 with 0x00
| end - 32         | Sub-image hash (32B) | calculated with encrypted image if encryption is on; from header to last sub-image (from OTA signature for 1st sub-image)

Sub-image header format:

| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | length (4B) | size of sub-sections + FST
| 4                | next_offset (4B) |
| 8                | type (1B) | PARTAB = 0, BOOT = 1, FWHS_S = 2, FWHS_NS = 3, FWLS = 4, ISP = 5, VOE = 6, WLN = 7, XIP = 8, WOWLN = 10, CINIT = 11, CPFW = 9, UNKNOWN = 63
| 9                | is_encrypted (1B)
| 10               | idx_pkey (1B)
| 11               | flags (1B) |      HAS_KEY1 = 1 << 0, HAS_KEY2 = 1 << 1
| 12               | 8 byte padding
| 20               | serial (4B) | 23 bytes
| 24               | 8B padding
| 32               | user key 1 (32B)
| 64               | user key 2 (32B)

FST format:

| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | encryption algo | 16bit, 0 = AES_EBC, 1 = AES_CBC 
| 2                | hash algo | 16bit, 0 = MD5, 1 = SHA256
| 4                | part_size | 32bit 
| 8                | valid_pattern | 8 bytes, default is 0x0, 0x1, 0x2 ... 0x7
| 16               | 4 bytes padding
| 20               | flags | 8 bit, ENC_EN = 1 << 0, HASH_EN = 2 << 0
| 21               | cipher_key_iv_valid | 8 bit, boolean
| 22               | 10 bytes padding 
| 32               | cipher_key | 23 bytes
| 64               | cipher_iv  | 16 bytes
| 80               | 16 bytes padding

Section format:

| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | Section header (96B) | 
| 96               | Entry Header (32?B) |
| 128?             | Section data     | length set in the entry header

Section header format:

| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | length (4B) | section entry + data length
| 4                | next_offset (4B) |
| 8                | type (1B)     | UNSET = 0x00, DTCM = 0x80, ITCM = 0x81, SRAM = 0x82, PSRAM = 0x83, LPDDR = 0x84, XIP = 0x85, UNKNOWN = 0xFF
| 9                | sce_enabled (1B)
| 10               | xip_page_size (1B)
| 11               | xip_block_size (1B)
| 12               | 4B padding
| 16               | valid_pattern (8B) | default is 0x0, 0x1, 0x2 ... 0x7
| 24               | sce_key_iv_valid (1B) 
| 25               | 7 bytes padding
| 32               | sce_key (16B) | defaults to 0xFF
| 48               | sce_iv (16B)  | defaults to 0xFF
| 64               | 32 bit padding

Entry Header format:

| Offset           | Section | Comment |
| ---------------- | ------- | ------- |
| 0                | length (4B) | data length
| 4                | address (4B) |
| 8                | entry_table (24?B) | default is 6x(4B, 0xFF). 


With that in mind, here's how the firmware gets parsed:
* For each subimage,
  * For each section in the subimage section,
    * `objcopy` the sections in `section.elf` from .elf into a file (`f"raw.{section.name}.bin"`)
    * Build entry header (address=`nmap[section.entry]`, entry_table=[entrypoint])
    * build section (header.type = `section.type`, data is read from file above)
  * Build Firmware struct (with sections above, remove empty)
  * Set sce_key, sce_iv from config

Armed with that knowledge, in theory, we can extract each section one by one from the dump and reconstruct the .elf? Maybe? If the image is not encrypted? Let's see!

fw1 dump:
* Subimage 1 (0xE0):
  * Next offset - 32544
  * type: 02 (FWHS_S)
  * is_encrypted: 00 (yay!)
  * Section 1 (0x01A0):
    * Next offset - 0xFFFFFFFF (last section?)
    * type: 0x82 (SRAM)
    * Entry -> length: 21288
    * Entry -> address: 268436608
    * Entry -> entry_table: 0x80 04 00 10 (only one entry; Don't forget that it's little-endian!)
    * Data: starts at 0x220, 

* Subimage 2 (0xE0 + 32544 = 0x8000)
  * Next offset - 1146880
  * type: 08 (XIP)
  * is_encrypted: 00 (yay!)
  * Section 1 (0x80C0)
    * Next offset - 0xFFFFFFFF (last section?)
    * Entry -> length: 1140056
    * Entry -> address: 2600468800
    * Entry -> entry_table: 0xFF
    * Data: starts at 0x8140

* Subimage 3 (0x8000 + 1146880 = 0x120000)
  * Next offset - 0xFFFFFFFF (last subimage)
  * type: 08 (XIP)
  * is_encrypted: 00 (yay!)
  * Section 1 (0x1200C0)
    * Next offset - 0xFFFFFFFF (last section?)
    * Entry -> length: 223496
    * Entry -> address: 2608857408
    * Entry -> entry_table: 0xFF
    * Data: starts at 0x120140

Checking the addresses of the entries agains the .elf generated by ESPHome build, they do appear to be identical:
```
~$ nm raw_firmware.elf | grep "__ram_start_table_start__"
10000480 D __ram_start_table_start__
~$ nm raw_firmware.elf | grep "XIP_RamImgSignature_s"
9b000140 T XIP_RamImgSignature_s
~$ nm raw_firmware.elf | grep "__xip_code_rodata_start__"
9b800140 T __xip_code_rodata_start__
```
Pretty cool, less guesswork I guess.

Off to carving then! I'll cut out the following sections:
* FWHS_S (skip 544, copy 21288)
* fwhs.xip_c (33088, 1140056)
* fwhs.xip_p (1179968, 223496)

...and loading them all at the "correct" offsets into r2 does not help. Crap. A bunch of hours passed, a bunch of avenues were explored - and I don't think I got it. I've fiddled with the sources of `ltchiptool` and I'm fairly confident that I can reconstruct what the firmware image would actually unpack to in memory - boot section and all. Still, the disassembly in r2 isn't making any sense: I can't even find any references to the strings that are clearly used by the firmware.

Oh well, pick it up later I guess.
