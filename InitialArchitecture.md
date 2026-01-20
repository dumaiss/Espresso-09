# Project: Espresso-09 (uBITz 6309)
**"A High-Pressure Extraction of the 8-bit Era"**

## 1. Architecture Overview

* **CPU:** Hitachi **HD63C09E** running in **Native Mode**.
* **Clock:** 3.0 MHz - 4.0 MHz (Async from Video/Sound).
* **Performance:** ~3 MIPS (approx. 3-4x speed of standard Z80/6809 systems).
* **Acceleration:** Hardware block moves (`TFM`), 32-bit registers (`Q`), and barrel shifter.

### Subsystems
1.  **Memory ("GIME-Lite"):** 1 MB SRAM + 512 KB Flash, managed by a **74LS612** MMU with Task Switching.
2.  **I/O ("The Barista"):** **ATmega1284P** I/O Processor connected via a **6821 PIA** split-channel bus.
3.  **Interrupts ("Northbridge"):** **22V10 PLD** implementing a Vectored Priority Controller (Indirect Jump Table).
4.  **A/V:** **TMS9928A** Video + **Dual SN76489** Audio (Stereo).

---

## 2. The Interrupt Subsystem (22V10 PLD)

**Function:** Prioritizes 4 interrupt sources and provides a dynamic vector address to the CPU.
**Mechanism:** The CPU vector at `$FFF8` points to `JMP [$FF00]`. The PLD watches the address bus; when `$FF01` is read, it substitutes the data bus with the offset of the highest priority handler.

### Wiring: IRQ Controller (U1 - 22V10)

| Pin | Signal | Direction | Connection | Notes |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `A0` | Input | CPU `A0` | Selects High/Low byte of vector |
| 2 | `!REQ_DATA` | Input | 6821 `!IRQA` | **Priority 0** (Lowest) - Disk/Bulk I/O |
| 3 | `!REQ_AUX` | Input | Expansion | **Priority 1** - Sound/Expansion |
| 4 | `!REQ_VSYNC` | Input | TMS9928 `!INT` | **Priority 2** - Video Frame Sync |
| 5 | `!REQ_EVENT` | Input | 6821 `!IRQB` | **Priority 3** (Highest) - UI Events |
| 6 | `!CS` | Input | Decoder `Y0` | Active Low for Address `$FF00-$FF01` |
| 23 | `!CPU_IRQ` | Output | CPU `!IRQ` | Master Interrupt Line |
| 14-21 | `D0-D7` | Output | CPU `D0-D7` | Tri-stated (Active only when `!CS` is Low) |

### Logic: `IRQ_CTL.PLD`

```pld
Name     IRQ_CTL ;
Partno   U001 ;
Device   g22v10 ;

/* Inputs & Outputs */
Pin 1 = A0;   Pin 6 = !CS;
Pin [2..5] = [!REQ_DATA, !REQ_AUX, !REQ_VSYNC, !REQ_EVENT];
Pin [14..21] = [D0..D7]; 
Pin 23 = !CPU_IRQ; 

/* Logic Definitions */
$DEFINE Win_EVENT (REQ_EVENT)
$DEFINE Win_VSYNC (REQ_VSYNC & !REQ_EVENT)
$DEFINE Win_AUX   (REQ_AUX   & !REQ_VSYNC & !REQ_EVENT)
$DEFINE Win_DATA  (REQ_DATA  & !REQ_AUX   & !REQ_VSYNC & !REQ_EVENT)
$DEFINE NoReq     (!REQ_DATA & !REQ_AUX   & !REQ_VSYNC & !REQ_EVENT)

/* Equations */
CPU_IRQ = REQ_EVENT # REQ_VSYNC # REQ_AUX # REQ_DATA;

FIELD Data_Bus = [D7..D0];
Data_Bus = 
    (!A0 & 'h'C0)            # /* High Byte: $C0xx */
    (A0 & Win_EVENT & 'h'30) # /* Priority 3: $C030 */
    (A0 & Win_VSYNC & 'h'20) # /* Priority 2: $C020 */
    (A0 & Win_AUX   & 'h'10) # /* Priority 1: $C010 */
    (A0 & Win_DATA  & 'h'00) # /* Priority 0: $C000 */
    (A0 & NoReq     & 'h'E0);  /* Safety:     $C0E0 */

[D0..D7].oe = CS;

```

