# Six Sigma UDFs — GB Grid Frequency

A Power BI portfolio project applying Six Sigma statistical process control (SPC) to a real, actively-regulated industrial process: UK electricity grid frequency, published by the National Energy System Operator (NESO).

## Why grid frequency

This project went through a few data sources before landing here. Tide gauge levels don't really map onto "an industrial process" — nothing is actively holding the tide within spec. UK groundwater levels have too much natural long-term drift to make for a meaningful capability story. Grid frequency is different: NESO is statutorily required to keep it within 1% of 50 Hz (49.5–50.5 Hz) and actively controls it to a much tighter 49.8–50.2 Hz operating target day to day — a genuine, continuously-controlled process with a real, externally-set specification, which is exactly what Cp/Cpk/Pp/Ppk are designed to evaluate.

## Data source

[NESO system frequency data](https://www.neso.energy/data-portal/system-frequency-data) — 1-second resolution GB grid frequency, published as one CSV per calendar month. Pulled via a Power Query function (`fnGetGridFrequencyReadings` in `expressions.tmdl`) that looks up the current month's file dynamically through NESO's CKAN `package_show` API rather than a hardcoded URL, so it keeps working if NESO reshuffles resource IDs behind the scenes. The fact table targets the last *completed* calendar month and trims to a short recent window to stay a manageable import size.

## Model structure

- **Fact_GridFrequency** — `dtm` (timestamp), `f` (frequency, Hz), `Date`
- **Dim_GridThresholds** — a one-row lookup table holding the specification limits (USL/LSL) as data rather than hardcoded constants, so they can be corrected or updated without touching any DAX. Currently set to the statutory 49.5/50.5 Hz range, sourced from NESO directly (see comments in the query for the source link and the note about the tighter 49.8/50.2 operating target as an alternative).
- **Dim_Calendar** — auto-sizes its date range from `Fact_GridFrequency`, rather than being hardcoded to a fixed period.
- **_Measures** — all DAX measures, described below.

## The DAX UDFs (`functions.tmdl`)

All fourteen are written as genuine, model-agnostic user-defined functions — every table/column they touch is passed in as a typed parameter (`TABLE`, `COLUMNREF`, `SCALAR`), never hardcoded to a specific table or column name. That's what makes them portable: the same function library, unchanged, has already been reused across three completely different datasets (tide gauges, groundwater, grid frequency) in this project's history.

- `NormSInv` — inverse standard normal CDF
- `DPMO` — Defects Per Million Opportunities
- `SigmaLevel` — Sigma Level from a DPMO value, with the standard 1.5σ long-term shift
- `ProcessSigmaWithin` — short-term sigma via average moving range / d2
- `Cp` / `Cpk` — short-term process capability (potential / actual)
- `Pp` / `Ppk` — long-term process capability, based on population standard deviation
- `ControlLimits` — center line, UCL, LCL (mean ± 3σ)
- `IsNelsonRule1` through `IsNelsonRule4` — the first four Nelson Rules for control chart pattern detection
- `ChartSummary` — builds a short anomaly-flagging summary string for a given period (used as a chart subtitle, not a full narrative)

## Report page

**"National Grid Frequency Capability"** — four KPI cards (Sigma Level, DPMO, Cpk, Ppk, each with a status label), a control chart of `Fact_GridFrequency[f]` against the calculated control limits with Nelson Rule 1 violations flagged as distinct markers, and a subtitle that names which day(s) an anomaly was flagged on (or states plainly that none were).

## A methodological note worth reading before trusting the numbers

Cp/Cpk and Pp/Ppk can diverge sharply here, and that's a real finding rather than a bug. Cpk is derived from the *short-term* sigma (the average moving range between consecutive readings), and because grid frequency barely moves from one second to the next, that moving range is tiny — which inflates Cpk to an unrealistically large number. Ppk uses the *population* standard deviation across the whole window instead, which correctly captures the real spread including any flagged excursions, and gives a far more honest capability figure. This is a textbook illustration of why within-subgroup variation understates true process variation when consecutive samples are highly autocorrelated, which is exactly the case for 1-second-resolution readings of a slow-moving physical quantity.

## Opening this project

This is a Power BI Project (PBIP), which requires a recent version of Power BI Desktop with DAX user-defined functions enabled (compatibility level 1702+). Open `SixSigma_Frequency.pbip` directly in Power BI Desktop.

## License / attribution

Grid frequency data is published by NESO under their open data licence — see the [data portal page](https://www.neso.energy/data-portal/system-frequency-data) for current terms.
