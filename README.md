# OpenTyrian
<img src="linux/icons/tyrian-128.png" width="128" height="128" align="right" alt="OpenTyrian icon">

[![Linux](https://github.com/pedrocatalao/tyrian-sdl/actions/workflows/linux.yml/badge.svg)](https://github.com/pedrocatalao/tyrian-sdl/actions/workflows/linux.yml)
[![macOS](https://github.com/pedrocatalao/tyrian-sdl/actions/workflows/macos.yml/badge.svg)](https://github.com/pedrocatalao/tyrian-sdl/actions/workflows/macos.yml)
[![Windows](https://github.com/pedrocatalao/tyrian-sdl/actions/workflows/windows.yml/badge.svg)](https://github.com/pedrocatalao/tyrian-sdl/actions/workflows/windows.yml)

OpenTyrian is an open-source port of the DOS game Tyrian.

Tyrian is an arcade-style vertical scrolling shooter.  The story is set
in 20,031 where you play as Trent Hawkins, a skilled fighter-pilot employed
to fight MicroSol and save the galaxy.

Tyrian features a story mode, one- and two-player arcade modes, and networked
multiplayer.

## Downloads

Self-contained builds are attached to each
[release](https://github.com/opentyrian/opentyrian/releases).  They include
SDL2 and the freeware Tyrian 2.1 game data, so there is nothing to install:

| Platform | File | Run |
|---|---|---|
| macOS (Apple Silicon and Intel) | `opentyrian-macos-universal.zip` | Unzip and open `OpenTyrian.app` |
| Linux x86_64 / arm64 | `opentyrian-linux-<arch>.tar.gz` | Extract and run `./opentyrian` |
| Windows x86_64 / arm64 | `opentyrian-windows-<arch>.zip` | Unzip and run `opentyrian.exe` |

The macOS app is not notarized.  If Gatekeeper refuses to open it, right-click
the app, choose *Open*, and confirm once.

Configuration and saved games are kept per user, outside the game directory:

| Platform | Location |
|---|---|
| Linux / macOS | `$XDG_CONFIG_HOME/opentyrian`, or `~/.config/opentyrian` |
| Windows | `%APPDATA%\OpenTyrian` |

## Game Data

OpenTyrian needs the Tyrian 2.1 data files, which have been released as
freeware: <https://camanis.net/tyrian/tyrian21.zip>

The release builds above already contain them.  Otherwise, extract the
archive so that the files (lowercase names) are in one of these places,
searched in order:

1. the directory given with `--data=DIR`
2. a `data` directory next to the executable (inside `Contents/Resources`
   for the macOS app)
3. the system directory the build was configured with
   (`/usr/local/share/games/tyrian` by default; `C:\TYRIAN` on Windows)
4. a `data` directory in the current working directory

`./get_data.sh [dir]` downloads and extracts them for you.  Filenames in the
archive may be uppercase; the script lowercases them, as does
`lower-script.sh` for an existing directory.

## Building

Requirements: a C99 compiler, GNU make, pkg-config, SDL2 and, for network
play, SDL2_net.

    make

Network play is left out automatically when SDL2_net is not found
(`make WITH_NETWORK=false` forces that; `WITH_NETWORK=true` requires it).
`make debug` builds with `-Werror`, `-O0` and debug info.  `make install`
honours `DESTDIR` and `prefix`.

A Visual Studio solution is in `visualc/`.

The self-contained release builds are produced by the same scripts CI uses:

    ./make_mac.sh      # universal OpenTyrian.app in build/, SDL2.framework bundled
    ./make_linux.sh    # SDL2 built from source and linked statically
    ./get_data.sh      # fetches the game data into data/ (both scripts call it)

The Linux script needs the development headers listed at the top of the file;
SDL loads the matching X11, Wayland and audio backends at run time, so the
resulting binary depends on nothing but glibc.  The Windows packages are
built under MSYS2 by `.github/workflows/windows.yml`.

## Command-Line Options

    -h, --help                   Show help about options
    -s, --no-sound               Disable audio
    -j, --no-joystick            Disable joystick/gamepad input
    -x, --no-xmas                Disable Christmas mode
    -t, --data=DIR               Set Tyrian data directory
    -n, --net=HOST[:PORT]        Start a networked game
    --net-player-name=NAME       Set local player name in a networked game
    --net-player-number=NUMBER   Set local player number in a networked game
                                 (1 or 2)
    -p, --net-port=PORT          Set local port to bind (default is 1333)
    -d, --net-delay=FRAMES       Set lag-compensation delay (default is 1)

## Keyboard Controls

    alt-enter      -- toggle full-screen

    arrow keys     -- ship movement
    space          -- fire weapons
    enter          -- toggle rear weapon mode
    ctrl/alt       -- fire left/right sidekick

## Network Multiplayer

Currently OpenTyrian does not have an arena; as such, networked games must be
initiated manually via the command line simultaneously by both players.

syntax:

    opentyrian --net HOSTNAME --net-player-name NAME --net-player-number NUMBER

where HOSTNAME is the IP address of your opponent, NUMBER is either 1 or 2
depending on which ship you intend to pilot, and NAME is your alias

OpenTyrian uses UDP port 1333 for multiplayer, but in most cases players will
not need to open any ports because OpenTyrian makes use of UDP hole punching.

## Links

- project: <https://github.com/opentyrian/opentyrian>
- irc:     <ircs://irc.oftc.net/#opentyrian>
- forums:  <https://tyrian2k.proboards.com/board/5>
