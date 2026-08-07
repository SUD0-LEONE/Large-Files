# SGM-2026-06-PANEL — Run notes (Carlingford Bank plc)

Prepared 30 June 2026 (cut-off 17:00 UTC), model SGM-4.2, profile `scenario-generator-run-v1`.

## Deliverables
| File | Rows (excl. header) | Check |
|---|---|---|
| `scenario_panel_return.csv` | 6 | one row per selected scenario, scenario_id ascending |
| `factor_extrapolation_return.csv` | 21,600 | 6 scenarios x 3,600 catalogue factors, catalogue row sequence preserved |
| `portfolio_projection_return.csv` | 42 | 6 scenarios x 6 metrics + 6 PANEL metrics |

## Method applied
1. **Factor-state expansion.** For each candidate scenario, driver combination `d = Σ_j beta_j × shock_j`
   over the ten primary drivers. `ADD` rows → `base + d` (2,830 rows); `LOG` rows → `base × exp(d)`
   (770 rows). Values carried at full precision internally, written to 8 decimals.
2. **Portfolio projection.** Per factor: `contribution = exposure × (scenario value − base value) × loss_scale`,
   joined catalogue↔exposure map on `factor_id` only (3,600/3,600 matched, no orphans on either side).
   Contributions summed by catalogue `asset_class` (RATE, CREDIT, FX, EQUITY_OPTION, OTHER — identical to the
   exposure map's `coverage_theme` partition) to give the five-class feature vector, then to TOTAL_LOSS.
3. **Panel objective.** Complete enumeration of all C(30,6) = 593,775 six-member subsets. Every candidate
   carries 3 compute units, so every subset consumes exactly 18 units and the compute budget binds with
   equality. Objective = TAIL + 3.0×COVERAGE + 2.0×DIVERSITY where
   - TAIL = Σ max(TOTAL_LOSS, 0) / 100 (tail_scale);
   - COVERAGE = Σ theme weights (all 1.0) for drivers with |shock| ≥ 0.75 in **at least one** panel member
     (each theme counted once across the panel);
   - DIVERSITY = Σ over all 15 unordered pairs of (1 − cosine similarity of the five-class feature vectors).
   Tie-break (lexicographically smaller sorted scenario-ID tuple) was implemented but not triggered — the
   optimum is strictly unique at full precision.

## Selected panel
`SP-008, SP-011, SP-017, SP-019, SP-027, SP-029` — 18 / 18 compute units.

| Metric | Value |
|---|---|
| OBJECTIVE | 216.153766 |
| TAIL_TERM | 179.635375 |
| COVERAGE_TERM (3.0 × 8 themes) | 24.000000 |
| DIVERSITY_TERM (2.0 × 6.259196) | 12.518392 |
| COMPUTE_UNITS | 18 |
| FACTOR_ROWS | 21,600 |

Theme representation achieved (8 of 10): CREDIT_LEVEL (SP-008, SP-029), FX_USD & FX_CARRY (SP-011),
EQ_CURVATURE (SP-017), EQ_LEVEL & EQ_SKEW (SP-019), RATE_SLOPE (SP-027), CREDIT_BEND (SP-029).

## Observations / judgement calls
- **RATE_LEVEL and OTHER_LEVEL are not represented.** In the June library only SP-004 (RATE_LEVEL 1.35,
  also RATE_SLOPE 0.82) and SP-024 (OTHER_LEVEL 1.52) breach the 0.75 threshold on those drivers. Swapping
  either in would add 3.0 of coverage but costs far more in tail (SP-004 total loss 839.9, SP-024 negative,
  versus 1,427.8 for the marginal retained scenario SP-011). The registered objective therefore, correctly,
  drops them. This is a property of the weighting, not an omission — flagging it in case the panel is also
  used for driver-coverage attestation.
- The optimum coincides exactly with the six largest positive portfolio losses, because the tail term
  dominates at tail_scale = 100. Verified by enumeration that no coverage/diversity trade-off beats it.
- `COMPUTE_UNITS` and `FACTOR_ROWS` are integers but are written at the manifest's 6-decimal portfolio
  precision for column-type consistency with the other PANEL rows.
- No unreadable or partially-readable source material; all five inputs parsed cleanly and the
  Model Performance Assessment (SATISFACTORY, no open findings) required no method adjustment.
