# Interrupt Descriptor Table

A mapping between each vector interrupt number to a descriptor for the
instructions to be fired when the interrupt occur.

Each descriptor is 8 bytes and the first entry may contain an entry, unlike the
GDT.

There are only 256 identifiers for interrupts. IDT may be maximum of 256
entries. Entries are required for those interrupts that actually occur only.

The instruction `lidt` is used to load the address of an IDT descriptor into the
interrupt descriptor table register (IDTR). Can only be executed with CPL = 0

The instruction `sidt` stores the value of the IDTR to an operand memory
location. This can be executed on any previlige level.

The IDT descriptor is a 6-byte data structure that contains the base address of
the IDT and the limit (i.e. the size of IDT in bytes).

Interrupt IDT entry:

     31                23                15                7                0
      +-----------------+-----------------+-----------------+-----+-----------+
      |           OFFSET 31..16           | P |DPL|0 1 1 1 0|0 0 0|(NOT USED) |4
      |-----------------------------------+-----------------------------------|
      |             SELECTOR              |           OFFSET 15..0            |0
      +-----------------+-----------------+-----------------+-----------------+

