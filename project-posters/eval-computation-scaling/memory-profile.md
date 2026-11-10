# Memory profile of the forecast sandbox eval build (2026-11-10)

**Setup**
- Hub: reichlab/forecast-sandbox-2025-2026 at b426a44.
- Config: the sandbox dashboard's `predevals-config.yml` (2 targets × 12 eval sets × 5 disaggregations).
- Image: `ghcr.io/hubverse-org/hubpredevalsdata-docker:latest` (hubPredEvalsData 1.3.0, hubEvals 0.5.0, scoringutils 2.3.0).
- Machine: one Unity cluster job (4 cores, 200 GB limit).
- Totals: 2 h 34 min wall time; 36 GB peak RSS (Slurm MaxRSS).

**Method**
- `generate_eval_data()` ran with entry/exit markers on these functions:
  - `load_model_out_in_eval_set`, `get_and_save_scores`, `get_scores_for_output_type`;
  - `score_model_out`, `transform_quantile_model_out`, `validate_transform_values`, `add_relative_skill_metrics`, `left_join_scores`;
  - `transform_forecasts`, `score`, `summarize_scores`.
- At each marker it recorded the peak RSS since the previous marker (`VmHWM`, reset through `/proc/self/clear_refs`), current RSS, arrow's allocator usage and, per `get_and_save_scores()` call, R's own heap peak.
- The profiled run (2 h 34 min) was no slower than an unprofiled build on the same data the day before (3 h 15 min, on a different node).

## Peak memory by eval set

| target | eval set | rows loaded | peak (GB) | minutes |
|---|---|---|---|---|
| wk inc flu hosp | three seasons, state level | 16,994,424 | 32.4 | 69.9 |
| wk inc flu hosp | 2025-2026 season, state level | 7,352,364 | 18.1 | 26.7 |
| wk inc flu hosp | 2023-2024 season, state level | 5,410,704 | 15.4 | 19.7 |
| wk inc flu hosp | 2024-2025 season, state level | 4,316,640 | 12.3 | 15.4 |
| wk inc flu hosp | three seasons, US national level | 318,504 | 13.4 | 1.6 |
| wk inc flu prop ed visits | three seasons, state level | 2,116,000 | 11.8 | 7.8 |
| wk inc flu prop ed visits | 2025-2026 season, state level | 2,097,600 | 11.8 | 7.5 |
| all others | | ≤ 287,040 | ≤ 12.7 | ≤ 1.4 each |

The baseline of about 11.5–12 GB in the smaller sets is memory the process kept after the first large call, not live data.

## Stages within the largest eval set (flu hosp, three seasons, state level)

| disaggregation | oracle join + as_forecast | transform_forecasts (append) | score() | relative skill | R heap peak |
|---|---|---|---|---|---|
| overall | 67 s, 3.8 → 11.1 GB | 337 s, → 20.9 GB (peak 21.6) | 427 s, peak 25.7 | 8 s | 23.9 GB |
| location | 58 s, 11.7 → 18.8 | 321 s, → 28.0 (peak 30.2) | 416 s, peak 30.3 | 14 s | 21.7 GB |
| reference_date | 59 s, 11.8 → 17.7 | 320 s, → 30.5 | 423 s, peak 32.4 | 15 s | 23.9 GB |
| horizon | 61 s, 11.8 → 16.4 | 339 s, → 27.9 | 435 s, peak 29.1 | 8 s | 20.5 GB |
| target_end_date | 58 s, 13.3 → 21.1 | 326 s, → 24.8 | 429 s, peak 27.9 | 18 s | 19.3 GB |

**Takeaways**
1. The peak scales with the rows in the largest single eval set.
2. Every stage except relative skill and the summary is independent of the disaggregation, yet it is repeated 5×.
3. Starting from the memory kept after the first call, later calls peak 4–7 GB higher than the first.

## Comparison with FluSight

- FluSight has a similar number of files, models and MB per season (see the poster).
- It has similar rows per round: wk inc flu hosp quantiles at state level, horizons 0–3, on 2023-12-16 / 2024-12-14 / 2025-12-13:

  | | 2023-12-16 | 2024-12-14 | 2025-12-13 |
  |---|---|---|---|
  | FluSight | 154,376 | 179,975 | 221,628 |
  | sandbox | 186,576 | 191,360 | 267,904 |

- Its dashboard has no multi-season eval set; its largest is about 6M rows. Its eval build takes about 40 min on a GitHub runner (flusight-dashboard "Rebuild Data", 2026-11-09), and about 9 GB was reported in hubPredEvalsData#80.
