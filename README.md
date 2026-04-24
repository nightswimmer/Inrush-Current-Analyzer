# Inrush Current Analyzer

A zero-dependency, single-file browser tool for analyzing **DC capacitor inrush current**, sizing inrush limiters, and sanity-checking NTCs against their datasheet energy-absorption rating.

> Close the switch on a bank of caps — how many amps flow in the first instant, how long does it last, how much energy gets dumped into the limiter, and can the NTC you picked actually take it?

![dark oscilloscope-style UI — phosphor-green current trace and amber voltage trace on a dark grid](docs/screenshot.png)

## Features

- **Live inrush readouts** — peak current, time constant τ, 99 % charge time, and total energy dissipated (`½·C·V²`).
- **Dual-trace transient plot** — current `I(t)` and capacitor voltage `V_C(t)` over `0…5τ`, rendered on a hand-drawn canvas (no chart library).
- **Limiter sizing helper** — target-driven: enter the peak current you're willing to allow, it tells you the required `R`, additional `R` to add, peak power, absorbed energy, and resulting charge time.
- **NTC analysis panel** with two sub-sections:
  - *Steady-State (hot)* — voltage drop, delivered load voltage, continuous power dissipation on the NTC, and `V_drop` as a percentage of `V_dc`.
  - *Energy Absorption vs. Datasheet* — reverse-calculate the NTC's one-shot joule rating from its "max capacitance at reference voltage" spec, compare against your actual `½·C·V²`, and read the utilization % and headroom directly.
- **Traffic-light warnings** for high peak currents (>50 A / >200 A), no-series-resistance condition, significant rail drop (>5 %), continuous NTC power (>2 W), and absorption utilization (≥75 %, ≥100 %).
- **Unit-aware inputs** — capacitance in µF / mF / F, resistance in mΩ / Ω, everything else auto-scaled on output.
- **Oscilloscope aesthetic** — JetBrains Mono, phosphor green on current, amber on voltage, dark panels.

## Quick start

```bash
git clone https://github.com/nightswimmer/Inrush-Current-Analyzer.git
cd Inrush-Current-Analyzer
```

Then either:

- Open [`index.html`](index.html) directly in your browser, **or**
- Serve it with any static server: `python -m http.server` and visit <http://localhost:8000>, **or**
- Deploy it to GitHub Pages — the repo is already a single static file.

No build step, no dependencies, no tooling. One file.

## Inputs

**Circuit parameters (left panel)**

| Field                  | Meaning                                                |
| ---------------------- | ------------------------------------------------------ |
| Source Voltage `V_dc`  | DC rail voltage                                        |
| Capacitance `C`        | Total bank capacitance (µF / mF / F)                   |
| Series Resistance      | External limiter / NTC cold resistance                 |
| Cap ESR                | Equivalent series resistance of the capacitor bank     |
| Source + Wiring R      | PSU output impedance + cable + contact resistance      |
| Max Load Current       | Normal steady-state current through the NTC           |
| NTC Hot Resistance     | Self-heated NTC resistance in normal operation        |

`R_total = R_limit + ESR + R_source` is used for the inrush calculations. `R_hot` is used only for the steady-state block.

**Sizing helper input**

| Field                | Meaning                                         |
| -------------------- | ----------------------------------------------- |
| Target Peak Current  | Max inrush current you're willing to allow      |

**NTC datasheet inputs**

| Field                       | Meaning                                                   |
| --------------------------- | --------------------------------------------------------- |
| Datasheet Ref. Voltage      | Voltage at which the NTC is rated in its datasheet        |
| Datasheet Max Capacitance   | Maximum capacitance the NTC can survive at that voltage   |

## Model

Ideal first-order RC transient:

```
I(t)   = (V / R) · exp(−t / RC)
V_C(t) = V · (1 − exp(−t / RC))
τ      = R · C
E      = ½ · C · V²    (independent of R)
```

Steady-state across the NTC:

```
V_drop = I_load · R_hot
P_hot  = I_load² · R_hot
```

Datasheet reverse-calculation:

```
E_rated = ½ · C_ref · V_ref²
```

This is a **sizing tool**, not a SPICE replacement. It does not model NTC self-heating dynamics, loop inductance, diode drops, or PSU current limiting. Read the results accordingly.

## Typical workflow

1. Plug in your `V_dc`, `C`, ESR, source/wiring R, normal load current, and NTC hot resistance.
2. Look at the **Peak Inrush** readout — if red, your switch and PSU are in trouble.
3. Use the **Limiter Sizing Helper** to find a cold resistance that clamps peak current to something your circuit can survive.
4. Enter your candidate NTC's **datasheet reference voltage** and **max capacitance** into the NTC Analysis panel. Check the **utilization %** — green means you have margin, amber means tight, red means pick a bigger part.
5. Check the **Steady-State (hot)** readouts: if `V_drop` is a big fraction of `V_dc` or `P_hot` is over a couple of watts, consider bypassing the NTC with a contactor or MOSFET after pre-charge.

See the **Engineer's Notes** section at the bottom of the page for guidance on NTC limiters vs. pre-charge resistors.

## License

[MIT](LICENSE) © 2026 nightswimmer
