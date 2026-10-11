# EMTP / Simscape Electrical Translation Tool

**Msc. Semester Project**
- **Project name** : Software tool to translate power system dynamic models across simulation platforms 
- **Author** : Pernelle Paget
- **Supervisors** : Cesar García Veloso & Prof. Mario, Paolone
**Distributed Electrical Systems Laboratory (DESL) - EPFL**

A tool to migrate electrical-circuit models from **EMTPWorks 4.6.0** to
**Simscape Electrical / Specialized Power Systems (SPS)** (MATLAB R2024b), and to validate
that both simulators produce equivalent waveforms.

System operators maintain models of the same network across several platforms (EMTP,
Simscape Electrical, PSS/E, PSCAD); porting a model by hand is slow and error-prone. This
tool automates the **EMTP → Simscape** path through a single flat **JSON pivot** and a
reproducible validation step. *(PSS/E is part of the wider project brief but is out of
scope for this repository: the scope here is EMTP → Simscape only.)*

This file is the single source of documentation for both **users** (how to install and
run the tool) and **developers** (how it works internally, the conventions, and how to
extend it).

## Pipeline

```
 EMTP circuit  ──►  ConvertToJSON.dwj  ──►  circ_emtp_*.json  ──►  Python builder  ──►  circ_sim_*.slx
 (.ecf/.net,        (run in EMTPWorks)       (JSON pivot)          (matlab.engine)
  results .mda/.m)                                                       │
                                                                         ▼
                                              Validation  ◄────  EMTP results + Simscape results
                                           (RMSE, relative error, ρ, interactive plots)
```

1. **JSON extraction** — `exporter/ConvertToJSON.dwj` runs inside EMTPWorks and writes a
   flat JSON description of the circuit (topology, parameters, scope flags, solver
   settings) to `data/json_files/`.
2. **Builder** — `scripts/translate.py` reads the JSON and drives MATLAB through the
   MATLAB Engine API to assemble the Simscape model, saving a `.slx`.
3. **Validation** — `scripts/validate.py` compares the EMTP `.mda` results against the
   Simscape simulation CSV and reports Pearson ρ / RMSE / relative error with an
   interactive Plotly plot.

## Installation

### Prerequisites

- **MATLAB R2024b** with Simulink and Simscape Electrical (Specialized Power Systems).
  Required only on the machine that *translates* circuits and *runs Simscape simulations*.
- **Python ≥ 3.12**, managed with [`uv`](https://docs.astral.sh/uv/) (creates a venv at
  `.venv/`).
- **EMTPWorks 4.6.0** — only to run the exporter and to produce the reference `.mda`/`.m`
  results; not needed for the Python steps.

The work splits naturally in two: **translation + Simscape extraction** need MATLAB (done
on the Windows VM in this project); **validation** only reads CSVs and runs anywhere
(typically the Mac).

### Installing uv

`uv` is the Python package/venv manager used by this project; it reads `pyproject.toml`
and creates the `.venv/` for you. Install it once, then the `uv sync` commands below take
care of Python and all dependencies.

- **macOS / Linux**:
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```
- **Windows (PowerShell)**:
  ```powershell
  powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```
- **Alternatives**: `pip install uv`, `pipx install uv`, or `brew install uv` (macOS).

Restart the terminal afterwards (so `uv` is on the `PATH`) and check with `uv --version`.
See the [official install guide](https://docs.astral.sh/uv/getting-started/installation/)
for details. `uv` will download a suitable Python (≥ 3.12) automatically if one is not
already installed.

### On the Windows VM (with MATLAB) — full pipeline

```bat
git clone <repo-url> sp_translation_tool
cd sp_translation_tool

:: install Python deps + the MATLAB Engine for Python
uv sync --extra matlab

:: activate the venv
.venv\Scripts\activate
```

`matlabengine==24.2.*` matches MATLAB **R2024b** (release 24.2). If you use a different
MATLAB release, adjust the version in `pyproject.toml` accordingly (the engine version
must match the installed MATLAB).

### On the Mac (validation only, no MATLAB)

```bash
git clone <repo-url> sp_translation_tool
cd sp_translation_tool

