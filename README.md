OS 1.26 / NOSP
==============

This repository contains source code for OS 1.26 / NOSP, a copy of Acorn
OS 1.20 with some bugs fixed and a large space cleared for further
patching.

OS 1.26 / NOSP has the following modifications:

- An OSWRSC entry point is provided at &FFB3
- The OSFILE call handler in the Cassette and ROM filing systems (CFS,
  RFS) [always returns][1] the correct length of the file to the
  control block (thanks to John Kortink)
- Calling `*RUN` on a cassette or ROM file [does not overwrite][1]
  arbitrary I/O addresses after loading the file (thanks to John
  Kortink)
- In the `VDU 21` state, cursor motion codes are only sent to the
  printer while `VDU 2` applies; the parameter of `VDU 1` is printed
  once
- `VDU 28` checks the parameters correctly
- An error raised while printing the language banner [aborts the change
  of language, and is handled by the current language][2] (thanks to
  J.G.Harston)
- The paged ROM indirection routine places one byte less on the stack
- OSBYTE calls to write to I/O memory avoid causing a dummy read cycle
  before the write, which upsets some hardware
- OSFILE with A=0 (save file) reads data only from the source processor
- Main memory is cleared faster on power up or 'critical' BREAK
- Locations &02CF, &02D0 and &02D1 are not touched
- Locations &C4 and &CB are unused while the CFS or RFS is active
- Semantically transparent optimisations
- All speech processor driver code is removed (thanks to a [patch][3]
  by J.G.Harston)
- RFS file searching and `*CAT` terminate when an RFS ROM is present
  in slot 0 (thanks to J.G.Harston)
- 466 bytes cleared in the main section + 1 existing = 467 bytes free
- 21 bytes cleared in the top page

The free space is placed at the end of the \*command table, currently
located at address &DED1.

OS 1.26
-------

The [main branch][4] retains speech processor support and makes 164
bytes of the ROM available in total.

STARGO / NOSP
-------------

To demonstrate how the spare ROM area can be reapplied, the `STARGO`
option in `src/MOSHdr` enables:

- CFS/RFS implements OSARGS with A=1, Y=0 (read command line tail
  pointer)
- CFS/RFS `*RUN` enters code with A=1, X=&FF, Y=command tail offset, C=1
- New commands: `*GO` / `*GOIO`
  \[&lt;*address*&gt;\]\[`,`\] \[`;`\]\[&lt;*arguments*&gt;\]
  which do both of the above
- `*FX 5,n` flashes the keyboard LEDs while waiting for the printer
- 356 + 1 bytes free

The rest of this document describes vanilla OS 1.26 / NOSP.

Build requirements: OS 1.26 / NOSP
----------------------------------

See the upstream release notes, included below, on how to assemble OS
1.26 / NOSP from the prebuilt disc images, substituting the version
number as necessary.  The Stardot method builds the OS using a physical
Acorn 'Turbo' second processor (or a device emulating one) plugged into
a BBC Micro or Master series computer.

There is also the option to assemble using an emulated Turbo processor,
running under RISC OS 2 or 3.x.

On a Windows PC it may be convenient to emulate RISC OS itself along
with the Turbo, to build the ROM image on the local file system.  An
[article on 4corn][5] provides detailed instructions on how to install
RPCEmu, assemble a Turbo second processor emulator inside, and build OS
1.20 in the Turbo emulator.  
Once you have completed the OS 1.20 build, load the
`adfs/AcornOS126.adl` disc image into RPCEmu (Disc &rarr; Floppy &rarr;
Load Drive :0...).  Open drive :0, and copy all files from :0 into the
`Emu6502.HostEmuOS` directory *window* to replace the OS 1.20 source
files.  Then enter these commands into the existing command window to
build OS 1.26 / NOSP, assembled in a file named `nosp`:

    *Quit
    *EmulateTurbo
    *Exec MakeMOS
    *Quit

The current ROM image has an MD5SUM of
`eb3eb2b4325743d021c775262cd998a6`.

Build requirements: disc images
------------------------------

To build the disc images from the source, you will need:

- a POSIX environment such as Linux, or MinGW on Windows
- `unix2mac`, part of the `dos2unix` package
- for DFS: [MMB Utils][6] by Stephen Harris, which require Perl
- for ADFS: [Acorn FS Utils][7] by COEUS, which require a
  C compiler

Install the MMB Utils and Acorn FS Utils on your `PATH`, as needed.
Then `cd` to the respective `dfs/` or `adfs/` directory, and enter:

    sh make_disk_images.sh

Patching the \*command table
----------------------------

The space now available makes it practical to add *star commands* to the
built-in OS command set.  New entries can be appended in place of the
NUL terminator byte at `src/MOS38` line 283, currently located at
address &DED0.

Command table entries have the following form:

    NAME  addr_hi  addr_lo  aux

Each entry begins with the name of the command, which consists of one
or more capital letters.  Other characters are not allowed.  
Note that built-in commands *and their abbreviated forms* take
precedence over all other commands in the service chain.  It may
sometimes be wise to reject abbreviations; to do this create a pair of
entries, one missing the last letter which is set to pass abbreviated
calls down the chain (see below), preceding an entry naming the command
in full.

