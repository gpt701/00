# Industrial Equipment Data Analyzer

A Python application that analyzes industrial machine readings from CSV files and produces operational metrics, alarms, charts, and a PDF report.

## Version 1 scope

The first version is an offline, command-line application. It reads one CSV file, validates and cleans its readings, calculates machine and sensor metrics, detects threshold-based alarms, and writes its results to an output directory.

It will not connect to a PLC, ESP32, API, or database, and it will not use machine learning. Those are possible future extensions after the CSV workflow is reliable.

## Architecture

```text
CSV input
  -> loader and schema validation
  -> cleaning and quality checks
  -> statistics and machine performance analysis
  -> threshold-based alarm detection
  -> charts and PDF report
  -> output files
```

Planned Python modules:

| Module | Responsibility |
| --- | --- |
| `src/data_loader.py` | Read CSV input as raw strings and validate the required column names. |
| `src/data_cleaner.py` | Normalize timestamps, sort and deduplicate rows, validate measurements, and report data-quality issues. |
| `src/analyzer.py` | Calculate per-sensor statistics and per-machine operating time, downtime, availability, energy, and production metrics. |
| `src/anomaly_detector.py` | Detect configurable temperature, current, and low-RPM threshold violations and produce an alarm table. |
| `src/visualization.py` | Generate per-machine temperature, power, and RPM time-series charts. |
| `src/report_generator.py` | Assemble analysis results and charts into a PDF report. |
| `main.py` | Coordinate the workflow and expose the command-line entry point. |

## Input CSV schema

The input is UTF-8 CSV with one row per one-minute sample. Column names are case-sensitive. `machine_id` and `timestamp` identify the equipment and sample; sensor and production fields may be blank if a reading is unavailable.

| Column | Type / unit | Meaning |
| --- | --- | --- |
| `machine_id` | text | Stable identifier for the equipment. |
| `timestamp` | ISO 8601 datetime | Time the sample interval ends. Timestamps should include a UTC offset; the analyzer will normalize them to UTC. |
| `temperature_c` | decimal, °C | Machine temperature at the sample time. |
| `current_a` | decimal, A | Electrical current at the sample time. |
| `voltage_v` | decimal, V | Electrical voltage at the sample time. |
| `rpm` | decimal, revolutions/minute | Motor speed at the sample time. |
| `power_kw` | decimal, kW | Measured active power at the sample time. |
| `vibration_mm_s` | decimal, mm/s RMS | Vibration velocity at the sample time. |
| `production_count_delta` | non-negative integer, units | Units produced during the minute ending at this sample. The first row for each machine should be `0`. |
| `machine_status` | enum | State during the minute ending at this sample: `RUNNING`, `WARNING`, `ALARM`, or `STOPPED`. |

Rows should be ordered by `machine_id` and `timestamp`, with no duplicate machine/timestamp pairs. A missing timestamp or invalid machine identifier makes a row unusable; invalid or missing measurements are reported and excluded from the affected measurement's statistics rather than silently replaced. Outlier handling will be conservative: values outside physically configured limits are flagged as invalid, not automatically clipped or imputed.

### Time and performance metric conventions

- A row's `machine_status` describes the one-minute interval ending at its timestamp.
- A valid `STOPPED` row contributes one minute of downtime. Other valid statuses contribute operating time.
- The final row does not add an extra interval; it describes the minute ending at its own timestamp, just like every other row.
- Availability is `operating_time / (operating_time + downtime) * 100`. Missing or invalid status intervals are excluded from both totals and reported as unknown time.
- Energy is estimated by summing `power_kw * 1/60` kWh for each valid one-minute interval. It is an estimate based on the sampling interval, not a meter-integrated value.
- Production is the sum of valid `production_count_delta` values.
- Initial demonstration alarm thresholds are temperature above 85 °C, current above 7 A, and RPM below 90% of the configured nominal RPM (default nominal RPM: 1,500). They will be configurable and are examples only, not safety limits; real deployments must use equipment-specific limits approved by a qualified engineer.

## Outputs

For an input such as `data/machine_data.csv`, the command-line run will write:

```text
output/
  report.pdf
  summary.csv
  alarms.csv
  {machine_id}_temperature.png
  {machine_id}_power.png
  {machine_id}_rpm.png
```

Outputs should be reproducible from the same input and configuration. The application will report data-quality counts and analysis-period boundaries so users can interpret the results.

## Dependencies

- Python 3.11 or newer
- pandas: CSV and tabular data handling
- NumPy: numerical calculations
- Matplotlib: chart generation
- ReportLab: PDF generation
- pytest (development only): automated tests

Runtime packages are listed in `requirements.txt`. Test tooling will be added separately when the test suite is introduced.

## Development milestones

