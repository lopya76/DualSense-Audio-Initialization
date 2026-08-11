# DualSense-Audio-Initialization
A technical breakdown and specification for correctly initializing the broken DualSense audio implementation on Windows, which has remained unfixed for 6 years.

## Overview

When connected to a Windows PC via USB, the DualSense controller fails to initialize its internal audio endpoints (built-in speaker and the 3.5mm headset jack. This is a known issue where the controller defaults to an uninitialized audio state. Rel Reddit post: https://www.reddit.com/r/Dualsense/comments/1axqxj8/dualsense_controller_the_speaker_and_pc_support/

By analyzing the 48-byte payload periodically sent to the controller from a host PC, this document details the exact bit manipulation required to force-initialize the audio hardware, route signals to specific outputs, and control hardware-level volumes.

## Tools Used
* **Wireshark** & **USBPcap** (USB packet capture and analysis)
* **DualSenseX** (Parameter modification at the firmware level)


## Payload Structure Overview
The payload looks something like this upon first connection:
First connection / speaker doesn't work
```
0000   02 0f 55 00 00 00 00 00 00 01 00 00 00 00 00 00   ..U.............
0010   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0020   00 00 00 00 00 00 00 03 00 00 02 00 35 00 00 e6   ............5...
```

Correctly initialized, all volumes maxed out (how it SHOULD be)
```
0000   02 ff d7 00 00 00 66 40 7c 01 00 00 00 00 00 00   ......f@|.......
0010   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0020   00 00 00 00 00 00 05 03 00 00 02 00 35 00 00 e6   ............5...
```

Let's break down what each byte does. (0x02 is a report ID and can be ignored in our case.)

Bytes 1-2:
Before the controller will accept volume or routing commands, the audio must first be initialized by flipping the following bytes:
```0x0F``` ```0x55``` -> ```0xFF``` ```0xD7```

