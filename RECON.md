# Phase 1 reconnaissance

Date: 2026-10-20. Upstream state: ps3recomp master `749182e` (v0.16.0, merge of #377, 2026-10-19).
Read-only: nothing was built, installed or posted. No game files, firmware or lifted output were copied.

## The finding that changes the plan

**Someone already has LittleBigPlanet 1 running natively on Apple Silicon.**
[benjidotsh/LittleBigRecomp](https://github.com/benjidotsh/LittleBigRecomp) (Benjamin Janssens, created 2026-10-17, 27 commits, HEAD `10a129c` on 2026-10-18).

- Target: **BCES00141, disc v02.00** (EU disc). macOS arm64, Metal, SDL2.
- README claim (not verified by running it): boots through first-boot save creation, the "Important Information" notice, logos and the Bink intro movie, then loads **the Introduction level**. The level renders, the pad works and START opens the pause menu, at about 30 flips/s. Known limits: an occasional bright wedge on one frame, depth/stencil clears ignoring the scissor, and FMOD's relocated DSP plugins running in the SPU interpreter.
- It lifts every SPU program LBP runs (FMOD, Bink, the WWS job-manager policy module and its 43 job modules, a voice codec job) at build time with `tools/relift.sh` / `relift_spu.sh`. No game code or derived output is committed.
- ps3recomp is a submodule pointing at `benjidotsh/ps3recomp`, branch `lbp`.

**It cannot be reproduced from what's public.** The submodule is pinned at `8a6a0cb`, which is not on any public ref. The public `lbp` branch stops at `d15591b`: 7 commits on top of upstream v0.15.0 (`0fe7dbf`), 119 commits behind current master. The 18 "Bump ps3recomp" commits after that (`56506df`..`10a129c`) point at runtime fixes nobody else can fetch: FP DDX/DDY, 59.94 Hz vblank and semaphore-ACQUIRE flip pacing, SPURS workload maxContention and idle backoff, SPU interpreter decode cache, NV3089 swizzle, WPOS/AA scale, scissored colour clears, ring-wrap callback. Those are the fixes that took it from "boots" to "playable Introduction at 30".

benjidotsh has **no PRs** on sp00nznet/ps3recomp (open or closed), and isn't in the README's community list.

The public `lbp` commits:

| Commit | What |
|---|---|
| `629b8c3` | **macOS boot**: guest runs on a 512 MB-stack pthread, `CFRunLoopRun` on the main thread, guest VM via `mmap(MAP_NORESERVE)`. Only touches `runtime/ppu/tests/boot_main.cpp` |
| `72ac44e` | cellFs octal `CELL_FS_O_*` flags, `O_EXCL`, truncate/rename/unlink on the ppu_fs fd table |
| `2d039dc` | cellSaveData always writes PARAM.SFO on setParam; save root from `PS3_VFS_ROOT` |
| `76169a5` | synchronous loopback connect, `sys_net_get_sockinfo`, sockets without NetCtl online |
| `2b3b108` | SPU atomic event_status updates, SN re-assert on WrEventAck during list stalls |
| `ed4efb6` | RSX G8B8 lane order, GPU depth-as-ARGB sampling, faster texture hashing |
| `d15591b` | thread-exit trace hook, usleep log limiter |

None of these are upstream.

## Questions the plan asked

### Which LBP does sagemono target, and how far does it get?

- Title: **BCUS98148** (US disc), per comments in their `lbp/main.cpp`. Their EDAT fix (`3847bd2`) was checked against NPEA00241's `data.edat`. They run an update EBOOT from hdd0 (`9c4af3b` says "e.g. v1.30"); the exact version isn't recorded.
- Furthest documented state (July 2026): intro movie plays in lifted SPU code (`c13ff13`). Past the loading freeze only with the `LBP_HLE_JOBDONE` hack that fakes job completion (`8e8ee0c`): it streams 200+ archives, then hits a new wall at guest `0x00416C27`. September fixes (#174–#178) went after the audio tasks and FMOD overlay. **Nothing claims menu, pod or gameplay.** No sagemono commits since 2026-09-15.
- There is no separate LBP repo. The port was an in-tree `lbp/` directory (`CMakeLists.txt`, `main.cpp`, `spu_dispatch_mt.c`, `gen/`) that upstream deleted in `e281532` (2026-09-21, v0.13.0) and generalised into `templates/project/` plus `tools/extract_jobmods.py` / `lift_jobmods.py`.
- `lbp/` was Windows-only: clang-cl with `/bigobj /EHsc /MAP`, a prebuilt `.lib`, D3D12/XInput, and `main.cpp` lines 181–582 (present, vblank, watchdog, job-done hack) all inside `#ifdef _WIN32`.
- libsre: **HLE on the PPU side** (`ps3_load_prx_modules` a weak no-op, runtime `cellSpurs`), with lifted Sony SPU code on the SPU side (SPURS kernel and policy modules, WWS job manager, 89 job modules matched by 16-byte signature in `runtime/spu/spu_channels.c`). Per-build tables became optional in `023d8a4` (#164) because they only match one EBOOT.
- **Every sagemono branch is already merged upstream** (ancestors or squash-merged as #162–#178). Nothing unmerged.

**Legal hazard:** sagemono's `origin/main`, `origin/master`, `dev/cellmark` and `spu/faithful-adopt-caner` carry `fw_spu/` with decrypted Sony firmware PRXs and C lifted from libsre. Upstream untracked it in `0952e1c` but it's still in history, and old upstream history also has `lbp/gen/` tables fingerprinted from LBP's EBOOT. Don't base anything on those refs or check out old revisions for reuse.

### Where does `lbp/CMakeLists.txt` live?

Nowhere current. It was deleted by `e281532`. Before that, `docs/GETTING_STARTED.md:221` read: "`lbp/CMakeLists.txt` in this repository does exactly that and is the working reference; the starter template does not set it for you." The line now names `templates/project/CMakeLists.txt` but keeps the "starter template does not set it for you" clause, which contradicts itself (small docs fix for upstream).

### macOS boot patches: inFamous (NoahBPeterson)

The inFamous work is one 112-commit series on `NoahBPeterson/ps3recomp` branch `infamous/rebase-v0.13.0` (tip `770660a`), which is **upstream PR #213, still open**. Only two pieces have merged: the lifter (#286) and exact SPU float/VMX with NEON (#287). The maintainer's blockers on #213: 61.6 MB of test goldens, game SPU bytes in a test file, default-behaviour changes affecting every title, a Linux link failure, ~17 conflicting files. No public inFamous 1 game repo exists. The best public stand-in is `erikomis/infamous2-recomp` (`infamous2-lle/`), which pins `770660a` and applies a 76k-line patch.

Most of the POSIX boot plumbing predates Noah and is Caner Saka's (see next section). What only Noah's branch has:

| Area | Commits | Upstream? |
|---|---|---|
| LLE firmware boot: `runtime/syscalls/lv2_prx.c` (PRX syscalls), `ppu_run` enters liblv2 `module_start`, `sys_lwsync.c` | `f75dcac` | No |
| Firmware SPU image handling and lifter (`tools/lift_firmware_spu.py`, `FIRMWARE_SPU_DIR`) | `4d2f7b0`, `70c456e` | No |
| arm64 varargs on the stack for liblv2 printf | `4d2f7b0` | No |
| Real lwmutex/lwcond on POSIX | `950cfd7` | Fixed independently upstream in `25a19b9` (#225) |
| Darwin TLS off the hot path, `_setjmp` instead of `setjmp` per SPU entry | `b2f87a2`, `363430f` | No |
| TTAS lock-line lock, lock-free GETLLAR | `1b847cd`, `e3ab49d`, `b709a08` | No |
| Single-copy-atomic guest memory access (`guest_mem_atomic.h`) | `0a50968` | No |
| Shared MTLBuffer staging ring (+16% RSX fps) | `662fe45` | No |
| Template presented every flip twice | `73c271e` | **No, bug still in upstream `templates/project/main.cpp`** |
| POSIX page guard reading arm64 PC | `9e944e4` | No |
| Readable-memory checks before stack dumps | `8a113db`, `e907607` | Fixed differently upstream in `8ca1d05`; will conflict |
| POSIX audio pacing (mixer ran 2.6x real time) | `93aa8df` | No |
| Keyboard pad from the Metal window | `3072636` | No |

Only in erikomis's tree: guest off the AppKit main thread with a `CFRunLoopRunInMode` loop on main (`b395749`). Without it the Metal window deadlocks grey because the backend `dispatch_sync`s to the main queue. benjidotsh's `629b8c3` solves the same problem the same way. **Neither upstream's template nor Noah's has it.**

### macOS boot patches: Yakuza (canersaka)

`canersaka/yakuza-dead-souls-ex` is a diverged in-tree copy of ps3recomp (no submodule). Every public ref is Windows/D3D12 only; no ref has ever had a Metal or macOS file. The Yakuza macOS runner is not public: upstream's `tools/prepare_yakuza_macos.py` patches an external `main.cpp` that already uses `CFRunLoop`, and that file isn't in any public ref. Whether Yakuza plays on macOS is unconfirmed (Caner's PR says "controllable gameplay has not been verified").

Caner's macOS work went upstream as **#160** (merged `972ce85` via #171, v0.12.0; 117 commits, +16.8k lines):

| Piece | Commit / file |
|---|---|
| Win32 shim over POSIX: interlocked ops, SRW/CS/condvars, events, threads with stack sizes, `VirtualAlloc` over mmap | `787d716`, `runtime/platform/win32_compat.{c,h}` |
| Vectored exception handlers over `sigaction` | `955ef96` |
| `SuspendThread`/`GetThreadContext` via Mach, arm64 branch | `8b89c04` |
| `VirtualQuery`/`IsBadReadPtr` via `mach_vm_region` | `84b34ce` |
| Guest-pointer trap on SIGSEGV **and** SIGBUS (Darwin) | `abad89e`, `runtime/ppu/ppu_loader.cpp:147-260` |
| 4 GB-aligned reservation, host page size (16 KB on arm64) | `e12cf5d`, `d77930a`, `runtime/memory/vm.h` |
| SPU MFC DMA demand-commit on POSIX | `eaface5`, `a10c9b4` |
| Backtraces via `backtrace(3)`/`dladdr` | `ba3ff18`, `runtime/platform/win32_backtrace.h` |
| 256 MB PPU host stacks | `8f4f5f3`, `runtime/syscalls/sys_ppu_thread.c:33` |
| Synthetic boot-smoke title (scaffold links and runs on macOS in CI) | `51abaa4` |
| Portable `templates/project` runner | `9ac5fe0` |
| Metal window on main queue, draw-engine feed | `620324d`, `25272dd`, `2f95b5d` |

Later upstream pieces: ld64 weak-symbol linking (`838921e`, `d8dc41b`, `018f880`, sp00nz, v0.16.0); LP64 spinlock width fix (`67e18b9`, tekulvw, #326, confirmed: `volatile long*` → `LONG*`, test `tools/test_spinlock_word_width.py`).

### What's upstream vs downstream (summary)

| | Upstream master | Downstream only |
|---|---|---|
| macOS build + CI, Metal backend, guest shaders via glslang + spirv-cross | yes | |
| POSIX shim, signal trap, stacks, backtraces, `__thread` TLS | yes (Caner) | |
| Boot scaffold links and runs a synthetic title on macOS | yes | |
| **Boots a real title on macOS** | **no** (README still says no) | benjidotsh (LBP1), Noah (inFamous), erikomis (inFAMOUS 2) |
| Guest off the main thread + CFRunLoop | no | benjidotsh `629b8c3`, erikomis `b395749`, Caner (private) |
| VM via mmap reserve on POSIX | no: `calloc(0xE0000000)` in template and `boot_main.cpp:1142` (and `PLATFORM_ABSTRACTION.md:234` describes an mmap scheme the code doesn't use) | benjidotsh `629b8c3` |
| LLE liblv2/libsre boot | no (hook is an empty stub; FIRMWARE_LLE proven on Yakuza only) | Noah #213 |
| All of sagemono's LBP fixes | yes | |
| benjidotsh's LBP runtime fixes | no | 7 public + ~18 unpushed |
| Template double-present fix | no | Noah `73c271e` |

Still Windows-only upstream: write-guard VEH (`ppu_loader.cpp:1049-1178`), hang watchdog, `PS3_DEBUG` debug console (so `DEBUG_CONSOLE.md` doesn't apply on macOS), `PS3_SAMPLE` sampler. No tree calls `sigaltstack()`, so a guest stack overflow faults on the overflowed stack.

### Other repos

- `shipa-2/ps3recomp-planetary`: nothing LBP-specific. `master` is 9 commits (a Plume/Vulkan renderer) on a September base, 558 behind; `backup/spurs-ef-work` is 9 sp00nz commits already upstream. Ignore.
- `AlvBarros/littlebigplanet-recomp`: 404. Private or deleted; no mirror found.
- Issue #139 (POSIX CI red) is closed. No open issue or PR mentions LittleBigPlanet or "boot scaffold".

## Corrections to the plan

- "Do not start a fourth independent LBP port": there are now effectively two existing LBP1 ports, and the one that matters (benjidotsh) is already on macOS and further than sagemono's Windows port ever documented. Phases 4–6 as written would redo work benjidotsh has done.
- CHANGELOG.md stops at v0.14.0. The v0.15.0 and v0.16.0 notes are in README.md's "Changelog" section.
- `docs/BUILDING.md`'s macOS section is stale: CI installs `ninja sdl2 glslang spirv-cross`, and without the last two Metal is fixed-function only. Phase 2's install list should include them.
- Phase 3 says lift the v1.30 update EBOOT. benjidotsh used disc v02.00 with no update mentioned. Per-build SPU/jobmod tables only match the EBOOT they were generated from, so matching benjidotsh's build (or not) decides how much of their tooling and debugging carries over.
- GETTING_STARTED's `lbp/CMakeLists.txt` reference is gone; `templates/project/CMakeLists.txt` is the reference now.

## Recommendation: which tree to base on

**Not master alone, not sagemono's fork.** sagemono's fork has nothing unmerged and its old refs carry firmware. Master alone means re-solving what benjidotsh already solved.

Base on **benjidotsh's work, rebased onto current upstream master**, and contribute to it rather than beside it:

1. First, get `8a6a0cb` (and whatever followed) pushed. Ask benjidotsh directly. Without it there's no reproducible LBP-on-Mac tree anywhere.
2. Until then, the fallback base is upstream `749182e` + benjidotsh's 7 public `lbp` commits rebased onto it (119 commits of drift, mostly Metal and SPURS; expect conflicts in `rsx_metal_backend.m`, `rsx_draw_engine.c`, `spu_channels.c`, `cellSpurs.c`). That reproduces boot-to-something, not the 30 fps Introduction.
3. Keep LLE (Noah's #213) out for now. benjidotsh got to the Introduction on HLE SPURS, and #213 is a 112-commit open PR with known blockers.

Where the useful contribution now lies, given that state:

- **Upstreaming.** benjidotsh's boot fix (`629b8c3`) is the piece that makes upstream's README table say "yes" for macOS. Moving it from `boot_main.cpp` into `templates/project/main.cpp` with a test, plus the template double-present fix, are small general PRs. benjidotsh's runtime fixes need to go upstream as one-concern PRs too; offering to do that splitting (with credit) is the most valuable thing I can see.
- **Past the Introduction.** Phases 7–8 (physics determinism against RPCS3, the pod, story levels, Create mode, saves, audio) are untouched by anyone on macOS.
- **The listed known limits** (wedge flash, scissored depth/stencil clears, FMOD DSP plugins in the interpreter).

## Open questions for you

1. Which LBP copy do you have (title ID, update version)? If it's BCES00141 v02.00, you can follow benjidotsh's build directly.
2. Are you fine with the plan pivoting from "port LBP" to "contribute to benjidotsh's port and upstream it"? Phases 4–6 change shape if so.
3. Phase 2 needs your Mac; this session ran in a Linux cloud container, so nothing here has been built or run.

## Sources

Local clones used for this (all read-only): upstream `sp00nznet/ps3recomp`, `sagemono/ps3recomp`, `NoahBPeterson/ps3recomp`, `erikomis/infamous2-recomp`, `canersaka/yakuza-dead-souls-ex`, `shipa-2/ps3recomp-planetary`, `benjidotsh/LittleBigRecomp`, `benjidotsh/ps3recomp` (`lbp`). PR and issue state from github.com pages on 2026-10-20 (the GitHub API is blocked from this environment).
