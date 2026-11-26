# OpenSmash-Ultimate, fork for program-driven fights

> **This is a fork** of [OpenSmash-Ultimate](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate). It follows upstream `main` (last merge: **v0.5.1**) and adds a handful of switches to the runtime, all off by default. Installing, building, playing and going online are upstream's work and are described in their README, kept unchanged [below](#opensmash-ultimate).
>
> Like upstream, **this repository does not contain the game**: no game files, keys, firmware or saves.

## What this fork is for

Running fights **under the control of a program** rather than of a person:

- an agent plays through a virtual pad, alone or against a person on a real controller;
- a recorded fight is replayed input for input and compared with the original, frame by frame;
- a search tries inputs from a saved state of the game, goes back to it and tries others, thousands of times;
- several such games run at once on one PC, without drawing anything.

Upstream already has most of what this needs: fights set up without the menus (`SSBU_DEBUG_MATCH`), scripted menu input (`SSBU_SEQ`), fixed randomness (`SSBU_DETERMINISTIC`), a rollback core that saves and restores the whole game (`SSBU_RB`), and native mods that read the game's state. See upstream's [testing runs](docs/TESTING-RUNS.md) and [how it works](docs/TECHNICAL.md). This fork adds the missing pieces. The programs that drive the game (the mod, the agent, the search) are not part of this repository.

## What it adds

Each addition is an environment variable read by the runtime (`runtime/`). Nothing changes unless it is set, with one exception noted in the last row.

| Switch | What it does | Where |
| :--- | :--- | :--- |
| `SSBU_RB_REQ=save` / `load` | **Savestates on request.** Code inside the process (a native mod) sets the variable; the runtime serves it at its next safe point. `save` keeps the game's full state before the next frame is played; `load` goes back to the newest state kept. `SSBU_RB_DONE` then reads `<saves> <loads> <pinned>`: counts of what was really done, so the caller knows on which frame a request took effect (a request can be served one frame later; a refused load is tried again). Needs the rollback core (`SSBU_RB=1`). | `runtime/hle_rb_core.c` |
| `SSBU_RB_REQ=base` / `loadbase` | **A state to come back to, however many saves follow**: typically the first frame of a fight, to play it again and again with other inputs without starting the game again. `base` saves a state and pins it; `loadbase` returns to it, in about 15 ms. The third number of `SSBU_RB_DONE` says whether the pinned state is still there. | `runtime/hle_rb_core.c`, `runtime/hle_rb_state.c` |
| `SSBU_NO_DRAW=1` | Nothing is drawn or cleared: state-only runs, several games side by side. | `runtime/gpu_queue.c` |
| `SSBU_SEQ_RELEASE=1` | With `SSBU_SEQ`: once the menu sequence has ended, player 1 gets its pad back, so that a person or a program plays the fight the sequence set up. | `runtime/hle_hid.c` |
| `SSBU_XINPUT_EXTRA=<n>` | With SDL pads: XInput pad `n` is read as well, as player `n + 1`. A program's virtual pad plays beside a person's own controller. | `runtime/hle_pad.c` |
| `SSBU_DEBUG_MATCH=p1=cpu,...` | `p1` takes `cpu` or `human`, as `p2` does: a match between two CPU players. | `runtime/fight_test.h` |
| `SSBU_RB_FIBER_FREE=0` | Turns **off** a fix this fork makes by default: host fibers that no savestate needs any more are deleted. Without the fix, a search that loads one state thousands of times leaked about 0.27 GB a minute. | `runtime/hle_fiber.c` |

Also: menu sequences can be twice as long (`SEQ_STEPS`, `runtime/input_seq.h`).

### The pinned state, in short

The rollback core keeps a ring of states: one full copy of guest memory, which is the newest state, and for each older state the pages a later save overwrote. When the ring is full, the oldest state goes. A pinned state stays: the state just newer than it goes instead, and its record is folded into the pinned one's, so that the pinned state can still be rebuilt from the newest one. A new memory mapping still drops it, and `SSBU_RB_DONE` says so.