---

## 3. The I/O Channel (6821 PIA + ATmega1284)

**Concept:** A "Split PIA" architecture. Port A is a bidirectional Data Highway (polled/low priority). Port B is an Input-Only Event Ticker (high priority).

### Wiring: PIA to MCU

| Signal | 6821 Pin | ATmega Pin | Function |
| --- | --- | --- | --- |
| **DATA BUS** | `PA0-PA7` | `PORTC[0..7]` | The Main Highway (Command/Data) |
| **EVENT BUS** | `PB0-PB7` | `PORTA[0..7]` | The "News Ticker" (Status codes) |
| **CMD_STROBE** | `CA2` | `INT0 (PD2)` | CPU Output: "I sent a command" |
| **DATA_RDY** | `CA1` | `PD3` | CPU Input: "MCU has data" |
| **EVENT_IRQ** | `CB1` | `PD4` | CPU Input: "User Event Occurred" |

---

## 4. The Memory Subsystem (74LS612 + 22V10)

**Concept:** 1MB Physical RAM mapped into 64KB Logical Space using 8KB Pages. Bit 7 of the MMU output acts as a "Type Select" to switch between RAM and Flash instantly.

### Wiring: MMU Interconnect (Critical)

| 6309 CPU | 74LS612 (MMU) | RAM (1MB) | Flash (512KB) | 22V10 (MMU Logic) |
| --- | --- | --- | --- | --- |
| `D0-D7` | `D0-D7` | `D0-D7` | `D0-D7` | - |
| `A13` | `RS0` | - | - | - |
| `A14` | `RS1` | - | - | - |
| `A15` | `RS2` | - | - | `A15` (Decode) |
| - | `RS3` | - | - | `MMU_SEL` (Output) |
| - | `MO0` | `A13` | `A13` | - |
| - | `MO1` | `A14` | `A14` | - |
| ... | ... | ... | ... | ... |
| - | `MO6` | `A19` | `A19` (NC) | - |
| - | `MO7` | - | - | Pin 1 (`TYPE_BIT`) |
| - | `D8-D11` -> GND | - | - | - |

*Note: Flash A19 is not connected effectively, mirroring the 512KB Flash twice in the upper ID space.*

### Logic: MMU Controller (Conceptual)

```cupl
/* 1. MMU Bank Select (RS3) */
/* If writing to $FF6x, let A3 choose bank (allows cross-task updates). */
/* Else use Task Bit. Force Task 0 (System) for Vectors/IO. */
MMU_SEL = (Is_MMU_Write & A3) 
        # (!Is_MMU_Write & !Is_Vector & TASK_BIT);

/* 2. Chip Selects */
/* RAM: Active if not I/O, not Booting, and MO7 is Low */
!RAM_CS   = !Is_IO_Hole & !BOOT_EN & !MO7;

/* Flash: Active if Booting OR MO7 is High */
!FLASH_CS = (BOOT_EN # (!Is_IO_Hole & MO7));

```

---

## 5. Software Definition (`uBITz_def.asm`)