1. [x] Finalize the input and output contracts and create a realistic synthetic dataset.
2. [x] Implement CSV loading, schema validation, and data-quality reporting.
3. [x] Implement descriptive statistics and machine performance metrics.
4. [x] Implement configurable alarm detection.
5. [x] Generate measurement charts.
6. Generate the PDF report.
7. Connect the full command-line workflow and write summary/alarm CSV outputs.
8. Add tests, usage examples, and portfolio-ready documentation.

Each milestone will be implemented and checked before moving to the next one.

## Synthetic demo data

`data/machine_data.csv` is generated, reproducible demo data, not measurements from real equipment. It contains 100,000 one-minute samples for `MILL-001`, with scheduled temperature, current, and RPM alarm scenarios, stopped periods, correlated sensor changes, and occasional blank sensor readings.

Regenerate the default dataset from the project root:

```bash
python scripts/generate_sample_data.py
```

The generator accepts `--output`, `--rows`, `--seed`, and `--start` options. For example, `python scripts/generate_sample_data.py --rows 500 --seed 7 --output data/preview.csv` creates a smaller repeatable sample. Timestamps are written in UTC; the default start is `2026-09-01T00:00:00+00:00`.

## Loading and cleaning input

The data pipeline can be used from the project root:

```python
from src.data_cleaner import clean_machine_data
from src.data_loader import load_machine_data

raw_data = load_machine_data("data/machine_data.csv")
machine_data, quality = clean_machine_data(raw_data)
print(quality.to_dict())
```

Missing required columns, duplicate column names, and empty files are rejected with explicit errors. Rows with missing machine IDs or invalid/missing timezone-aware timestamps are dropped; duplicate machine/timestamp rows keep the first occurrence, and out-of-order samples are sorted chronologically. Blank or invalid measurements are left missing and counted rather than imputed. Broad configurable-in-code sanity ranges catch implausible values; they are data-quality checks, not safety thresholds. Invalid or missing machine statuses are counted as `UNKNOWN`.

Focused tests for loading and cleaning can be run with:

```bash
python -m unittest discover -s tests
```

## Machine and sensor analysis

Analyze cleaned readings using:

```python
from src.analyzer import analyze_machine_data

result = analyze_machine_data(machine_data)
print(result.summary.to_string(index=False))
print(result.sensor_statistics.to_string(index=False))
```

The summary contains one row per machine, including observed time boundaries, sample count, operating/downtime/unknown-status minutes, availability, production total, and estimated energy. Sensor statistics are in a separate long-form table with valid and missing counts, mean, median, min/max, sample standard deviation, and 95th percentile. Availability is `NaN` when there are no known status intervals. Power readings with missing values are omitted from estimated energy and counted separately; production is summed over valid samples. These calculations follow the fixed one-minute sample convention; gaps in timestamp sequences are not interpolated.

## Threshold-based alarms

Detect alarms in cleaned data:

```python
from src.anomaly_detector import AlarmThresholds, detect_alarms

alarms = detect_alarms(machine_data)
custom_thresholds = AlarmThresholds(
    high_temperature_c=85,
    over_current_a=7,
    nominal_rpm=1500,
    low_rpm_fraction=0.9,
)
alarms = detect_alarms(machine_data, custom_thresholds)
```

The default demo rules are strict threshold comparisons: temperature above 85 °C, current above 7 A, and RPM below 90% of nominal RPM (1,350 RPM by default). RPM alarms are suppressed when the sample status is `STOPPED`, since zero speed is expected then. Each violating sample produces its own alarm row; consecutive rows are not grouped into incidents. These example values are not safety limits and must be configured for the actual equipment before operational use.

## Measurement charts

Create charts after loading and cleaning the input:

```python
from src.visualization import generate_charts

chart_paths = generate_charts(machine_data, "output")
```

The chart generator writes `{machine_id}_temperature.png`, `{machine_id}_power.png`, and `{machine_id}_rpm.png` for each machine, with UTC time axes. Temperature alarm samples are highlighted; temperature and RPM charts show their configured threshold lines. Missing readings are skipped, but if a machine has no valid readings for one of the charted metrics, generation stops with a clear error rather than silently producing an incomplete set. The x-axis automatically selects date ticks suitable for the time range.

## PDF report

Generate a PDF after calculating the analysis, alarms, and charts:

```python
from src.analyzer import analyze_machine_data
from src.anomaly_detector import detect_alarms
from src.report_generator import generate_pdf_report
from src.visualization import generate_charts

analysis = analyze_machine_data(machine_data)
alarms = detect_alarms(machine_data)
charts = generate_charts(machine_data, "output")
report_path = generate_pdf_report(
    "output/report.pdf",
    analysis,
    alarms,
    quality,
    charts,
)
```

The report includes machine performance summaries, sensor statistics, data-quality counts, alarm counts and the latest 25 alarm readings, followed by the generated charts for each machine. `alarms` retains the complete alarm table for the future CSV export. The PDF explicitly notes that energy is estimated and alarm thresholds are demonstration values. Missing chart files or incomplete chart mappings raise explicit errors.
