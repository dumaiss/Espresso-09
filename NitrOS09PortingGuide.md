# Espresso-09: NitrOS-9 Porting Guide & BSP

**Target Architecture:** Hitachi 6309 (Native Mode)
**Memory Model:** 1MB RAM (Dynamic Address Translation via 74LS612)
**I/O Model:** PIA-based Command/Event Channel

---

## Phase 1: System Definitions (`uBITz_def.asm`)

This file defines the hardware register map. It is the equivalent of `coco3.d` or `sys.d` in the standard NitrOS-9 source tree.

```assembly
*******************************************************************************
* uBITz 6309 System Definition File
* Architecture: Hitachi 6309 (Native) + 1MB RAM + PIA I/O + Vectored IRQ
*******************************************************************************

* -----------------------------------------------------------------------------
* 1. MEMORY MAP (I/O Space $FF00 - $FFFF)
* -----------------------------------------------------------------------------
IO_BASE         EQU $FF00

* --- Interrupt Controller (22V10) ---
IRQ_VECTOR_H    EQU $FF00       ; Read: Returns Vector High Byte ($C0)
IRQ_VECTOR_L    EQU $FF01       ; Read: Returns Vector Low Byte (Priority Encoded)

* --- PIA Subsystem (I/O Channel) ---
PIA_BASE        EQU $FF40
PIA_PORTA       EQU $FF40       ; Data Channel (CPU <-> MCU Bulk Data)
PIA_CRA         EQU $FF41       ; Control Register A
PIA_PORTB       EQU $FF42       ; Event Channel (MCU -> CPU Status/Keys)
PIA_CRB         EQU $FF43       ; Control Register B

* --- Sound Subsystem (Dual SN76489) ---
SOUND_LEFT      EQU $FF50       ; Write-Only: Left Channel Chip
SOUND_RIGHT     EQU $FF51       ; Write-Only: Right Channel Chip

* --- MMU Subsystem (74LS612) ---
MMU_REGS_SYS    EQU $FF60       ; System Task Map (Regs 0-7)
MMU_REGS_USR    EQU $FF68       ; User Task Map (Regs 8-15)

* --- System Control Latch (PLD) ---
SYS_CTRL        EQU $FF90
* Bit Definitions for SYS_CTRL:
SYS_BOOT_BIT    EQU %10000000   ; 1=Boot Mode (Flash Everywhere), 0=Normal
SYS_TASK_BIT    EQU %00000001   ; 1=User Task, 0=System Task

* -----------------------------------------------------------------------------
* 2. INTERRUPT VECTORS (Indirect Jump Table at $C000)
* -----------------------------------------------------------------------------
* The 22V10 outputs these Low Byte offsets based on priority.
VEC_DATA_IO     EQU $C000       ; Priority 0 (Lowest): Disk/Bulk Data
VEC_AUX         EQU $C010       ; Priority 1: Expansion/Sound
VEC_VSYNC       EQU $C020       ; Priority 2: Video VBLANK
VEC_EVENT       EQU $C030       ; Priority 3 (Highest): Keyboard/Mouse
VEC_SPURIOUS    EQU $C0E0       ; Safety Vector (No IRQ active)

* -----------------------------------------------------------------------------
* 3. MMU CONFIGURATION (74LS612)
* -----------------------------------------------------------------------------
* Bits 0-6: Physical Page Index (0-127)
* Bit 7:    Type Select (0=RAM, 1=Flash)
MMU_TYPE_RAM    EQU %00000000
MMU_TYPE_FLASH  EQU %10000000

* Page Size = 8KB. Total Logical Pages = 8 ($0000-$FFFF)
PAGE_0          EQU 0           ; $0000 - $1FFF
PAGE_1          EQU 1           ; $2000 - $3FFF
PAGE_6          EQU 6           ; $C000 - $DFFF (Used for Shadow Copy)
PAGE_7          EQU 7           ; $E000 - $FFFF (System/Vectors)

* -----------------------------------------------------------------------------
* 4. PIA CONFIGURATION
* -----------------------------------------------------------------------------
* Port A: Data (IRQ Enabled, Pulse Mode CA2)
PIA_CFG_DATA    EQU %00101101   
* Port B: Events (IRQ Enabled, Strobe Mode CB2)
PIA_CFG_EVENT   EQU %00100101   

* -----------------------------------------------------------------------------
* 5. MACROS
* -----------------------------------------------------------------------------
* Switch to User Memory Map
SET_TASK_USER MACRO
        LDA   SYS_CTRL
        ORA   #SYS_TASK_BIT
        STA   SYS_CTRL
        ENDM

* Switch to System Memory Map
SET_TASK_SYS MACRO
        LDA   SYS_CTRL
        ANDA  #~SYS_TASK_BIT
        STA   SYS_CTRL
        ENDM

```

---

## Phase 2: The Cold Boot Loader (`boot.asm`)

This code initializes the 6309 Native Mode, sets up the MMU, and prepares the I/O. It replaces the standard CoCo BASIC ROM.