Following the name of the command is the high byte of the *action
address* which the command interpreter will call.  The high byte always
has bit 7 set to mark the end of the command name; this means that bit 6
must be set as well, to address the constant OS memory between &C000 and
&FFFF.  It takes a jump from OS ROM code to reach a routine in main
memory (`JMIUSR` is one, described below).  Code in paged ROM can be
reached via the [extended vector][8] entry points at &FF00..&FF4E, which
must be set up before use.  The low byte of the address comes next.

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
command name.  
The routine should exit with `RTS` to return from OSCLI.  Registers and
flags on exit are forfeit, and code intercepting the OSCLI vector may
alter them *en route* to the caller.  They are not passed to the second
processor.

Remember to replace the terminator byte at the end of the new table!

### Useful addresses

Pointing a \*command at `CLIEND` (&E129) passes it to paged ROMs or the
current filing system.  This is convenient for disposing of the
abbreviated forms of a command; the most efficient auxiliary byte value
is &FF.

To bypass utility ROMs, an action address equal to `JMIFSC - &07`
(&E132) sends the command straight to the filing system control vector,
defined at &021E.

`JMIUSR` (&F090) sends a \*command to USERV, defined at &0200.  An
auxiliary byte value of &01 emulates `*LINE`; other values (between &02
and &DF inclusive) cause entry into the USERV routine with non-standard
reason codes.

In a routine handling the new command, `SKIPSP` (&E142) may be passed
the current offset into the command in Y.  It returns a non-space
character in A, its offset in Y, and `EQ` if that character is CR.
`SKIPSN` (&E141) is the same but ignores the current character by
advancing Y over it.

As an example, a command named `I` whose routine begins with

     BEQ STARI
     JMP CLIEND
    STARI
     ...

will reject all invocations with arguments, allowing `*I AM FRED` to
reach the NFS ROM.

After making changes
---------------------

Adjust the amount of padding at line 285 of `src/MOS38` to ensure that
`src/MOS76` assembles code up to &FBFF exactly.  The assembler will warn
if the code overruns into the FRED area - but not if it falls short.

More space at a pinch
---------------------

If it comes to the crunch a little more room can be made by
reassembling, minus some frills.  Delete lines 75..76 and 80..85
from `src/MOS34`:

     TAY
     BEQ CLEARA ;branch always

     INY
     STAIY &0000 ;saves 24 ms
     INY
     STAIY &0000 ;saves 8 ms
     INY
     STAIY &0000 ;saves 4 ms

Modify line 285 of `src/MOS38` accordingly:

     % 479 ;padding

Twenty-one more bytes can be saved by reverting portions of source code
to the original.  They are:

- 3 bytes providing the OSWRSC entry (in `src/MOS99`).
- 6 bytes calculating the cassette file size (in `src/MOS72`)
- 5 bytes ensuring RFS file searching terminates (in `src/MOS54`)
- 7 bytes freeing &02CF..D1 for programs (in `src/MOS34`, `src/MOS38`)

An archive of [OS 1.25][9] is available separately.  The source code in
this archive is manifolded and builds OS 1.20, 1.25, 1.26, STARGO and
NOSP according to the choice of header file.  A conditional assembly
reference to `MOS125` or `NOSP` introduces each variation from the
standard code.  
Also in this distribution is OS 1.26 patched for [GoSDC][10] tape
emulation support, and a copy of Acornsoft's Graphics Extension ROM
suitable for all the modified OS ROMs.

Known problems
--------------

- Certain \*commands in the Opus DDOS and Challenger ROMs corrupt the
  stack, causing a crash on exit ([patched disassemblies][11]
  are available).
- Acornsoft's Graphics Extension ROM (GXR) 1.20 ignores all graphics
  commands, as it contains hard-coded internal references to OS 1.20
  (it too can be [reassembled][12] to work with this OS).
- Slogger's Tape to Challenger 3 ROM (T2C3) 1.00 jumps to the hard-coded
  address of the OSBYTE handler in OS 1.20, causing a crash on the next
  call to OSBYTE. (Patch &8F15 = `JMP &E860`.)
- Many software titles, especially games, decrypt themselves using the
  contents of the OS ROM as a key.  These titles are incompatible
  with OS 1.26.

[1]:  http://mdfs.net/Archive/BBCMicro/2006/10/14/174712.htm
[2]:  https://stardot.org.uk/forums/viewtopic.php?p=358510#p358510
[3]:  https://mdfs.net/System/ROMs/AcornMOS/BBC_JGH/MOSnosp.src
[4]:  https://github.com/regregex/OS126/tree/master/
[5]:  https://www.4corn.co.uk/articles/65hostandmos/
[6]:  https://github.com/sweharris/MMB_Utils
[7]:  https://github.com/SteveFosdick/AcornFsUtils
[8]:  https://beebwiki.mdfs.net/Paged_ROM#Extended_vectors
[9]:  http://regregex.bbcmicro.net/#prog.os126
[10]: https://www.zeridajh.org/hardware/gosdc/index.html
[11]: http://regregex.bbcmicro.net/#features.bbc
[12]: https://github.com/regregex/GXR

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
