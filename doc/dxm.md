# Tyrian SDL as a DOS ex Machina core

This tree builds the game twice from one source: the standalone SDL2 game
(the Makefile, or CMake), and a **DXM core** — a dependency-free library that
[DOS ex Machina](https://github.com/pedrocatalao/dos-ex-machina) loads and
runs inside its simulated PC.  The contract is DXM's `PORTING.md`; the
vendored `src/dxm/dxm_core.h` is the interface both sides compile against.

## Layout

```
src/
  platform.h        THE SEAM.  Everything the game needs from a host:
                    present a 320x200 frame, key/mouse events, a clock,
                    an audio pull, logging, paths, exit.  Game code
                    includes this and never SDL.
  scancode.h/.c     the game's key space (SDL2's numbering), key names for
                    config files, US-layout characters
  surface.h/.c      the 8-bit software surface the game draws into
  video.c           shared: the screen surfaces, JE_showVGA -> plat_present
  keyboard.c        shared: input queues; receives input_* callbacks

  platform_sdl.c    standalone: events, timing, audio device, log, paths
  video_sdl.c       standalone: window, renderer, texture, scalers
  video_scale*.c    standalone: the software scalers (SDL textures)
  joystick.c        standalone: SDL joysticks
  main.c            standalone: main()

  dxm/dxm_core.h    vendored contract header (DXM_ABI)
  dxm/platform_dxm.c the adapter: platform.h on top of dxm_host
  dxm/video_stub.c  no window: one scaler, no fullscreen
  dxm/joystick_stub.c
  dxm/dxm_entry.c   the info block and the three fixed-name exports
```

The Makefile builds only `src/*.c`, so the core files never enter the
standalone build.  CMake lists both sets explicitly.

## Building the core

```
cmake -S . -B build-core -DOT_CORE=ON -DOT_MODULE=ON
cmake --build build-core
```

produces `build-core/libopentyrian_core.a` and `build-core/opentyrian.dxm`,
the loadable module (hidden visibility; exactly `dxm_core_get_info`,
`dxm_core_main`, `dxm_core_audio` exported).  CI builds it for every platform
and asserts with `nm` that no `SDL_*`, `exit`, `getenv`, `pthread_*` or
`plat_*` symbol is left undefined.

## How the eight rules are met

| Rule | How |
|---|---|
| 3.1 no `exit()` | every call site uses `plat_exit()`; the adapter `longjmp`s to `dxm_core_main` |
| 3.2 restartable | see *Host requirements* — the shell reopens the module per run |
| 3.3 no stdio/signals | `logging.c` goes through `plat_log()` -> `host->log()` |
| 3.4 no cwd | data dir passed as `--data=`; saves go to `host->pref_dir` |
| 3.5 no SDL | the core links `src/dxm/*` instead of the `*_sdl.c` files; `nm` in CI |
| 3.6 no threads | audio is pulled by the shell via `dxm_core_audio`; `plat_pump()` checks `should_quit()` |
| 3.7 no `getenv` | only `platform_sdl.c` reads the environment |
| 3.8 prefixed symbols | moot for a hidden-visibility module: three exports |

Timing is wall-clock (`host->now()`), never frame-count.  Video is Mode 13h:
320x200, INDEX8, PAR 5:6, 400 CRT lines.  Audio: the game mixes mono s16 at
44100 Hz; the adapter duplicates to stereo for the shell.

## What the host had to provide

Two changes were made in DXM for this port, and both are general rather than
Tyrian-specific:

1. **Full keymap.**  `sc_from_sdl()` mapped 15 keys.  Tyrian needs the whole
   keyboard: Ctrl and Alt fire the sidekicks, letters and Backspace type save
   names and cheats, F-keys and the keypad are bindable.  It is now a table
   from SDL scancode to XT set 1 make code covering 0x01..0x58.  Extended
   keys (arrows, Home/End/PgUp/PgDn, Ins/Del, right Ctrl/Alt, keypad Enter
   and `/`) report their **base** code (0x48 for Up, 0x1D for right Ctrl),
   since the key-state array is indexed by the plain byte and that is what
   the adapter reads.
2. **The module is reopened per run.**  It used to stay loaded.  The game has
   ~850 file-scope globals plus function-local statics, so a fresh `dlopen`
   per run resets all of them for free, which is the only restart strategy
   that will not rot.  `corehost_stop()` also fences against the audio
   callback before the unload, so nothing is executing inside the image when
   it goes.

Still open:

- **No mouse.**  `dxm_host` has no mouse at all, so the adapter reports none
  and the game is keyboard-only inside DXM.  The standalone build has the
  mouse.  Adding one means a new `dxm_host` entry (relative motion plus a
  button mask, the way a DOS mouse driver reported it) and an ABI bump.
- **Text input** comes from key state and shift on a US layout, because
  `getch()` only carries ASCII for Esc, Enter and Space.  A host `getch()`
  with real layout characters would be an improvement, not a requirement.

## Catalogue entry

For DXM's `catalogue.json`, once a release exists:

    id      tyrian
    exe     TYRIAN.EXE
    abi     2
    source  https://github.com/pedrocatalao/tyrian-sdl
    module  the .dxm asset per platform-arch, with its sha256
    data    kind freeware, https://camanis.net/tyrian/tyrian21.zip,
            probe tyrian1.lvl

The zip nests its files in a folder and may name them in UPPERCASE;
`unzip_extract` already flattens, and the game opens uppercase names as a
fallback, so neither needs a host change.