```assembly
*******************************************************************************
* uBITz 6309 Bootloader & Initialization
*******************************************************************************
    INCLUDE "uBITz_def.asm"
    ORG     $E000           ; Runs in Top 8KB Page (Flash Mirror at boot)

RESET:
    ORCC    #$50            ; Disable FIRQ/IRQ
    
    * 1. ENTER NATIVE MODE (The "Turbo Button")
    LDMD    #$01            ; Switch 6309 to Native Mode

    * 2. INITIALIZE MMU MAP (System Task 0)
    LDU     #MMU_REGS_SYS   ; Point to $FF60
    LDA     #0              ; Start Physical Page 0
    
    * Map RAM Pages 0-6
INIT_RAM_LOOP:
    STA     ,U+             
    INCA                    
    CMPA    #7              
    BNE     INIT_RAM_LOOP

    * Map Page 7 to Flash (Physical Page $80)
    * Keeps execution alive before we Shadow RAM
    LDA     #(MMU_TYPE_FLASH | $00) 
    STA     ,U              

    * 3. SETUP STACK
    LDS     #$DFFF          ; Top of RAM Page 6

    * 4. DISABLE BOOT ROM MIRROR
    * Clears SYS_BOOT_BIT. MMU takes full control.
    LDA     #0              
    STA     SYS_CTRL        

    * 5. INITIALIZE PIA (I/O Channel)
    CLR     PIA_CRA         
    LDA     #$FF            ; DDRA = Output
    STA     PIA_PORTA       
    LDA     #PIA_CFG_DATA   
    STA     PIA_CRA         

    CLR     PIA_CRB         
    CLR     PIA_PORTB       ; DDRB = Input
    LDA     #PIA_CFG_EVENT  
    STA     PIA_CRB         

    * 6. SHADOW ROM TO RAM (See Section 3)
    JSR     SHADOW_ROM

    * 7. SETUP RAM VECTORS (Jump Table)
    * Now that vectors are in RAM, we perform 'Transfer Memory' fill 
    * to clear the table, then install a safety handler.
    LDX     #$C000
    LDW     #$00FF
    TFM     X+,X+           ; Clear Table
    
    LDX     #VEC_SPURIOUS
    LDA     #$3B            ; 'RTI' Opcode
    STA     ,X

    * 8. HANDOFF TO OS/MONITOR
    ANDCC   #$EF            ; Enable IRQ
    JMP     OS_START        ; Jump to Kernel entry

```

---

## Phase 3: Shadow RAM Routine

This routine copies the OS code from slow Flash to fast RAM using the 6309 `TFM` instruction, then re-maps the MMU so the CPU runs from RAM seamlessly.

```assembly
*******************************************************************************
* SHADOW_ROM
* Copies Flash ($E000) to RAM and swaps it in.
*******************************************************************************
SHADOW_ROM:
    * 1. MAP TARGET RAM
    * Map Physical RAM Page 7 to Logical Page 6 ($C000)
    LDA     #7              
    STA     MMU_REGS_SYS+6  

    * 2. BURST COPY (Flash -> RAM)
    * Source: $E000 (Flash)
    * Dest:   $C000 (RAM)
    * Size:   8KB
    LDU     #$E000          
    LDY     #$C000          
    LDW     #$2000          
    TFM     U+, Y+          ; Hardware Block Copy (Native Mode)

    * 3. THE SWAP
    * Map Physical RAM Page 7 (Bit 7=0) to Logical Page 7 ($E000)
    * Execution continues from RAM instantly.
    LDA     #7              
    STA     MMU_REGS_SYS+7  

    * 4. CLEANUP
    * Restore Logical Page 6 to RAM Page 6
    LDA     #6
    STA     MMU_REGS_SYS+6
    
    RTS

```

---

## Phase 4: Hardware Vector Table (Flash)

This must be burned at the very end of the 512KB Flash. It contains the 6309 hardware vectors.

**Critical Porting Detail:** Unlike standard CoCo which jumps to a specific address, Espresso-09 uses **Indirect Vectoring** via the 22V10 PLD.

```assembly
    ORG     $FFF0
    
    FDB     RESET           ; Reserved
    FDB     RESET           ; SWI3
    FDB     RESET           ; SWI2
    FDB     RESET           ; FIRQ
    
    * IRQ VECTOR (The 22V10 Link)
    * Opcode $6E $9F $00 $FF = JMP [$FF00]
    FCB     $6E, $9F
    FDB     IRQ_VECTOR_H    ; $FF00
    
    FDB     RESET           ; SWI
    FDB     RESET           ; NMI
    FDB     RESET           ; RESET ($FFFE)

```

---

## Phase 5: Software Jump Table (RAM)

Once `SHADOW_ROM` runs, the Vectors are in RAM. The OS Kernel must populate the jump table at `$C000` to route interrupts to the actual drivers.

```assembly
    ORG     $C000

* Priority 0: Data (Disk I/O)
ISR_DATA:
    LBRA    DRIVER_DISK     ; Jump to Disk Driver
    NOP                     ; Padding to align to 16 bytes

    ORG     $C010
* Priority 1: Aux (Sound/Exp)
ISR_AUX:
    LBRA    DRIVER_SOUND

    ORG     $C020
* Priority 2: VSync (Video)
ISR_VSYNC:
    LBRA    DRIVER_VIDEO

    ORG     $C030
* Priority 3: Event (Keyboard/Mouse)
ISR_EVENT:
    LBRA    DRIVER_INPUT    ; Jump to Input Driver

```
