# Findings

Measured on the test machine in `README.md`. Each entry names the build and what showed it.

## 2026-11-24

### Rendering reaches the window
- `main` + Agility SDK: the title presents at a steady 30 fps; frames are black for the first ~12 s,
  then the loading screen artwork (frame dumps from ~frame 380 on, 8153–8160 of 8160 samples non-dark).
- Without the Agility SDK (inbox D3D12 on Windows 10), root signatures with flags `0x400`/`0x800`
  (`*_HEAP_DIRECTLY_INDEXED`, SM 6.6) are refused and the title dies at `game+0xb5a5f8` within seconds.

### How far each build gets (seconds to the first fault, frames presented)

| Build | Runs | Result |
|---|---|---|
| `main` | 2 | 95 s / 2100, 55 s / 1200 – AV in `VCRUNTIME140.dll+0x1dcf3` (memcpy) |
| `main` + PR #541 | 2 | 125 s / 3000, 100 s / 2400 – same memcpy AV |
| `main` + #541 + #684 (`openworld` before `XCL_PATCH`) | 3 | ~165 s / 3300 ×2, 330 s / 8100 ×1 – title watchdog, null write at `game+0x23cf683` |
| same + `XCL_PATCH` on `+0x23cf683` only | 2 | 315 s / 7800, 225 s / 4800 – watchdog's second kill, null write at `game+0x2215c35` |
| same + `XCL_PATCH=…;0x2215bdc:732e:9090` | – | not run yet |

`XCL_PSO_WAIT_MS=200` did not help: one run ended with `DXGI_ERROR_DEVICE_HUNG`, one in the watchdog.

### The title's stall watchdog
- Loop at `game+0x142215af6` (VA). It reads a table of registered threads at `0x146844a20` (64 rows of
  0x48 bytes, last-beat times at `r13 + 8*rdi`, `r13` starting at `0x146844a28`) and compares each thread's
  silence with a float threshold at `0x146159e20` = **300.0** (seconds).
- Over the threshold: `jae` at `game+0x142215bdc` → report path → `game+0x23cf683`
  (`mov dword [0], 0x4d8bd421`, a tagged deliberate kill) or `game+0x2215c35` (`mov dword [0], 0`; `int3`).
- So a fault there means **one registered thread has not beaten for 300 s**. The kill is the symptom;
  the hung thread is the bug. Not yet known: which row/thread, and what it waits on.

### The GPU hangs right before the watchdog fires
- Run `nowd-1` (`XCL_PATCH` on `+0x23cf683`): `D3D12 device removed: 0x887a0006` (DXGI_ERROR_DEVICE_HUNG) at
  314.986 s, the watchdog's null write at 314.988 s. Run `psowait200-1`: device removed at 102.7 s.
- So the thread the watchdog names is most likely the render thread waiting on a hung GPU, not a CPU-side
  deadlock. Hypothesis (not verified here): a translated shader loop that never ends. Five Star Recomp
  (GPL-3.0, ideas only) reports the same class of hang in this title and bounds it.
- XCL has no general loop iteration bound in translated shaders (only the split-wave64 `s_barrier` spin,
  upstream #798, already in `main`), and no open upstream PR for one (searched 2026-11-24).

### The GPU hang is the split-wave64 spin waits (NVIDIA, wave32)
- Translated loops are already bounded (`LOOP_GUARD`, `xcl_budget = 1u << 16`); the split-wave64 meet and
  the safe `s_barrier` spin up to `1u << 20` atomic polls each. Before every hang the fps fell 30 → 21 → 16 → 13,
  then `0x887a0006` – consistent with spins growing past Windows' 2 s TDR (`TdrDelay` unset here = 2 s).
- `XCL_WAVE64_SPIN=0` (real group barriers instead of spin waits) + both watchdog patches: **600 s, 17 700
  frames, steady 30 fps, no device removal, no fault** (run `nospin-1`). Before: device removed or watchdog
  at 100–315 s on every run.
- `XCL_DRED=1` is not usable here: with it the title froze at 9 s (no log line for 9 min).
- `XCL_WAVE64_SPIN=0` **without** watchdog patches: watchdog at 262 s, still 0 device removals – so a
  second, CPU-side stall remains (task 2). Frame dumps: profile selector at ~50 s, story-loading art at
  ~200 s. One DirectStorage request failed in that run (`0x89240009`, 128 KiB of
  `data\ttdb\props\octa_imposters.ttpf` at 206 s); it did not recur in the next run (traced, 0 failures).
- With both patches + `XCL_WAVE64_SPIN=0` (run `dsdiag-1`): 600 s, 17 700 frames, the title sits at the
  **profile selector waiting for A** – no run so far had any input, so nothing past it was reached.

### Mesh pipelines
- `main` refuses ~40 mesh pipelines whose console MS sits in the AS slot (no MS subobject). PR #541
  translates them; with it, 0 refused and the picture is unchanged.
