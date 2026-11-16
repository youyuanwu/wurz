# How Stage0 bootstraps a C compiler

Stage0 builds a compiler toolchain by starting with a very small machine-code
program and adding capabilities in small, inspectable steps. It replaces the
usual dependency on a prebuilt C compiler with a chain that can be traced back
to individual instruction bytes.

The central rule is:

> Every new tool must be buildable using tools that already exist.

This document focuses on the Linux/POSIX path in
[stage0-posix](https://github.com/oriansj/stage0-posix), using its AMD64
implementation as a concrete example. The original
[stage0 project](https://github.com/oriansj/stage0) also covers lower-level
bootstrapping, including a monitor for entering hexadecimal programs.

For the earlier boundary of creating and loading the first executable without
a compiler, see [Hex0: the first steps from no compiler](./hex0-from-nothing.md).

## 1. What is the starting point?

The first executable is a small program called `hex0`. Its job is to translate
hexadecimal text into bytes. It is not a C compiler.

To create the initial executable without an assembler:

1. Write down the machine instructions needed to read, decode, and write bytes.
2. Encode those instructions using the CPU instruction-set specification.
3. Calculate addresses and jump offsets manually.
4. Include the executable header required by the loader, where applicable.
5. Put those bytes into a file or load them directly into memory.

Normal users of Stage0 receive an existing seed executable instead of entering
it manually. That seed should be audited or independently reconstructed; its
small size makes this more practical than auditing an entire prebuilt compiler.

### What must already exist?

For the POSIX path, the seed still needs:

- A compatible CPU or emulator.
- A running kernel with the system-call interface expected by that port.
- A loader capable of executing the seed.
- Source files and a filesystem it can read and write.
- A way to invoke programs and pass their arguments.

The Linux seed uses system calls rather than depending on a preinstalled C
standard library. This does not mean that it is independent of the kernel.

Starting on bare hardware requires additional work for loading programs,
storage, and device I/O. Stage0 does not make those dependencies disappear.
See [Bootstrapping hex0 without an operating system](./bare-metal-bootstrap.md)
for that starting point.

### How small is the seed?

The stage0-posix README describes a 256-byte starting seed, but that is not a
universal size for every architecture and revision. At the inspected
[bootstrap-seeds revision](https://github.com/oriansj/bootstrap-seeds/tree/cedec6b8066d1db229b6c77d42d120a23c6980ed/POSIX/AMD64),
the AMD64 `hex0-seed` is **229 bytes**. The optional `kaem-optional-seed`
command runner is a separate **618-byte** executable.

These are seed sizes, not the size of the resulting C compiler or the total
trusted computing base.

## 2. The bootstrap ladder

The main compiler-building path looks like this:

```text
Machine-code hex0 seed
        |
        v
hex0: hexadecimal text -> bytes
        |
        v
hex1: add simple labels and relative-address calculation
        |
        v
hex2-0: add long labels and absolute addresses
        |
        v
M0: macro assembly -> hex2 text
        |
        v
cc_amd64: a minimal C compiler written in assembly
        |
        v
Initial M2-Planet: a C-subset compiler written in C
        |
        +--> build better assembler, linker, runtime, and build tools
        |
        v
M2-Planet rebuilt using M2-Planet
```

The arrows show build dependencies, not a single program processing all inputs.
For example, after M0 exists, its output still needs to pass through hex2 to
become an executable.

### Step A: Rebuild hex0

Hex0 recognizes pairs of hexadecimal digits, whitespace, and line comments.
Conceptually:

```text
Read input characters.
Skip whitespace and comments.
Combine each pair of hexadecimal digits into a byte.
Write the byte.
```

For example, this hex0 input represents four data bytes:

```text
00 01 7F FF  # Example data, not a runnable program.
```

There are no instruction names or symbolic labels. The author supplies the
actual instruction encodings, executable header bytes, and offsets.

The seed first translates the hexadecimal source of hex0 itself into a new
executable. The AMD64 seed script expects that executable to be byte-for-byte
identical to the supplied seed. This connects the executable to a readable,
commented representation of its bytes.

It does not, by itself, prove that the seed is correct or trustworthy.

### Step B: Build hex1

Hex1 is written in hex0 format, so hex0 can build it.

Hex1 introduces single-character labels and relative references. This removes
some manual jump-offset arithmetic: the programmer can identify a destination
symbolically, and the tool calculates the displacement.

The new capability is small, but it makes writing the next tool much easier.

### Step C: Build hex2-0

The initial hex2 implementation is written in hex1 format. It adds longer label
names and absolute-address references.

These features let it resolve addresses across larger inputs and act as a
simple linker. It is not a general-purpose linker for arbitrary ELF object
files; it works with the bootstrap's hex2 representation.

The early architecture-specific executable is called `hex2-0`. A more capable
C implementation replaces it later.

### Step D: Build M0

M0 is built from hex2 source. It adds macro definitions and string handling,
making it possible to give names to instruction encodings instead of repeatedly
writing raw hexadecimal bytes.

The resulting pipeline is:

```text
Macro-assembly source -> M0 -> hex2 text -> hex2-0 -> executable
```

On the inspected AMD64 path, a small helper called `catm` is also built before
M0. It concatenates files using explicit input and output filenames, avoiding a
dependency on a host `cat` command or shell redirection. Among other things,
this joins an executable-header template with generated program text.

### Step E: Build the first C compiler

The AMD64 implementation provides `cc_amd64.M1`, assembly source that M0 can
process. Despite the `.M1` filename, this initial build uses M0, not the later
C-written M1 assembler.

The assembled result, `cc_amd64`, supports only the C subset needed to build
the initial M2-Planet compiler. It does not need to implement the entire C
standard or compile arbitrary modern C projects.

The important distinction is:

- Its **implementation language** is assembly.
- Its **input language** is a restricted form of C.

No existing C compiler is required for this step.

### Step F: Build M2-Planet

The first C compiler compiles M2-Planet's C source together with bootstrap
support code. Its output is macro assembly, which still passes through M0 and
hex2-0:

```text
M2-Planet C source + bootstrap support
        |
        v
cc_amd64
        |
        v
Macro assembly + architecture definitions + runtime assembly
        |
        v
M0 -> hex2 text + ELF header -> hex2-0
        |
        v
Initial M2-Planet executable
```

M2-Planet implements a larger C subset and can compile its own source.
However, it is still part of a toolchain: compiling C to assembly is not the
same as producing the final executable without assembler and linker support.

## 3. Replace the temporary tools

Once the initial M2-Planet works, the build uses it to compile C implementations
of the supporting tools:

| Tool | Purpose |
| --- | --- |
| M1 | A more capable, cross-platform macro assembler that replaces M0. |
| hex2 | A more capable linker that replaces the initial hex2-0. |
| blood-elf | Generates ELF/DWARF support for more inspectable, debuggable binaries. |
| kaem | Runs build scripts without relying on a conventional host shell. |
| get_machine | Supplies machine information for architecture-dependent build decisions. |

[M2libc](https://github.com/oriansj/M2libc) supplies bootstrap and fuller runtime
support. Early builds use restricted support code; later builds can use more
capable library implementations.

The inspected AMD64 scripts also build M2-Mesoplanet, a compiler-driver tool,
before rebuilding M2-Planet.

### What runs the build before kaem has been compiled?

There are two distinct responsibilities:

- `hex0` and its successors translate source into programs.
- `kaem` invokes those programs in the required order.

An existing shell can run the initial commands, or an optional prebuilt
`kaem-optional-seed` can run them. The seed-stage script builds a minimal
`kaem-0` from hex0 source; later, the toolchain builds a fuller kaem from C.

Using the optional runner avoids relying on the host shell for those steps,
but introduces another seed executable to audit. It is not part of hex0.

## 4. Reach self-hosting

The initial M2-Planet compiles M2-Planet's source again. M1 assembles the output,
and hex2 produces the final executable using the runtime and ELF support.

```text
Assembly-written C compiler
        |
        | compiles M2-Planet source
        v
Initial M2-Planet
        |
        | compiles M2-Planet source
        v
Rebuilt M2-Planet
```

There is no circular dependency: the assembly-written compiler creates the
first executable of the C-written compiler.

Self-hosting does not mean that the first and rebuilt binaries must be
identical. Bootstrap flags, runtime implementations, and debug information can
differ. Byte comparisons are meaningful when source, configuration, supporting
tools, and other relevant build inputs are held constant.

## 5. What does verification establish?

A careful bootstrap audit includes:

1. Checking the seed instructions, executable layout, and system-call usage.
2. Rebuilding hex0 and comparing it with the seed.
3. Inspecting each translator and the source it accepts for the next step.
4. Preserving intermediate assembly, hex2 text, and executables for inspection.
5. Checking expected hashes and reproducibility where applicable.
6. Accounting for the kernel, loader, hardware, source provenance, and any host
   tools still used.

The AMD64 build finishes its tool-building sequence with a checksum-manifest
check. Such checks detect differences from expected outputs; they do not prove
the absence of malicious behavior or compiler bugs. Hashes are also not digital
signatures.

The goal is to reduce opaque binary dependencies and make the chain auditable,
not to eliminate all trust.

## 6. From a C subset to a larger toolchain

Stage0-posix provides an early compiler and supporting tools. It does not
directly turn the hex0 seed into a current GCC or Clang in one compilation.

[live-bootstrap](https://github.com/fosslinux/live-bootstrap) extends the chain
through additional libraries, utilities, and compiler stages. Later stages need
not all be written in C; building modern GCC or Clang eventually requires
C++-capable tooling.

The exact versions and dependency graph belong to the selected bootstrap
project and revision. The general strategy stays the same: use the tools
already available to build the next, more capable set.

## 7. Where to read the actual build

For a concrete walkthrough, follow these upstream files in order:

1. [stage0-posix AMD64 entry script](https://github.com/oriansj/stage0-posix/blob/643598041bf7639883874fe2cdc9d9693c9b03d5/kaem.amd64):
   dispatches the seed, minimal-tool, and full-tool phases.
2. [AMD64 seed-stage script](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/mescc-tools-seed-kaem.kaem):
   rebuilds hex0 and builds minimal kaem.
3. [AMD64 minimal-tool script](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/mescc-tools-mini-kaem.kaem):
   builds hex1, hex2-0, M0, cc_amd64, initial M2-Planet, and supporting tools.
4. [AMD64 full-tool script](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/mescc-tools-full-kaem.kaem):
   builds further tools and rebuilds M2-Planet.
5. [AMD64 remaining-build script](https://github.com/oriansj/stage0-posix-amd64/blob/15535f88e25825f01a0de275b6d45f77e618bd6b/kaem.run):
   dispatches the full-tool build, additional utilities, and checksum checks.

These references were inspected on 2026-11-15. The architecture-script links
identify the inspected architecture revision; a particular stage0-posix checkout
can pin a different revision through its submodules. When reproducing a build,
use that checkout's recorded submodule revisions rather than mixing versions.
Phase numbering and script organization can differ from the overview README.

Additional references:

- [Original Stage0 design and hex-language descriptions](https://github.com/oriansj/stage0/blob/master/README.org).
- [stage0-posix overview](https://github.com/oriansj/stage0-posix/blob/643598041bf7639883874fe2cdc9d9693c9b03d5/README.md).
- [Seed audit guidance](https://github.com/oriansj/bootstrap-seeds/blob/cedec6b8066d1db229b6c77d42d120a23c6980ed/README.md).
- [M2-Planet project](https://github.com/oriansj/M2-Planet).

In short: **start with bytes, use them to build progressively better
translators, build a small C compiler in assembly, then use C to rebuild and
extend the toolchain.**
