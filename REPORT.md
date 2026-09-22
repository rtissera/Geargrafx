# GearGrafx performance work — handover report

Branch: `perf-optimizations` on `fork` (https://github.com/rtissera/Geargrafx.git)
Baseline for every number below: `78fc3d7` (merge-base with `main`, pre-optimization).

## 1. What this branch is

It started as five commits produced by another agent (Qwen). Those were reviewed,
two were dropped as measurably worthless or harmful, and three further
optimizations were added. Every change is **bit-identical in output** to the
pre-optimization baseline, verified on the full PCE library.

Current history (oldest first):

| commit | change | measured |
|---|---|---|
| `9c1e0c9` | VCE: remove dead `cycles_to_hsync` computation | −2.39% |
| `2e4f1e2` | VDC: `RenderBackground` bitplane LUT | −1.34% |
| `a7b8c3f` | VCE: 32-bit load/store for RGBA frame fill (non-SGX) | −2.31% |
| `07d3051` | VCE: specialise `Clock` loop on `m_speed` | **−4.17%** |
| `f1c57db` | VCE: same 32-bit store for the SGX path | −1.3…−2.0% (SGX only) |
| `3f749c6` | VCE: divider-3 modulo → lookup table | +2.5…+2.9% (6 of 192 games) |

Dropped from Qwen's original set:

| commit | why |
|---|---|
| `e47ff22` | hoist `m_clock_divider` into a local — **+0.52% mean (a regression)**; +2.94% on Aoi Blink |
| `c66e4d0` | sprite bitplane LUT — **−0.13%**, i.e. nothing; cost +8 instrs, +5 loads, a stack frame |

Backup of Qwen's original tip is at local branch `backup-perf-optimizations` (`c66e4d0`).

## 2. Results

**x86-64 wall clock, 192 games × 1800 frames, best-of-3, pinned core:**

| | speedup vs `78fc3d7` | games slower |
|---|---|---|
| Qwen's 5 commits as submitted | 7.17% | 7 |
| after dropping the two bad ones | ~7.2% | — |
| after the `m_speed` specialisation (`07d3051`) | **22.23%** | **0** |

median 22.51%, stdev 3.74%, range 5.5%–58.1%.

**AArch64 / ARMv7 / RV64GC, exact instruction counts** (qemu + TCG `insn` plugin),
measured at the `a7b8c3f` point:

| target | reduction | n | stdev |
|---|---|---|---|
| AArch64 | 5.62% | 30 | 0.52% |
| ARMv7 | 7.12% | 24 | 0.41% |
| RV64GC | 8.50% | 22 | 0.50% |

`07d3051` added a further −4.17% on AArch64 on top of that.

Note wall-clock gain far exceeds the instruction-count reduction (22% vs 11% on
the same code): removing an unpredictable branch from a 45M-iteration loop pays
extra on an out-of-order core. ARM handhelds should land between the two figures.

## 3. Where the time actually goes (current baseline)

callgrind self cost, 900 frames, games actually running (see §5 on why that matters):

| | HuCard Bomb | HuCard R-Type | SGX Aldynes | CD Sapphire | CD Dracula |
|---|---|---|---|---|---|
| **VCE `Clock`** (huc6260_inline) | **31.1%** | **33.5%** | **30.5%** | **26.4%** | **27.9%** |
| **VDC `Clock`** (huc6270_inline) | 15.4% | 17.9% | **22.2%** | 12.6% | 13.4% |
| `RenderFrameTemplate` | 3.9% | 4.8% | **11.7%** | 3.3% | 2.8% |
| `RenderBackground` | 3.8% | 5.4% | 3.0% | 3.1% | 3.8% |
| `RenderSprites` | 2.9% | — | — | — | — |
| HuC6280 CPU | 7.4% | 5.6% | 3.3% | 6.5% | 6.2% |
| core clock loop | 6.4% | 5.5% | 4.7% | 6.1% | 6.0% |
| audio | 4.4% | 3.8% | 2.7% | 5.1% | 5.0% |
| **ADPCM** | — | — | — | **4.1%** | **4.1%** |
| **SCSI** | — | — | — | **3.1%** | 2.7% |
| `LineEvents` | — | 2.2% | 2.5% | — | — |
| `FetchSprites` | — | — | 2.2% | — | — |

The CPU core is only 3–7%. This is a **video-bound** emulator; optimizing the
HuC6280 is not where the time is.

## 4. Next candidates, ranked

**1. CD guard-check elimination — ~4–5% on CD titles.**
SCSI, ADPCM and audio are clocked every master cycle and early-out almost every
time. `scsi_controller_inline.h`'s compound guard
(`m_next_event == SCSI_EVENT_NONE && m_next_load_cycles <= 0 && !m_bus_changed && m_auto_ack_cycles <= 0`)
alone is **1.81%**; ADPCM's two `IS_SET_BIT(m_control, …)` guards are 1.01% each.
ADPCM's actual work (`Sample()`) is 663K instructions against ~170M in its guards.
Fix shape: give each subsystem a "next event in N cycles" countdown so the core
loop skips the call entirely. Care needed: every countdown must be invalidated
when the relevant registers change. This is the biggest remaining win and the
only one that is structural rather than a peephole.

**2. VDC `Clock` — 12.6–22.2%.** Largest unexplored block. Hot lines:
`if (m_active_line && m_h_state == HDW)` 3.75%, `pixel = m_line_buffer[idx]` 3.48%,
the two transfer-pending guards 3.9% combined (already tried merging — no effect).

**3. VCE `Clock` — 26–33%.** Still the top item but now spread thin; no line above
3.6%. Peepholes are exhausted here. Would need restructuring, e.g. precomputing
the visible span per line to hoist the window test, or batching pixels instead of
stepping master clocks.

## 5. Methodology — read this before trusting or extending any measurement

**The emulator is not deterministic by default.** `geargrafx_core.cpp:101` seeds
its RNG from `time(NULL)`. Two runs of the *same* binary disagree. Every A/B
comparison must pin it — via `LD_PRELOAD` of a fake `time()` for dynamic builds,
or a source patch for static cross-compiled ones.

**Games do not boot on their own.** With a no-op input pump the CD titles sit on
the syscard3 splash forever — two different CD games produced byte-identical
output at 1800 frames. HuCard titles sit in attract mode. The harness must inject
periodic RUN/I presses. An early CD profile in this session was invalid for
exactly this reason and had to be redone.

**qemu wall-clock time is not a valid proxy.** TCG over-weights branches and
loads; it reported the sprite LUT as 2× *slower* when instruction counts showed
−18%. Use the `insn` TCG plugin for exact instruction counts instead — they are
deterministic and immune to CPU contention, so parallel jobs don't corrupt them.

**Source-level "this folds away" reasoning is unreliable** at `-O3` with this much
inlining. The rejected phase-counter variant cost ~1% even on divider-4 games
where its code is behind a compile-time-false condition and should vanish
entirely; merely introducing the local perturbed register allocation across all
four `ClockTemplate` specialisations.

**Always include a control** that the change cannot possibly affect. It caught a
false DIFF in this session (a baseline binary accidentally built with a different
harness) and confirmed the SGX store was confined to its branch (−0.000%).

## 6. Scoreboard — what kind of change actually wins

Nine changes were measured this session. The split is sharp:

**Won** (all remove work the compiler cannot remove):
`9c1e0c9` dead code · `2e4f1e2` variable-shift chain → LUT · `a7b8c3f`/`f1c57db`
byte-copy → word copy · `07d3051` branch chain → compile-time constant ·
`3f749c6` integer division → LUT

**Measured zero or worse** (all tidy things GCC already handles):
`e47ff22` hoist a member load · `c66e4d0` sprite LUT · merging two VDC guards ·
removing a provably-dead clamp in the min-chain · carrying the mod-3 phase

Do not commit a peephole on the strength of reading the source. Measure it, on
several workloads, with a control.

## 6b. Lookup table sizing — measured, do not "optimize" these

Both LUTs are larger than the functions they encode, and both can be shrunk
*correctly*. Both shrinks were implemented, verified exhaustively and on 203
titles, measured — and **reverted, because both are slower.**

| table | committed | reduced form | cost of reducing |
|---|---|---|---|
| `k_huc6260_mod3` (huc6260.h) | 1365 B, `v[m_hpos]` | 84 B — `64 == 1 (mod 3)` so index by `(m_hpos>>6)+(m_hpos&63)`, max 83 | **−1.90%, −1.78%** (R-Type II, Aoi Blink — the divider-3 games) |
| `k_bg_bitplane_lut` (huc6270.cpp) | 1024 B, `values[byte]` | 32 B — nibbles 0-3 come from the byte's high nibble, 4-7 from its low, so `t[b>>4] \| (t[b&15]<<16)` with `u16 t[16]` | **−0.57%, −0.08%** (Bomberman '94, 1943 Kai; scales with `RenderBackground` share) |

