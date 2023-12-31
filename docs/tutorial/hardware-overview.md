# Brief hardware overview

Before trying to design homebrew - be it a game, a demo or a tool - for a platform, it is a good idea to be aware of its general features and constraints.

The WonderSwan is a handheld console released by Bandai in March 1999, with an enhanced Color model released in December 2000. This part of the tutorial serves as a brief overview of its capabilities.

## CPU

The CPU is an NEC V30MZ CPU, clocked at 3.072 MHz. This CPU is compatible with the Intel 80186, an expanded version of the 16-bit 8086 CPU whose distant grandchildren power PCs to this day. However, note that the CPU has little to do with the older NEC V20/V30 - most of the extensions and quirks specific to that architecture are missing in the V30MZ.

The timing characteristics of the CPU are very different - in a good way! - to those of the early 80s chips. The V30MZ was designed in the 1990s, and features significantly better IPC (instructions per clock) than the original Intel part. Of particular note to prior 8-bit developers is the presence of a reasonably fast 32/16 divider (23-25 cycles), 16/8 divider (16-18 cycles) and 16/16 multiplier (3-4 cycles). Conversely, a prior 8086 developer should note that using look-up tables with the XLAT opcode (5 cycles) can often be slower than some simple operations, such as multiplication or shifting.

Note that while NEC provides their own opcode and register names, the Wonderful toolchain uses the Intel names. As such, you are better off using tutorials and guides targetting the 8086/80186.

### Banking and segmentation

While banking is common on many retro consoles, memory segmentation implemented in the 16-bit 8086 architecture is fairly unique - and the WonderSwan does banking *on top* of that. As such, it's a good idea to look at it closer.

The CPU exposes a 20-bit memory bus. This means it's capable of directly addressing 1 megabyte of memory. However, it's also a 16-bit CPU - the instruction pointer, data offsets and other registers are all, at most, sixteen bits in width. How does it address more memory?

The answer is *segmentation*. In addition to general registers, the CPU provides four *segment* registers:

- **CS** - the code segment; the instruction pointer is provided relative to the code segment.
- **SS** - the stack segment; the stack pointer is provided relative to the stack segment.
- **DS** - the data segment; any data pointers are accessed relative to the data segment.
- **ES** - the extra segment; some opcodes, such as `MOVSB` and `STOSW`, use it as an additional data segment.

A segment covers the top 16 bits of the 20-bit address (referred to as a "linear address"); that is, adding `1` to a segment register is equivalent to adding `16` to the address. Therefore, any memory access is done by multiplying the segment value by `16` and adding it to the 16-bit offset: `segment * 16 + offset`. For example, if the data segment is set to `0x3108`, while the data offset is set to `0x4240`, the linear address resolved will be `0x352c0`:

``` mermaid
graph TD
A[Data segment<br><b>0x3108</b>] --->|x 16| C
B[<b>Logical pointer</b><br>Data offset<br><b>0x4240</b>] --->|plus| C
C[<b>Linear pointer</b><br>CPU address space<br>0x3108 * 16 + 0x4240<br><b>0x352c0</b>]
```

This means that programming for the 8086 distinguishes between `near` (16-bit, containing only the offset) and `far` (32-bit, containing the segment *and* the offset) pointers.

In the Wonderful toolchain, we generally assume near data and stack pointers point to internal RAM, and therefore the segments are both typically set to `0x0000`. However, this means that placing even read-only data in the plentiful space of the cartridge's ROM, as opposed to the console's internal RAM, requires a special modifier - `__far`. However, the toolchain instead recommends using the `__wf_rom` modifier, which is also correctly configured on targets which *don't* utilize the ROM in the same manner.

!!! warning
    On the WonderWitch, a different assumption is made - the data segment is placed in the cartridge's SRAM, with only the stack in internal RAM. To ensure your code works with WonderWitch, it's a good idea to use the `__wf_iram` modifier on such pointers.

However, because *banking* is performed on top, a linear address in the cartridge ROM's space may not always point to the same physical location in the ROM! This is further discussed in the Cartridge section below.
As the naming can get confusing, the following convention has been established by this manual:

* **logical pointer** - 16-bit offset in the CPU address space, relative to a **segment**.
* **linear pointer** - 20-bit pointer to the CPU address space.
* **physical pointer** - pointer to a physical address space, such as the cartridge ROM.

## Memory

The WonderSwan features 16 kilobytes of internal memory. The WonderSwan Color bumps that to a total of 64 kilobytes. The address space is divided so that CPU addresses `0x00000 - 0x0FFFF` point to internal memory, while addresses `0x10000` - `0xFFFFF` are redirected to the cartridge.

Unlike many other 8/16-bit platforms, the WonderSwan has a unified memory architecture - CPU, video and audio data are all stored in the same internal memory at different locations.

The WonderSwan Color also adds general DMA, capable of efficiently copying data from the internal or cartridge memory to the internal memory.

## Graphics

