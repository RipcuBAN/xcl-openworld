# Task board

Status: `todo` → `doing (<owner>, <date>)` → `review` → `done (<commit>)`. Take a task by editing its
row in the same commit that starts the branch. One owner per task. Acceptance is a run, not an opinion.

Owners: **C** = Claude, **G** = Gemini Neon, **H** = human.

## Milestone 1 – past the loading screen without the watchdog

| # | Task | Owner | Status | Acceptance |
|---|---|---|---|---|
| 1 | Run `openworld` with both watchdog patches; record how far it goes and what is on screen (frame dumps every 300) | C | todo | `FINDINGS.md` row with frames, time, last frame description |
| 2 | Name the hung thread: decode the watchdog's thread table (`0x146844a20`, 64×0x48), find which row passes 300 s and which game thread it is (name, entry point) | G | todo | row index, thread entry RVA, and the code it waits in, from static reading + one run's thread dump |
| 3 | Thread dump at the watchdog: when the report path is reached (`game+0x2215c0c`), log every title thread's RIP and the wait it is in (`XCL_TRACE_WAITS`, upstream PR #924, may already do it) | C | todo | the log names the waiting thread and its wait object |
| 4 | Fix what the hung thread waits for (likely a GPU wait bridged wrongly, a DirectStorage request, an XGameRuntime async, or an audio processor command); upstream PR first if one exists | G (design) + C (runs) | todo | watchdog does not fire within 10 min without `XCL_PATCH` |
| 5 | Pull upstream XCL daily onto `openworld`; test open PRs that touch our walls (#1046 GPU wait release, #889 compute preload, #924 trace waits, #214 PSO concurrency) | C | todo | each tested PR gets a FINDINGS row: better / same / worse |

## Milestone 2 – the world renders

| # | Task | Owner | Status | Acceptance |
|---|---|---|---|---|
| 6 | World scene is dark after the title card: find the first missing pass (frame dumps of render targets, `report.json` untranslated shaders) | G | todo | named pass + the shader/feature it lacks |
| 7 | Untranslated shaders that drop world draws: list them from `report.json`, group by missing instruction/feature | C | todo | table in FINDINGS |
| 8 | wave64 on NVIDIA (wave32): measure which programs differ (`docs/wave64-on-wave32.md`) and their cost | G | todo | list of programs + ms per frame |
| 9 | Memory: commit grows past 30–40 GB in long runs (upstream #534) – measure here, find the owner | C | todo | commit curve over 10 min + top allocations |

## Milestone 3 – playable

| # | Task | Owner | Status | Acceptance |
|---|---|---|---|---|
| 10 | Input reaches gameplay (pad + keyboard), camera and movement | H + C | todo | a person walks and drives for 5 min |
| 11 | Frame time in the world ≥ 20 fps on the RTX 3060 | G + C | todo | fps line over 300 frames in the world |
| 12 | Save/load in the world | C | todo | save, quit, load returns to the same place |
