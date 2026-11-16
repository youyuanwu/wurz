# Hex0: the first steps from no compiler

How can a machine run the first compiler if there is no compiler to build it?
The answer is to start below compilers: choose machine-code bytes by hand and
provide a way to place them where the CPU can execute them.

Hex0 makes the next step easier. It translates a readable hexadecimal
description of bytes into the bytes themselves. It does not understand C,
assembly mnemonics, or executable formats.

This document explains that initial boundary. For the later path to a
self-hosting C compiler, see [How Stage0 bootstraps a C compiler](./stage0.md).

For startup code, serial input, and loading programs without a kernel, see
[Bootstrapping hex0 without an operating system](./bare-metal-bootstrap.md).

## 1. What does "from nothing" mean?

There is no useful bootstrap from literally nothing. A CPU needs power, memory,
a defined starting state, and some mechanism for receiving instructions.
Different starting assumptions move the boundary:

| Starting environment | What is already available | What must be supplied |
| --- | --- | --- |
| Linux, but no compiler | Kernel, executable loader, filesystem, program launcher | A compatible hex0 executable and source inputs. |
| Firmware monitor, but no OS | Memory access, program loading, some I/O and a way to start execution | A program written for that monitor's machine state and interfaces. |
| Hardware with no software monitor | CPU, memory, reset behavior, and some physical way to load code | Initial code, machine initialization, and any required I/O support. |

A shell, firmware monitor, ROM programmer, and front-panel switches are
different ways of getting across this boundary. None is automatically present
on every machine.

In particular, the Linux hex0 seed is **not** a bare-metal bootloader. It expects
Linux services and cannot simply replace a board's reset firmware.

## 2. Text describing bytes is not the bytes

Consider this text:

```text
31 C0
```

In an ASCII-compatible file, those characters occupy five bytes:

```text
33 31 20 43 30
```

Hex0 converts them to two bytes:

```text
31 C0
```

On x86-64, those two output bytes encode `xor eax, eax`. The CPU executes the
binary instruction, not the characters spelling its hexadecimal representation.
On a different CPU architecture, the same bytes need not mean the same thing.

This is the initial bootstrap problem: **something must convert or directly
enter the first instruction bytes before hex0 exists.**

## 3. Design the first program on paper

First choose a target and an execution contract:

- Which CPU instruction set and operating mode?
- Where will the program be loaded?
- Where does execution begin?
- Is a stack already established, or must the program create one?
- How does the program receive input and deliver output?
- How does it stop or return control?

For Linux AMD64, the kernel supplies the process environment and initial stack,
and the program can open, read, and write files through system calls. A
bare-metal implementation must instead initialize or explicitly rely on the
corresponding hardware or firmware facilities.

The conceptual hex0 algorithm is small:

```text
For each input character:
    If inside a line comment:
        Leave comment mode at the end of the line.
    Otherwise, if the character starts a comment:
        Enter comment mode.
    Otherwise, if it is a hexadecimal digit:
        Save the first digit of a pair.
        On the second digit, output 16 * first_digit + second_digit.
    Otherwise:
        Skip separators such as whitespace.
```

The Stage0 format supports comments beginning with `#` or `;`. A source file
should provide complete byte pairs. Do not assume a tiny seed has the error
checking or malformed-input diagnostics of a general-purpose assembler.

There are no symbolic labels, instruction names, or automatic jump calculations
at this stage.

## 4. Turn the design into machine-code bytes

Using the CPU's instruction documentation, a programmer selects instructions
and encodes them manually.

For example, on x86-64:

| Instruction | Encoding |
| --- | --- |
| `xor eax, eax` | `31 C0` |
| `syscall` | `0F 05` |

These are instruction examples, not a complete runnable program. A system call
also needs the appropriate number and arguments in registers.

The programmer must also calculate:

- Instruction lengths and positions.
- Immediate values and their byte order.
- Relative jump displacements.
- Any absolute addresses.
- Loader headers, if the execution environment requires them.

For a concrete jump calculation, an x86 short jump beginning at address
`0x0100` occupies two bytes. To jump to `0x0108`, the displacement is measured
from the next instruction, at `0x0102`:

```text
displacement = 0x0108 - 0x0102 = 6
encoding     = EB 06
```

When instructions move, affected offsets must be recalculated. Removing this
manual work is one reason the later hex1 stage adds labels.

### The Linux executable also needs headers

An instruction stream alone is not normally a Linux executable file. The
inspected AMD64 hex0 source includes an ELF header and a program header before
its instructions. They tell the loader how to map the file and where to begin.

That particular source describes:

- A mapping starting at virtual address `0x600000`.
- An entry point at `0x600078`, after 120 bytes of headers.
- A complete file length of `0xE5`, or 229 bytes.

These values belong to the
[inspected AMD64 implementation](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/hex0_AMD64.hex0),
not to a universal hex0 specification.

## 5. Produce the first executable

