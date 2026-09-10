[README.md](https://github.com/user-attachments/files/32055604/README.md)
# Agent 2 — AWS Anomaly Pre-Screening Agent

A working implementation of the 8-phase design: it screens Automatic
Weather Station readings for physical, spatial, and temporal
consistency, then reasons about spatial deviations instead of
immediately calling them sensor faults — so it doesn't drown out real
localized events (like the flood example) with false alarms.

## Install / requirements

Pure Python 3.8+, **no external dependencies**.

```bash
cd agent2_project
python3 tests/test_agent.py          # run the test suite
python3 examples/run_flood_example.py  # reproduce the design doc's flood scenario
```

## Structure

```
agent2/
  schema.py      Phase 1 — StationReading input contract
  physical.py    Phase 2 — range checks + cross-variable relationship checks
  spatial.py     Phase 3 — deviation vs. neighboring stations
  temporal.py    Phase 4 — trend / sudden-jump / stale detection
  reasoning.py   Phase 5 — does NOT auto-reject spatial deviations;
                 classifies NO_ANOMALY / POSSIBLE_REAL_EVENT / SUSPICIOUS /
                 LIKELY_SENSOR_FAULT using physical+temporal+cross-variable evidence
  scoring.py     Phase 6 — weighted 0-100 confidence score -> CLEAN/REVIEW/SUSPICIOUS
  pipeline.py    Phase 8 — Agent2 class, the single entry point
  validation.py  Phase 7 — backtesting harness + synthetic test dataset
                 (FP, FN, detection rate, processing time)
tests/
  test_agent.py  Regression tests, including the flood false-positive check
examples/
  run_flood_example.py  End-to-end run of the design doc's Region-1 scenario
```

## Usage

```python
from agent2 import Agent2, StationReading, Coordinates
from datetime import datetime

agent = Agent2()

current = StationReading(
    station_id="A", timestamp=datetime.now(),
    location=Coordinates(19.07, 72.87),
    temperature_c=27.5, humidity_pct=95, pressure_hpa=1003.2,
    wind_speed_ms=10.0, wind_direction_deg=218, rainfall_mm=6.5,
)
neighbors = [...]   # list[StationReading] from nearby stations
history = [...]     # list[StationReading], this station's own recent readings

report = agent.evaluate(current, neighbors, history)

print(report.final_label)          # CLEAN / REVIEW / SUSPICIOUS
print(report.reasoning.verdict)    # e.g. POSSIBLE_REAL_EVENT
print(report.score.total)          # 0-100
print(report.to_dict())            # full JSON-serializable report for logging / ML input
```

`Agent2.evaluate_batch(...)` is provided for scoring a whole network tick
(one reading per station) at once.

## Phase 7 — running validation

```python
from agent2 import Agent2
from agent2.validation import generate_synthetic_cases, run_validation

agent = Agent2()
cases = generate_synthetic_cases()          # synthetic stand-in for real historical episodes
metrics = run_validation(agent, cases)
print(metrics.summary())
```

Once Data Engineering / met department provides real labeled historical
episodes (thunderstorms, floods, cyclones, heat waves, sensor/comm
failures), load them with:

```python
from agent2.validation import load_episodes_from_json, run_validation
cases = load_episodes_from_json("historical_episodes.json")
metrics = run_validation(agent, cases)
```

See the docstring in `validation.py` for the expected JSON schema.

## Tuning

All thresholds are defaults, not laws of physics — they're meant to be
refit using Phase 7 results on real data:

- `agent2.physical.RangeConfig` — per-variable physical bounds
- `agent2.spatial.DEFAULT_ABS_THRESHOLD` / `DEFAULT_Z_THRESHOLD` — how much
  deviation from neighbors counts as suspicious
- `agent2.temporal.MAX_RATE_PER_MIN`, `STALE_MIN_READINGS` — jump/stale sensitivity
- `agent2.scoring.ScoringWeights`, `ScoreThresholds` — the 30/30/20/20 weights
  and CLEAN(<30)/REVIEW(<60)/SUSPICIOUS thresholds from the design doc

Pass custom configs into `Agent2(range_config=..., spatial_z_threshold=...,
scoring_weights=..., score_thresholds=...)`.

## Integration (Phase 8)

```
AWS Stations -> Data Engineering Stream -> Agent2.evaluate() per reading
             -> CLEAN/REVIEW/SUSPICIOUS + full report -> ML model -> final decision
```

`Agent2` is stateless and thread-safe per call — deploy it as a library
inside your stream processor (e.g. a Kafka consumer / Flink UDF), calling
`evaluate()` once per incoming reading with that station's resolved
neighbor list and recent history window.

## Known limitations / next steps

- Neighbor resolution (which stations count as "nearby" to A) is assumed
  to be done upstream and passed in; a haversine-based auto-resolver could
  be added to `spatial.py` if needed.
- Thresholds are reasonable defaults, not empirically fit — Phase 7 against
  real historical data is the intended mechanism to calibrate them.
- `validation.py`'s synthetic generator is for pipeline testing only; it is
  not a substitute for real historical validation.
