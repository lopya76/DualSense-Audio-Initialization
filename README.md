# DualSense Audio Initialization on Windows
A technical breakdown and specification for correctly initializing the broken DualSense audio implementation on Windows, which has remained unfixed for 6 years.


## Overview

When connected to a Windows PC via USB, the DualSense controller fails to initialize its internal audio endpoints (built-in speaker and the 3.5mm headset jack). This is a known issue where the controller defaults to an uninitialized audio state. Related Reddit post: https://www.reddit.com/r/Dualsense/comments/1axqxj8/dualsense_controller_the_speaker_and_pc_support/

By analyzing the 48-byte payload, which is periodically sent to the controller from a host PC, this document details the exact bit manipulation required to force-initialize the audio hardware, route signals to specific outputs, and control hardware-level volumes.
<img width="990" height="366" alt="Screenshot 2026-08-11 132949" src="https://github.com/user-attachments/assets/c38bb882-6454-44cb-bdf3-6271502c87b2" />

## Tools Used
* **Wireshark** & **USBPcap** (USB packet capture and analysis)
* **DualSenseX** (Parameter modification at the firmware level)
<img width="1919" height="1079" alt="Screenshot 2026-08-11 140017" src="https://github.com/user-attachments/assets/a7f52640-b974-4400-87b1-a917c697d5e9" />
<br>
<br>

## Payload Structure Overview
The payload looks something like this:

First connection / speaker doesn't work
```
0000   02 0f 55 00 00 00 00 00 00 01 00 00 00 00 00 00   ..U.............
0010   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0020   00 00 00 00 00 00 00 03 00 00 02 00 35 00 00 e6   ............5...
```

Correctly initialized, all volumes maxed out (how it SHOULD be)
```
0000   02 ff d7 00 00 9d 66 40 6c 01 00 00 00 00 00 00   ......f@l.......
0010   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0020   00 00 00 00 00 00 05 03 00 00 02 00 35 00 00 e6   ............5...
```
<br>
<br>

## Byte breakdown
> Note: 0x02 is a report ID and can be ignored in our case.


## Bytes 1-2 [*`ff d7`* 00 00 9d 66 40 6c]
Before the controller will accept volume or routing commands, the audio must first be initialized by flipping the following bytes:
`0x0F` `0x55` -> `0xFF` `0xD7`

<br>

## Byte 8 [ff d7 00 00 9d 66 40 *`6c`*]

This byte acts as a hardware multiplexer, determining which physical outputs receive the audio signal. If an output is disabled via this value, its corresponding volume byte in the payload is forced to ```0x00```

There are 3 output modes and changing this value selects the corresponding mode:
| Output mode | Byte 8 Value | Effect |
| --- | --- | --- |
| Speaker Only| 0x7C | Headset volume byte (Byte 5) must be 0x00. |
| Headset Only | 0x4C | Speaker volume byte (Byte 6) must be 0x00. |
| Combined | 0x6C | Both volume bytes are active simultaneously. |

<br>

## Bytes 5, 6 and 7 [ff d7 00 00 `9d` `66` `40` 6c]
If the aforementioned values are set correctly, we unlock volume controls for the microphone, speaker and headset.

<br>
<br>

**Byte 5** follows a linear curve across its full range `(hex ≈ 1.28 × volume% + 28)`. At 0%, this naturally lands on 0x1E rather than 0x00, which floor keeps the headphone amp engaged, avoiding a hard cutoff, but may produce audible hiss on particularly sensitive IEMs.

0% (Floor): ```0x1E```
10%: ```0x29```
50%: ```0x5D```
80%: ```0x82```
100%: ```0x9D```

<br>
<br>

**Byte 6** also scales linearly `(hex ≈ 0.41 × volume% + 61)`, but unlike the 3.5mm jack, 0% is a hard mute (0x00).


0% (Muted): ```0x00```
10%: ```0x41```
50%: ```0x51```
80%: ```0x5D```
100%: ```0x66```

<br>
<br>

**Byte 7** is the microphone, which uses a 1:1 linear mapping to integer values, its max value being 64.

Formula: `Volume (0-64) = Hex Value`
`64` directly corresponds to `0x40`, `38` to `0x26`, etc.


<br>
<br>

## Payload Construction Reference

To recap, this is the byte structure controlling the states of different audio endpoints of a DualSense controller: <br>
Byte 0: 0x02 (Report ID) <br>
Byte 1: 0xFF (Initialized Flag) <br>
Byte 2: 0xD7 (Initialized Flag) <br>
Byte 3-4: 0x00 0x00 <br>
Byte 5: [Headset Volume Hex] (Must be 0x00 if Headset is off) <br>
Byte 6: [Speaker Volume Hex] (Must be 0x00 if Speaker is off) <br>
Byte 7: [Mic Volume Hex] <br>
Byte 8: [Output Mode Hex] <br>
<br>
The rest of the bytes are currently undocumented by me, but I think they're for LEDs and adaptive trigger states.
<br>
<br>

All of the research was done by me, by hand, to practice reading and deciphering bytes (which i recently learned how to do :D)
<br>
I hope my research will be useful to someone, someday.




