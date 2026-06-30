# VRI Framework

**Valid / Reference / Issue — A Governance and Auditing Convention for Production Forecasting**

---

## Purpose

VRI is a governance and auditing convention for production forecasting systems. It provides a strict operational vocabulary for managing temporal leakage, data eligibility, and forecast reproducibility.

It does not introduce a new forecasting methodology or temporal theory.

---

## Core Concepts

### Valid Time
Valid Time is the business-defined time point or interval that a record or prediction describes. It is a property of the modelled reality, independent of when the record was created, collected, or published.
- For the half-hourly load of 08:00–08:30 on a given day, the Valid Time is 08:00, the start of the settlement period.
- For a weather forecast for tomorrow, the Valid Time is the future time described by that forecast.

### Issue Time
Issue Time is the time at which a record, prediction, or record version becomes available to its intended consumer.

| Type | Valid Time | Issue Time | Description |
| :--- | :--- | :--- | :--- |
| Outturn          | 2026-06-18 08:00:00 | 2026-06-19 02:00:00 | Published at 02:00 on the following day |
| Outturn revision | 2026-06-18 08:00:00 | 2026-06-19 10:00:00 | Revised value published later the same day |
| Forecast         | 2026-06-20 08:00:00 | 2026-06-19 08:15:00 | Forecast released at 08:15 the day before |

Each revision or forecast release is a separate record version with its own Issue Time.

### Reference Time

Reference Time is the temporal anchor of a forecasting sample. It defines the information state used to construct that sample.
- Each training, validation, test, or live sample has one Reference Time.
- Every input value included in the sample must satisfy: `Issue Time ≤ Reference Time`

For example, consider a half-hourly solar-generation model using `Solar Radiation Forecast` and `Cloud Cover Forecast` as input features. The model jointly produces `HH0`, `HH1`, and `HH2` as separate outputs.

The final training row constructed by the I/O builder may be:

| Reference Time | Solar Radiation Forecast | Cloud Cover Forecast | Solar Generation HH0 | Solar Generation HH1 | Solar Generation HH2 |
| :--- | ---: | ---: | ---: | ---: | ---: |
| 08:00 | 320 W/m² | 65% | 110 MW | 145 MW | 180 MW |

The first test input row contains the same input features without realised outputs:

| Reference Time | Solar Radiation Forecast | Cloud Cover Forecast |
| :--- | ---: | ---: |
| 08:30 | 410 W/m² | 48% |

After inference:

| Reference Time | Predicted Solar Generation HH0 | Predicted Solar Generation HH1 | Predicted Solar Generation HH2 |
| :--- | ---: | ---: | ---: |
| 08:30 | 150 MW | 190 MW | 225 MW |

The Forecast Schedule maps each output horizon to its Valid Time:

| Reference Time | Horizon | Valid Time | Solar Generation Forecast |
| :--- | :--- | :--- | ---: |
| 08:30 | HH0 | 08:30 | 150 MW |
| 08:30 | HH1 | 09:00 | 190 MW |
| 08:30 | HH2 | 09:30 | 225 MW |

The Forecast Schedule maps each output horizon from the sample's Reference Time to its Valid Time.

### Forecast Schedule

A Forecast Schedule defines how each forecast horizon maps from a sample's Reference Time to an output Valid Time. For a regular half-hourly schedule:

| Reference Time | Horizon | Valid Time |
| :--- | :--- | :--- |
| 08:30 | H0 | 08:30 |
| 08:30 | H1 | 09:00 |
| 08:30 | H2 | 09:30 |

The schedule may also be irregular. A horizon therefore represents an ordered output position and does not necessarily imply a fixed elapsed duration. Forecast Schedule is a forecasting-system configuration, not an additional VRI time concept.

### Cutoff
Cutoff is the global data boundary applied to one dataset construction, training run, backtest, or inference job. A run normally has one Cutoff, while the samples or inferences within that run may have many Reference Times.

Cutoff constrains:
- which record versions can be retrieved;
- which samples can be constructed;
- whether required labels are available;
- where a training or evaluation dataset must end.

