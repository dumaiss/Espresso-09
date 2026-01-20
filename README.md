# Project: Espresso-09

**"A High-Pressure Extraction of the 8-bit Era"**

### 1. Core Processing

* **CPU:** Hitachi **HD63C09E** running in **Native Mode**.
* **Clock:** 3.0 MHz - 4.0 MHz (Async from Video/Sound).
* **Performance:** ~3 MIPS (approx. 3-4x speed of standard Z80/6809 systems).
* **Acceleration:** Hardware block moves (`TFM`), 32-bit registers (`Q`), and barrel shifter.

### 2. Memory Architecture (The "GIME-Lite")

* **Physical Memory:** **1 MB SRAM** + **512 KB Flash ROM**.
* **Logical Memory:** 64 KB Address Space.
* **MMU:** **74LS612** Memory Mapper.
* **Page Size:** 8 KB (8 slots).
* **Modes:** Dual-Map Task Switching (System vs. User Task).
* **Features:** Hardware "Shadowing" (Copy ROM to RAM and swap instantly).



### 3. The I/O Channel (The "Barista")

* **I/O Processor:** **ATmega1284P** (20 MHz).
* **Interface:** **6821 PIA** (Parallel Interface Adapter).
* **Port A:** High-Speed Bi-directional Data Bus (Command/Response).
* **Port B:** Dedicated Event Bus (Keyboard/Mouse Interrupts).


* **Peripherals:**
* **Storage:** SD Card (FAT32, DMA-like transfers via Port A).
* **Input:** 4x USB Ports (HID Host for Keyboards/Gamepads).
* **System:** Real-Time Clock, WiFi (optional via ESP on MCU).



### 4. Audio / Video

* **Video:** **TMS9928A** VDP.
* **Output:** Component Video (YPbPr) or RGB.
* **VRAM:** 16 KB (Private Bus).
* **Resolution:** 256x192, 32 Sprites, Hardware Scrolling.


* **Audio:** **Dual SN76489** PSGs.
* **Channels:** 6 Tone + 2 Noise (Stereo Panning).
* **Clock:** 3.579545 MHz (Independent "Concert Pitch" Oscillator).
* **Integration:** Hardware Wait-State Logic via 6309 `MRDY`.



### 5. System Logic

* **Interrupts:** **22V10 PLD** "Northbridge".
* **Type:** Vectored Priority Controller (Indirect Jump Table).
* **Priorities:** Event (UI) > VSYNC > Sound > Bulk Data.


* **Form Factor:** Modular PCB "Bricks" or Integrated Single Board.

1. **Import** your existing Power, Video, and Sound blocks (they remain 90% identical).
2. **Delete** the Z80, SIO, and discrete RAM logic.
3. **Place** the 6309, 74LS612, 6821, and the two 22V10s (MMU Decoders + IRQ Controller).

Would you like me to generate a specific **Netlist / Pin Connection Table** for the 6309 <-> 74LS612 <-> RAM interconnection? That is usually the most error-prone part of the schematic (getting the address lines A0-A19 sorted correctly).
