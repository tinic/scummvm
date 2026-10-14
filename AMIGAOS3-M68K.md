# AmigaOS 3 (m68k) port notes

Branch `amigaos3-m68k`, living in the `tinic/scummvm` fork only. It carries the
classic 68k AmigaOS 3 port of ScummVM. Nothing here is proposed upstream in this
shape; the aim is a real C++ program on OS 3 that a source-level debugger can be
pointed at.

## Status

Step 1 is done: the null backend plus the `testbed` engine cross-build with the
AmigaPorts 68k toolchain, and the resulting binary runs on a classic Amiga (an
emulated A3000 with Kickstart 3.1). The testbed engine executes its
non-interactive testsuites, writes its report, and then stays in its summary
dialog - see "Known limitations" below.

The SDL 1.2 backend for OS 3 (AmigaPorts libSDL12) is *not* started; `--backend=null`
is the only backend wired up so far.

## Toolchain and build

AmigaPorts `m68k-amigaos-gcc`, release **AmigaDev v16.2-rc16**
(`m68k-amigaos-g++ (AmigaDev v16.2-rc16) 16.2.0b 20261010182711`). Check for a
newer release before building; the old pinned toolchain is not used.

    export PATH=/path/to/m68k-amigaos-gcc-16.2-rc16/m68k-amigaos-gcc-16.2/bin:$PATH
    ./configure --host=m68k-amigaos --backend=null \
                --disable-all-engines --enable-engine=testbed
    make -j6

`--enable-engine=testbed` is not optional: with `--disable-all-engines` alone
the binary only ever shows the launcher.

The debug build carries `-g -gdwarf-5 -fvar-tracking-assignments -O0 -noixemul`
(the configure host case adds the DWARF request and `-g` on the link line, which
is what makes the linker emit the debug hunk). The resulting binary is
69,350,316 bytes, of which ~11 MB is `.text`; the rest is the `.dwarf2` hunk,
which is a `HUNK_DEBUG` hunk and is not loaded by `LoadSeg`.

## What runs today

    DH0:scummvm -c DH0:scummvm.ini --debuglevel=6 -p DH0:tb testbed

Non-interactive mode comes from `[scummvm] interactive-mode=false` in
`scummvm.ini`; there is no such command line option. Suites that need a display,
sound, input or the network skip themselves, as they do on any headless backend.
Observed report:

    GFX         Tests Executed: 0   (all skipped: pixel formats, blits, modes, cursors)
    Image       Tests Executed: 0   (all skipped)
    FS          Tests Executed: 3   Passed: 3
    SaveGames   Tests Executed: 5   Passed: 5
    Misc        Tests Executed: 3   Passed: 1 (Datetime), Failed: 2
    Events / SoundSubsystem / Networking / Printing   Tests Executed: 0

The full table is written to `DH0:tb/testbed.log`.

## The commits on this branch

1. `configure`: AmigaOS 3 (m68k) host support - the `m68k-amigaos` host triple
   had no case of its own and fell through to the generic fallback (which also
   left the host binutils in place, so the archive had no symbol index). The
   AmigaOS case kept PowerPC assumptions: `-mlongcall` (which the 68k compiler
   does not know) and `-gstabs` (dropped by GCC 16, and 68k wants DWARF).
   PowerPC behaviour is unchanged.
2. `backends/module.mk` + `backends/fs/morphos/*`: the classic AmigaOS file
   system backend. MorphOS and AmigaOS 3 share the same `dos.library` interface;
   only the AmigaOS 4 backend uses the DOS object API.
3. `backends/platform/null/null.cpp`: a millisecond clock for AmigaOS 3, built
   from `DateStamp()` (1/50 s ticks) plus `Delay()` in `delayMillis()`. Without
   it `getMillis()` returned 0 on every non-POSIX, non-Windows host, which turns
   any timed wait into an infinite loop - `initGraphics()`'s splash loop never
   ended, so no engine ever started.
4. `backends/graphics/null/null-graphics.h`: `lockScreen()` now hands out a
   scratch `Graphics::Surface` instead of a null pointer. Engines draw even when
   nothing is ever displayed, and the null backend used to assert in
   `graphics/font.cpp` (the testbed writes its report on the screen).
5. `engines/testbed/misc.cpp`: the mutex test armed a timer whose callback
   removes itself, kept a pointer to its own stack frame and to a `Common::Mutex`
   freed right after the test, and could still fire from the next event poll -
   proven, the guest logged the callback with a freed mutex. Disarmed before the
   test returns instead of relying on the callback to uninstall itself.

## Known limitations

* `Misc/Timers` and `Misc/Mutexes` fail. Both need a timer to fire during a
  `delayMillis()` window. On this backend timers are only checked from
  `pollEvent()` (`DefaultTimerManager::checkTimers()`), and `delayMillis()` does
  not poll, so the 100 ms timer never fires while the test waits - the same
  reason the mutex test used to see its callback only afterwards. Not a port
  defect; a backend-architecture property.
* The engine never returns. `TestbedEngine::run()` shows its
  `TestbedExitDialog` unconditionally as its last action, and that modal loop
  waits for input or a quit event which the null backend never delivers
  (`pollEvent()` always returns false). So the guest runs all tests and then sits
  there. For a debugger target that is arguably fine, but it means there is no
  clean exit status.
* Debug builds are `-O0` and the binary is ~69 MB, ~58 MB of it debug info.

## Running it under Amiberry

The port itself needs nothing special, but a bare emulated drive does not have
what a Workbench 3.1 install has. The rig used here wraps the binary in a small
AmigaDOS shim that, before `SystemTagList()`-ing ScummVM:

* copies `mathieeedoubbas.library`, `mathieeedoubtrans.library`,
  `mathieeesingtrans.library` and `locale.library` into `LIBS:` (libnix wants
  them, and a bare rig leaves `LIBS:` empty),
* unpacks the GUI themes to `DH0:gui/themes/`,
* creates the `DH0:tb` test data the filesystem and savegame tests expect.

The shim is rig scaffolding, not part of the port. Two more rig facts: the guest
needs a large stack (>8 KB; 1 MB is used here), and it must run on a 32-bit
machine (A3000 class) with Z3 memory - an A1200/A600 cannot load this program at
all, its 24-bit address space leaves too little free memory for `LoadSeg`.