Cutoff does not replace Reference Time.
- **Reference Time** governs the information state of an individual sample.
- **Cutoff** governs the overall data boundary of the run.

---

## Training

Training samples follow the same input eligibility rule as inference.

For each sample:

- A historical point on the Valid Time axis is assigned as the sample's Reference Time.
- Every input record used by the sample satisfies `Issue Time ≤ Reference Time`.
- Labels are realised outcomes associated with the required Valid Times.
- Labels may be issued after the Reference Time and attached later for model fitting.
- The model architecture, such as XGBoost, MLP, TCN, or LSTM, is independent of VRI.

### Label Completeness and Past-Covariate Availability
A training sample may be included only when all required labels are available by the dataset Cutoff: 

`Required Label Issue Time ≤ Cutoff`

This is a label-completeness rule, not an input eligibility rule. Label Issue Times may be later than the sample's Reference Time.

An observation is not automatically eligible because its Valid Time is earlier than the Reference Time. It must still satisfy: `Issue Time ≤ Sample Reference Time`

For each input series, the input-selection policy must define:

- which eligible record version is selected;
- any required safety lag;
- whether an eligible forecast or nowcast replaces an unavailable outturn;
- how missing values are handled;
- whether the sample is excluded when historical availability cannot be reproduced reliably.

Historical samples must reproduce the source-specific availability state that existed at each Reference Time. Later publications, revisions, corrections, and backfills are ineligible unless they were already available at that time.

For a regular half-hourly schedule where labels are issued immediately at their Valid Times:

`Latest Training Reference Time = Cutoff − Maximum Horizon × 30 minutes`

This is a special case. For delayed labels or irregular Forecast Schedules, use the actual label-completeness rule:

`Required Label Issue Time ≤ Cutoff`

---

## Testing & Inference

Testing follows the same input eligibility rule as training. A model may be evaluated through historical inference or used for live prediction.

- Each test or live sample has one Reference Time.
- Only input records satisfying `Issue Time ≤ Reference Time` are eligible.
- Future covariates are allowed when their record versions were issued by the Reference Time.
- Realised target values are never used as model inputs.
- Realised targets are attached only for scoring after predictions have been generated.
- For out-of-sample backtesting, the completed model is frozen before the first simulated test inference.
- No test-period samples or labels enter model fitting.

### Single Inference and Multiple Inferences

- **Single inference**: one Reference Time, one logical model input, and one set of outputs.
- **Multiple inferences**: multiple logical model inputs, each governed by its own Reference Time.

Multiple inferences may be processed individually or together in a batch. The number of inferences is independent of the number of forecast horizons:

- one inference may produce one or multiple horizons;
- an evaluation may contain multiple inferences;
- each inference may use a single-horizon or multi-horizon output structure.

Predictions are mapped to their Valid Times through the Forecast Schedule and may then be evaluated against realised values.

---

## Multi-horizon and I/O Schema

### Multi-horizon
Multi-horizon forecasting produces values for multiple Valid Times:

`y(t+1), y(t+2), ..., y(t+H)`

A horizon is an ordered position in the Forecast Schedule. It does not necessarily represent a fixed elapsed duration.

Multi-horizon describes the temporal coverage of a forecast. It is independent of the number of input and output dimensions presented to the model.

### I/O Schema

SISO, SIMO, MISO, and MIMO describe the logical structure presented to a model:

- **SISO**: single input, single output
- **SIMO**: single input, multiple outputs
- **MISO**: multiple inputs, single output
- **MIMO**: multiple inputs, multiple outputs

Inputs and outputs may represent semantic variables or separately constructed model dimensions.

A jointly generated multi-horizon forecast may therefore appear as multiple output dimensions even when all outputs belong to the same target series.

Multi-horizon and I/O schema describe different aspects of the model:

- **Multi-horizon** describes which future Valid Times are predicted.
- **I/O schema** describes the model's input and output dimensions.

---

## I/O Schema Examples

The examples below describe the input and output structures presented to a forecasting model.