There are two broad approaches. They solve the same byte-entry problem with
different assumptions.

### Approach A: Use an existing computer

On a working development machine:

1. Write or obtain the commented hexadecimal source.
2. Audit the instruction bytes and executable layout.
3. Use a hex decoder, assembler, or independently written conversion program
   to write the intended binary.
4. Inspect the result and set executable permissions if required.
5. Transfer it to the target if the target is another machine.

The [bootstrap-seeds documentation](https://github.com/oriansj/bootstrap-seeds/blob/cedec6b8066d1db229b6c77d42d120a23c6980ed/README.md)
describes approximating hex0 with existing `sed` and `xxd` tools.

An editor is convenient for writing the source, but not fundamental. The text
could also arrive from another machine, removable storage, or a simple input
program. The tools used to manufacture the seed do not need to be installed
on the target afterward.

However, tools used without independent verification remain part of the trust
assumptions of that manufacturing process. This is a way to construct or
reconstruct a seed, not a claim about how every distributed seed was
historically produced.

### Approach B: Enter bytes through hardware or a minimal monitor

Without a development environment, possible entry mechanisms include:

- Front-panel controls that deposit bit patterns into memory.
- A programmer that writes a ROM or flash device.
- A firmware monitor that writes memory and transfers control to an address.

A programmer attached to another computer moves part of the work to that other
computer; it does not eliminate the dependency.

For a suitable bare-metal target, the early sequence could be:

```text
Physical byte-entry mechanism
        |
        v
Small machine-code monitor
        |
        v
Hexadecimal input -> bytes in memory or storage
        |
        v
Stored hex0 translator and source inputs
        |
        v
More capable bootstrap tools
```

The monitor itself must be entered as bytes or supplied by existing firmware.
Its I/O and storage routines must match the hardware. This is a conceptual
route, not a claim that the Linux seed includes those routines.

The original Stage0 design distinguishes a `Hex0_monitor` from the later hex0
translator. Its monitor can accept hexadecimal programs and record text, so
it can help create source files before a fuller editor exists.

## 6. Does this require a terminal or editor?

No particular terminal or editor is mandatory, but input cannot happen without
some mechanism that the machine understands.

| Component | Role | Required? |
| --- | --- | --- |
| Terminal | Sends input and displays output through an existing interface. | Only if the chosen loading or interaction method uses it. |
| Editor | Creates and changes source text. | No; source can be prepared or supplied by other means. |
| Monitor | Receives commands to inspect memory, load bytes, or start programs. | One option; its own implementation must already exist or be entered. |
| Loader | Establishes the execution environment and starts the program. | Some equivalent mechanism is necessary; it need not be an OS loader. |
| Byte-entry mechanism | Gets the initial code into accessible memory or storage. | Yes, in some form. |

A serial terminal does not magically make a CPU accept hexadecimal text.
Firmware, software, or hardware must interpret the input and place the
corresponding bytes in memory.

The POSIX hex0 program itself reads and writes named files using system calls.
It does not need an interactive terminal or editor during translation.
Something must still launch it and supply its arguments.

## 7. Let hex0 rebuild itself

Once an executable seed exists, supply the hexadecimal source describing that
same program:

```text
First hex0 executable + hex0 source text
                    |
                    v
             Rebuilt executable
```

The AMD64 seed-stage script expects the rebuilt executable to match the seed
byte-for-byte. This checks an important consistency property: the readable
byte description and executable agree when processed by the seed.

This is not a proof of correctness. A faulty or malicious seed could behave
correctly on its own source and differently on other inputs. Auditing or
independently reconstructing the seed remains important.

Also, hex0 is not generating its own source. A human or an earlier process must
provide that source, and the sources for subsequent tools.

## 8. The next steps

After hex0, Stage0 incrementally removes the most painful manual work:

```text
hex0    Write explicit bytes and calculate addresses manually.
hex1    Use simple labels and calculated relative references.
hex2    Use longer labels and absolute-address references.
M0      Name instruction encodings through macros.
C       Build an assembly-written compiler, then a C-written compiler.
```

On the POSIX path, build orchestration is a separate concern: an existing
shell or the optional kaem seed launches the commands. Hex0 translates bytes;
it is not a shell or a build-script interpreter.

See the [Stage0 walkthrough](./stage0.md) for the remaining compiler and
toolchain stages.

## References

- [Original Stage0 monitor and stage descriptions](https://github.com/oriansj/stage0/blob/master/README.org).
- [Commented AMD64 hex0 source](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/hex0_AMD64.hex0).
- [AMD64 seed-stage build script](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/mescc-tools-seed-kaem.kaem).
- [Seed construction and audit guidance](https://github.com/oriansj/bootstrap-seeds/blob/cedec6b8066d1db229b6c77d42d120a23c6980ed/README.md).

The essential boundary is physical: **humans choose the initial bytes, some
mechanism loads them, and only then can software take over the translation
work.**