```asm
*******************************************************************************
* uBITz 6309 System Definition File
* Architecture: Hitachi 6309 (Native) + 1MB RAM + PIA I/O + Vectored IRQ
*******************************************************************************

* --- Memory Map ---
IO_BASE         EQU $FF00
IRQ_VECTOR_H    EQU $FF00       ; Vector High Byte ($C0)
IRQ_VECTOR_L    EQU $FF01       ; Vector Low Byte (Dynamic)

PIA_BASE        EQU $FF40
PIA_PORTA       EQU $FF40       ; Data Channel
PIA_CRA         EQU $FF41       ; Control A
PIA_PORTB       EQU $FF42       ; Event Channel
PIA_CRB         EQU $FF43       ; Control B

SOUND_LEFT      EQU $FF50       ; SN76489 #1
SOUND_RIGHT     EQU $FF51       ; SN76489 #2

MMU_REGS_SYS    EQU $FF60       ; System Task Map (Regs 0-7)
MMU_REGS_USR    EQU $FF68       ; User Task Map (Regs 8-15)
SYS_CTRL        EQU $FF90       ; System Latch

* --- System Control Bits ---
SYS_BOOT_BIT    EQU %10000000   ; 1=Boot Mode (Flash Mirror), 0=Normal
SYS_TASK_BIT    EQU %00000001   ; 1=User Task, 0=System Task

* --- Interrupt Vectors ($C0xx) ---
VEC_DATA_IO     EQU $C000       ; Priority 0
VEC_AUX         EQU $C010       ; Priority 1
VEC_VSYNC       EQU $C020       ; Priority 2
VEC_EVENT       EQU $C030       ; Priority 3 (Highest)
VEC_SPURIOUS    EQU $C0E0       ; Safety

* --- MMU Constants ---
MMU_TYPE_RAM    EQU %00000000
MMU_TYPE_FLASH  EQU %10000000

```

---

## 6. Bootloader Source (`boot.asm`)

```asm
    INCLUDE "uBITz_def.asm"
    ORG     $E000           ; Runs in Top 8KB Page

RESET:
    ORCC    #$50            ; Disable Interrupts
    LDMD    #$01            ; ENTER 6309 NATIVE MODE

    * -- Configure MMU (74LS612) --
    LDU     #MMU_REGS_SYS   ; Point to $FF60
    LDA     #0              ; Start Physical Page 0
INIT_RAM_LOOP:
    STA     ,U+             ; Store Page ID
    INCA                    ; Next Page
    CMPA    #7              ; Done 0-6?
    BNE     INIT_RAM_LOOP

    * Map Page 7 to Flash initially
    LDA     #(MMU_TYPE_FLASH | $00) 
    STA     ,U              ; Store at $FF67

    * -- Disable Boot ROM --
    LDA     #0              
    STA     SYS_CTRL        ; Clear BOOT_BIT, MMU takes over

    * -- Initialize PIA --
    CLR     PIA_CRA         ; Select DDRA
    LDA     #$FF            ; All Output
    STA     PIA_PORTA       
    LDA     #PIA_CFG_DATA   ; Load Config
    STA     PIA_CRA         

    * -- Setup RAM Vector Table --
    LDX     #$C000
    LDW     #$00FF          
    TFM     X+,X+           ; Clear Table
    
    * -- Shadow Flash to RAM --
    JSR     SHADOW_ROM

    * -- Go --
    ANDCC   #$EF            ; Enable IRQ
    JMP     MAIN            

*******************************************************************************
* SHADOW_ROM: Copies 8KB ROM at $E000 to RAM, then swaps RAM into place
*******************************************************************************
SHADOW_ROM:
    LDA     #7              ; Target Physical RAM Page ID
    STA     MMU_REGS_SYS+6  ; Map it to Window at $C000
    
    LDU     #$E000          ; Source (ROM)
    LDY     #$C000          ; Dest (RAM Window)
    LDW     #$2000          ; 8KB
    TFM     U+, Y+          ; Block Copy

    LDA     #7              ; Physical RAM Page 7
    STA     MMU_REGS_SYS+7  ; Overwrite mapping at $E000 (The Swap!)
    
    LDA     #6              ; Restore Window
    STA     MMU_REGS_SYS+6
    RTS
```

```

```
