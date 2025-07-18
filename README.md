OS 1.26
=======

This repository contains source code for OS 1.26, a copy of Acorn OS
1.20 with some bugs fixed and a large space cleared for further
patching.

OS 1.26 has the following modifications:

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
- `*HELP` is stable while `*SPOOL` is active
- OSCLI dispatches \*commands faster, especially to USERV
- The paged ROM indirection routine places one byte less on the stack
- OSBYTE calls to write to I/O memory avoid causing a dummy read cycle
  before the write, which upsets some hardware
- OSBYTE 152 returns the next buffer entry in Y as documented, and
  the correct implementation is detectable
- OSBYTE 206 [does not affect the stability][3] of other OSBYTE calls
  (thanks to TobyLobster)
- OSWRCH enters NETV with C=0 so OSBYTE 208 does not disrupt printing
- OSFILE with A=0 (save file) reads data only from the source processor
- File handle 0 is not valid in RFS
- Main memory is cleared faster on power up or 'critical' BREAK
- Locations &02CF, &02D0 and &02D1 are not touched
- Locations &C4 and &CB are unused while the CFS or RFS is active
- RFS file searching and `*CAT` terminate when an RFS ROM is present
  in slot 0 (thanks to J.G.Harston)
- Character recognition is faster in two-colour display `MODE`s
- Semantically transparent optimisations
- 250 bytes cleared in the main section + 1 existing = 251 bytes free
- 21 bytes cleared in the top page

The free space is placed at the end of the \*command table, currently
located at address &DF35.

OS 1.26 / NOSP
--------------

In addition to the above, the [NOSP branch][4] strips speech processor
support and makes 589 bytes of the ROM available in total.  
An optional paged ROM module, `SPDRV`, restores speech system
functions.

STARGO
------

To demonstrate how the spare ROM area can be reapplied, the `STARGO`
option in `src/MOSHdr` enables:

- CFS/RFS implements OSARGS with A=1, Y=0 (read command line tail
  pointer)
- CFS/RFS `*RUN` enters code with A=1, X=&FF, Y=command tail offset, C=1
- New commands: `*GO` / `*GOIO`
  \[&lt;*address*&gt;\]\[`,`\] \[`;`\]\[&lt;*arguments*&gt;\]
  which do both of the above
- `*::` \[&lt;*slot*&gt;\]\[`,`\] \[&lt;*command*&gt;\] sends a command
  to paged ROMs only, or to the paged ROM slot number given in hex
- `*:::` \[&lt;*command*&gt;\] sends a command to the filing system only
- `*FX 5,n` flashes the keyboard LEDs while waiting for the printer
- 83 + 1 bytes free

The rest of this document describes vanilla OS 1.26.

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
[article on 4corn][5] provides detailed instructions on how to install
RPCEmu, assemble a Turbo second processor emulator inside, and build OS
1.20 in the Turbo emulator.  
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
`56ac0a078f6852c4186318e7f0a114ca`.

Build requirements: disc images
-------------------------------

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
terminator sequence at `src/MOS38` line 310, currently located at
address &DF34.

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
memory (one is `JMIUSR`, described below).  Code in paged ROM can be
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
The routine should exit with `RTS` to return from OSCLI.  All registers
and flags are surrendered on exit, and code intercepting the OSCLI
vector may alter them *en route* to the caller.  Their values are not
passed to the second processor.

Be sure to replace the terminator byte at the end of the new table.
Formerly NUL, its present value is &80.  The parser now also examines
the empty string between it and the last command, and to prevent a match
here, there must remain a `""` entry earlier in the table.

### Useful addresses

Pointing a \*command at `CLIEND` (&E0BB) passes it to paged ROMs or the
current filing system.  This is convenient for disposing of the
abbreviated forms of a command; the most efficient auxiliary byte value
is &FF.

To bypass utility ROMs, an action address equal to `JMIFSC - &07`
(&E0C4) sends the command straight to the filing system control vector,
defined at &021E.

`JMIUSR` (&E6F1) sends a \*command to USERV, defined at &0200.  An
auxiliary byte value of &01 emulates `*LINE`; other values (between &02
and &DF inclusive) cause entry into the USERV routine with non-standard
reason codes.

In a routine handling the new command, `SKIPSP` (&E0CF) may be passed
the current offset into the command in Y.  It returns a non-space
character in A, its offset in Y, and `EQ` if that character is CR.
`SKIPSN` (&E0CE) is the same but ignores the current character by
advancing Y over it.

As an example, a command named `I` whose routine begins with

     BEQ STARI
     JMP CLIEND
    STARI
     ...

will reject all invocations with arguments, allowing `*I AM FRED` to
reach the NFS ROM.

After making changes
--------------------

Adjust the amount of padding at line 313 of `src/MOS38` to ensure that
`src/MOS76` assembles code up to &FBFF exactly.  The assembler will warn
of code overrunning into the FRED area - but not of falling short.

More space at a pinch
---------------------

If it comes to the crunch a little more room can be made by
reassembling, minus some frills.  Sixty-two bytes can be saved by
reverting portions of source code to the original.  They are:

- 27 bytes unrolling the memory clearing loop (in `src/MOS34`)
- 5 bytes ensuring RFS file searching terminates (in `src/MOS54`)
- 6 bytes calculating the cassette file size (in `src/MOS72`)
- 3 bytes providing the OSWRSC entry (in `src/MOS99`)
- 9 bytes making faster palette changes (in `src/MOS03`, `src/MOS11`)
- 4 bytes speeding up character recognition (in `src/MOS11`)
- 8 bytes freeing &02CF..D1 for programs (in `src/MOS34`, `src/MOS38`).

Applying all but the last three changes yields 41 bytes total and
results in [OS 1.25][9], available separately.  The source code in this
archive is manifolded and builds OS 1.20, 1.25, 1.26, STARGO and
[NOSP][4] according to the choice of header file: NOSP eliminates a
further 312 bytes of speech processor driver code, based on
J.G.Harston's [patch][10].  A conditional assembly reference to `MOS125`
or `NOSP` introduces each variation from the standard code.  
Also in this distribution is OS 1.26 patched for [GoSDC][11] tape
emulation support, and a copy of Acornsoft's Graphics Extension ROM
suitable for all the modified OS ROMs.

Installing
----------

OS 1.26 is suitable for programming into a 27128 or 27C128 EPROM to be
installed in IC 51, the dedicated OS ROM socket of the Model A/B
motherboard.  
Due to the large amount of published software relying on the exact
contents of OS 1.20, it is recommended to install OS 1.26 in such a way
that OS 1.20 remains available.

An OS RAM module from [BooBip.com][12] fits between the OS ROM and its
socket, and allows the ROM to start the computer at power up; it can
then host an alternative OS subsequently softloaded into RAM, while
providing a second bank of RAM to other code.
OS 1.26 is one such alternative OS, though like OS 1.20 it is not aware
of the module and makes no use of the RAM.  Either OS may be installed
in ROM and the other softloaded.
The method of softloading and OS selection is up to the user.

A more economical option is to prepare a 27C256 one-time programmable
EPROM and permanently modify it to supply one of two 16 KiB images,
selected by a switch or jumper.  
After programming the EPROM, the bases of pins 14, 27 and 28 are wired
to a miniature SPDT switch or three header pins that connect pin 27
(A14) *either* to pin 14 (0V) *or* to pin 28 (+5V).  The end of pin 27
is then cropped to keep it clear of the socket, and if a header is used,
a jumper is slid over two appropriate header pins, connecting them
together.  The EPROM is fitted with care.  The computer then runs the OS
in the first or second half of the device, respectively.  
The switch or jumper must not be toggled while the computer is running.

OSBYTE 152
----------

The OS call to examine the next entry in a system buffer had had a
fault, which was later fixed in Electron OS 1.00 and OS 2.00 for the
Model B+.

Although it is also possible to fix the bug by intercepting the REMV
vector, the [established workaround][13] for this call relies on the
value returned in Y instead of the expected entry.  Cross-platform
applications need to know which OS is running to engage the workaround,
and until this release it was enough to verify that a series 1 OS was
installed, by receiving X=1 from an [OSBYTE call with A=0][14] and X=1,
before using the OSBYTE 152 result to look up the actual entry.

With the bug fixed in this edition, the test must be refined.
REMV handlers featuring the bug are entered at points between &E45B and
&E5BD inclusive, but it suffices to test the high byte of REMV, at
location &022D in the I/O processor, for values &E4 or &E5.  Other
values indicate that OSBYTE 152 behaves as documented.  
Where this test is expensive, such as on the coprocessor, it need only
be performed if the OS test mentioned above confirms a series 1 OS.

Known problems
--------------

- Certain \*commands in the Opus DDOS and Challenger ROMs corrupt the
  stack, causing a crash on exit ([patched disassemblies][15]
  are available).
- Acornsoft's Graphics Extension ROM (GXR) 1.20 ignores all graphics
  commands, as it contains hard-coded internal references to OS 1.20
  (it too can be [reassembled][16] to work with this OS).
- Slogger's Tape to Challenger 3 ROM (T2C3) 1.00 jumps to the hard-coded
  address of the OSBYTE handler in OS 1.20, causing a crash on the next
  call to OSBYTE. (Patch &8F15 = `JMP &E7F5`.)
- Many software titles, especially games, decrypt themselves using the
  OS ROM contents as a key.  These titles are incompatible with OS 1.26.

[1]:  https://mdfs.net/Archive/BBCMicro/2006/10/14/174712.htm
[2]:  https://stardot.org.uk/forums/viewtopic.php?p=358510#p358510
[3]:  https://stardot.org.uk/forums/viewtopic.php?p=388738#p388738
[4]:  https://github.com/regregex/OS126/tree/nosp/
[5]:  https://www.4corn.co.uk/articles/65hostandmos/
[6]:  https://github.com/sweharris/MMB_Utils
[7]:  https://github.com/SteveFosdick/AcornFsUtils
[8]:  https://beebwiki.mdfs.net/Paged_ROM#Extended_vectors
[9]:  http://regregex.bbcmicro.net/#prog.os126
[10]: https://mdfs.net/System/ROMs/AcornMOS/BBC_JGH/MOSnosp.src
[11]: https://www.zeridajh.org/hardware/gosdc/index.html
[12]: http://www.boobip.com/hardware/osram
[13]: https://beebwiki.mdfs.net/OSBYTE_%2698
[14]: https://beebwiki.mdfs.net/OSBYTE_%2600
[15]: http://regregex.bbcmicro.net/#features.bbc
[16]: https://github.com/regregex/GXR

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