- In a **tabular** setup, each sample is represented as one row containing its inputs and outputs. The row has one Reference Time, while individual values may describe different Valid Times.
- In a **sequential** setup, each sample contains an ordered input sequence and one or more outputs. The sample has one Reference Time, while each sequence position and output has its own Valid Time.

Any input used to construct a sample must satisfy: `Issue Time ≤ Reference Time`

In the tabular examples, `t` denotes the sample's Reference Time on the underlying time axis. Expressions such as `t-1` and `t+1` identify the Valid Times of individual input and output values relative to that Reference Time.

---

### SISO Example 1: Tabular Single-Horizon Autoregressive Forecast

A tabular model uses one historical load value to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SISO | Load at `t-1` | Load at `t+1` |

Training samples may look like:

| Reference Time | Load `t-1` | Load `t+1` |
| :--- | ---: | ---: |
| 2026-06-19 07:00 | 24,100 MW | 24,900 MW |
| 2026-06-19 07:30 | 24,450 MW | 25,300 MW |
| 2026-06-19 08:00 | 24,900 MW | 25,650 MW |

A test input sample may look like:

| Reference Time | Load `t-1` |
| :--- | ---: |
| 2026-06-19 08:30 | 25,300 MW |

After inference:

| Reference Time | Valid Time | Load Forecast |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 09:00 | 25,650 MW |

---

### SISO Example 2: Tabular Forecast from One Future Covariate

A tabular model uses one temperature forecast value to predict one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SISO | Temperature forecast | GB Zone A load |

A training sample may look like:

| Reference Time | Temperature Forecast | Load |
| :--- | ---: | ---: |
| 2026-06-19 08:00 | 18.1°C | 25,650 MW |

The input and output describe the following Valid Times:

| Reference Time | Item | Valid Time | Issue Time |
| :--- | :--- | :--- | :--- |
| 2026-06-19 08:00 | Temperature forecast | 2026-06-19 08:30 | 2026-06-19 07:40 |
| 2026-06-19 08:00 | Load label | 2026-06-19 08:30 | 2026-06-19 08:35 |

The temperature forecast is eligible because:

`2026-06-19 07:40 ≤ 2026-06-19 08:00`

Its Valid Time is later than the Reference Time because it is a future covariate.

---

### SISO Example 3: Sequential Single-Horizon Forecast

A sequential model uses one historical load sequence to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SISO | Historical load sequence | Future load |

A training sample contains one ordered input sequence:

| Reference Time | Valid Time | Historical Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 06:00 | 23,800 MW |
|                  | 2026-06-19 06:30 | 24,100 MW |
|                  | 2026-06-19 07:00 | 24,450 MW |
|                  | 2026-06-19 07:30 | 24,900 MW |

The corresponding output is:

| Reference Time | Valid Time | Future Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 08:30 | 25,300 MW |

A test input sample may contain:

| Reference Time | Valid Time | Historical Load |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 06:30 | 24,100 MW |
|                  | 2026-06-19 07:00 | 24,450 MW |
|                  | 2026-06-19 07:30 | 24,900 MW |
|                  | 2026-06-19 08:00 | 25,300 MW |

After inference:

| Reference Time | Valid Time | Load Forecast |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 09:00 | 25,650 MW |

This is SISO because one ordered load sequence produces one future load value.

---

### SIMO Example 1: Tabular Multi-Horizon Forecast

A tabular model uses one temperature forecast value to jointly forecast load across three horizons.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SIMO | Temperature forecast | Load H1; Load H2; Load H3 |

Training samples may look like:

| Reference Time | Temperature Forecast | Load H1 | Load H2 | Load H3 |
| :--- | ---: | ---: | ---: | ---: |
| 2026-06-19 07:00 | 17.2°C | 24,900 MW | 25,300 MW | 25,650 MW |
| 2026-06-19 07:30 | 17.6°C | 25,300 MW | 25,650 MW | 25,900 MW |
| 2026-06-19 08:00 | 18.1°C | 25,650 MW | 25,900 MW | 26,100 MW |

A test input sample may look like:

| Reference Time | Temperature Forecast |
| :--- | ---: |
| 2026-06-19 08:30 | 18.5°C |

After inference:

| Reference Time | Load H1 | Load H2 | Load H3 |
| :--- | ---: | ---: | ---: |
| 2026-06-19 08:30 | 25,850 MW | 26,050 MW | 26,200 MW |

The Forecast Schedule maps the outputs to:

| Reference Time | Horizon | Valid Time | Load Forecast |
| :--- | :--- | :--- | ---: |
| 2026-06-19 08:30 | H1 | 2026-06-19 09:00 | 25,850 MW |
| 2026-06-19 08:30 | H2 | 2026-06-19 09:30 | 26,050 MW |
| 2026-06-19 08:30 | H3 | 2026-06-19 10:00 | 26,200 MW |

This is SIMO because one tabular input value produces multiple output values.

---

### SIMO Example 2: Sequential Multi-Horizon Forecast

A sequential model uses one historical load sequence to jointly forecast three future load values.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SIMO | Historical load sequence | Load H1; Load H2; Load H3 |

A training sample may look like:

**Input**

| Reference Time | Valid Time | Historical Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 06:00 | 23,800 MW |
|                  | 2026-06-19 06:30 | 24,100 MW |
|                  | 2026-06-19 07:00 | 24,450 MW |
|                  | 2026-06-19 07:30 | 24,900 MW |

**Output**

| Reference Time | Horizon | Valid Time | Load |
| :--- | :--- | :--- | ---: |
| 2026-06-19 08:00 | H1 | 2026-06-19 08:30 | 25,300 MW |
|                  | H2 | 2026-06-19 09:00 | 25,650 MW |
|                  | H3 | 2026-06-19 09:30 | 25,900 MW |

This is SIMO because one ordered load sequence produces multiple future load values.

---

### SIMO Example 3: One Input and Two Output Variables

A tabular model uses one solar-radiation forecast value to jointly predict two solar-generation variables.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SIMO | Solar-radiation forecast | Grid-connected solar; embedded solar |

A training sample may look like:

| Reference Time | Solar-Radiation Forecast | Grid-Connected Solar | Embedded Solar |
| :--- | ---: | ---: | ---: |
| 2026-06-19 07:00 | 410 W/m² | 3,250 MW | 1,180 MW |
| 2026-06-19 07:30 | 380 W/m² | 3,350 MW | 1,070 MW |
| 2026-06-19 08:00 | 220 W/m² | 3,270 MW | 1,210 MW |

The input and outputs describe the following Valid Times:

| Reference Time | Item | Valid Time |
| :--- | :--- | :--- |
| 2026-06-19 08:00 | Solar-radiation forecast | 2026-06-19 08:30 |
| 2026-06-19 08:00 | Grid-connected solar | 2026-06-19 08:30 |
| 2026-06-19 08:00 | Embedded solar | 2026-06-19 08:30 |

This is SIMO because one input value produces two output variables.

---

### MISO Example 1: Tabular Single-Horizon Load Forecast

A tabular model uses several input values to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MISO | Load at `t-1`; temperature forecast at `t+1`; wind forecast at `t+1` | Load at `t+1` |

Training samples may look like:

| Reference Time | Load `t-1` | Temperature Forecast `t+1` | Wind Forecast `t+1` | Load `t+1` |
| :--- | ---: | ---: | ---: | ---: |
| 2026-06-19 07:00 | 24,100 MW | 17.2°C | 8,400 MW | 24,900 MW |
| 2026-06-19 07:30 | 24,450 MW | 17.6°C | 8,250 MW | 25,300 MW |
| 2026-06-19 08:00 | 24,900 MW | 18.1°C | 8,100 MW | 25,650 MW |

A test input sample may look like:

| Reference Time | Load `t-1` | Temperature Forecast `t+1` | Wind Forecast `t+1` |
| :--- | ---: | ---: | ---: |
| 2026-06-19 08:30 | 25,300 MW | 18.5°C | 7,950 MW |

After inference:

