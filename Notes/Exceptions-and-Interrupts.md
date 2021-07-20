# Exceptions and Interrupts

Both of them alter the control flow of execution.

* Interrupts: Used to handle asynchronous events external to the processor
* Exceptions: Handle conditions detected by the processor itself while 
executing insns

There are two sources of interrupts and two sources of exceptions

* Interrupts
  * Maskable interrupts; signalling via the INTR pin.
  * Nonmaskable interrupts; signalling via the NMI pin.
* Exceptions
  * Processor-detected; these can be classified as _traps_, _faults_, or 
_aborts_
  * Programmed; using instructions like `INT n`. Those are usually called 
"software interrupts" but they're handled as exceptions.

## Identifying interrupts

Non-maskable interrupts and exceptions are assigned the interrupt numbers 0 
through 31.

Maskable interrupts are identifying by programming the external interrupt 
controller (e.g. Intel's Programmable Interrupt Controller).

Programming the PIC can be done by the software. Any number in the range \[32, 
255\] can be used.

| Identifier   | Description
---------------|-------------
| 0            | Divide error
| 1            | Debug exceptions
| 2            | Nonmaskable interrupt
| 3            | Breakpoint (one-byte INT 3 instruction)
| 4            | Overflow (INTO instruction)
| 5            | Bounds check (BOUND instruction)
| 6            | Invalid opcode
| 7            | Coprocessor not available
| 8            | Double fault
| 9            | (reserved)
| 10           | Invalid TSS
| 11           | Segment not present
| 12           | Stack exception
| 13           | General protection
| 14           | Page fault
| 15           | (reserved)
| 16           | Coprecessor error
| 17-31        | (reserved)
| 32-255       | Available for external interrupts via INTR pin

Exceptions are classified depending on the way they're reported and whether 
restarting the insn that caused the exception is supported

* Faults: Exceptions reported _before_ the insn causing the exception. Faults 
can be detected before or during the insn. If reported during the insn, the 
machine's state is first restored to its state before the insn.
* Traps: Exceptions reported at the insn boundary immediately _after_ the insn 
in which the exception was detected.
* Aborts: An abort is an exception which does not always report the location of
the instruction causing the exception and does not allow restart of the program
which caused the exception. Aborts are used to report severe errors, such as
hardware errors and inconsistent or illegal values in system tables.

Interrupts and exceptions are only served at instruction boundaries, i.e., 
between finishing one instruction and beginning another.

Instructions with repetitions may exhibit interrupt between repititions so that 
such instructions do not delay interrupt response.

While an NMI is executing, any NMI-signalled interrupts are differed until the 
`IRET` instruction is executed.

The `IF` _interrupt enable flag_, when set, INTR-signalled interrupts are 
serviced. When the processor is RESET, IF is cleared. `IF` is altered using 
`CLI` and `STI` instructions.

* `IF` is resotred by `POPF` when it follows a `PUSHF`. 
* `IRET` restores the state before the exception handler.
* Interrupts using interrupt gates automatically resets `IF`.

## Altering SS:

Altering SS that's followed by an ESP modification with an interrupt between 
the two instructions may result in an inconsistent pair of `SS:ESP`. 80386 
masks NMI and INTR, debug exceptions, and single-step traps between the two 
instructions. Example:  

```
  MOV SS, AX
  MOV ESP, StackTop
```

Page faults and protection faults may still occur on the instruction boundary 
between the two instructions.

## Priority of interrupts and exceptions

If more than one interrupt or exception is pending at in insn boundary, the 
processor services them one at a time.
The interrupt or exception of the highest-priority class; other lower-priority 
interrupts are held pending and other lower-priority exceptions are discarded; 
they're rediscovered when control flow returns from the interrupt handler.

|Priority    | Class of Interrupt or Exception      |
|------------|--------------------------------------|
| HIGHEST    | Faults except debug aults            |
|            | Trap instructions INTO, INT n, INT 3 |
|            | Debug traps for this instruction     |
|            | Debug faults for next instruction    |
|            | NMI interrupt                        |
| LOWEST     | INTR interrupt                       |


## Stack of Interrupts

Interrupts use stack to save data just like the CALL instruction. EFLAGS is 
saved before ther ISR is jumped-to.

Some exceptions (which?) push an error code to the stack.

Interrupts disable the trap flag (TF) that enables single-stepping while 
executing the ISR code. TF is restored to its previous value when IRET is
called.

## How an interrupt occurs?

How would a device, for example, the timer generate an interrupt?

### The PIC:

1. The timer is connected to the IR0 pin of the PIC, it asserts that pin.
2. PIC sets the bit representing the IRQ in the _Interrupt Request Register_.
3. The PIC examines the _Interrupt mask Register_ to see whether the interrupt
must be serviced or not.
4. The PIC notifies the processor about the interrupt

### The CPU:

1. First, it has to complete the instruction it's currently executing.
2. The CPU then checks the `IF` flag in `EFLAGS` (only if set the interrupt is serviced)
3. The CPU sends an acknowledgement for the PIC through the `INTA` pin.
4. The PIC places the vector of the interrupt on the data lines.
5. The PIC places the IRQ number on the data lines.
6. PIC's _In Service Register_ gets the bit corresponding to the interrupt asserted.

### Interruption Process:

1. The CPU saves the state of the current process (EFLAGS, CS, EIP, ..., etc)
2. Interrupt vector # given by the PIC is used to identify the ISR address.
    1. In protected mode, vector # is used as an index into the IDT
    2. In real mode, vector # is used as an index into the IVT.

#### Real Mode

1. CS:IP is loaded from the IVT entry, which transfers execution to the ISR.

#### Protected Mode

1. The interrupt gate is extracted from IDT[vector].
2. The selector field of the interrupt gate is loaded into CS.
3. The offset field is loaded into EIP.
5. CS:EIP is now loaded and control flow is transfered to the ISR.

### Completion of an Interrupt

Control flow was transferred to the ISR, which would take the appropriate action
to serve the device request for attention.

During the ISR, all interrupts are masked by the PIC waiting for an _End of
Interrupt_ (`EOI`) signal from the processor to unmask interrupts.

When the `EOI` signal is sent to the PIC, it clears the set bit in `In Service
Register`. 

## MISC

* Supporting exceptions/setting up IDT without programming the PIC would result
in unwanted interrupt 8 (exception fault). That's because the timer interrupt
IRQ line is connected to IR0 of the master PIC, which is mapped initially to
0x08.

* IRQ # -> Relevant to PIC. Numbers assigned to lines connected to devices.
* Vector # -> Index in the Interrupt vector table.
