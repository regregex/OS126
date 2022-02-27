OS 1.26
=======

This repository contains source code for OS 1.26, a copy of Acorn OS
1.20 with some bugs fixed and a large space cleared for further
patching.

OS 1.26 has the following modifications:

- An OSWRSC entry point is provided at &FFB3
- The OSFILE call handler in the Cassette and ROM filing systems (CFS,
  RFS) always returns the correct length of the file to the control
  block
- Calling `*RUN` on a cassette or ROM file does not overwrite arbitrary
  I/O addresses after loading the file
- In the `VDU 21` state, cursor motion codes are only sent to the
  printer while `VDU 2` applies; the parameter of `VDU 1` is printed
  once
- `VDU 28` checks the parameters correctly
- The paged ROM indirection routine places one byte less on the stack
- OSBYTE calls to write to I/O memory avoid causing a dummy read cycle
  before the write, which upsets some hardware
- Main memory is cleared faster on power up, or on a 'critical' BREAK	
- Locations &02CF, &02D0 and &02D1 are not touched
- Locations &C4 and &CB are unused while the CFS or RFS is active
- Semantically transparent optimisations
- 123 bytes cleared in the main section + 1 existing = 124 bytes free
- 21 bytes cleared in the top page

The free space is placed at the end of the \*command table, currently
located at address &DF5C.

Build requirements: OS 1.26
---------------------------

See the upstream release notes, included below, on how to assemble OS
1.26 from the prebuilt disc images, substituting the version number as
necessary.  The Stardot method builds the OS using a physical Acorn
'Turbo' second processor (or a device emulating one) plugged into a BBC
Micro or Master series computer.

There is also the option to assemble using an emulated Turbo processor,
running under RISC OS 2 or 3.x.

