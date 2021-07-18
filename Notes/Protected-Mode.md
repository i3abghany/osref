# Protected Mode

Protected mode memory addressing allows the processor to address more memory
than the real-mode memory (the first 1M bytes of memory)

Protected mode allows more modern features of microprocessors such as memory
protection and addressing segments larger than 64K bytes.

## Difference between real mode and protected mode

* Registers are extended to 32 bits with backward compatibility for `*X` registers.
* Two new segment registers: `fs` and `gs`.
* 32-bit offsets can be used. Thus, 4G bytes of memory could be addressed.
* More sophisticated memory segmentation mechanism:
  * Segments can be protected. Example: kernel code protected from user-space apps.
  * CPU can implement virtual memory.
* No BIOS interrupts.

## Switching to Protected Mode

1. Segmentation is supported by introducing a data structure called the __global descriptor table__.
This data structure defines memory segments, their lengths and their attributes.
2. Loading the `GDT` address to the `GDTR` by using a special instruction **XXX**
3. Setting the **XXX** bit in **XXX** register to instruct the processor to switch to PM.

## Living Without BIOS Interrutps?

BIOS routines are _not_ available in PM.
The OS must implement its own drivers for devices (screen, keyboard, ...)

### Printing to Screen

Screen can be configured into one of two modes:
* Text mode
* Graphics mode

What's displayed on the screen is an example of memory-mapped hardware.

Most computers boot up in a _Video Graphics Array_ `VGA` mode with 80*24
dimensions.

In text mode, a font is already defined in the VGA driver, so we don't need
pixel-by-pixel rendering.

Each character cell is represented by two bytes:
* The first: ASCII code fo the char.
* The second: Cell attributes.

VGA memory usually starts at `0xB8000`

## Global Descriptor Table

GDT is specific to the IA-32 architecture, and it contains information about
different segments in memory.

In protected mode, segment registers contain `Selector`, which is used as an
index to the GDT to reach for a certain `Segment Descriptor`.

GDT is loaded using the `lgdt` instruction, which expects an address of a data
strucutre called the GDT descritpor.

The GDT descriptor is a 6-byte data structure that consists of:
* The size of the GDT - 1 (two bytes)
* The starting address of the GDT (4 bytes)

Note: size - 1 because the GDT can be as much as 2^16 bytes (8192 entries * 8 bytes / entry), 
thats why size - 1 is passed.

A segment descriptor is an 8-byte data structure that holds information about
a segment in memory. Descriptor layout:

```
    31                23                15                7               0
    +-----------------+-----------------+-----------------+-----------------+
    |                 | | | |A|         | |     | |       |                 |
    |   BASE 31..24   |G|X|O|V| LIMIT   |P| DPL |0|  TYPE |  BASE 23..16    | 4
    |                 | | | |L| 19..16  | |     | |       |                 |
    |-----------------------------------+-----------------------------------|
    |                                   |                                   |
    |        SEGMENT BASE 15..0         |       SEGMENT LIMIT 15..0         | 0
    |                                   |                                   |
    +-----------------+-----------------+-----------------+-----------------+

            A      - ACCESSED
            AVL    - AVAILABLE FOR USE BY SYSTEMS PROGRAMMERS
            DPL    - DESCRIPTOR PRIVILEGE LEVEL
            G      - GRANULARITY
            P      - SEGMENT PRESENT
```

Descriptor fields:
* Base: The starting address of the segment
* G: Granularity.
  * If G == 0, length is in bytes, which means a segment can be at most 1M bytes.
  * If G == 1, length is in 4K bytes, which means that a segment can span a 4G bytes of memory
* A: Accessed. This bit is set when the the selector refering to this descriptor is loaded in a segment register.
* AVL: Ignored by the hardware, OS is free to use this bit however it wants.
* Limit: Length of the segment
* P: Segment Present. The descriptor is _not_ valid for use. The processor will raise an exception when a segment register is loaded with this descriptor's selector.
* DPL: Descriptor Privilage Level. Used for protection. Segment selector contains a privilege level that has to match DPL in order to gain access for the segment.
* TYPE: the type of descriptor. (code or data? extends-upwards or downwards? conforming or non-conforming? ... etc)

Intel CPUs require the first descriptor in the GDT to be a _null_ descriptor; i.e.
all zeros. This can be useful when a segment register is not loaded with a selector
so it has a zero. The CPU won't trigger an exception when a null selector is loaded
in a segment register, but it will trigger an exception when a null selector is used
to access memory.

## Flat Memory Model

In 80386, there's no way to entirely disable segmentation. However, the same
effect can be achieved through having static descriptors that do not change
and span all the 4G bytes address space. Once they're set, no need to change
them at all.

