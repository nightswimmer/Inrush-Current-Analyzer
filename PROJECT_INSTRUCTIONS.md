# Project Instructions — Inrush Current Analyzer

This file is the entry-point context for Claude conversations on this project. Keep it up to date at the end of every session.

---

## 1. What this project is

A browser-based **inrush current analyzer** for DC capacitor charging through a series resistance (the classic RC transient). It answers the practical engineering question: *"If I close the switch on this power rail, how much current flows in the first instant, how long does it last, and how much energy gets dumped into my limiter?"*

Targeted use cases:
- Sizing an NTC inrush limiter (e.g. Ametherm SL-series) from a target peak current.
- Reverse-calculating the energy absorption rating from a datasheet's "max capacitance at reference voltage" spec and checking margin.
- Sizing a pre-charge resistor + bypass contactor/MOSFET for persistent DC loads.
- Sanity-checking whether a MOSFET/relay/PSU can survive switch-on into a bank of capacitors.
- Checking the NTC's steady-state behaviour: voltage drop and power dissipation at normal load current.

## 2. Tech stack

- **Single static file**: [index.html](index.html) contains HTML, CSS, and vanilla JavaScript. No build step, no bundler, no dependencies.
- **Runtime**: any modern browser. Open `index.html` directly (file://) or serve it from any static host / GitHub Pages.
- **Fonts**: JetBrains Mono via Google Fonts (only external request).
- **Chart**: hand-rolled on a single `<canvas>` — no Chart.js / D3 / etc.

Keep it this way unless there is a strong reason to add tooling. Part of the appeal is "one file, drop it anywhere, it works."

## 3. Current features

### Inputs (left panel)
- **Source Voltage** `V_dc` — volts.
- **Capacitance** `C` — µF / mF / F selectable.
- **Series Resistance** `R_limit` — the external limiter / NTC cold resistance (mΩ / Ω).
- **Cap ESR** — equivalent series resistance of the capacitor bank, in mΩ.
- **Source + Wiring R** — PSU output impedance + cable + contact resistance, in mΩ.
- **Max Load Current** `I_load` — normal steady-state current through the NTC, in A.
- **NTC Hot Resistance** `R_hot` — self-heated resistance once the NTC is up to temperature, in mΩ / Ω.

`R_total = R_limit + ESR + R_source` is used for the inrush calculations. `R_hot` is used independently for the steady-state analysis.

### Main readouts
- **Peak Inrush** `I_peak = V / R_total` — at `t = 0`. Also shows peak power. Turns red if > 100 A or if R = 0.
- **Time Constant** `τ = R · C` — auto-scaled units.
- **Charge Time (99%)** `≈ 4.6 τ` (also reports 99.9% at `ln(1000)·τ`).
- **Energy Dissipated** `½·C·V²` — the total energy dumped in the series resistance during charging (independent of R).

### Transient chart
- `I(t) = I_peak · exp(-t/τ)` — phosphor-green trace.
- `V_C(t) = V · (1 − exp(-t/τ))` — amber trace.
- Time axis spans `0 … 5τ` with dashed markers at each τ.
- Dual y-axis: current left (green), voltage right (amber).

### Inrush warning banner
- `R = 0` or infinite peak → hard warning about parasitic-only limiting.
- `I_peak > 200 A` → switch/MOSFET survivability warning.
- `I_peak > 50 A` → PSU OCP / cable rating warning.

### Limiter Sizing Helper
Target-driven: enter the max peak current you want to allow, it computes:
- Required `R_total = V / I_target`
- Additional R needed (vs. current `R_total`)
- Peak power in the limiter (`V · I_target`)
- Total energy absorbed (`½·C·V²`)
- Resulting new `τ` and 99% charge time

### NTC Analysis panel
Two sub-sections sharing one warning banner.

**Steady-State (hot)** — uses `I_load` and `R_hot`:
- `V_drop = I_load · R_hot` — voltage across the NTC at normal load.
- `V_load = V_dc − V_drop` — what the load actually sees (turns red if negative).
- `P_hot = I_load² · R_hot` — steady-state power the NTC has to dissipate continuously.
- `V_drop as % of V_dc` — rail-loss indicator.

Hot-state warnings:
- `V_load < 0` → inputs contradict (I·R > V_dc).
- `V_drop > 5% of V_dc` → suggests using contactor bypass after pre-charge.
- `P_hot > 2 W` → verify NTC continuous dissipation rating.

**Energy Absorption vs. Datasheet** — reverse-calc from the datasheet's "max capacitance at reference voltage" spec:
- Inputs: **Datasheet Ref. Voltage** `V_ref` and **Datasheet Max Capacitance** `C_ref`.
- `E_rated = ½ · C_ref · V_ref²` — implied one-shot joule rating.
- `E_real = ½ · C · V_dc²` — actual pulse energy in this circuit.
- `Utilization = E_real / E_rated` — color-coded: green < 75 %, amber 75–100 %, red ≥ 100 %.
- `Headroom = E_rated − E_real` — positive (green) or negative (red).

Absorption warnings:
- Utilization ≥ 100 % → exceeds rating; pick larger NTC, or split in series/parallel, or use pre-charge + bypass.
- Utilization 75–99 % → thin margin; consider next size up for derating.

### Engineer's Notes
Static prose block explaining peak-current formula, the invariance of ½CV², NTC limiters, and the pre-charge resistor + bypass pattern.

## 4. Physics / model

Ideal lumped RC for the inrush:
```
I(t)   = (V / R) · exp(-t / RC)
V_C(t) = V · (1 − exp(-t / RC))
E_R    = ½ · C · V²   (total energy dissipated in R, independent of R)
```
Steady-state across the NTC once fully charged:
```
V_drop = I_load · R_hot
P_hot  = I_load² · R_hot
```
Datasheet reverse-calc (assumes the datasheet rating is the ½CV² energy at the stated V_ref, C_ref):
```
E_rated = ½ · C_ref · V_ref²
```

No inductance, no NTC self-heating transient curve, no diode drops, no voltage-dependent ESR, no PSU current-limit clamp. This is a first-order sizing tool, not a SPICE replacement.

## 5. Styling conventions

"Oscilloscope/lab instrument" aesthetic:
- Dark background (`--bg: #0b0d0e`), panels a shade lighter.
- Phosphor green `#3ee07a` for current / primary.
- Amber `#ffb347` for voltage and mid-severity highlights.
- Red `#ff5a4a` for warnings.
- JetBrains Mono everywhere, uppercase small-caps labels with wide letter-spacing.
- Tabular numerals on all readouts.

All colors are CSS custom properties in `:root` — extend by adding there, not by hard-coding. In-panel section headings use the `.subhead` / `.subhead-note` classes introduced for the NTC Analysis panel; reuse them rather than inventing new heading styles.

## 6. Repo layout

```
.
├── index.html              # entire application
├── README.md               # GitHub landing page
├── PROJECT_INSTRUCTIONS.md # this file
├── LICENSE                 # MIT
├── CLAUDE.md               # Claude workflow rules
└── .claude/                # Claude Code local settings
```

## 7. Working agreements (from [CLAUDE.md](CLAUDE.md))

- Read this file at the start of every conversation.
- When something is unclear, ask — don't guess.
- For questions phrased as "is there a bug in X" or "could we add Y", answer the question; **do not** change code until explicitly told to.
- Before big structural changes, remind the user to push to GitHub first.
- At end of session / on commit: update this file and `README.md`, then propose a commit message. The user does the commit.

## 8. Current status

- **Stage**: working, feature-complete for the scope of "first-order RC sizing + NTC datasheet check". Single file, everything in place.
- **Known gaps / ideas** (not committed to — discuss before implementing):
  - NTC self-heating model (R(T) curve) for a more realistic peak — right now `R_hot` and `R_limit (cold)` are independent scalar inputs.
  - Inductance in the loop (turns RC into RLC — oscillatory / critically damped regimes).
  - PSU current-limit clamp (flat-top then exponential).
  - Export chart as PNG / transient data as CSV.
  - Shareable URL (params encoded in query string).
  - Parametric sweep (e.g. "peak current vs. R_limit" for a given V, C).

## 9. Session log

- **2026-04-22**: Initial project setup. Repo initialized from a VS Code + Claude template. `index.html` with full analyzer UI, calculations, canvas chart, and sizing helper already in place. Filled in `PROJECT_INSTRUCTIONS.md` and `README.md` (previously template stubs).
- **2026-04-23**: Added a **Steady-State Analysis** panel with two new inputs (`I_load`, `R_hot`) and readouts for V_drop, V_load, P_hot, and V_drop as % of V_dc. Warning banner covers negative V_load, >5 % rail loss, and >2 W continuous NTC dissipation.
- **2026-04-24**: Moved `I_load` and `R_hot` inputs to the left parameters panel. Renamed the panel to **NTC Analysis** and split it into two sub-sections: *Steady-State (hot)* and *Energy Absorption vs. Datasheet*. The absorption block takes a datasheet reference voltage and max capacitance, computes `E_rated = ½·C_ref·V_ref²`, and compares it against the circuit's actual `½·C·V²` with utilization % (traffic-light coloured) and headroom in joules. Extended warnings to cover ≥ 75 % and ≥ 100 % utilization. Added `.subhead` / `.subhead-note` CSS for in-panel section headings.