On a Windows PC it may be convenient to emulate RISC OS itself along
with the Turbo, to build the ROM image on the local file system.  An
[article on 4corn](https://www.4corn.co.uk/articles/65hostandmos/)
provides detailed instructions on how to install RPCEmu, assemble a
Turbo second processor emulator inside, and build OS 1.20 in the Turbo
emulator.  
Once you have completed the OS 1.20 build, load the
`adfs/AcornOS126.adl` disc image into RPCEmu (Disc &rarr; Floppy &rarr;
Load Drive :0...).  Open drive :0, and copy all files from :0 into the
`Emu6502.HostEmuOS` directory *window* to replace the OS 1.20 source
files.  Then enter these commands into the existing command window to
build OS 1.26:

    *Quit
    *EmulateTurbo
    *Exec MakeMOS
    *Quit

The current ROM image has an MD5SUM of
`bc779c4ee7826f8e44090d9bfd4f0ecf`.

Build requirements: disc images
------------------------------

To build the disc images from the source, you will need:

- a POSIX environment such as Linux, or MinGW on Windows
- `unix2mac`, part of the `dos2unix` package
- for DFS: [MMB Utils](https://github.com/sweharris/MMB_Utils) by
  Stephen Harris, which require Perl
- for ADFS:
  [Acorn FS Utils](https://github.com/SteveFosdick/AcornFsUtils) by
  Steve Fosdick, which require a C compiler

Install the MMB Utils and Acorn FS Utils on your `PATH`, as needed.
Then enter the respective `dfs/` or `adfs/` directory, and enter:

    sh make_disk_images.sh

Patching the \*command table
----------------------------

With the space made available, it is now practical to add \*commands to
the built-in OS command set.  New entries can be inserted in place of
the NUL terminator byte, currently located at address &DF5B.

Command table entries have the following form:

    NAME  addr_hi  addr_lo  aux

Each entry begins with the name of the \*command, which must consist of
one or more capital letters.  Other characters are not allowed.  
Note that built-in commands *and their abbreviated forms* take
precedence over all other commands in the resolution chain.  It may
sometimes be wise to reject abbreviations; to do this create a pair of
entries, one without the last letter which passes abbreviated calls down
the chain (see below), preceding an entry that names the command in
full.

Following the name of the command is the high byte of the *action
address* which the command interpreter will call.  The high byte always
has bit 7 set to mark the end of the command name.  Bit 6 must be set
too, to address constant OS memory; code in paged ROM can be reached via
the extended vector entry points at &FF00..&FF4E.  The low byte of the
address comes next.

The entry ends with an auxiliary byte that controls the register values
on entry to the \*command code.  Its value is passed to the routine in
the accumulator.  
When the value is less than &80, the X and Y registers contain,
respectively, the low and high bytes of the address of the first
*argument* in the command line following the command name.  
When it is &80 or more, the Y register contains the offset of the first
argument from the start of the command line (addressed by the GSINIT
pointer at &F2..3), and the X register is undefined.  Passing Y to
the GSINIT system call will select the argument.

OSCLI calls the action address with the carry flag clear (`CC`).  The
zero flag is set (`EQ`) if and only if there are no arguments after the
\*command.  
The routine should exit with `RTS` to return from OSCLI.  Registers and
flags on exit are forfeit, and code intercepting the OSCLI vector may
alter them *en route* to the caller.  They are not passed to the second
processor.

Remember to replace the terminator byte at the end of the new table!

### Useful addresses

Pointing a \*command at `CLIEND` (&E05C) passes it to paged ROMs or the
current filing system.  This is convenient for disposing of the
abbreviated forms of a command; the most efficient auxiliary byte value
is &FF.

To bypass utility ROMs, an action address equal to `JMIFSC - &07`
(&E065) sends the command straight to the filing system control vector,
defined at &021E.

`JMIUSR` (&E681) sends a \*command to USERV, defined at &0200.  An
auxiliary byte value of &01 emulates `*LINE`; other values (between &02
and &7F inclusive) cause entry into the USERV routine with non-standard
reason codes.  X and Y must contain the address of the first argument.

In routines, `SKIPSP` (&E075) returns a non-space character in A, its
offset in Y, and `EQ` if that character is CR.  `SKIPSN` (&E074) is the
same but ignores the current character by advancing Y over it.

As an example, a command named `I` whose routine begins with

     BEQ STARI
     JMP CLIEND
    STARI
     ...

will reject all invocations with arguments, allowing `*I AM FRED` to
reach the NFS ROM.

After making changes
---------------------

Adjust the amount of padding at line 225 of `src/MOS38` to ensure that
`src/MOS76` assembles code up to &FBFF exactly.  The assembler will warn
if the code overruns into the FRED area - but not if it falls short.

More space at a pinch
---------------------

If it comes to the crunch a little more room can be made by
reassembling, minus some frills.  Delete lines 80..85 from `src/MOS34`:

     INY
     STAIY &0000 ;saves 24 ms
     INY
     STAIY &0000 ;saves 8 ms
     INY
     STAIY &0000 ;saves 4 ms

Modify line 225 of `src/MOS38` accordingly:

     % 133 ;padding

Known problems
--------------

- Certain \*commands in the Opus DDOS and Challenger ROMs corrupt the
  stack, causing a crash on exit
  ([patched disassemblies](http://regregex.bbcmicro.net/#features.bbc)
  are available).
- Many software titles, especially games, decrypt themselves using the
  contents of the OS ROM as a key.  These titles are incompatible
  with OS 1.26.

* * *

The README from the original
[Acorn OS 1.20 repository](https://github.com/stardot/AcornOS120)
follows.

<pre>
Introduction
============

This repository contains the original source code for Acorn OS 1.20.

The structure of the repository is as follows:

    original_sources.zip
              - a copy of the original OS 1.20 source files, with the
                original <cr> line endings, a couple of build bugs (in
                MosHdr and MakeMOS) and a 1-bit character error in
                MOS44 (in a comment).

    adfs/     - disk images in ADFS (adl and dat) formats;
                these are generated by the make_disk_images.sh script
                which uses COEUS's AcornFsUtils package.

    dfs/      - disk images in DFS (ssd and dsd) formats;
                these are generated by the make_disk_images.sh script
                which uses SWEH's Perl mmb_utils.

    src/      - the OS 1.20 source files (with various build fixes)

    tools/    - binaries for MASM (both DFS and non-DFS versions)

Assembling the OS Source Code
=============================

An Acorn Turbo (256K) 6502 Co Processor is needed to assemble the OS
1.20 sources. As originals are exceedingly rare, an alternative is the
emulated version provided by PiTubeDirect Fer-De-Lance (and later) as
Co Pro 17. The reset banner should say: "Acorn TUBE 6502 256K".

The source code is in Acorn MASM format, and a copy of the "Turbo"
version of MASM (called TurMASM) is included in the disk images.

Using ADFS:
===========

The disk organization on ADFS involves a single disk image containing
the tools (TurMASM), build scripts, source code, and sufficient free
space for the build process.

In the adfs/ directory are the following versions:

   AcornOS120.adl - this is a 640KB interleaved disk image, suitable
   for writing to a floppy disk.

   AcornOS120.dat - this is a 640KB non-interleaved disk image,
   suitable for using with BeebSCSI (e.g. rename it to scsi0.dat).

Transfer one of these disk images onto your physical hardware.

To assemble the sources, you just need to boot the disk (it contains
an appropriate !BOOT file)

The OS 1.20 binary is generated in a file called MOS1_20 in the root directory
    (this has a MD5SUM of 0a59a5ba15fe8557b5f7fee32bbd393a)

Using DFS:
==========

(or MMFS)

The disk organization on DFS involves four disk images:
- drive 0: Tools (TurMASM)
- drive 2: Working files (initially empty)
- drive 1: OS 1.20 source code
- drive 3: OS 1.20 source code (continued)

In the dfs/ directory are following versions:

Four seperate single sided disk images:
     ssd/AcornOS120_disk0.ssd
     ssd/AcornOS120_disk1.ssd
     ssd/AcornOS120_disk2.ssd
     ssd/AcornOS120_disk3.ssd

Two seperate double-sided disk images:
    dsd/AcornOS120_disk02.dsd
    dsd/AcornOS120_disk13.dsd

Transfer one of these disk image sets onto your physical hardware.

To assemble the sources, you just need to boot disk 0 (it contains an
appropriate !BOOT file)

The OS 1.20 binary is generated in a file called MOS1_20 on disk 2
    (this has a MD5SUM of 0a59a5ba15fe8557b5f7fee32bbd393a)

Acknowledgements:
=================

Many thanks to Stuart Swales and Paul Fellows (ex Acornsoft) for
discovering these long-lost sources and making them available to the
Acorn community.
</pre>
