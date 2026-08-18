# Espresso-09

**A 6309 member of the pBITz coffee-machine family**

Espresso-09 is a modular homebrew computer built around the Hitachi HD63C09E/HD63C09EP. It combines a real 6309 CPU, a 74LS612 memory mapper, 512 KiB of SRAM, 512 KiB of ROM, and the shared pBITz expansion architecture used by the other coffee-series machines.

The primary operating-system target is **NitrOS-9 Level 2**. The memory system is deliberately organized around NitrOS-9's native 8 KiB DAT model rather than reproducing the CoCo 3 GIME in programmable logic.

Espresso-09 is currently an architecture and hardware-design project. The repository is being used to capture the machine definition before schematic capture and firmware bring-up begin.

## Current status

| Subsystem | Status |
| --- | --- |
| HD63C09E/HD63C09EP CPU architecture | Selected |
| 74LS612-based MMU architecture | Defined at architectural level |
| 512 KiB RAM + 512 KiB ROM physical map | Defined |
| 8 KiB NitrOS-9-compatible DAT model | Defined |
| Fixed pBITz I/O window | Defined |
| ROM-to-RAM instant-on boot model | Defined at architectural level |
| ATF15xx control/decode CPLD | Architecture defined; device/package and equations pending |
| W65C22S / RX660 I/O Controller interface | Architecture selected; electrical assignment and byte protocol pending |
| NitrOS-9 Level 2 port | Target architecture defined; implementation not started |
| CPU board schematic and PCB | Not started |
| Emulator support | Future work |

## Hardware architecture

### Processor

- Hitachi **HD63C09E / HD63C09EP**
- 6309 Native Mode is the normal software target
- Initial CPU clock target is approximately **3 MHz**; final clock-generation details remain to be finalized
- 16-bit logical address space and 8-bit data bus
- 6309-specific instructions such as `TFM` are expected to be useful for ROM shadowing and bulk-memory operations

### Physical memory

Espresso-09 has exactly **1 MiB of physical memory**:

| Physical range | Size | Function |
| --- | ---: | --- |
| `00000h-7FFFFh` | 512 KiB | SRAM |
| `80000h-FFFFFh` | 512 KiB | ROM |

The 20-bit physical address space is divided into 128 blocks of 8 KiB each. Blocks `00h-3Fh` select RAM and blocks `40h-7Fh` select ROM.

### Memory mapper

The **74LS612** is used as the actual address-translation store and mapper. Its sixteen mapping registers are divided into two complete 64 KiB hardware maps:

- entries 0-7: **task 0**, the system/kernel map
- entries 8-15: **task 1**, the currently active user-process map

CPU `A15..A13` select one of eight 8 KiB logical blocks. The LS612 `MA3` input is the hardware task-select bit. This is intentionally close to the NitrOS-9 Level 2 DAT model: the CPLD controls policy and fixed overlays, but it does not re-create a second MMU or a GIME-style translation engine.

With 128 physical 8 KiB blocks, each mapper entry needs seven physical-block bits. The remaining five bits in the LS612's 12-bit entry are available for Espresso-specific attributes such as write protection, invalid-page detection, and future diagnostics.

See [Architecture.md](Architecture.md) for the current memory architecture and boot sequence.

## Logical address space

The current architectural baseline keeps almost the entire 64 KiB CPU address space available as DAT-translated memory while reserving small fixed windows at the top:

| Logical range | Function |
| --- | --- |
| `0000h-FDFFh` | DAT-translated memory |
| `FE00h-FEFFh` | Fixed pBITz I/O window |
| `FF00h-FF1Fh` | Fixed MMU/control registers |
| `FF20h-FFEFh` | Fixed common RAM / task-switch trampoline area |
| `FFF0h-FFFFh` | Fixed vector window |

The exact implementation of the common/vector backing store remains part of the hardware design, but these windows are intentionally independent of the currently selected DAT task.

### pBITz I/O compatibility

The `FE00h-FEFFh` window preserves the pBITz 4-bit Device Select convention:

```text
FE D R
   | |
   | +-- A3..A0: card-local register 0..15
   +---- A7..A4: Device Select 0..15
```

This gives each selected expansion device at least sixteen directly addressable registers while keeping I/O permanently visible regardless of the current memory task.

