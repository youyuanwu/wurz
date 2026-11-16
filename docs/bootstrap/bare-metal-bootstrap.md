# Bootstrapping hex0 without an operating system

Hex0 does not fundamentally need an operating system. It needs a way to receive
hexadecimal characters and somewhere to put the decoded bytes. On bare
hardware, a small monitor can provide those facilities directly.

The existing Linux seed cannot run unchanged: its file operations, startup
environment, and system calls depend on Linux. A bare-metal version must replace
those dependencies with an explicit hardware and execution contract.

This is a conceptual starting-point guide, not a tested implementation for a
particular board. For the physical creation of the first executable, see
[Hex0: the first steps from no compiler](./hex0-from-nothing.md). For the later
compiler stages, see the [Stage0 walkthrough](./stage0.md).

## 1. Choose what already exists

"No OS" can describe several different starting points:

| Starting point | Facilities available | Work still required |
| --- | --- | --- |
| Existing firmware monitor | Some memory access, I/O, and program-loading commands | A hex0 program compatible with that monitor. |
| Boot ROM or flash containing your code | A reset path into your instructions | Hardware startup, I/O, loading, and execution control. |
| Emulator with no guest OS | Emulated CPU, memory, devices, and a loading mechanism | Guest startup code and drivers for the selected machine. |

Using firmware avoids writing some startup code, but makes that firmware a
dependency. An emulator similarly remains part of the execution environment.
Neither approach requires an OS inside the target machine.

For a first implementation, choose a documented target with:

- One active CPU core, or a way to keep other cores idle.
- RAM that is usable immediately or simple to initialize.
- A UART serial interface.
- A known reset or entry address.
- A way to install the first binary.

A system with directly usable SRAM can be easier than one requiring complex
DRAM initialization. A screen, keyboard, disk controller, network stack, and
filesystem are not necessary for the first milestone.

## 2. Get the first instructions onto the machine

The CPU must reach code through its reset mechanism or an existing loader.
Possible installation methods include:

- Programming ROM or flash.
- Loading RAM through a hardware debugger.
- Using an existing firmware loader.
- Using an emulator's image-loading facility.
- Manually depositing bytes through suitable memory-entry hardware.

That initial code must already be machine code. A CPU does not decode a text
file containing hexadecimal digits unless some code or hardware performs that
conversion first.

During development, an assembler on another computer can produce this binary.
For a bootstrap with no compiler or assembler seed, the initial instruction
bytes must instead be manually encoded or independently reconstructed. External
programmers, development tools, and firmware remain trust assumptions unless
they are separately verified.

## 3. Establish the startup environment

The first code must establish, or explicitly rely on, the machine state it
needs:

1. Select the required execution mode and privilege level.
2. Make the intended RAM accessible.
3. Configure memory mapping and execution permissions if applicable.
4. Establish a correctly aligned stack before using routines that require it.
5. Mask ordinary interrupts until handlers exist, and keep other cores from
   interfering.
6. Initialize the UART, including any required clocks and pin configuration.
7. Initialize program data as required by the implementation.

For example, a C-written monitor generally needs its initialized data copied
into RAM and its zero-initialized data cleared before entering ordinary C code.
An assembly-only seed can have a smaller startup contract.

These are responsibilities, not a universal instruction sequence. Their order
and implementation depend on the target. A Linux process normally receives many
of these facilities from the kernel; bare-metal code does not.

## 4. Bring up serial I/O first

Start with two small operations:

```text
get_character:
    Poll the UART until a received character is available.
    Read and return that character.

put_character:
    Poll the UART until it can accept a character.
    Write the character.
```

The actual register addresses, status bits, and I/O instructions come from the
board and device documentation. Some machines use memory-mapped registers;
others have separate I/O instructions.

Before attempting hex0, verify that the monitor can:

1. Print a fixed startup message.
2. Receive and echo a character.
3. Write a known value into a reserved RAM location and read it back.

Polling avoids needing an interrupt-driven driver or scheduler. Match serial
settings on both ends, and use flow control or acknowledgments when needed:
fast transmission can overflow a small receiver.

A terminal is merely an interface to this channel. It does not interpret
machine code or make the target understand hexadecimal input. The monitor does.

## 5. Decode hexadecimal input into RAM

The Linux arrangement is:

```text
Input file -> system-call reads -> hex0 -> system-call writes -> output file
```

A simple bare-metal arrangement is:

```text
Serial input -> UART driver -> hex0 decoder -> reserved RAM buffer
```

The decoder combines hexadecimal digit pairs into bytes and supports the chosen
hex0 comment and separator rules. It does not need to understand instructions
or executable headers.

However, a serial stream has no file EOF. A surrounding loading protocol must
define:

- The destination address and maximum permitted size.
- How a transfer starts and ends.
- The expected decoded byte count.
- How to detect corruption, such as a checksum.
- How to acknowledge success or report and recover from failure.

For example, a monitor could accept a load request, receive a framed block of
hexadecimal text, then verify its decoded length and checksum. This framing is
an additional monitor protocol, not part of plain hex0.

Bounds-check writes before they happen, including address arithmetic. Reject
transfers that would overwrite the monitor, stack, device registers, or reserved
memory. Report incomplete byte pairs, transfer errors, and checksum mismatches;
do not treat a partial image as successfully loaded.

A checksum detects some transmission errors. It does not authenticate code or
make it safe to execute.

## 6. Keep the loader and loaded program separate

A conceptual memory layout is:

```text
Boot ROM / flash
    Reset code and possibly the monitor

RAM
    Monitor code, if copied into RAM
    Monitor data and receive buffer
    Monitor stack
    Loaded program and its data
    Loaded program's stack and working memory
```

Actual addresses must come from the target's memory map. Define non-overlapping
ranges and ensure that stack growth and temporary buffers stay inside them.

Without a filesystem, source input can be received into one buffer and generated
output placed into another. Preserve both until translation has completed.
Results can be sent back over serial for storage on another machine.

An editor is unnecessary if sources arrive through this channel. If sources
must be created locally, a primitive text-entry facility can precede a fuller
editor.

## 7. Define how a loaded program starts and finishes

Writing bytes into RAM is not enough. The monitor and loaded program must agree
on an execution contract:

| Item | Required agreement |
| --- | --- |
| Instruction set | CPU architecture, mode, and supported instructions. |
| Image layout | Load address, entry address, code, data, and working-memory ranges. |
| Addressing | Fixed addresses, position-independent code, or explicit relocation. |
| Stack | Its location, size, alignment, and which component initializes it. |
| Input/output | Direct device access or calls to monitor-provided routines. |
| Arguments | Where input buffers, lengths, and service addresses are passed. |
| Completion | Return to the monitor, invoke an exit service, or require reset. |
| Failure | How faults are reported and how control can be recovered. |

A monitor can expose a few ordinary callable routines such as character input
and output. These do not need Linux system-call semantics. Their calling
convention and register-preservation rules must still be defined.

Before transferring control:

1. Check that the entire image was received and validated.
2. Confirm that its entry point is in the intended executable region and meets
   the target's alignment requirements.
3. Complete any required data/instruction cache synchronization.
4. Establish the agreed stack, registers, and interrupt state.
5. Branch or call according to the agreed return convention.

On some architectures, instruction fetch will not see newly written code
without explicit synchronization. Consult the processor's requirements.

For a first test, use a program that prints one character and returns through
the agreed interface. Do not start by loading a compiler.

Bare-metal programs commonly run with enough privilege to overwrite the
monitor. Loader bounds checks prevent loading mistakes, not arbitrary behavior
after execution. Only run code you trust, and retain a reset or debugger path.

## 8. Why the POSIX seed is not directly reusable

The Linux AMD64 seed contains an ELF image and relies on Linux startup state and
system calls. A bare-metal port must account for all of those dependencies.

Removing the ELF header does not fix them. The program can still contain:

- Addresses calculated for its original placement.
- Assumptions about the initial stack and command-line arguments.
- Instructions requesting kernel services that do not exist on bare hardware.

A simple monitor can load a flat binary at a fixed address. Alternatively, it
can implement an appropriate executable loader. Supporting ELF loading alone
still does not supply the Linux services a Linux executable expects.

Port the decoder's input/output boundary and startup contract rather than
assuming that changing its file format is sufficient.

## 9. Rebuild hex0 and continue toward C

Once the monitor and bare-metal hex0 work:

1. Supply the hexadecimal source of the bare-metal hex0 implementation.
2. Decode it into a separate output buffer.
3. Compare the output with the expected seed bytes.
4. Load and execute it under the same execution contract.
5. Repeat the process for increasingly capable tools.

If the rebuilt program is executed at a different address, it must be
position-independent, relocated, or rebuilt for that address. A byte-for-byte
copy of fixed-address code is not automatically runnable elsewhere.

The conceptual progression remains:

```text
Reset / initial loader
        |
        v
Startup code + UART routines + monitor
        |
        v
Bare-metal hex0
        |
        v
hex1 -> hex2 -> macro assembler -> assembly-written C compiler
        |
        v
C-written compiler and supporting tools
```

This is not an unchanged execution of the Stage0 POSIX scripts. Each tool needs
compatible startup and runtime support. File-based programs may need buffer or
stream adapters, a simple storage layer, or broader OS-like services. A compiler
also needs enough memory for its sources, internal state, and generated output.

It is possible to stop well short of a full OS: one program at a time, fixed
memory regions, serial transport, and a small shared runtime can support an
initial toolchain. Increasing compatibility with POSIX programs adds work that
the Linux path gets from an existing kernel.

## 10. A practical milestone checklist

- [ ] Reset or the initial loader reaches your startup code.
- [ ] RAM and the stack behave as expected.
- [ ] A startup message and serial echo work.
- [ ] Hexadecimal input produces the expected bytes in reserved RAM.
- [ ] Bad lengths, invalid addresses, incomplete input, and corrupt transfers
      produce explicit failures without executing the image.
- [ ] A small loaded program prints a character and returns.
- [ ] The bare-metal hex0 source reproduces its expected executable bytes.
- [ ] Sources and generated outputs can be preserved between tool invocations.
- [ ] The next translator runs using the same documented execution contract.

An emulator is a useful first target because memory and CPU state are easier to
inspect. It can run with no guest OS, even though its host uses one. Installing
the initial image through the emulator is still part of the bootstrap boundary.

## Further reading

- [Hex0 and the initial byte-entry problem](./hex0-from-nothing.md).
- [Stage0's path to a C compiler](./stage0.md).
- [Original Stage0 monitor and stage descriptions](https://github.com/oriansj/stage0/blob/master/README.org).
- [Linux AMD64 hex0 implementation](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/hex0_AMD64.hex0).

The core idea is to replace implicit OS services with a small, explicit
execution environment: **startup code, input/output, memory ownership, and a
defined way to load and run the next program.**