Both reductions are exactly correct (verified over every input: 0 mismatches on
all 1365 hpos values and all 256 bytes, then 203/203 titles bit-identical). They
lose because the index arithmetic costs more than the L1 footprint it saves:
1365 B + 1024 B is ~2.4 KB, about 7% of a 32 KB L1D, and both tables are hot
enough to stay resident. One load beats shift+and+add, and one load beats two
loads plus a shift and an or.

**1365 B and 1024 B are the floor for the single-load form.** Anything smaller
requires computing the index, which is the thing the table exists to avoid. If a
future target has a much smaller or more contended D-cache this is worth
re-measuring, but on AArch64 the answer is unambiguous.

## 7. Test harness

Not in the repo — lives in this session's scratchpad
(`/tmp/claude-1000/-home-romain-geargrafx/*/scratchpad`). Worth recreating and
committing; the repo currently has **no rendering test at all**, only a 65x02 CPU
JSON runner in `tests/`.

- `headless.cpp` — drives `RunToVBlank`, FNV-hashes framebuffer + audio every N
  frames, injects RUN/I, accepts optional syscard/gexpress BIOS paths.
- `fixtime.so` — `LD_PRELOAD` shim pinning `time()`.
- `build_a64.sh` / `build_both.sh` — cross-build static AArch64 + x86 binaries.
- qemu 10.1.3 built with `--enable-plugins` at `/home/romain/qemu/build-user`,
  plus `libinsn.so` compiled from `tests/tcg/plugins/insn.c`.

Content used: 192 HuCards (`~/ClaudeSega/tas_roms_extracted/PCE`), 5 SuperGrafx
(`~/pcetang_hwtest_2026-08-31/rom_staging`), 22 CD CHDs (`/userdata/roms/pcenginecd`)
with BIOS at `/userdata/bios/`.

**Note:** `perf` does not work on this machine (kernel 6.17.0-1032-oem has no
matching `linux-tools`). callgrind was used instead.

## 8. Library facts worth knowing

Measured with an instrumented per-speed pixel counter over all 192 HuCards:

- **96.7%** of pixel time runs at divider 4 (5.36 MHz), **3.3%** at divider 3
  (7.16 MHz), ~0% at divider 2.
- Only **6 of 192** games are predominantly divider 3: Aoi Blink, Image Fight,
  Mr. Heli, Populous, R-Type II, Shanghai. This is why `3f749c6` does not move
  the aggregate despite being worth ~2.6% on those titles.
- The libretro core runs **RGB565**, not RGBA8888 (`libretro.cpp:272`). Anything
  touching `RenderFrameTemplate<…, 4>` is dead code on RetroArch/handheld targets
  and only helps the desktop SDL build. `a7b8c3f` falls in this category.