# base deps only (no MATLAB Engine)
uv sync

# activate the venv
source .venv/bin/activate
```

You can import and run everything in `src/validation/` and the parameter/units helpers
offline. The builder can be exercised with a stub MATLAB engine, but a real translation or
Simscape extraction requires the VM.

### Paths configuration

All data paths live in [`sptool.toml`](sptool.toml). `base = ""` (the default) auto-detects
the repository root from the code location, so the tool works as-is on any machine. Set
`base` only if you want the data directories **outside** the repo.

## Usage

```bash
# (Windows VM, MATLAB) Translate one circuit: JSON -> .slx
python scripts/translate.py circ_emtp_Vsine_R

# (Windows VM, MATLAB) Run the Simscape simulation -> results CSV
python scripts/extract_simscape.py circ_sim_Vsine_R

# (anywhere) Extract the EMTP reference results .mda/.m -> CSV
python scripts/extract_emtp.py circ_emtp_Vsine_R

# (anywhere) Compare EMTP vs Simscape and open the interactive plot
python scripts/validate.py circ_emtp_Vsine_R          # --mpl static plot, --no-plot to skip
```

- Circuit names are accepted with or without the `.json`/`.slx`/`.csv` suffix, and in
  either the `circ_emtp_*` or `circ_sim_*` form.
- `validate.py` auto-generates the EMTP CSV from `.mda`/`.m` when missing (no MATLAB). With
  `--extract` it will also run the Simscape simulation if its CSV is missing.
- `translate_all.py --only <names>` translates several circuits in one MATLAB session;
  the whitelist/exclude lists live in `sptool.toml`.
- `python -m compileall -q src scripts` is a quick syntax check.

The validation plot is an interactive Plotly HTML: a dropdown isolates one signal, the
legend toggles individual traces, the |error| curves are hidden by default (click to show)
on a secondary log axis, and units / RMSE / ρ appear in the labels. A PNG is also exported.

## Supported components

- **Sources** — AC voltage (1φ / 3φ), AC current, DC voltage, DC current
- **RLC branches** — R, L, C and combinations; 1φ and 3φ (with bus auto-promotion)
- **PI lines** — 1φ and 3φ PI sections (matrix / sequence parameters)
- **Switches** — ideal switches
- **Transformers** — linear 1φ (Tr0); 3φ two-winding and three-winding (with saturation)
- **Nonlinear** — nonlinear resistor (Rnonl), nonlinear inductor (Lnonl)
- **Loads** — PQ load (load-flow and RLC variants) → constant-Z Three-Phase RLC Load
- **Probes** — explicit voltage / current probes, 1φ (3φ probes not yet built; see below)

## Repository tree

```
sp_translation_tool/
├── README.md                    project overview + engineering reference (this file)
├── pyproject.toml               Python project + dependencies (uv)
├── sptool.toml                  paths + translation config (whitelist / exclude lists)
├── exporter/
│   └── ConvertToJSON.dwj        EMTP-side exporter, run inside EMTPWorks (frozen, UTF-16)
├── src/
│   ├── config.py                EMTP -> Simscape block-path mapping + resolved paths
│   ├── translator/              JSON pivot -> Simscape .slx
│   │   ├── pipeline.py          orchestrates load -> init -> build -> save
│   │   ├── builder.py           dispatch loop over the circuit components
│   │   ├── model_init.py        model creation, powergui (Discrete), solver
│   │   ├── context.py           BuildContext: engine, nodes, wiring helpers
│   │   ├── json_loader.py       read the JSON pivot (utf-8 / utf-8-sig / utf-16)
│   │   ├── param_reader.py      flat -> nested {value, unit} normalization + scope folding
│   │   ├── utils.py             unit conversion, coordinates
│   │   ├── measurements.py      VM / CM / 3-phase V-I blocks -> To Workspace
│   │   ├── wiring.py            island detection + single consolidated Ground
│   │   └── components/          one builder per component family
│   │       ├── base.py          Comp: shared read-only view over a JSON component
│   │       ├── sources.py       AC / DC voltage and current sources
│   │       ├── rlc.py           RLC branches (1-phase / 3-phase)
│   │       ├── pi_line.py       PI section lines
│   │       ├── switch.py        ideal switches
│   │       ├── transformer.py   linear 1-phase transformer (Tr0)
│   │       ├── transformer3p.py 3-phase transformers (2- and 3-winding)
│   │       ├── loads.py         PQ loads
│   │       ├── nonlinear.py     nonlinear R / L
│   │       └── probe.py         explicit voltage / current probes
│   └── validation/              EMTP vs Simscape comparison
│       ├── emtp_reader.py       read EMTP .mda (Fortran binary) + .m metadata
│       ├── simscape_reader.py   run the sim, auto-discover To Workspace -> CSV
│       ├── compare.py           time alignment + column matching + ρ
│       ├── column_matching.py   pair EMTP <-> Simscape signal columns
│       ├── errors.py            RMSE, mean/max |Δ|, relative error, notes
│       ├── naming.py            circuit-name / result-path resolution
│       └── plot.py              interactive Plotly + static matplotlib plots
├── scripts/                     command-line entry points
│   ├── translate.py             JSON -> .slx (one circuit)
│   ├── translate_all.py         translate several circuits in one MATLAB session
│   ├── extract_emtp.py          EMTP .mda/.m -> res_*.csv
│   ├── extract_simscape.py      run the Simscape sim -> res_*.csv
│   ├── validate.py              compare EMTP vs Simscape + plot
│   └── plot_sim.m               plot a model's signals in MATLAB
├── data/
│   ├── json_files/              input JSON pivots (versioned)
│   ├── emtp_circuits/           .ecf/.net + _pj results (gitignored)
│   ├── emtp_results/            extracted EMTP CSVs (gitignored)
│   ├── simscape_circuits/       generated .slx (gitignored)
│   └── simscape_results/        Simscape CSVs (gitignored)
├── docs/
│   ├── probes.md                probe detection + N:1 mapping rules
│   ├── EMTP_Simscape_translation_tool.xlsx   component mapping spreadsheet
│   ├── reference/               EMTP info scripts + PDFs
│   └── test-reports/            validation test reports
```

---

# Engineering reference

The rest of this document is the technical reference for developers extending the tool.

## JSON pivot format (flat)

The exporter writes parameters **flat**, not nested:

- per-phase scalars `X_A`/`X_B`/`X_C` + a shared `X_units` (e.g. `Voltage_A` + `Voltage_units`,
  `R_A` + `R_units`);
- scope flags `scope_<kind>_<phase>` (`scope_voltage_A`, `scope_current_B`, …);
- `phases`: `1` or `3`. There is **no `Vsine3p` type** — a 3-phase source is a `Vsine` with
  `phases: 3`.

`param_reader.normalize_parameters()` converts this to the nested `{value, unit}` form the
builders consume; `scope_dict()` folds the per-phase flags into one `{voltage, current,
power}` dict (OR-ed over phases). PI-line `R/L/C` arrive as dicts (`mode`/`data`/`zero`/
`pos`); the normalizer only injects their `unit`. The `components/base.Comp` dataclass wraps
all this so each builder gets a clean read-only view.

Minimal example (a source and a resistor):

```json
{"id":"AC1","type":"Vsine","parameters":{"phases":1,
  "Voltage_units":"kVRMSLL","Voltage_A":230,"Frequency_A":60,"Phase_A":0,
  "scope_voltage_A":true,"scope_current_A":true},
  "connections":{"pin_0":"s1","pin_1":"GND"}}
{"id":"R1","type":"RLC","parameters":{"phases":1,"R_units":"Ohm","R_A":1,
  "scope_voltage_A":true,"scope_current_A":true},
  "connections":{"pin_0":"s1","pin_1":"GND"}}
