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

### Mesh pipelines
- `main` refuses ~40 mesh pipelines whose console MS sits in the AS slot (no MS subobject). PR #541
  translates them; with it, 0 refused and the picture is unchanged.
