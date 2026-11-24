# coop: getting the retail title to a playable open world

This folder coordinates two agents working on this fork of [XCL](https://github.com/ivall/xcl):

- **Claude** (Claude Code, Claude Opus 5.5): fast loop – builds, runs, measurements, integrating upstream
  PRs, small runtime fixes, instrumentation.
- **Gemini Neon**: slow, deep work – static analysis of the title's code, root causes that need long
  reading, larger runtime designs.

The human (RipcuBAN) owns the machine, runs anything that needs a person, and decides priorities.

**Goal:** the title boots, loads the world and is playable in the open world (walk/drive, camera, HUD)
on the test machine below, without the stall watchdog firing.

## Files

| File | What it is |
|---|---|
| [`TASKS.md`](TASKS.md) | The task board: who owns what, status, acceptance check. Edit it when you take or finish a task. |
| [`FINDINGS.md`](FINDINGS.md) | Measured facts only, with the run, commit and log that show them. |
| [`PROMPT_CLAUDE.md`](PROMPT_CLAUDE.md) | Start-up prompt for a Claude session. |
| [`PROMPT_GEMINI.md`](PROMPT_GEMINI.md) | Start-up prompt for a Gemini Neon session. |

## Test machine

Windows 10 22H2 (19045), Ryzen 5 7600X, NVIDIA RTX 3060 12 GB (wave32 only), Rust 1.98.1, DXC 1.9.2602.
Windows 10's inbox D3D12 stops at SM 6.5, so the D3D12 Agility SDK 1.620.2 `D3D12Core.dll` sits in
`%LOCALAPPDATA%\XCL\D3D12\` (XCL loads it itself; see `crates/xcl/src/gpu/agility.rs`).

## Branch `openworld`

`main` of upstream XCL plus:

| Commits | From | What |
|---|---|---|
| `3c1dfd1`…`6204385` | upstream PR #541 (+ the recompiler commits it is stacked on) | console MS in the AS slot translated as a mesh shader (0 refused mesh pipelines, was ~40) |
| `dbee491` | upstream PR #684 | XG `CopyFromSubresource` 7-argument ABI – fixes the `memcpy` AV in `VCRUNTIME140` at 55–125 s |
| `20d3d90` | this fork | `XCL_PATCH=<rva>:<old>:<new>` checked byte patches (diagnostic) |

## Run

```powershell
cargo build --release --workspace
# NVIDIA / wave32 GPUs: real group barriers instead of split-wave64 spin waits (the spins hung the GPU)
$env:XCL_WAVE64_SPIN = '0'
# keep the title's stall watchdog from ending the process (until the loading stall is fixed)
$env:XCL_PATCH = '0x23cf683:c704250000000021d48b4d:9090909090909090909090;0x2215bdc:732e:9090'
target\release\xcl.exe run <game folder>\game_scarlett_master.exe
```

With these two the title runs 10 min at 30 fps on the test machine (twice) and waits at the profile
selector for **A** (Enter on the keyboard, with the game window focused).

Logs and `report.json`: `%LOCALAPPDATA%\XCL\<title>\logs\`. `XCL_FRAME_DUMP=N` dumps every Nth frame there.

## Rules (same as upstream, plus two)

- **Never commit game data**: no archives, executables, shaders, textures, audio, dumps, logs that quote
  game strings, or anything taken from the game folder. Offsets, hashes and counts are fine.
- **Licence: GPL-2.0-only.** No code from VIctory or Leonida (GPL-3.0), nor from WinDurango (LGPL-3.0).
  Reading them for ideas is fine; copying code is not.
- One task per branch (`claude/<task>`, `gemini/<task>`), merged into `openworld` after a run shows it
  does not regress (frames reached and time to the first fault, against the `openworld` head).
- Results in `FINDINGS.md` and `TASKS.md` come from runs you did; say which run.
- Upstream first: before starting, check `gh pr list -R ivall/xcl --search "<keywords>"`. If upstream has
  a PR for it, test that PR instead of writing a second one.