The WonderSwan features a 224x144 display, capable of simultaneously[^1] displaying up to 8 out of 16 shades of gray, or 241 out of 4096 colors on the WonderSwan Color through:

[^1]: Without mid-line register manipulation tricks.

* a background color,
* two overlapping 32x32 tile layers ("screens"),
* up to 128 8x8 sprites, 32 per line.

Unusually, the display is clocked at 75.47 Hz.

The default mode for tile storage is to use 2-bit-per-pixel data plus one of sixteen palettes to select the shades or colors. The palettes are configured as follows:

* Palettes 0-15 are available for the screens, while only 8-15 are available for spirtes.
* Palettes 0-3 and 8-11 feature an opaque color "zero", while 4-7 and 12-15 assume it to be transparent.

The WonderSwan Color augments this with two additional modes, which allow displaying 4-bit-per-pixel tile data, albeit with color "zero" always transparent.

In addition to the display, there are also six user-controlled segments to the right or bottom of the console, which can be used as indicators.

Additional hardware features include:

* displaying sprites both beneath and on top of the second screen,
* limiting the second screen to be drawn inside or outside of a specified rectangle,
* limiting the sprites to be drawn inside a specified rectangle,
* hardware screen scrolling,
* hardware tile horizontal and vertical flipping.

## Sound

The WonderSwan features four channels of audio, each capable of playing 32 x 4-bit wavetable samples with a specified frequency and separate left/right volume.
The audio circuit is digital, outputting samples at 24000 Hz; the built-in speaker is mono, whereas the headphone output allows for stereo.

Channel 2 can be optionally configured to play 8-bit unsigned PCM samples, while channel 4 can be optionally configured to play pseudorandom noise.

The WonderSwan Color augments this with:

* "Hyper Voice", a fifth channel capable of playing stereo PCM samples,
* Sound DMA, allowing for relatively inexpensive copying of sample data from ROM or RAM to channel 2 or Hyper Voice.

## Input

The WonderSwan features eleven keypad inputs: the two directional diamonds on the left (Y1-Y4, X1-X4), as well as the A, B and Start buttons.

Notably, the arrangement of the directional diamonds allows an alternate layout, where the WonderSwan is held vertically.

## Cartridge

The cartridge is capable of anything, depending on the mapper chip. For the purposes of this guide, we're going to discuss the features of the official Bandai mapper chips - the 2001 and 2003.

The Bandai 2001 allows up to 16 megabytes of ROM data, while the 2003 allows up to 64 megabytes.

!!! warning
    Even though the theoretical limit of Bandai's chips is 64 MB, there are no commercial games larger than 16 MB, and no broadly available flash cartridges capable of playing games larger than 8 MB. *Caveat emptor.*

### Banking

Memory banking is provided by splitting the cartridge's part of the linear address space into four areas:

* `0x10000 - 0x1FFFF` - SRAM
* `0x20000 - 0x2FFFF` - ROM, bank 0
* `0x30000 - 0x3FFFF` - ROM, bank 1
* `0x40000 - 0xFFFFF` - ROM, "linear" bank 2

Banks 0 and 1 can be moved at runtime, as to point to any 64KB part of the ROM. Conversely, bank 2 can point to any 1 MB part of the ROM, but only its top[^2] 768 KB is exposed.

[^2]: When discussing memory, "bottom" typically refers to the *first* address of memory, while "top" refers to the *last* address of memory.

Let's look at a more practical example. Suppose that `bank 0` is set to point to the `0x680000` - `0x68FFFF` portion of the ROM's address space ("physical address"); as with the previous example in the *Segmentation* section, the data segment is set to `0x3108`, while the data offset is set to `0x4240`:

``` mermaid
graph TD
A[Data segment<br><b>0x3108</b>] --->|x 16| C
B[<b>Logical pointer</b><br>Data offset<br><b>0x4240</b>] --->|plus| C
C[<b>Linear pointer</b><br>CPU address space<br>0x3108 * 16 + 0x4240<br><b>0x352c0</b>] --->|cartridge bus| E
D[Bank 1 offset<br><b>0x68</b>] ---> F
E[Cartridge ROM bank 1<br><b>0x52c0</b>] ---> F
F[<b>Physical pointer</b><br>Cartridge ROM address space<br><b>0x6852c0</b>]
```

Optionally, a cartridge can provide:

* external EEPROM (2001 mapper),
* a real-time clock (2003 mapper),
* NOR flash, with write/erase access from the SRAM bank (2003 mapper).

## Other

Additional hardware features include:

* 128 bytes (extended to 2 kilobytes in the Color) of internal EEPROM, of which a small amount of bytes can be used by user programs,
* A 9600/38400 baud serial port, usable for connecting to a PC, another WonderSwan or a peripheral.

## More information

If you'd like to learn more about the WonderSwan's inner workings, I highly recommend [the WSdev Wiki](https://ws.nesdev.org/wiki/WSdev_Wiki) as a hardware reference guide. Additional resources are also listed as part of [awesome-wsdev](https://github.com/WonderfulToolchain/awesome-wsdev).
