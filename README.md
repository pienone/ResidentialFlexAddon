# ResidentialFlex — a residential flexibility add-on for BALMOREL

Soft-linked modelling framework that quantifies the **demand-side flexibility of electric vehicles and heat pumps in Danish households**, and its effect on the wider power system, by coupling price-responsive household optimisation (Julia/JuMP) with the [BALMOREL](https://github.com/balmorelcommunity/Balmorel) energy-system model (GAMS).

Developed as part of the MSc thesis *"Investigation of residential electricity consumption profiles with a focus on the energy system flexibility potential"* — Sustainable Energy Engineering, Energy Systems Analysis, Technical University of Denmark (DTU), 2021.

**Authors:** Pietro Nonis, Francesco Gaballo

---

## Research question

How much flexibility can residential electric vehicles and heat pumps provide to the Danish power system, and how does price-responsive ("smart") operation change system prices and dispatch compared with passive consumption, under different adoption pathways up to 2040?

## Method

The framework iterates between a **consumer-side** optimisation and the **system-side** BALMOREL model until prices and flexible demand are consistent:

```
            ┌──────────────────────────────────────────────┐
            │                                              │
            ▼                                              │
 1. Household optimisation (Julia / JuMP)                  │
    cost-minimising load scheduling per                    │
    consumer category, given electricity prices            │
            │                                              │
            ▼                                              │
 2. Aggregation (Python)                                   │
    scale profiles by adoption scenario and                │
    technology shares → DE_RESEVHP time series             │
            │                                              │
            ▼                                              │
 3. BALMOREL run (GAMS) with the RESEVHP add-on            │
    flexible residential demand enters the                 │
    electricity balance of DK1 and DK2                     │
            │                                              │
            ▼                                              │
 4. Price extraction (Python)                              │
    hourly DK1/DK2 prices from BALMOREL results ───────────┘
```

**Step 1 — household optimisation.** One linear program per consumer category, region and year minimises the cost of meeting each household's energy needs at the prevailing prices:

| Technology | Flexibility representation |
|---|---|
| Battery EVs (BEV) | State-of-charge balance, charging only when the car is at home (driving-pattern availability), full battery required at departure |
| Plug-in hybrids (PHEV) | As BEV, with a high-cost fuel fallback |
| Air-to-water heat pumps | Heat demand can be shifted within a 3-hour window (hot-water storage) |
| Air-to-air heat pumps | Heat demand can be shifted within a 1-hour window |
| Summer houses (air-to-air) | As above, with summer-house demand profiles |

**Step 2 — aggregation.** Category profiles are weighted by EV and heat-pump adoption shares per category, region and year, and converted into the `DE_RESEVHP` parameter for BALMOREL.

**Step 3 — BALMOREL add-on.** The `RESEVHP` add-on declares `DE_RESEVHP(YYY,SSS,TTT,RRR)` and adds it to the electricity balance equation via BALMOREL's add-on hooks.

**Step 4 — price feedback.** Electricity prices (`EL_PRICE_YCRST`) for DK1 and DK2 are read from the BALMOREL GDX output and passed back to step 1 for the next iteration.

### Scope

- **Regions:** DK1 and DK2 (Denmark), within BALMOREL's wider European system
- **Years:** 2020, 2025, 2030, 2040
- **Time resolution:** hourly, four representative weeks per year
- **Scenarios:** Expected, Rapid and Slow adoption pathways; flexible-participation rates of 10%, 25%, 40% and 75%
- **Consumer categories:** household types (e.g. houses and apartments) derived from Danish residential consumption data

## Repository structure

```
BALMOREL add-on/
├── ResidEVHPprivateoptim_step1.jl   # Step 1: household LP models (EV, PHEV, heat pumps)
├── ResidEVHPaggr_step2.py           # Step 2: aggregation into DE_RESEVHP
├── ResidEVHPprices_step4.py         # Step 4: price extraction from BALMOREL results
├── ResidEVHP/                       # BALMOREL add-on: parameter declaration, definition, equation term
└── _hooks/                          # BALMOREL hook files with the RESEVHP add-on switched in
Data/
├── Julia/       # Inputs for step 1: demand profiles, driving patterns, price series per iteration
├── Python/      # Adoption shares per scenario (EVs, heat pumps) and EV category coefficients
├── Inc/         # Generated DE_RESEVHP .inc files per scenario, participation rate and iteration
├── Analysis/    # Result plots, private cost reduction, consumption time-series analysis
└── DataRESEVHP.xlsm   # Excel/VBA conversion of the aggregated series to BALMOREL format
```

## How to run

**Requirements**

- [BALMOREL](https://github.com/balmorelcommunity/Balmorel) and GAMS
- Julia with JuMP, CSV, DataFrames and a linear-programming solver (the thesis used Gurobi)
- Python 3 with pandas, numpy, matplotlib and [gdxpds](https://github.com/NREL/gdx-pandas)

**Steps**

1. Copy `BALMOREL add-on/ResidEVHP/` into BALMOREL's `base/addons/` folder and merge the `_hooks/` lines into your BALMOREL hook files. Enable the add-on with `RESEVHP=yes`.
2. Run a base BALMOREL scenario and extract prices with `ResidEVHPprices_step4.py`.
3. Run `ResidEVHPprivateoptim_step1.jl` with the chosen scenario, participation rate and iteration.
4. Aggregate with `ResidEVHPaggr_step2.py` and convert the result to `DE_RESEVHP.inc` (`Data/DataRESEVHP.xlsm`).
5. Run BALMOREL with the new `DE_RESEVHP.inc`, then repeat from step 2 until prices converge.

Scenario, participation rate and iteration are set at the top of each script.

## Results

<!-- Add 2–4 headline results from the thesis, e.g. peak-load reduction, price effects, or private cost savings per household under smart operation, with one figure. -->

[FILL: headline results and a figure]

## Known limitations

This is research code from 2021 and is shared as-is:

- File paths are hard-coded for the original Windows setup and must be adapted.
- The Julia code uses pre-1.0 JuMP syntax (`Model(solver=...)`, `solve`) and needs updating for current JuMP versions.
- Gurobi is used as the solver; an open solver such as HiGHS can be substituted for these linear programs.
- Flexible demand enters BALMOREL as an exogenous parameter, updated iteratively, rather than as endogenous decision variables.

## License

MIT — see [LICENSE](LICENSE).
