# Six Sigma UDFs for Power BI

A reusable library of Six Sigma statistical process control (SPC) functions for Power BI, written as genuine DAX user-defined functions (UDFs) rather than one-off measures. Every function is fully model-agnostic — tables and columns are passed in as typed parameters (`TABLE`, `COLUMNREF`, `SCALAR`), never hardcoded — so the same library drops into any dataset with a value column and a time column. This repo includes a full worked example (UK grid frequency) to prove that claim, but the UDFs themselves are the point.

## The library (`functions.tmdl`)

Fourteen functions covering the standard Six Sigma capability toolkit:

- `NormSInv` — inverse standard normal CDF (underpins `SigmaLevel`)
- `DPMO` — Defects Per Million Opportunities
- `SigmaLevel` — Sigma Level from a DPMO value, with the standard 1.5σ long-term shift
- `ProcessSigmaWithin` — short-term sigma via average moving range / d2
- `Cp` / `Cpk` — short-term process capability (potential / actual)
- `Pp` / `Ppk` — long-term process capability, based on population standard deviation
- `ControlLimits` — center line, UCL, LCL (mean ± 3σ)
- `IsNelsonRule1` through `IsNelsonRule4` — the first four Nelson Rules for control chart pattern detection
- `ChartSummary` — builds a short anomaly-flagging summary string for a given period, for use as a chart subtitle

None of these reference a specific table or column by name. That's what's actually being demonstrated here: the same unchanged function library has now been proven to work, without modification, across three completely different real-world datasets over the life of this project — UK tide gauges, UK groundwater levels, and (in this repo) GB grid frequency. Swapping datasets is a matter of writing a handful of measures that call the functions with different table/column arguments; the functions themselves never change.

## Worked example: GB grid frequency

To exercise the library end to end, this repo includes a full Power BI Project (PBIP) applying it to a real, actively-regulated process: UK electricity grid frequency, published by the National Energy System Operator (NESO). It was chosen deliberately over tide gauges (not really "an industrial process") and groundwater levels (too much natural long-term drift for a meaningful capability story) because NESO is statutorily required to keep frequency within 1% of 50 Hz (49.5–50.5 Hz) and actively controls it to a tighter 49.8–50.2 Hz day-to-day target — a genuine, continuously-controlled process with a real, externally-set specification.

**Data source**: [NESO system frequency data](https://www.neso.energy/data-portal/system-frequency-data) — 1-second resolution, published as one CSV per calendar month, pulled via a Power Query function (`fnGetGridFrequencyReadings` in `expressions.tmdl`) that looks up the current month's file dynamically through NESO's CKAN `package_show` API rather than a hardcoded URL.

**Model structure**:
- **Fact_GridFrequency** — `dtm` (timestamp), `f` (frequency, Hz), `Date`
- **Dim_GridThresholds** — a one-row lookup table holding the specification limits (USL/LSL) as data rather than hardcoded constants. Currently set to the statutory 49.5/50.5 Hz range, sourced from NESO directly.
- **Dim_Calendar** — auto-sizes its date range from `Fact_GridFrequency`
- **_Measures** — the DAX measures that call the UDF library against this specific fact table

**Report page**: "National Grid Frequency Capability" — four KPI cards (Sigma Level, DPMO, Cpk, Ppk, each with a status label), a control chart with calculated control limits and Nelson Rule 1 violations flagged as distinct markers, and a subtitle naming which day(s), if any, an anomaly was flagged on.

## A methodological note worth reading before trusting the numbers

In the worked example, Cp/Cpk and Pp/Ppk diverge sharply, and that's a real finding rather than a bug — worth understanding since it'll show up again in any high-frequency dataset you point this library at. Cpk is derived from the *short-term* sigma (the average moving range between consecutive readings), and because grid frequency barely moves from one second to the next, that moving range is tiny — which inflates Cpk to an unrealistically large number. Ppk uses the *population* standard deviation across the whole window instead, which correctly captures the real spread including any flagged excursions, and gives a far more honest capability figure. This is a textbook illustration of why within-subgroup variation understates true process variation when consecutive samples are highly autocorrelated.

## Opening the worked example

Requires a recent version of Power BI Desktop with DAX user-defined functions enabled (compatibility level 1702+). Open `SixSigma_Frequency.pbip` directly in Power BI Desktop.

## Reusing the library on your own data

Copy `functions.tmdl` into your own PBIP project's semantic model definition folder, then write measures that call the functions against your own fact table's value/time columns and your own USL/LSL source — a Dim table like `Dim_GridThresholds` here, or a hardcoded constant if you don't need one to be data-driven. No changes to `functions.tmdl` itself should be necessary.

## License / attribution

Grid frequency data is published by NESO under their open data licence — see the [data portal page](https://www.neso.energy/data-portal/system-frequency-data) for current terms.