| Reference Time | Valid Time | Load Forecast |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 09:00 | 25,850 MW |

This is MISO because multiple tabular input values produce one output value.

---

### MISO Example 2: Sequential Multi-Input Forecast

A sequential model uses a multivariate sequence of load, temperature, and wind values to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MISO | Load, temperature, and wind feature channels | Future load |

One training sample may look like:

| Reference Time | Valid Time | Load | Temperature | Wind |
| :--- | :--- | ---: | ---: | ---: |
| 2026-06-19 08:00 | 2026-06-19 06:30 | 24,100 MW | 17.0°C | 8,600 MW |
|                  | 2026-06-19 07:00 | 24,450 MW | 17.2°C | 8,400 MW |
|                  | 2026-06-19 07:30 | 24,900 MW | 17.6°C | 8,250 MW |

The corresponding output is:

| Reference Time | Valid Time | Future Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 08:30 | 25,300 MW |

This is MISO because the sequential input contains multiple feature channels while the model produces one output value.

---

### MISO Example 3: Inputs with Different Availability States

A tabular model is configured with historical load, a temperature forecast, and a wind forecast.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MISO | Historical load; temperature forecast; wind forecast | Future load |

For a sample with Reference Time `2026-06-19 08:00`:

| Input | Valid Time | Issue Time |
| :--- | :--- | :--- |
| Historical load | 2026-06-19 07:30 | 2026-06-19 07:55 |
| Temperature forecast | 2026-06-19 08:30 | 2026-06-19 07:40 |
| Wind forecast | 2026-06-19 08:30 | 2026-06-19 08:05 |

The historical load is eligible because:

`2026-06-19 07:55 ≤ 2026-06-19 08:00`

The temperature forecast is eligible because:

`2026-06-19 07:40 ≤ 2026-06-19 08:00`

The wind forecast issued at 08:05 is not eligible because:

`2026-06-19 08:05 > 2026-06-19 08:00`

An earlier eligible wind-forecast version must be used. If no eligible version exists, the sample must follow the model's defined missing-input policy or be excluded.

---

### MIMO Example 1: Tabular Multi-Input, Multi-Horizon Forecast

A tabular model uses solar-radiation and cloud-cover forecast values to jointly forecast solar generation across three horizons.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MIMO | Solar-radiation forecast; cloud-cover forecast | Solar H0; Solar H1; Solar H2 |

Training samples may look like:

| Reference Time | Solar-Radiation Forecast | Cloud-Cover Forecast | Solar H0 | Solar H1 | Solar H2 |
| :--- | ---: | ---: | ---: | ---: | ---: |
| 2026-06-19 08:00 | 320 W/m² | 65% | 110 MW | 145 MW | 180 MW |
| 2026-06-19 08:30 | 410 W/m² | 48% | 150 MW | 190 MW | 225 MW |

A test input sample may look like:

| Reference Time | Solar-Radiation Forecast | Cloud-Cover Forecast |
| :--- | ---: | ---: |
| 2026-06-19 09:00 | 470 W/m² | 40% |

After inference:

| Reference Time | Horizon | Valid Time | Solar Forecast |
| :--- | :--- | :--- | ---: |
| 2026-06-19 09:00 | H0 | 2026-06-19 09:00 | 185 MW |
| 2026-06-19 09:00 | H1 | 2026-06-19 09:30 | 220 MW |
| 2026-06-19 09:00 | H2 | 2026-06-19 10:00 | 250 MW |

This is MIMO because multiple tabular input values produce multiple output values.

---

### MIMO Example 2: Multiple Inputs and Multiple Output Variables

A tabular model uses several input values to jointly predict future load and future price.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MIMO | Load at `t-1`; price at `t-1`; temperature forecast at `t+1`; wind forecast at `t+1` | Load at `t+1`; price at `t+1` |

A training sample may look like:

| Reference Time | Load `t-1` | Price `t-1` | Temperature Forecast `t+1` | Wind Forecast `t+1` | Load `t+1` | Price `t+1` |
| :--- | ---: | ---: | ---: | ---: | ---: | ---: |
| 2026-06-19 08:00 | 24,900 MW | £72/MWh | 18.1°C | 8,100 MW | 25,650 MW | £75/MWh |

