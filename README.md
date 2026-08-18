# Espresso-09

**A 6309 member of the pBITz coffee-machine family**

Espresso-09 is a modular homebrew computer built around the Hitachi HD63C09E/HD63C09EP. It combines a real 6309 CPU, a 74LS612 memory mapper, **1 MiB of SRAM**, **512 KiB of ROM**, and the shared pBITz expansion architecture used by the other coffee-series machines.

The primary operating-system target is **NitrOS-9 Level 2**. The memory system is deliberately organized around NitrOS-9's native 8 KiB DAT model rather than reproducing the CoCo 3 GIME in programmable logic.

Espresso-09 is currently an architecture and hardware-design project. The repository is being used to capture the machine definition before schematic capture and firmware bring-up begin.

## Current status

| Subsystem | Status |
| --- | --- |
| HD63C09E/HD63C09EP CPU architecture | Selected |
| 74LS612-based MMU architecture | Defined at architectural level |
| 1 MiB RAM + 512 KiB ROM physical map | Defined |
| 2 MiB physical address envelope | Defined |
| 8 KiB NitrOS-9-compatible DAT model | Defined |
| Fixed pBITz I/O window | Defined |
| Independent `DAT_BANK` / `SYS_MODE` privilege model | Defined at architectural level |
| BA/BS hardware privilege-entry gate | Defined at architectural level; timing/equations pending |
| 16-bit unprivileged pBITz permission mask | Defined at architectural level |
| ROM-to-RAM instant-on boot model | Defined at architectural level |
| ATF15xx control/decode CPLD | Architecture defined; device/package, budget, and equations pending |
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
- CPU `BA`/`BS` bus-state signals are required by the current privilege-entry architecture
- 6309-specific instructions such as `TFM` are expected to be useful for ROM shadowing and bulk-memory operations

### Physical memory

Espresso-09 uses a **2 MiB physical address envelope** with **1.5 MiB populated**:

| Physical range | Block range | Size | Function |
| --- | --- | ---: | --- |
| `000000h-0FFFFFh` | `00h-7Fh` | 1 MiB | SRAM |
| `100000h-17FFFFh` | `80h-BFh` | 512 KiB | Reserved / unpopulated |
| `180000h-1FFFFFh` | `C0h-FFh` | 512 KiB | ROM |

The physical address space is divided into 256 blocks of 8 KiB each. Normal NitrOS-9 RAM allocation therefore has a clean contiguous pool of 128 blocks, `00h-7Fh`, while firmware occupies the top 64 blocks, `C0h-FFh`.

The reserved `80h-BFh` region is intentionally left outside the initial RAM and ROM population. It may remain unused or support a future hardware revision without changing the DAT block format.

### Memory mapper and privilege state

The **74LS612** is used as the actual address-translation store and mapper. Its sixteen mapping registers are divided into two complete 64 KiB hardware DAT banks:

- entries 0-7: **hardware DAT bank 0**, the resident system map
- entries 8-15: **hardware DAT bank 1**, the working map used for the currently required NitrOS-9 software task

CPU `A15..A13` select one of eight 8 KiB logical blocks. The LS612 `MA3` input is driven by the CPLD's `DAT_BANK` state. NitrOS-9 software task numbers remain a separate concept: their eight-byte DAT images live in kernel RAM and are copied into the working hardware bank when required.

Privilege is also separate from DAT-bank selection. The CPLD holds a one-bit `SYS_MODE` latch:

```text
DAT_BANK=0, SYS_MODE=1   normal kernel/system execution
DAT_BANK=1, SYS_MODE=1   privileged working-map execution
DAT_BANK=1, SYS_MODE=0   ordinary unprivileged process execution
```

The `DAT_BANK=0, SYS_MODE=0` combination is forbidden.

On the CPU's interrupt/reset acknowledge bus state, the CPLD latches `DAT_BANK=0, SYS_MODE=1`. Interrupt and software-interrupt vectors come from the fixed vector overlay and enter fixed trampoline code, so unprivileged software never has to be allowed to write the task/privilege control register in order to enter the kernel.

With a 2 MiB physical envelope, each physical block number is exactly eight bits. Eight of the LS612's twelve stored bits therefore form the physical block number, leaving four bits for Espresso-specific attributes such as write protection, system-only mappings, invalid-page detection, and one future use.

See [Architecture.md](Architecture.md) for the current memory, privilege-entry, and boot architecture.

## Logical address space

The current architectural baseline keeps almost the entire 64 KiB CPU address space available as DAT-translated memory while reserving small fixed windows at the top:

| Logical range | Function |
| --- | --- |
| `0000h-FDFFh` | DAT-translated memory |
| `FE00h-FEFFh` | Fixed pBITz I/O aperture |
| `FF00h-FF1Fh` | Fixed MMU/control registers |
| `FF20h-FFEFh` | Fixed common RAM / DAT-bank and privilege trampoline area |
| `FFF0h-FFFFh` | Fixed vector window |

The exact implementation of the common/vector backing store remains part of the hardware design, but these windows are intentionally independent of the currently selected DAT bank.

### pBITz I/O compatibility

The `FE00h-FEFFh` window preserves the pBITz 4-bit Device Select convention. On an access to this window, the Espresso CPLD uses CPU address bits `A7..A4` to drive the four pBITz bus control signals `CS3..CS0`:

```text
CPU address within FE00h-FEFFh

A7..A4  -------->  Espresso CPLD  -------->  pBITz CS3..CS0
                                                    |
                                                    v
                                      compared with card rotary setting
```