```

## The exporter `ConvertToJSON.dwj` (frozen)

`exporter/ConvertToJSON.dwj` is a JavaScript script run from EMTPWorks' *User scripts* menu.
It reads each device through the simulator's own scripting API and dumps the active circuit
to a JSON file. It is **frozen — do not modify**, must stay **UTF-16 LE BOM** (never
re-encode as UTF-8), and never mutates the circuit on disk.

## Translator modules (leaves first)

- `matlab_engine.py` — `start_matlab_engine()`; the **only** import of `matlab.engine`.
- `json_loader.py` — `load_json()`; tries utf-8, utf-8-sig, utf-16.
- `param_reader.py` — flat → nested normalization + scope folding.
- `utils.py` — unit conversion: `convert_to_si` (SI prefixes), `to_inductance` (Ohm =
  reactance → `L = X/2πf`, else H), `to_capacitance` (Siemens = susceptance → `C = B/2πf`,
  else F); `get_circuit_frequency`; coordinate transform.
- `context.py` — `BuildContext`: engine, model name, `el_nodes` (node → ports),
  `galvanic_links`, 3-phase bus set, helpers (`set_pos`, `connect`, `safe_set_param`,
  `set_param_first_match`).
- `components/*.py` — one builder per family; `components/__init__.dispatch(ctx, c)` routes
  by type/phases.
- `measurements.py` — explicit Voltage/Current/Three-Phase V-I Measurement blocks → `To
  Workspace`.
- `wiring.py` — union-find island detection + single consolidated `Ground`.
- `model_init.py` — model name `circ_emtp_*` → `circ_sim_*`; `powergui` **Discrete**;
  `ode23tb`; step/stop time from `simulation_settings`.
- `pipeline.py` — `translate_circuit()` orchestrates load → init → build → save.

## Measurement strategy — explicit blocks (not the Multimeter)

The SPS **Multimeter cannot be driven reliably** from the MATLAB Engine: its
signal-selection mask params (`L`, `yselected`) are GUI-oriented and a programmatic
selection does not persist (a diagnostic returned `L='3'`, selection lost). So **every
recorded signal is an explicit measurement block** with a real Simulink output → `To
Workspace`, one named variable per signal (`V_<comp>`, `I_<comp>`; 3-phase = 3-column
A/B/C).

| Scope | Block | Wiring |
|---|---|---|
| 1φ voltage | `Voltage Measurement` (VM) | **parallel** across the two terminals (LConn1 +, LConn2 −) |
| 1φ current | `Current Measurement` (CM) | **series**, spliced into a pin |
| 3φ V and/or I | `Three-Phase V-I Measurement` | **series** on the bus (LConn1..3 in, RConn1..3 out, 2 signal outports: V then I) |

**Current sign — two EMTP conventions** (validated on `circ_emtp_Vsine_R`):
- branches/loads: EMTP branch current enters pin0 → CM `+` on the external-node side
  (`entering=True`);
- sources: EMTP `ivs` leaves the `+` terminal → CM `+` on the component side
  (`entering=False`, `source=True`). The two are 180° apart by KCL.

Series CM/VI splices create an intermediate node; `ctx.galvanic_links` tells the ground
union-find they are the same net (an ideal ammeter is a short), so no stray ground appears.

## Component mapping EMTP → SPS (methods & approximations)

`EMTP_SIMSCAPE_MAPPING` in `src/config.py` holds the SPS library paths. Block paths are
version-specific — confirm an unknown one with `get_param(gcb,'ReferenceBlock')` (or
`...'DialogParameters'` for param names).

### Sources
- **`Vsine` 1φ → AC Voltage Source.** SPS wants **peak** amplitude: `kVRMSLL → V·√2/√3`,
  `kVRMS → V·√2`, else passthrough. Phase **+90°** (EMTP cosine vs SPS sine); **+180°** more
  if `pin_0` is grounded (inverted source → pins swapped). `pin_0`(+)→`RConn1`,
  `pin_1`(−)→`LConn1`.
- **`Vsine` 3φ (`phases:3`) → Three-Phase Source.** SPS wants **RMS line-to-line**:
  `kVRMSLL → V`, `kVRMS → V·√3`, else `V·√2`. `PhaseAngle = Phase_A + 90°`;
  `InternalConnection = Yg`. **`NonIdealSource = on` with `R = 1e-3 Ω, L = 0`** — a fully
  ideal source cannot sit across the PI line shunt capacitor (SPS error); the tiny series R
  breaks that topology with a negligible drop (~1e-3 %).
- **`Isine` 1φ → AC Current Source**; **`Vdc`/`Idc` → DC Voltage/Current Source.** Same
  phase (+90°) and polarity rules.

### RLC branch
- **1φ → Series RLC Branch**; **3φ → Three-Phase Series RLC Branch.** A 1φ branch whose
  node touches a 3-phase bus is **auto-promoted** to 3φ (a pre-pass marks 3φ buses).
- R via `convert_to_si`; **L via `to_inductance`** (Ohm ⇒ reactance ⇒ `L=X/2πf`, else H);
  **C via `to_capacitance`** (Siemens ⇒ susceptance ⇒ `C=B/2πf`, else F). `BranchType` is
  derived from which values are non-zero.

### PI section line
- **1φ → Pi Section Line**; **3φ → Three-Phase PI Section Line** (params
  `Resistances`/`Inductances`/`Capacitances`, each a sequence vector `[positive zero]`).
- **Matrix → sequence**: from `Z_self = data[0][0]` and `Z_mutual = data[0][1]`,
  `Z1 = Z_self − Z_mutual`, `Z0 = Z_self + 2·Z_mutual`. A diagonal (decoupled) line →
  `Z1 = Z0` (exact). An asymmetric matrix can't be reproduced by a sequence model (warns;
  phases averaged → **approximation**).
- **L/C floor for propagation speed** (`_floor_speed`): SPS forbids a zero L or C and warns
  past 300000 km/s. We raise L or C minimally so `v = 1/√(L·C) ≤ 2e5 km/s` (a series-RL line
  thus gets a negligible shunt C ~25 nF).
- **Initial conditions** are not settable on the basic block → left 0 (small `t=0`
  transient on phases with non-zero CI: **approximation**).

### Switch
- **→ Ideal Switch.** Initial state from `tclose < 0` (closed). `Ron=1e-4`, `Rs=1e5`,
  `Cs=inf`. A `Step` block drives the gate.

### Transformer (1φ, Tr0)
- **→ Linear Transformer.** `UNITS=SI`, `NominalPower=[1000e3 60]`, `winding1=[V1 0 0]`,
  `winding2=[V2 0 0]`, **`RmLm=[1e6 1e6]`** (≈ideal; cannot be `inf` in series with an
  inductance). Explicit VM + CM on both windings.

### Three-phase transformer (Tr3p, 2- or 3-winding)
- **→ Three-Phase Transformer (Two/Three Windings).** `UNITS=pu`; `NominalPower=[Pn fn]`
  (`Pn=SW1·1e6` VA). Per winding `Winding<i>=[V_phph(Vrms) R(pu) L(pu)]` (voltage stays in
  **Volts even in pu mode**). Connection `Yg/Yn/Y` or Delta `D11`/`D1`.
- **Saturation always applied when a `magnetization_branch` curve is present** (the
  `Exclude_magnetization_branch` flag is ignored — the exporter reads it unreliably).
  `Saturation = [i φ]` prepended with `[0 0]`. Ports: w1=LConn1..3, w2=RConn1..3,
  w3=RConn4..6. Windings are isolated islands.

### PQ load
- **→ Three-Phase Parallel RLC Load** (or Series if `useRLmodel`). `LoadType = "constant Z"`
  — EMTP models a PQ load as a frozen RLC impedance; the SPS constant-Z load is the faithful
  equivalent (the Dynamic Load is **not**). `Q>0 → InductivePower`, `Q<0 → CapacitivePower`.
  Connection `Yg → Y (grounded)`, `Y → Y (floating)`, `D → Delta`. `P=Q=0 ⇒ not placed`.

### Nonlinear R / L
- **`Rnonl` → Nonlinear Resistor** (`VI = [V I; …]`); **`Lnonl` → Nonlinear Inductor**
  (`IFL = [I φ(Wb); …]`). Both 1-phase two-terminal; scopes via explicit VM/CM.

### Probes
- **voltage → Voltage Measurement**, **current → Current Measurement**, output → `To
  Workspace`. 1-pin voltage probes are deprecated (default `pin_1=GND` + warning). 3φ probes
  (`Vprobe3`, `vMeter3p`) are **not yet built** (see `docs/probes.md`).

## Grounding (`wiring.py`)

A **single consolidated `Ground`** references the whole model: all `GND`-token ports plus
one chosen node per electrically-isolated island, tied to one common ground node
(single-point ⇒ no loop ⇒ measured signals unchanged). Islands are found by union-find with
two key rules: **3-phase bus nodes are expanded** to `<bus>_A/B/C`; **Yg 3-phase sources and
Yg PQ loads ground their own neutral internally**. An all-Yg network (e.g. the load-flow
circuit) builds with **0 external Ground**.

## Validation (`src/validation/`)

- `emtp_reader.py` — `read_mda_binary()` (Fortran unformatted records) + `.m` metadata →
  typed DataFrame (`AC1_v[V]`, `AC1a_i[A]`, …).
- `simscape_reader.py` — runs `sim('<model>')`, auto-discovers every `To Workspace` block,
  expands multi-column signals to `_Phase_A/B/C`, writes `res_<model>.csv`.
- `compare.py` — **time alignment**: EMTP writes one extra off-grid sample at `t=Dt/2`; we
  drop every EMTP sample off the Simscape step grid, then intersect grid indices so both
  sides carry identical time points (strict point-to-point, no interpolation).
- `column_matching.py` — pairs EMTP `<comp>_<kind>[unit]` (and 3-phase `<comp><a|b|c>`) with
  Simscape `V_<comp>`/`I_<comp>` + `_Phase_A/B/C`, scored on component + type + **phase**.
  Power signals (`unit=W`) are excluded (power is intentionally not translated).
- `errors.py` — Pearson ρ, RMSE, mean/max |Δ|, relative error per matched signal. Formulas:
  `Δᵢ=|eᵢ−sᵢ|`, `RMSE=√(mean Δᵢ²)`, `relative error = max|Δ| / max(max|e|, max|s|)`,
  `ρ = cov(e,s)/(σ_e σ_s)` (NaN when a signal is constant). ρ is invariant to scale/offset
  so it is reported together with the amplitude-sensitive errors.
- `plot.py` — `plot_plotly` (default, interactive HTML + PNG) and `plot_simulation_results`
  (static matplotlib, `validate.py --mpl`).

## Conventions (easy to get wrong)

- Phase: EMTP cosine vs SPS sine → +90°; +180° more for an inverted (grounded-`pin_0`)
  source.
- Vsine 1φ peak vs 3φ RMS-LL scaling (see Sources above).
- Source polarity `pin_0`(+)→`RConn1`, `pin_1`(−)→`LConn1`; current sign per the two
  conventions (loads enter pin0, sources leave +).
- 3-phase conserving ports are **`LConn1..3` / `RConn1..3`**, never `Block/A` (raises
  *Invalid Simulink object name*). The V-I block signal outports are ports `1` (V) and `2` (I).
- `to_capacitance`/`to_inductance` for Siemens/Ohm-stored line data.

## Validation status

- **1-phase**: `circ_emtp_Vsine_R` — all 4 signals ρ = 1.000000 (validated).
- **3-phase PI + load-flow**: `circ_emtp_Vsine3p_pi_R` and `circ_emtp_loadflow` — ρ ≈
  0.9999–1.0, relative error < 1.3 % on all matched signals (validated).
- **Per-component vs in-circuit**: the 3φ transformer and nonlinear R/L map and
  parameterize correctly in isolation, but two full circuits
  (`circ_emtp_TR3p_YDD_PQloadRLC`, `circ_emtp_Vsine3p_Rnonl_PQloadLF`) do not yet fully agree
  with EMTP in-circuit — open work.
- **Excluded**: `circ_emtp_TR0_V1_V2` (invalid source circuit — shorted secondary; in
  `sptool.toml` `exclude`).

## Adding a new component type

1. Add the SPS block path to `EMTP_SIMSCAPE_MAPPING` in `src/config.py` (confirm with
   `get_param(gcb,'ReferenceBlock')`).
2. Add `components/<family>.py` with `build_<family>(ctx, c)`: read params via the
   normalized `Comp` view; `add_block` → `set_pos` → `set_param` (convert units) →
   `ctx.connect` each pin → register measurements (`measurements.py`).
3. Register the route in `components/__init__.dispatch` (specific patterns before generic).
4. Drop a JSON in `data/json_files/` and run `scripts/translate.py <circuit>`.

## Known limitations & future work

- **PQ load** uses the **nominal** voltage for the frozen Z; should use the load-flow
  voltage `V_LF` (the only `TODO` left in the code, `components/loads.py`).
- **PI line**: general (non-diagonal) sequence matrices; per-phase initial conditions;
  inverse-L matrix form; the 1φ `format_pi_param` does not yet apply the Ohm/Siemens
  semantics.
- **In-circuit mismatch** (components OK in isolation): `circ_emtp_TR3p_YDD_PQloadRLC` and
  `circ_emtp_Vsine3p_Rnonl_PQloadLF` do not yet fully agree with EMTP.
- **Meshed multi-source load-flow networks**: steady-state initialization is out of scope
  (Discrete `powergui`); both runs start from zero.
- **Not yet mapped**: 3φ probes (`Vprobe3`/`vMeter3p`). Power signals are intentionally not
  translated.
- **Measurement layer**: revisit the strategy and reassess the SPS Multimeter now that its
  limitation is characterised, to reduce the block count on large circuits.
- **New validation protocols** worth adding: transient/event tests (switch, fault),
  time-step and parameter sweeps, automated per-component unit tests, pass/fail thresholds
  for a regression suite, RMS / phase / spectral comparisons, and scaling tests on a
  standard network (e.g. IEEE 9/14 bus).

## Developer notes / housekeeping

Suggested clean-ups for the next maintainer (not yet done):

- `requirements.txt` is stale and contradicts `pyproject.toml` (the real dependency source
  via `uv`). Remove it or regenerate from `uv export`.
- `simscape_reader.py` still carries the abandoned Multimeter "sidecar" machinery
  (`_load_sidecar`, `_expand_internal_meas`, the `internal_meas` branch). The tool no longer
  writes a `.signals.json` sidecar — this code is effectively dead and can be removed.
- `column_matching.py` keeps a backward-compatible alias `match_columns_intelligent` that is
  no longer referenced.
- A stray Excel lock file `docs/~$EMTP_Simscape_translation_tool.xlsx` should be removed and
  `~$*` added to `.gitignore`.
- `scripts/check_source_polarity.m` is an untracked one-off diagnostic — document it or move
  it under `scripts/diagnostics/`.
- Minor: `build_vsine3p` uses magic constants (`1.73205`, `1.22474`) where `build_source_1p`
  uses `math.sqrt(2)/math.sqrt(3)`; unify on a shared `rms_to_peak` helper.
- No test suite yet: a smoke test (build with a stub engine + a reference-CSV comparison)
  would protect future refactors. Offline you can import `param_reader`, `utils`, and
  `src/validation/*`.

See [docs/probes.md](docs/probes.md) for the probe detection and N:1 mapping rules, and
[docs/EMTP_Simscape_translation_tool.xlsx](docs/EMTP_Simscape_translation_tool.xlsx) for the
full per-field component mapping.