The two outputs share the same Valid Time:

| Reference Time | Output | Valid Time |
| :--- | :--- | :--- |
| 2026-06-19 08:00 | Load `t+1` | 2026-06-19 08:30 |
| 2026-06-19 08:00 | Price `t+1` | 2026-06-19 08:30 |

This is MIMO because multiple tabular input values produce multiple output variables.

---

### MIMO Example 3: Multiple Zones and Multiple Horizons

A tabular model jointly forecasts load for Zone A and Zone B across two future days.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MIMO | Zone A load; Zone B load; Zone A weather H1 and H2; Zone B weather H1 and H2 | Zone A H1; Zone A H2; Zone B H1; Zone B H2 |

A test input sample may look like:

| Reference Time | Zone A Load | Zone B Load | Zone A Weather H1 | Zone A Weather H2 | Zone B Weather H1 | Zone B Weather H2 |
| :--- | ---: | ---: | :--- | :--- | :--- | :--- |
| 2026-06-19 03:00 | 12,400 MW | 9,800 MW | Mild | Warm | Cool | Mild |

After inference:

| Reference Time | Output | Valid Time | Forecast |
| :--- | :--- | :--- | ---: |
| 2026-06-19 03:00 | Zone A H1 | 2026-06-20 | 12,750 MW |
| 2026-06-19 03:00 | Zone A H2 | 2026-06-21 | 12,900 MW |
| 2026-06-19 03:00 | Zone B H1 | 2026-06-20 | 10,050 MW |
| 2026-06-19 03:00 | Zone B H2 | 2026-06-21 | 10,200 MW |

This is MIMO because multiple input values produce multiple output values.

---

## Additional VRI Examples

### Example: Batched Historical Inference

A backtest evaluates the model at several historical Reference Times.

| Reference Time | Valid Time |
| :--- | :--- |
| 2026-06-01 08:00 | 2026-06-01 08:30 |
| 2026-06-01 08:30 | 2026-06-01 09:00 |
| 2026-06-01 09:00 | 2026-06-01 09:30 |

Each row is a separate logical sample with its own Reference Time.

The samples may be processed together in one batch. Batching does not merge their Reference Times or change the model's I/O schema.

---

### Example: Cutoff and Label Completeness

A training dataset is constructed with:

`Cutoff = 2026-06-19 08:30`

For every input used by a sample:

`Issue Time ≤ Sample Reference Time`

For every required label:

`Label Issue Time ≤ Cutoff`

| Reference Time | Label Valid Time | Label Issue Time | Cutoff |
| :--- | :--- | :--- | :--- |
| 2026-06-19 07:30 | 2026-06-19 08:00 | 2026-06-19 08:05 | 2026-06-19 08:30 |
| 2026-06-19 07:30 | 2026-06-19 08:30 | 2026-06-19 09:00 | 2026-06-19 08:30 |

The first label is available by the Cutoff.

The second label is not available by the Cutoff. If both labels are required, the sample is not label-complete and must be excluded.

For a regular half-hourly schedule where labels are available immediately at their Valid Times:

`Latest Training Reference Time = Cutoff − Maximum Horizon × 30 minutes`

This is a special case. When labels are issued later or the Forecast Schedule is irregular, completeness must be evaluated using the actual Label Issue Times.

---

### Example: Decision Time Is Outside VRI

A forecast is issued before a later business decision.

| Reference Time | Valid Time | Issue Time | Decision Time |
| :--- | :--- | :--- | :--- |
| 2026-06-19 08:00 | 2026-06-19 08:30 | 2026-06-19 08:15 | 2026-06-19 10:00 |

VRI governs the temporal semantics of the inputs, samples, and predictions.

The later business Decision Time is outside VRI.

---

## Auditing