The shared backplane and expansion-bus hardware are maintained in the [pBITzPlatform repository](https://github.com/dumaiss/pBITzPlatform).

## Boot and ROM shadowing

Espresso-09 is intended to provide an **instant-on NitrOS-9** experience from ROM while executing normal runtime code from RAM.

At reset, the LS612 enters pass mode so the first 64 KiB of RAM is available with a known identity mapping. The CPLD overlays a boot ROM region at the top of the logical address space so the 6309 can fetch its reset vector and execute the bootstrap without relying on uninitialized mapper contents.

The normal startup sequence is expected to:

1. reset into LS612 pass mode with task 0 selected
2. execute the bootstrap from the ROM overlay
3. initialize the system and user DAT entries
4. enter LS612 map mode while keeping the boot ROM overlay active
5. copy the ROM image into RAM in 8 KiB blocks using temporary source/destination windows and `TFM`
6. install the runtime NitrOS-9 system map and fixed common/vector state
7. transfer execution to a RAM-resident handoff stub
8. disable the boot ROM overlay
9. continue into NitrOS-9 from RAM

Normal runtime mappings are expected to use RAM. ROM remains the reset/firmware source and may still be explicitly mapped for diagnostics or recovery if the final CPLD policy permits it.

## NitrOS-9 Level 2

NitrOS-9 Level 2 is the primary operating-system target.

The hardware is designed around its native memory-management model:

- eight 8 KiB DAT blocks per 64 KiB process address space
- a permanent system hardware map
- one hardware map for the currently running user process
- additional process DAT images maintained by the kernel in software
- context switches implemented by loading eight physical-block bytes into LS612 task-1 entries

The Espresso port will still require platform-specific boot, MMU, interrupt, timer, console, storage, and device drivers, but the goal is to preserve the existing NitrOS-9 Level 2 memory-management concepts rather than emulate CoCo-specific peripherals.

## I/O Controller

Modern storage and human-interface services are expected to be offloaded to an **RX660-based I/O Controller**. The selected host-facing interface is a **W65C22S VIA** rather than dual-port shared RAM.

The VIA presents an ordinary 16-register peripheral to the 6309 through the fixed pBITz I/O window. It is the electrical and handshake endpoint for IOCALL-style transactions; it is **not** intended to hold the I/O Controller's complete state or buffers.

The richer interface lives in RX660 SRAM. Firmware can expose logical registers, command queues, replies, events, and bulk-transfer buffers through a byte-level protocol carried by the VIA. A typical transaction can therefore use one VIA port as an 8-bit data path, the other for command/index information, and the VIA control lines for host-to-controller and controller-to-host handshaking. The exact port and control-line assignment is not yet frozen.

The same transport is expected to support:

- short host-to-IOC commands and replies
- software-defined IOC register access
- bulk byte streams for blocks such as storage sectors
- asynchronous events and host interrupts

Expected I/O Controller responsibilities include:

- storage
- keyboard and controller input
- real-time clock and system services
- asynchronous host events

The VIA's timers and shift register remain available, but the final source of the NitrOS-9 system tick and other host timing services is still an implementation decision.

The IOC transport is not intended to grow a parallel shared-memory path. Any future high-bandwidth DMA or shared-memory facility should be designed separately with explicit physical-address semantics rather than becoming part of the RX660 mailbox interface.

## Video and sound

Video and sound are expansion functions rather than fixed properties of the CPU board. Espresso-09 is intended to reuse compatible pBITz and Percolator-series hardware where practical.

Multimedia-card development is maintained separately in the [PercolatorLabs repository](https://github.com/dumaiss/PercolatorLabs).

## Repository layout

The repository is currently small while the architecture is being established.

| Path | Contents |
| --- | --- |
| `README.md` | Project overview and current status |
| `Architecture.md` | Current Espresso-09 machine and memory architecture |
| `Espresso-09-Logo.jpg` | Project artwork |
| `LICENSE.md` | Project license |

Schematic, programmable-logic, firmware, and emulator directories will be added as those parts of the project begin.

## Development notes

- Treat [Architecture.md](Architecture.md) as the current architectural reference when older commits disagree.
- The 8 KiB DAT model and use of LS612 `MA3` as the task selector are intentional architectural choices.
- The W65C22S is the selected host transport for the RX660 I/O Controller; exact VIA/RX660 signaling and the byte-level IOCALL protocol still require definition.
- Exact CPLD package, pin assignment, memory timing, and fixed-common backing still require implementation validation.
- Do not infer CoCo/GIME peripheral compatibility from the use of NitrOS-9 Level 2; Espresso-09 is its own pBITz machine.

## Related repositories

| Repository | Scope |
| --- | --- |
| [pBITzPlatform](https://github.com/dumaiss/pBITzPlatform) | Shared pBITz backplane, bus, mezzanine I/O hardware, and platform tools |
| [PercolatorLabs](https://github.com/dumaiss/PercolatorLabs) | Shared video and sound expansion-card development |
| [NitrOS-9](https://github.com/nitros9project/nitros9) | Target operating-system source tree |

## License

See [LICENSE.md](LICENSE.md).