Peripheral cards select themselves by comparing the bus `CS3..CS0` value with their configured 4-bit Device Select. The address bits themselves are not the card-select interface.

Once selected, a card uses whichever normal address lines it needs for local register decoding—typically `A0`, `A1`, `A2`, and, for a full sixteen-register aperture, `A3`.

Privileged execution (`SYS_MODE=1`) always has pBITz access. Unprivileged access (`SYS_MODE=0`) is controlled by a **16-bit `USER_IO_MASK`**, one permission bit per Device Select. The reset/default NitrOS-9 policy is `0000h`, forcing ordinary user processes through operating-system drivers unless the kernel deliberately grants direct access to a device.

The shared backplane and expansion-bus hardware are maintained in the [pBITzPlatform repository](https://github.com/dumaiss/pBITzPlatform).

## Boot and ROM shadowing

Espresso-09 is intended to provide an **instant-on NitrOS-9** experience from ROM while executing normal runtime code from RAM.

At reset, the LS612 enters pass mode so the first 64 KiB of RAM is available with a known identity mapping. The CPLD starts in `DAT_BANK=0, SYS_MODE=1` and overlays a boot ROM region at the top of the logical address space so the 6309 can fetch its reset vector and execute the bootstrap without relying on uninitialized mapper contents.

The final ROM block is physical block `FFh`, so the reset vectors can live naturally at physical addresses `1FFFFEh-1FFFFFh` and be overlaid into logical `FFFEh-FFFFh` during boot.

The normal startup sequence is expected to:

1. reset into LS612 pass mode with `DAT_BANK=0, SYS_MODE=1`
2. execute the bootstrap from the ROM overlay
3. initialize hardware DAT bank 0 and hardware DAT bank 1
4. initialize unprivileged I/O permissions
5. enter LS612 map mode while keeping the boot ROM overlay active
6. copy the ROM image into RAM in 8 KiB blocks using temporary source/destination windows and `TFM`
7. install the runtime NitrOS-9 system map and fixed common/vector state
8. transfer execution to a RAM-resident handoff stub
9. disable the boot ROM overlay
10. continue into NitrOS-9 from RAM

A full 512 KiB shadow maps ROM blocks `C0h-FFh` to RAM blocks `00h-3Fh`, leaving RAM blocks `40h-7Fh`—the upper 512 KiB—untouched and immediately available. Runtime code may later reuse any shadowed block that does not contain live system/module data.

Normal runtime mappings are expected to use RAM. ROM remains the reset/firmware source and may still be explicitly mapped for diagnostics or recovery if the final CPLD policy permits it.

## NitrOS-9 Level 2

NitrOS-9 Level 2 is the primary operating-system target.

The hardware is designed around its native memory-management model:

- eight 8 KiB DAT blocks per 64 KiB process address space
- a permanent system hardware DAT bank
- one working hardware DAT bank
- NitrOS-9 software task numbers `0..31`, each represented by an eight-byte DAT image in kernel memory
- context switches implemented by loading the selected software task's eight physical-block bytes into LS612 hardware DAT bank 1 when required
- cached working-bank reuse when the same unchanged DAT image remains resident
- 128 physical RAM blocks (`00h-7Fh`) available to the Espresso NitrOS-9 port

The **1 MiB RAM target** is intended to provide comfortable headroom for an EOU-style NitrOS-9 environment, multiple resident processes, graphics/window services, modules, buffers, and development tools without changing the native 8 KiB DAT model.

The Espresso port deliberately adds a real privilege boundary that stock CoCo 3 NitrOS-9 does not have. User→kernel entry therefore relies on the hardware `BA`/`BS` acknowledge gate, while kernel return to user execution is explicit from the fixed privileged trampoline.

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
- The **1 MiB RAM + 512 KiB ROM / 2 MiB physical envelope** is the current target memory configuration.
- The two LS612 halves are **hardware DAT banks**, not NitrOS-9 software task numbers.
- `DAT_BANK` selects LS612 `MA3`; `SYS_MODE` is the independent hardware privilege state.
- The `BA`/`BS` interrupt/reset acknowledge state provides the one-way hardware entry from unprivileged execution to `DAT_BANK=0, SYS_MODE=1`.
- The 16-bit `USER_IO_MASK` controls pBITz permissions only while `SYS_MODE=0`; privileged execution bypasses it regardless of DAT bank.
- Non-default `MMU_ATTR` programming is a two-write critical sequence: IRQ/FIRQ must be masked, and every mapper-entry write clears the attribute staging-valid state.
- The W65C22S is the selected host transport for the RX660 I/O Controller; exact VIA/RX660 signaling and the byte-level IOCALL protocol still require definition.
- **CPLD macrocell/product-term/pin budgeting is now a near-term design task** before the final ATF15xx device/package is selected.
- Exact CPLD package, pin assignment, memory timing, privilege-entry timing, and fixed-common backing still require implementation validation.
- Do not infer CoCo/GIME peripheral compatibility from the use of NitrOS-9 Level 2; Espresso-09 is its own pBITz machine.

## Related repositories

| Repository | Scope |
| --- | --- |
| [pBITzPlatform](https://github.com/dumaiss/pBITzPlatform) | Shared pBITz backplane, bus, mezzanine I/O hardware, and platform tools |
| [PercolatorLabs](https://github.com/dumaiss/PercolatorLabs) | Shared video and sound expansion-card development |
| [NitrOS-9](https://github.com/nitros9project/nitros9) | Target operating-system source tree |

## License

See [LICENSE.md](LICENSE.md).