A VRI-compliant system should be able to reconstruct, for each logical inference:
- Reference Time
- Valid Time
- Forecast horizon
- Applicable Forecast Schedule
- Input record identifiers with their Valid Times and Issue Times
- Selected record versions and the deterministic rule used to select them
- Output record identifiers with their Valid Times and Issue Times
- Model version
- Run Cutoff

For training datasets, the system should additionally record:
- Sample Reference Time
- Target labels with their Valid Times and Issue Times
- Dataset Cutoff
- Training and test periods
- Target-buffer or label-completeness rule

---

## What VRI Is Not

VRI is not:
- a forecasting algorithm;
- a temporal database theory;
- a leakage detection theory;
- a replacement for feature stores;
- a replacement for forecasting methodology.

It is a governance and auditing convention for production forecasting systems.

---

## References

[1] **Databricks.** (n.d.). "Point-in-time feature joins." *Databricks Feature Store Documentation.*  
Accessed 2026-06-19.  
https://docs.databricks.com/aws/en/machine-learning/feature-store/time-series

[2] **Tecton.** (n.d.). "Construct Training Data." *Tecton Documentation.*  
Accessed 2026-06-19.  
https://docs.tecton.ai/docs/reading-feature-data/reading-feature-data-for-training/constructing-training-data

[3] **Simha, N.** (2023). "Chronon — A Declarative Feature Engineering Framework." *Airbnb Engineering Blog.*  
Accessed 2026-06-19.  
https://medium.com/airbnb-engineering/chronon-a-declarative-feature-engineering-framework-b7b8ce796e04

[4] **Hyndman, R. J., & Athanasopoulos, G.** (2021). *Forecasting: Principles and Practice.* 3rd ed. OTexts.  
https://otexts.com/fpp3/

[5] **Ben Taieb, S., Bontempi, G., Atiya, A. F., & Sorjamaa, A.** (2012). "A review and comparison of strategies for multi-step ahead time series forecasting based on the NN5 forecasting competition." *Expert Systems with Applications*, 39(8), 7067–7083.  
https://doi.org/10.1016/j.eswa.2012.01.039

[6] **Salinas, D., Flunkert, V., Gasthaus, J., & Januschowski, T.** (2020). "DeepAR: Probabilistic forecasting with autoregressive recurrent networks." *International Journal of Forecasting*, 36(3), 1181–1191.  
https://doi.org/10.1016/j.ijforecast.2019.07.001

[7] **Lim, B., Arık, S. Ö., Loeff, N., & Pfister, T.** (2021). "Temporal Fusion Transformers for interpretable multi-horizon time series forecasting." *International Journal of Forecasting*, 37(4), 1748–1764.  
https://doi.org/10.1016/j.ijforecast.2021.03.012

[8] **Bontempi, G., Ben Taieb, S., & Le Borgne, Y.-A.** (2013). "Machine Learning Strategies for Time Series Forecasting." In *Business Intelligence* (pp. 62–77). Springer.  
https://doi.org/10.1007/978-3-642-36318-4_3

[9] **Kaufman, S., Rosset, S., Perlich, C., & Stitelman, O.** (2012). "Leakage in data mining: Formulation, detection, and avoidance." *ACM Transactions on Knowledge Discovery from Data*, 6(4), Article 15, 21 pages.  
https://doi.org/10.1145/2382577.2382579

[10] **Snodgrass, R. T., & Ahn, I.** (1985). "A taxonomy of time in databases." *Proceedings of the 1985 ACM SIGMOD International Conference on Management of Data*, 236–246.  
https://doi.org/10.1145/22733.22745

[11] **Jensen, C. S., et al.** (1998). "The consensus glossary of temporal database concepts — February 1998 version." In *Temporal Databases: Research and Practice* (pp. 367–405). Springer.  
https://doi.org/10.1007/BFb0053710

---

## License

Licensed under the [Apache License 2.0](LICENSE).

---

## Citation

```bibtex
@misc{vri-framework,
  author       = {Fan Zhang},
  title        = {VRI: A Governance and Auditing Convention for Production Forecasting},
  year         = {2026},
  howpublished = {\url{https://github.com/roguewatt/vri-framework}}}
