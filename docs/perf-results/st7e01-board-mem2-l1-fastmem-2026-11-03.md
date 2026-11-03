# Fortune Street (ST7E01): MEM2 and locked-L1 fastmem — 2026-11-03

## Question

On the Castle Trodain board, Fortune Street ran at about 61% game speed on
hardware WebGPU with the JIT forced on. What is the CPU thread spending its
time on, and can the JIT get it closer to full speed?

## Setup

- Machine: Intel i9-14900KF, NVIDIA RTX 5080, Windows, Chrome 156 (headed).
- Disc: Fortune Street (USA), `ST7E01`, NTSC-U. Not committed.
- Harness: Playwright-driven headed Chrome. Boot the disc, load a board save
  state 40 s in, wait for the JIT compile count to level off (≥ 90 s), then
  sample `gameSpeed` every 2 s for 60 s. The save state is local and not
  committed.
- URL: `http://127.0.0.1:8090/?video=wgpu&forcejit=1` plus the flags below.
- Cores:
  - before: `517dbbc4df9cea6f07dd8b2ad0752610840e8759eeaaa2ae78ccb4a35c6e8e7f`
  - after: `b91fb991634eb26a42a0f7d0e5e7d6c086e6c4756884a5aa5d2db851d0d015c6`

## Profile

A Chrome CPU profile of the emulation thread on the board (self time):

| Bucket | Share |
|---|---|
| `ExecuteOneBlock` | 26.4% |
| `RunWasmBlock` | 9.6% |
| `Dispatch` | 5.6% |
| JIT-compiled block bodies | 26.5% |

None of the existing JIT flags helped:

| Flags | Mean | Median |
|---|---|---|
| `forcejit=1` (run 1) | 61.8% | 61% |
| `forcejit=1` (run 2) | 61.3% | 61% |
| `metrics=1` | 61.5% | 61% |
| `jittier=mixed` | 61.0% | 61% |
| `blockmerge=1` | 59.1% | 61% |
| `shortprefix=1` | 55.7% | 58% |
| `fastmemhoist=1` | 60.6% | 61% |

## Cause

Every JIT and import-helper RAM fast path covered only MEM1 (`0x8xxxxxxx`).
Fortune Street keeps much of its hot data in Wii MEM2 (`0x9xxxxxxx`) and
uses the locked L1 cache at `0xE0000000` as a scratchpad for paired-single
math. Those accesses went through import calls, and for `psq_l`/`psq_st` the
full interpreter and MMU.

Import calls during one benchmark run (boot to end of sample, `metrics=1`):

| Counter | Before | MEM2 only | MEM2 + L1 |
|---|---|---|---|
| `fp` imports | 328,766,248 | 258,343,661 | 35,266,435 |
| `lfs` imports | 78,274,877 | 5,658 | 5,772 |
| `stfs` imports | 12,896,390 | 1,723 | 1,769 |
| `lfd` imports | 1,160,193 | 2,122 | 2,165 |
| `psq-mem` (psq outside fast RAM) | 294,856,391 | 223,876,508 | 0 |
| `psqfast` (C++ psq fast path) | 23,956,018 | 34,337,021 | 35,140,684 |

After the MEM2 change, the two hottest `psq-mem` sites were float `psq_st` at
`0xe0000390` (121.5M) and float `psq_l` at `0xe0000120` (102.3M).

## Change (patch 0069)

- The import helpers resolve MEM1 first, then MEM2, through one
  `ResolveFastRamAccess` helper.
- On Wii, the JIT RAM check computes the host address for MEM1 or MEM2 into a
  new Wasm local, and every direct RAM access uses that local. GameCube
  layouts have no MEM2, so their code generation is unchanged by this part.
- Float `psq_l`/`psq_st` get an inline locked-L1 path, using the same EA check
  as the existing `lfs`/`stfs` L1 paths. This also applies to GameCube discs.

Like the existing MEM1 fast paths, the MEM2 check maps
`EA & 0x3fffffff` straight to physical memory and ignores BATs. Games map
`0x9xxxxxxx` to MEM2 with the standard BAT setup.

## Result

| Core | Mean | Median | p10 |
|---|---|---|---|
| before (`forcejit=1`) | 61.3% | 61% | 60% |
| MEM2 only | 66.5% | 67% | 64% |
| MEM2 only, `fastmemhoist=1` | 67.6% | 67% | 66% |
| MEM2 + L1, `metrics=1` | 68.7% | 71% | 67% |
| MEM2 + L1 | 69.4% | 69% | 67% |

On the board, game speed rose from about 61% to about 69% (+13% relative).
Static menus were already near 100%.

## Correctness

- The board frames match the old core by eye.
- Gameplay smoke on the new core: rolled by hitting the die with the 2
  button, then walked with the D-pad from the bank to the next junction. The
  camera, HUD, and die result rendered correctly. The old core behaved the
  same with the same inputs, only slower.
- `npm run check` and `node --test tests/*.test.mjs` pass (914 pass,
  15 skipped).
- No GameCube disc was available locally, so the GameCube effect of the L1
  `psq` path (for example on Melee) was not measured.

## Next

The remaining 35M `fp` imports are non-float `psq` types (U16, U8, S16), which
already take the C++ fast path. Block dispatch (about 42% of CPU time in the
profile) is the next target.
