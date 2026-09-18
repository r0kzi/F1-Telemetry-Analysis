# F1 Telemetry Analysis

Compare two Formula 1 drivers' laps -- from any session, any circuit, any year -- using real telemetry pulled from the [FastF1](https://docs.fastf1.dev/) API.

Given two driver codes (e.g. `VER` vs `LEC`), it produces:

- An **interactive multi-channel dashboard** (speed, RPM, gear, throttle, brake, a steering estimate, a wheel-speed estimate, and delta time) synchronised on a shared distance axis, with corner markers overlaid.
- A **track map** with both racing lines and corner numbers.
- A **corner-by-corner breakdown**: entry/apex/exit speed, braking point, and throttle application at every corner.
- A **plain-text race engineer report** summarising who was faster, where, and by how much.

Corner locations are pulled directly from FastF1's circuit info for the session, so this works for **any track on the F1 calendar** -- it isn't limited to a single hard-coded circuit.

## Example

```python
config = AnalysisConfig(
    year=2025,
    event="Monaco Grand Prix",
    session="Q",
    driver1="VER",
    driver2="LEC",
)
```

Running the notebook with this config compares Verstappen's and Leclerc's fastest laps in 2025 Monaco Qualifying, corner by corner.

## Project structure

```
f1-telemetry-analysis/
├── analysis.ipynb          # Main notebook -- set the config, run all cells
├── f1_telemetry/
│   ├── config.py            # AnalysisConfig dataclass
│   ├── data_loader.py       # Session loading + lap selection
│   ├── telemetry.py         # Distance-synced telemetry, delta time, steering/wheel-speed estimates
│   ├── corners.py           # Track-agnostic corner detection + corner metrics
│   ├── visualization.py     # Plotly dashboard + Matplotlib track map
│   └── report.py            # Sector table, summary stats, text report
├── requirements.txt
└── README.md
```

The analysis logic lives in the `f1_telemetry` package rather than the notebook, so each piece (loading data, syncing telemetry, corner detection, plotting, reporting) can be tested, reused, or swapped independently.

## Setup

```bash
git clone https://github.com/<your-username>/f1-telemetry-analysis.git
cd f1-telemetry-analysis
pip install -r requirements.txt
jupyter notebook analysis.ipynb
```

FastF1 caches downloaded session data locally in `cache/` so repeat runs for the same session are fast and don't re-hit the API.

## How it works

1. **Load** -- `fastf1.get_session()` fetches lap and telemetry data for the requested year/event/session.
2. **Select laps** -- either each driver's fastest lap, or an explicit lap number.
3. **Synchronise** -- both drivers' car data is sampled at different times, so every channel is resampled with `np.interp` onto a shared 0..max_distance axis. This is what makes lap comparisons possible.
4. **Derive channels** -- delta time comes from the resampled time channel; the steering estimate comes from the rate of change of heading (`arctan2` of the gradient of XY position); wheel speed is approximated as ground speed inflated by how hard the car is turning.
5. **Corner detection** -- `session.get_circuit_info().corners` gives corner number, letter, and distance-from-start for the current circuit, straight from FastF1 -- no manual per-track data required.
6. **Corner metrics** -- for each corner, entry speed (50 m before the apex), apex speed, exit speed (50 m after), throttle application (30 m after), and first braking point within 150 m of the apex.

## Notes and limitations

- The **steering estimate** is a proxy derived from position data, not the car's real steering-wheel angle (FastF1 does not expose that channel). It's useful for relative comparison, not absolute angles.
- FastF1's circuit corner data is community-maintained (via MultiViewer) for visualization purposes and is not perfectly precise.
- Telemetry availability varies by session and year; very old sessions may have incomplete data.

## Tech stack

Python, [FastF1](https://docs.fastf1.dev/), pandas, NumPy, Plotly, Matplotlib.

## Possible extensions

- Batch-compare every driver in a session, not just two.
- Tyre-compound and stint-by-stint pace comparison across a race.
- Weather-adjusted pace normalisation.