Checked on our runs: recorded fights replayed after `loadbase`, one after the other in the same game, are identical frame for frame to the same fights in a freshly started game, and a search of some 49,000 frames gives the same result twice in a row.

## Building and staying in sync

Build exactly as upstream does ([setup guide](docs/BUILDING.md)). The fork differs from upstream `main` by eight files under `runtime/` and by this README; `git diff <upstream commit> main -- runtime` shows all of it. It is kept up to date by merging upstream `main` from time to time; the one place that usually conflicts is `runtime/hle_pad.c`.

Tested on Windows only. These switches are not supported by the upstream project: please do not report problems with them there.

License: GPL-3.0-or-later, as upstream.

---

*Below: the original project's README, unchanged.*

# OpenSmash-Ultimate

[![Discord](https://img.shields.io/badge/Discord-join-5865F2?logo=discord&logoColor=white)](https://discord.gg/pkgyYGf3YB)
[![License: GPL-3.0-or-later](https://img.shields.io/badge/license-GPL--3.0--or--later-blue)](LICENSE)

**[Website: live players, rooms and leaderboard](https://denkishi.github.io/opensmash-site/) · [Discord](https://discord.gg/pkgyYGf3YB) · [Releases](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/releases) · [Installation guide](docs/INSTALL.md) · [Issues](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/issues)**

OpenSmash-Ultimate turns *Super Smash Bros. Ultimate* into a native PC program by static recompilation. It translates the game's ARM64 code into C once, ahead of time, compiles it, and links it with a runtime that stands in for the Nintendo Switch. The game's code and data come from a copy you dumped from your own console.

It is the merge of [WindowSmash](https://github.com/OpenSmash-Ultimate/WindowSmash) and [ssbu-recomp](https://github.com/OpenSmash-Ultimate/ssbu-recomp); new work, issues and pull requests go here.

## Online

Play online from the launcher or the game's own menus. The **[website](https://denkishi.github.io/opensmash-site/)** shows who is playing right now, the open rooms and arenas, and the GSP leaderboard, overall and per fighter.

- **A server near you.** Official lobby servers by region (EU, NA East and South America; **Auto** picks the one that answers soonest). They relay every match, so no player's internet address reaches another player, and they take signed-in players: sign in once in the launcher (Discord, or e-mail and password). Epic Online Services is still there for those who choose it (`eos` in `online_server.txt`).
- **Rollback netplay** for fights, with no router setup.
- **GSP ranked** matches and a public leaderboard. Sign in with Discord on the website to see your own GSP and rank and to choose whether your name is shown.
- **Discord Rich Presence** with Join: friends can see what you play and join your room.
- Details and troubleshooting: [Multiplayer guide](docs/MULTIPLAYER.md).

> [!IMPORTANT]
> **This repository does not contain the game.** It holds no game files, keys, firmware or saves, and never will. You need your own copy of the game, dumped from your own Switch.

## Install

1. Download **[OpenSmash.exe](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/releases/latest/download/OpenSmash.exe)** from the latest [release](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/releases) and run it. It does the rest.
2. Choose where OpenSmash goes (about 60 GB free; not the Desktop or Documents) and click **Download and set up**. It downloads the project from GitHub into that folder, makes the **OpenSmash-Launcher** and **OpenSmash-Ultimate** shortcuts (desktop and Start menu), and opens the launcher there.
3. In the installer window, check that it found your game files (game plus its 13.0.5 update), keys and optionally firmware; fix any folder with **Browse**. Click **Set up and build** and wait: 30 minutes to a few hours.
4. Press **Play** in the launcher, or use the **OpenSmash-Ultimate** shortcut (the game without the launcher).
5. To update, open **Settings -> Update** in the launcher. Mods are in the launcher's **Mods** view; online is in **Settings -> Online**.

The first start is faster than before.

You need your own copy of the game, version 13.0.5, dumped from your own Switch, and a 64-bit Windows 10 or 11 PC; AVX2 is recommended, SSE4.2 works too. Linux and macOS (Apple silicon) are experimental. The [installation guide](docs/INSTALL.md) explains each step; [docs/BUILDING.md](docs/BUILDING.md) is the step-by-step setup and building from source.

## What's in it

| Area | What you get | Where |
| :--- | :--- | :--- |
| The game | The whole game recompiled for x86-64: menus, every mode, DLC fighters and stages, movies (VP9), music. | here |
| Online | Official lobby servers by region (EU, NA East, South America) that relay every match, one account for all of them, rollback netplay, GSP ranked and a leaderboard on the website, Discord sign-in and Rich Presence, room codes (`XXXXX-XXXXX`) and LAN rooms. | [Online](#online) |
| Mods | The OpenSmash launcher and mod manager, Skyline plugins, ARCropolis mods, a switch for every mod in the game's settings and in the launcher; native mods written against `hook.h`. | [#62](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/62), [#63](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/63), [#67](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/67) |
| New fighters | Fighters with a kind and files of their own (up to 16), all eight colours; fighters written as C++ source. | [#67](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/67), [#68](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/68) |
| Graphics | Vulkan by default (Direct3D 11 stays a choice in `display.ini` and the settings menu); Metal on macOS; shader prewarm and Prepare Shaders; a faster first start. | [#64](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/64), [#26](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/26) |
| Settings | Game language, master volume, controllers (dead zones, invert, ignore), GameCube adapters, keyboard keys, low-latency display mode, the album. | [#56](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/56), [#57](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/57), [#60](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/60), [#3](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/3) |
| Memory | Caps on the shader prewarm and on the copy kept for mod reloads; a memory line in the log. | [#66](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/pull/66) |

## Guides

| Guide | Read it to |
| :--- | :--- |
| [Installation guide](docs/INSTALL.md) | install, update, add mods and go online. |
| [Setup guide](docs/BUILDING.md) | do every setup step by hand, and build from source. |
| [Playing](docs/PLAYING.md) | change the controls or the window, and find saves, screenshots and problem reports. |
| [Multiplayer](docs/MULTIPLAYER.md) | play online, in a private room, or on a LAN. |
| [Mods and Skyline plugins](docs/SKYLINE.md) | install mods and plugins and turn them on or off. |
| [DLC and unlocks](docs/DLC-AND-UNLOCKS.md) | use your DLC, or unlock every base-game fighter on your save. |
| [The 13.0.5 update](docs/UPDATES.md) | see which copies of the game count as 13.0.5 and how the update is applied. |
| [Linux](docs/LINUX.md) · [macOS](docs/MACOS.md) | install, build and run the experimental versions. The OpenSmash Launcher (`opensmash-launcher.sh`, `OpenSmash Launcher.command`) does setup, updates, Play, settings and mods in a window, with a game pad too (Steam Deck Game Mode). |
| [How it works](docs/TECHNICAL.md) | understand the recompiler, the runtime and every tool. |

UI language: the launcher and external menu offer English, Simplified Chinese, Japanese, Korean, Spanish, French and German, with a shared **Interface language** setting. See [UI languages and translation guide](docs/LOCALIZATION.md) for adding a language or translating new UI text.

## `ssbu-recomp/`

[`ssbu-recomp/`](ssbu-recomp/) holds ssbu-recomp as it was imported, with its history, so `git log` and `git blame` still show who wrote what. Its parts move out of the folder as they are ported to the root runtime, and the folder goes when it is empty. Material taken from the game was removed from every commit of its history on import.

## Getting help

- **Questions and finding players:** the OpenSmash-Ultimate Discord, **[discord.gg/pkgyYGf3YB](https://discord.gg/pkgyYGf3YB)**.
- **Setup problems:** see [If something goes wrong](docs/BUILDING.md#if-something-goes-wrong) in the setup guide.
- **Bugs:** use the **Report a problem** button at the bottom of the launcher, or open the [bug form](https://github.com/OpenSmash-Ultimate/OpenSmash-Ultimate/issues/new?template=bug_report.yml). **`play.log` is required**: it is at `<OpenSmash folder>\build\play.log` (the button opens that folder with the file selected). Drag it into the form. After a crash, also add the `.txt` report from the `reports\` folder. The form also asks for your version, renderer, graphics card and driver, operating system, and the steps that cause the bug.
- **Security problems:** report them privately, as [SECURITY.md](SECURITY.md) explains. Don't open a public issue.

> [!WARNING]
> Never post game files, keys, saves or generated game code in issues or pull requests. Logs and reports can contain your folder names, player names and internet addresses, so read them before posting. A crash report's `.dmp` file holds part of the game's memory: share it only privately, with someone who asked for it.

## For developers

Read [CONTRIBUTING.md](CONTRIBUTING.md) first. Its three rules hold for every change: nothing derived from the game goes into the repository, clean sources only, and every claim is measured.

### Built with AI coding tools

Much of OpenSmash-Ultimate was written with AI coding assistants, directed and checked by its maintainers: reading the game's code in a disassembler, writing the runtime and the tools, running the game headless to prove each change with its own logs and counters, and reviewing pull requests. That lowers the bar to contribute: you don't need to know ARM64, Direct3D or the Switch's services before you start. Describe the problem, let an assistant read the code with you, and hold it to the same rules as anyone else: [CONTRIBUTING.md](CONTRIBUTING.md)'s three rules, a measurement for every claim, and a test that fails without the fix. Say in your pull request what you checked yourself.

OpenSmash's source-only tests use made-up data and need no game files. Run them from the repository root, in PowerShell or a Unix shell:

```
python -m pip install -r tests/requirements-dev.txt
python -m unittest discover -s tests -v
```

CI runs these workflows from [`.github/workflows/`](.github/workflows/):

| Workflow | Runs on | What it checks |
| :--- | :--- | :--- |
| OpenSmash tests | Windows, Linux, macOS | The unit tests. |
| Runtime compile | Windows, Linux, macOS | The whole runtime, with its platform layer, compiles and links into the game's program as a player's build makes it, on a made-up game (`tools/dev/runtime_compile_check.py`); on Windows and Linux for both instruction sets. Every runtime source is in the program unless the script lists it as left out on purpose, and none is in a subfolder no build looks in. |
| ssbu-recomp Linux runtime | Linux | ssbu-recomp's host runtime compiles; its synthetic differential test, platform self-test and compile-scheduler test pass; `setup.py --tools-only` finds every tool and library the build needs; its log collector leaves game files out of a report. Its installer script, `install_deps.sh`, is only checked for syntax: the workflow installs the packages itself. |
| ssbu-recomp macOS runtime | macOS (Apple silicon) | The same on macOS, where `install_mac.sh` does run. It also builds the glslang shim, checks that `setup.py` stops at the dump check when given an empty dump, and reports what MoltenVK offers on the runner. |
| No game data | Linux | No Switch files (`.nca`, `.nsp`, `.nsz`, `.xci`, `.nso`, `.nro`, `.arc`), no key files, and no file over 10 MB, in any commit. |

## License

OpenSmash-Ultimate is free software under the GNU General Public License, version 3 or (at your option) any later version: **GPL-3.0-or-later**. [`LICENSE`](LICENSE) holds the license text. Third-party parts keep their own licenses, listed in [NOTICE.md](NOTICE.md).

The license covers this project's own source. It gives no rights to Super Smash Bros. Ultimate, its files, or anything built from them.

Super Smash Bros. Ultimate and related names belong to their respective owners. OpenSmash-Ultimate is an independent project and is not affiliated with or endorsed by Nintendo.
