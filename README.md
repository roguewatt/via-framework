# VIA Framework

**Valid / Issue / Sample As-of Time — A Governance and Auditing Convention for Production Forecasting**

## Purpose

The VIA Framework provides a structured convention for designing, constructing, and auditing production forecasting systems.

It separates two levels:

- **Run Design**: how a training, backtest, evaluation, or inference run is organised.
- **Sample Design**: how each forecasting sample is temporally governed and represented.

The three core VIA concepts are:

- **Valid Time**
- **Issue Time**
- **Sample As-of Time**

The framework also uses **Cutoff** as a separate run-level control.

VIA does not introduce a new forecasting algorithm, model architecture, or temporal theory. VIA uses established ideas from forecasting, temporal databases, point-in-time feature retrieval, and real-time data vintages. Its contribution is to organise these ideas into a production forecasting governance convention with explicit run-level, sample-level, record-version-level, and audit-level controls.

## Framework Structure

| Level | Component | Purpose |
| :--- | :--- | :--- |
| Run Design | Forecasting Strategy | Defines whether forecasts are generated directly or recursively |
| Run Design | Inference Organisation | Defines whether a run processes one or multiple forecasting samples |
| Run Design | Cutoff | Defines the point-in-time visibility boundary of the run |
| Run Design | Horizon Coverage | Defines the forecast periods and Valid Times that must be covered |
| Sample Design | VIA | Defines the temporal meaning and information eligibility of each sample |
| Sample Design | Forecast Sample | Defines the input and output structure of a sample |
| Sample Design | Forecast Period Mapping | Relates Sample As-of Time to output Valid Time |
| Sample Design | I/O Schema | Defines the logical input and output structure presented to a model |

---

# Run Design

## Forecasting Strategy

Forecasting Strategy defines how forecasts are generated.

- **Direct forecasting**: forecasts are generated directly from eligible inputs without using earlier predictions as inputs for later forecasts.
- **Recursive forecasting**: earlier predictions are used as inputs when generating later forecasts.

Forecasting Strategy is independent of the number of Forecast Periods, forecasting samples, and model outputs.

## Inference Organisation

Inference Organisation defines how many forecasting samples are processed.
- **Single inference**: one forecasting sample is processed to produce one set of outputs.
- **Multiple inferences**: multiple forecasting samples are processed, each with its own Sample As-of Time.

Multiple inferences may be processed individually or together in a batch. The number of inferences is independent of the number of Forecast Periods:

- one inference may produce outputs for one or multiple Forecast Periods;
- an evaluation may contain multiple inferences;
- each inference may use a single-output or multi-output structure.

## Cutoff

Cutoff is the global point-in-time boundary applied to one dataset construction, training run, backtest, evaluation, or inference job.

A run normally has one Cutoff, while the samples within that run may have many Sample As-of Times.

Cutoff constrains:

- which record versions are visible to the run;
- which samples may be constructed;
- whether required labels are available;
- where a training or evaluation dataset must end.

Run-level visibility requires:

$$
\text{Record Issue Time} \leq \text{Cutoff}
$$

Visibility by the Cutoff does not automatically make a record eligible for every sample.

Sample-level eligibility still requires:

$$
\text{Input Issue Time} \leq \text{Sample As-of Time}
$$

Therefore:

- **Cutoff** governs the overall point-in-time boundary of the run.
- **Sample As-of Time** governs the information state of an individual sample.

## Horizon Coverage

Horizon Coverage defines how far into the future a forecasting run must produce predictions.

For an output with Sample As-of Time $\tau$ and Valid Time $u$, the corresponding Forecast Period is:

$$
p = u - \tau
$$

A run may cover one Forecast Period or multiple Forecast Periods. The required Valid Times follow from the Sample As-of Time and the applicable Forecast Periods.

Terms such as `H0`, `H1`, and `H2` may be used by a model or application as output labels. In VIA, these labels do not replace the underlying temporal definitions: Sample As-of Time, Valid Time, and Forecast Period.

Horizon Coverage is independent of:

- forecasting strategy;
- inference organisation;
- model I/O schema.

---

# Sample Design

## Core VIA Concepts

### Record

A record is a data value or prediction value used by a forecasting system. In VIA, a record has temporal meaning: it describes a Valid Time and is issued at an Issue Time. A record may serve as an input, a target, or a prediction, depending on how it is used by a forecasting sample or run.

Each revision or forecast release is treated as a separate record version with its own Issue Time.

### Valid Time

Valid Time is the business-defined time point or interval that a record or prediction describes.

It is a property of the modelled reality, independent of when the record was created, collected, or issued.

- For half-hourly load covering 08:00–08:30, the Valid Time may be 08:00.
- For a weather forecast for tomorrow, the Valid Time is the future time described by the forecast.

### Issue Time

Issue Time is the time at which a specific record, prediction, or record version is issued by its source.

The precise meaning of issuance follows the source definition for that data product. Each revision, correction, or forecast release is treated as a separate record version with its own Issue Time.

| Type | Valid Time | Issue Time | Description |
| :--- | :--- | :--- | :--- |
| Outturn | 2026-06-18 08:00 | 2026-06-19 02:00 | Issued the following day |
| Outturn revision | 2026-06-18 08:00 | 2026-06-19 10:00 | Revised version issued later |
| Forecast | 2026-06-20 08:00 | 2026-06-19 08:15 | Forecast issued the previous day |

### Sample As-of Time

Sample As-of Time is the information-state anchor of a forecasting sample.

For the input side of the sample, it defines the information boundary: every input record used by the sample must have been issued by that time.

$$
\text{Input Issue Time} \leq \text{Sample As-of Time}
$$

For the output side, Sample As-of Time is the reference point from which a Forecast Period may be expressed. The output itself is defined by its Valid Time and Target.

Each training, validation, test, or live sample has one Sample As-of Time.

### Output

A forecast output is defined by:

$$
\text{Output} = (\text{Valid Time}, \text{Target})
$$

Valid Time states **when** the output applies. Target states **what** is being forecast.

Examples include:

- `(2026-06-19 09:00, Load)`
- `(2026-06-19 09:00, Price)`
- `(2026-06-19 09:30, Load)`

Multiple outputs may share the same Valid Time, and the same Target may appear at multiple Valid Times.

### Forecast Sample

For a single output, let:

- $\tau$ be the Sample As-of Time;
- $u$ be the output Valid Time;
- $X_{\tau,u}$ be the inputs used by the sample;
- $Y_u$ be the Target at Valid Time $u$.

The forecast sample is:

$$
S_{\tau,u} = \left(X_{\tau,u}, Y_u\right)
$$

For multiple outputs, the Valid Time becomes a vector:

$$
\mathbf{u} = (u_1, \ldots, u_m)
$$

and the corresponding Target vector is:

$$
\mathbf{Y}_{\mathbf{u}} = \left(Y^{(1)}_{u_1}, \ldots, Y^{(m)}_{u_m}\right)
$$

so the sample becomes:

$$
S_{\tau,\mathbf{u}} = \left(X_{\tau,\mathbf{u}}, \mathbf{Y}_{\mathbf{u}}\right)
$$

The single-output form is the special case $m=1$.

The roles of $\tau$ and $\mathbf{u}$ are different:

- $\tau$ defines the information boundary for the inputs;
- $\mathbf{u}$ contains the Valid Times of the outputs.

Let $\mathcal{F}_\tau$ denote the information available by Sample As-of Time $\tau$. The VIA information-admissibility condition is:

$$
\sigma(X_{\tau,\mathbf{u}}) \subseteq \mathcal{F}_\tau
$$

The condition applies only to the input side of the sample. It does not require the future Targets to be known at $\tau$.

Operationally, this is enforced by:

$$
\text{Input Issue Time} \leq \text{Sample As-of Time}
$$

### Forecast Period Mapping

Forecast Period is the time interval between Sample As-of Time and an output's Valid Time.

For one output:

$$
p = u - \tau
$$

or equivalently:

$$
\text{Valid Time} = \text{Sample As-of Time} + \text{Forecast Period}
$$

For multiple outputs:

$$
\mathbf{p} = \mathbf{u} - \tau
$$

with the subtraction applied component-wise.

For a regular half-hourly forecast with Sample As-of Time `08:30`:

| Output Label | Forecast Period | Valid Time |
| :--- | :--- | :--- |
| H0 | 0 minutes | 08:30 |
| H1 | 30 minutes | 09:00 |
| H2 | 60 minutes | 09:30 |

`H0`, `H1`, and `H2` are output labels in this example. The temporal meaning is carried by the Forecast Period and Valid Time.

## I/O Schema

SISO, SIMO, MISO, and MIMO describe the logical structure presented to a model:

- **SISO**: single input, single output
- **SIMO**: single input, multiple outputs
- **MISO**: multiple inputs, single output
- **MIMO**: multiple inputs, multiple outputs

Inputs and outputs may represent business variables, target series, covariates, or separately constructed model dimensions.

A forecast covering multiple Valid Times may therefore appear as multiple output dimensions even when all outputs belong to the same Target series.

Forecast-period coverage and I/O schema describe different aspects of the model:

- **Forecast-period coverage** describes which future Valid Times are predicted.
- **I/O schema** describes the model's input and output dimensions.

---

# Training

Training samples follow the same input eligibility rule as inference.

For each sample:

- The sample is assigned a Sample As-of Time representing the historical information state being reconstructed.
- Every input record used by the sample satisfies $\text{Issue Time} \leq \text{Sample As-of Time}$.
- Labels are realised outcomes associated with the required Valid Times.
- Labels may be issued after the Sample As-of Time and attached later for model fitting.
- The model architecture, such as XGBoost, MLP, TCN, or LSTM, is independent of VIA.

## Label Completeness and Past-Covariate Availability
A training sample may be included only when all required labels are available by the dataset Cutoff: 

$$
\text{Required Label Issue Time} \leq \text{Cutoff}
$$

This is a label-completeness rule, not an input eligibility rule. Label Issue Times may be later than the Sample As-of Time.

An observation is not automatically eligible because its Valid Time is earlier than the Sample As-of Time. It must still satisfy:

$$
\text{Issue Time} \leq \text{Sample As-of Time}
$$

For each input series, the input-selection policy must define:

- which eligible record version is selected;
- any required safety lag;
- whether an eligible forecast or nowcast replaces an unavailable outturn;
- how missing values are handled;
- whether the sample is excluded when historical availability cannot be reproduced reliably.

Historical samples must reproduce the source-specific availability state that existed at each Sample As-of Time. Later publications, revisions, corrections, and backfills are ineligible unless they were already available at that time.

For a regular half-hourly schedule where labels are issued immediately at their Valid Times:

$$
\text{Latest Training Sample As-of Time} = \text{Cutoff} - \text{Maximum Forecast Period}
$$

This is a special case. For delayed labels or irregular forecast-period mappings, use the actual label-completeness rule:

$$
\text{Required Label Issue Time} \leq \text{Cutoff}
$$

---

# Testing & Inference

Testing follows the same input eligibility rule as training. A model may be evaluated through historical inference or used for live prediction.

- Each test or live sample has one Sample As-of Time.
- Only input records satisfying $\text{Issue Time} \leq \text{Sample As-of Time}$ are eligible.
- Future covariates are allowed when their record versions were issued by the Sample As-of Time.
- Future or unavailable realised target values must not be used as model inputs.
- Realised targets are attached only for scoring after predictions have been generated.
- For a fixed-model backtest, the completed model is frozen before the first test sample.
- No test-period samples or labels enter fitting for that fixed model.

---

# I/O Schema Examples

The examples below describe the input and output structures presented to a forecasting model.

- In a **tabular** setup, each sample is represented as one row containing its inputs and outputs. The row has one Sample As-of Time, while individual values may describe different Valid Times.
- In a **sequential** setup, each sample contains an ordered input sequence and one or more outputs. The sample has one Sample As-of Time, while each sequence position and output has its own Valid Time.

Any input used to construct a sample must satisfy: $\text{Issue Time} \leq \text{Sample As-of Time}$

In the tabular examples, `t` denotes the Sample As-of Time on the underlying time axis. Expressions such as `t-1` and `t+1` identify the Valid Times of individual input and output values relative to that Sample As-of Time.

## SISO

### SISO Example 1: Tabular Single-Horizon Autoregressive Forecast

A tabular model uses one historical load value to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SISO | Load at `t-1` | Load at `t+1` |

Training samples may look like:

| Sample As-of Time | Load `t-1` | Load `t+1` |
| :--- | ---: | ---: |
| 2026-06-19 07:00 | 24,100 MW | 24,900 MW |
| 2026-06-19 07:30 | 24,450 MW | 25,300 MW |
| 2026-06-19 08:00 | 24,900 MW | 25,650 MW |

A test input sample may look like:

| Sample As-of Time | Load `t-1` |
| :--- | ---: |
| 2026-06-19 08:30 | 25,300 MW |

After inference:

| Sample As-of Time | Valid Time | Load Forecast |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 09:00 | 25,650 MW |

### SISO Example 2: Tabular Forecast from One Future Covariate

A tabular model uses one temperature forecast value to predict one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SISO | Temperature forecast | GB Zone A load |

A training sample may look like:

| Sample As-of Time | Temperature Forecast | Load |
| :--- | ---: | ---: |
| 2026-06-19 08:00 | 18.1°C | 25,650 MW |

The input and output describe the following Valid Times:

| Sample As-of Time | Item | Valid Time | Issue Time |
| :--- | :--- | :--- | :--- |
| 2026-06-19 08:00 | Temperature forecast | 2026-06-19 08:30 | 2026-06-19 07:40 |
| 2026-06-19 08:00 | Load label | 2026-06-19 08:30 | 2026-06-19 08:35 |

The temperature forecast is eligible because:

$$
\text{2026-06-19 07:40} \leq \text{2026-06-19 08:00}
$$

Its Valid Time is later than the Sample As-of Time because it is a future covariate.

### SISO Example 3: Sequential Single-Horizon Forecast

A sequential model uses one historical load sequence to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SISO | Historical load sequence | Future load |

A training sample contains one ordered input sequence:

| Sample As-of Time | Valid Time | Historical Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 06:00 | 23,800 MW |
|                  | 2026-06-19 06:30 | 24,100 MW |
|                  | 2026-06-19 07:00 | 24,450 MW |
|                  | 2026-06-19 07:30 | 24,900 MW |

The corresponding output is:

| Sample As-of Time | Valid Time | Future Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 08:30 | 25,300 MW |

A test input sample may contain:

| Sample As-of Time | Valid Time | Historical Load |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 06:30 | 24,100 MW |
|                  | 2026-06-19 07:00 | 24,450 MW |
|                  | 2026-06-19 07:30 | 24,900 MW |
|                  | 2026-06-19 08:00 | 25,300 MW |

After inference:

| Sample As-of Time | Valid Time | Load Forecast |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 09:00 | 25,650 MW |

This is SISO because one ordered load sequence produces one future load value.

## SIMO

### SIMO Example 1: Tabular Multi-Horizon Forecast

A tabular model uses one temperature forecast value to jointly forecast load across three horizons.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SIMO | Temperature forecast | Load H1; Load H2; Load H3 |

Training samples may look like:

| Sample As-of Time | Temperature Forecast | Load H1 | Load H2 | Load H3 |
| :--- | ---: | ---: | ---: | ---: |
| 2026-06-19 07:00 | 17.2°C | 24,900 MW | 25,300 MW | 25,650 MW |
| 2026-06-19 07:30 | 17.6°C | 25,300 MW | 25,650 MW | 25,900 MW |
| 2026-06-19 08:00 | 18.1°C | 25,650 MW | 25,900 MW | 26,100 MW |

A test input sample may look like:

| Sample As-of Time | Temperature Forecast |
| :--- | ---: |
| 2026-06-19 08:30 | 18.5°C |

After inference:

| Sample As-of Time | Load H1 | Load H2 | Load H3 |
| :--- | ---: | ---: | ---: |
| 2026-06-19 08:30 | 25,850 MW | 26,050 MW | 26,200 MW |

Using the Forecast Period Mapping, the outputs resolve to:

| Sample As-of Time | Output Label | Forecast Period | Valid Time | Load Forecast |
| :--- | :--- | :--- | :--- | ---: |
| 2026-06-19 08:30 | H1 | 30 minutes | 2026-06-19 09:00 | 25,850 MW |
| 2026-06-19 08:30 | H2 | 60 minutes | 2026-06-19 09:30 | 26,050 MW |
| 2026-06-19 08:30 | H3 | 90 minutes | 2026-06-19 10:00 | 26,200 MW |

This is SIMO because one tabular input value produces multiple output values.

### SIMO Example 2: Sequential Multi-Horizon Forecast

A sequential model uses one historical load sequence to jointly forecast three future load values.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SIMO | Historical load sequence | Load H1; Load H2; Load H3 |

A training sample may look like:

**Input**

| Sample As-of Time | Valid Time | Historical Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 06:00 | 23,800 MW |
|                  | 2026-06-19 06:30 | 24,100 MW |
|                  | 2026-06-19 07:00 | 24,450 MW |
|                  | 2026-06-19 07:30 | 24,900 MW |

**Output**

| Sample As-of Time | Output Label | Forecast Period | Valid Time | Load |
| :--- | :--- | :--- | :--- | ---: |
| 2026-06-19 08:00 | H1 | 30 minutes | 2026-06-19 08:30 | 25,300 MW |
|                  | H2 | 60 minutes | 2026-06-19 09:00 | 25,650 MW |
|                  | H3 | 90 minutes | 2026-06-19 09:30 | 25,900 MW |

This is SIMO because one ordered load sequence produces multiple future load values.

### SIMO Example 3: One Input and Two Output Variables

A tabular model uses one solar-radiation forecast value to jointly predict two solar-generation variables.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| SIMO | Solar-radiation forecast | Grid-connected solar; embedded solar |

A training sample may look like:

| Sample As-of Time | Solar-Radiation Forecast | Grid-Connected Solar | Embedded Solar |
| :--- | ---: | ---: | ---: |
| 2026-06-19 07:00 | 410 W/m² | 3,250 MW | 1,180 MW |
| 2026-06-19 07:30 | 380 W/m² | 3,350 MW | 1,070 MW |
| 2026-06-19 08:00 | 220 W/m² | 3,270 MW | 1,210 MW |

The input and outputs describe the following Valid Times:

| Sample As-of Time | Item | Valid Time |
| :--- | :--- | :--- |
| 2026-06-19 08:00 | Solar-radiation forecast | 2026-06-19 08:30 |
| 2026-06-19 08:00 | Grid-connected solar | 2026-06-19 08:30 |
| 2026-06-19 08:00 | Embedded solar | 2026-06-19 08:30 |

This is SIMO because one input value produces two output variables.

## MISO

### MISO Example 1: Tabular Single-Horizon Load Forecast

A tabular model uses several input values to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MISO | Load at `t-1`; temperature forecast at `t+1`; wind forecast at `t+1` | Load at `t+1` |

Training samples may look like:

| Sample As-of Time | Load `t-1` | Temperature Forecast `t+1` | Wind Forecast `t+1` | Load `t+1` |
| :--- | ---: | ---: | ---: | ---: |
| 2026-06-19 07:00 | 24,100 MW | 17.2°C | 8,400 MW | 24,900 MW |
| 2026-06-19 07:30 | 24,450 MW | 17.6°C | 8,250 MW | 25,300 MW |
| 2026-06-19 08:00 | 24,900 MW | 18.1°C | 8,100 MW | 25,650 MW |

A test input sample may look like:

| Sample As-of Time | Load `t-1` | Temperature Forecast `t+1` | Wind Forecast `t+1` |
| :--- | ---: | ---: | ---: |
| 2026-06-19 08:30 | 25,300 MW | 18.5°C | 7,950 MW |

After inference:

| Sample As-of Time | Valid Time | Load Forecast |
| :--- | :--- | ---: |
| 2026-06-19 08:30 | 2026-06-19 09:00 | 25,850 MW |

This is MISO because multiple tabular input values produce one output value.

### MISO Example 2: Sequential Multi-Input Forecast

A sequential model uses a multivariate sequence of load, temperature, and wind values to forecast one future load value.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MISO | Load, temperature, and wind feature channels | Future load |

One training sample may look like:

| Sample As-of Time | Valid Time | Load | Temperature | Wind |
| :--- | :--- | ---: | ---: | ---: |
| 2026-06-19 08:00 | 2026-06-19 06:30 | 24,100 MW | 17.0°C | 8,600 MW |
|                  | 2026-06-19 07:00 | 24,450 MW | 17.2°C | 8,400 MW |
|                  | 2026-06-19 07:30 | 24,900 MW | 17.6°C | 8,250 MW |

The corresponding output is:

| Sample As-of Time | Valid Time | Future Load |
| :--- | :--- | ---: |
| 2026-06-19 08:00 | 2026-06-19 08:30 | 25,300 MW |

This is MISO because the sequential input contains multiple feature channels while the model produces one output value.

### MISO Example 3: Inputs with Different Availability States

A tabular model is configured with historical load, a temperature forecast, and a wind forecast.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MISO | Historical load; temperature forecast; wind forecast | Future load |

For a sample with Sample As-of Time `2026-06-19 08:00`:

| Input | Valid Time | Issue Time |
| :--- | :--- | :--- |
| Historical load | 2026-06-19 07:30 | 2026-06-19 07:55 |
| Temperature forecast | 2026-06-19 08:30 | 2026-06-19 07:40 |
| Wind forecast | 2026-06-19 08:30 | 2026-06-19 08:05 |

The historical load is eligible because:

$$
\text{2026-06-19 07:55} \leq \text{2026-06-19 08:00}
$$

The temperature forecast is eligible because:

$$
\text{2026-06-19 07:40} \leq \text{2026-06-19 08:00}
$$

The wind forecast issued at 08:05 is not eligible because:

$$
\text{2026-06-19 08:05} > \text{2026-06-19 08:00}
$$

An earlier eligible wind-forecast version must be used. If no eligible version exists, the sample must follow the model's defined missing-input policy or be excluded.

## MIMO

### MIMO Example 1: Tabular Multi-Input, Multi-Horizon Forecast

A tabular model uses solar-radiation and cloud-cover forecast values to jointly forecast solar generation across three horizons.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MIMO | Solar-radiation forecast; cloud-cover forecast | Solar H0; Solar H1; Solar H2 |

Training samples may look like:

| Sample As-of Time | Solar-Radiation Forecast | Cloud-Cover Forecast | Solar H0 | Solar H1 | Solar H2 |
| :--- | ---: | ---: | ---: | ---: | ---: |
| 2026-06-19 08:00 | 320 W/m² | 65% | 110 MW | 145 MW | 180 MW |
| 2026-06-19 08:30 | 410 W/m² | 48% | 150 MW | 190 MW | 225 MW |

A test input sample may look like:

| Sample As-of Time | Solar-Radiation Forecast | Cloud-Cover Forecast |
| :--- | ---: | ---: |
| 2026-06-19 09:00 | 470 W/m² | 40% |

After inference:

| Sample As-of Time | Output Label | Forecast Period | Valid Time | Solar Forecast |
| :--- | :--- | :--- | :--- | ---: |
| 2026-06-19 09:00 | H0 | 0 minutes | 2026-06-19 09:00 | 185 MW |
| 2026-06-19 09:00 | H1 | 30 minutes | 2026-06-19 09:30 | 220 MW |
| 2026-06-19 09:00 | H2 | 60 minutes | 2026-06-19 10:00 | 250 MW |

This is MIMO because multiple tabular input values produce multiple output values.

### MIMO Example 2: Multiple Inputs and Multiple Output Variables

A tabular model uses several input values to jointly predict future load and future price.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MIMO | Load at `t-1`; price at `t-1`; temperature forecast at `t+1`; wind forecast at `t+1` | Load at `t+1`; price at `t+1` |

A training sample may look like:

| Sample As-of Time | Load `t-1` | Price `t-1` | Temperature Forecast `t+1` | Wind Forecast `t+1` | Load `t+1` | Price `t+1` |
| :--- | ---: | ---: | ---: | ---: | ---: | ---: |
| 2026-06-19 08:00 | 24,900 MW | £72/MWh | 18.1°C | 8,100 MW | 25,650 MW | £75/MWh |

The two outputs share the same Valid Time:

| Sample As-of Time | Output | Valid Time |
| :--- | :--- | :--- |
| 2026-06-19 08:00 | Load `t+1` | 2026-06-19 08:30 |
| 2026-06-19 08:00 | Price `t+1` | 2026-06-19 08:30 |

This is MIMO because multiple tabular input values produce multiple output variables.

### MIMO Example 3: Multiple Zones and Multiple Horizons

A tabular model jointly forecasts load for Zone A and Zone B across two future days.

| I/O Schema | Inputs | Outputs |
| :--- | :--- | :--- |
| MIMO | Zone A load; Zone B load; Zone A weather H1 and H2; Zone B weather H1 and H2 | Zone A H1; Zone A H2; Zone B H1; Zone B H2 |

A test input sample may look like:

| Sample As-of Time | Zone A Load | Zone B Load | Zone A Weather H1 | Zone A Weather H2 | Zone B Weather H1 | Zone B Weather H2 |
| :--- | ---: | ---: | :--- | :--- | :--- | :--- |
| 2026-06-19 03:00 | 12,400 MW | 9,800 MW | Mild | Warm | Cool | Mild |

After inference:

| Sample As-of Time | Output | Valid Time | Forecast |
| :--- | :--- | :--- | ---: |
| 2026-06-19 03:00 | Zone A H1 | 2026-06-20 | 12,750 MW |
| 2026-06-19 03:00 | Zone A H2 | 2026-06-21 | 12,900 MW |
| 2026-06-19 03:00 | Zone B H1 | 2026-06-20 | 10,050 MW |
| 2026-06-19 03:00 | Zone B H2 | 2026-06-21 | 10,200 MW |

This is MIMO because multiple input values produce multiple output values.

## Additional Framework Examples

### Example: Batched Historical Inference

A backtest evaluates the model at several historical Sample As-of Times.

| Sample As-of Time | Valid Time |
| :--- | :--- |
| 2026-06-01 08:00 | 2026-06-01 08:30 |
| 2026-06-01 08:30 | 2026-06-01 09:00 |
| 2026-06-01 09:00 | 2026-06-01 09:30 |

Each row is a separate logical sample with its own Sample As-of Time.

The samples may be processed together in one batch. Batching does not merge their Sample As-of Times or change the model's I/O schema.

### Example: Cutoff and Label Completeness

A training dataset is constructed with:

$$
\text{Cutoff} = \text{2026-06-19 08:30}
$$

For every input used by a sample:

$$
\text{Issue Time} \leq \text{Sample As-of Time}
$$

For every required label:

$$
\text{Label Issue Time} \leq \text{Cutoff}
$$

| Sample As-of Time | Label Valid Time | Label Issue Time | Cutoff |
| :--- | :--- | :--- | :--- |
| 2026-06-19 07:30 | 2026-06-19 08:00 | 2026-06-19 08:05 | 2026-06-19 08:30 |
| 2026-06-19 07:30 | 2026-06-19 08:30 | 2026-06-19 09:00 | 2026-06-19 08:30 |

The first label is available by the Cutoff.

The second label is not available by the Cutoff. If both labels are required, the sample is not label-complete and must be excluded.

For a regular half-hourly schedule where labels are available immediately at their Valid Times:

$$
\text{Latest Training Sample As-of Time} = \text{Cutoff} - \text{Maximum Forecast Period}
$$

This is a special case. When labels are issued later or the Forecast Period Mapping is irregular, completeness must be evaluated using the actual Label Issue Times.

### Example: Decision Time Is Outside VIA

A forecast is issued before a later business decision.

| Sample As-of Time | Valid Time | Issue Time | Decision Time |
| :--- | :--- | :--- | :--- |
| 2026-06-19 08:00 | 2026-06-19 08:30 | 2026-06-19 08:15 | 2026-06-19 10:00 |

VIA governs the temporal semantics of the inputs, samples, and predictions.

The later business Decision Time is outside VIA.

---

# Auditing

A VIA-compliant system should be able to reconstruct, for each forecasting sample and prediction:

- Sample As-of Time
- Outputs, identified by Target and Valid Time
- Forecast Periods, where used
- Applicable Forecast Period Mapping
- Input record identifiers with their Valid Times and Issue Times
- Selected record versions and the deterministic rule used to select them
- Output record identifiers with their Valid Times and Issue Times
- Model version
- Inference-run identifier
- Inference-run Cutoff

For each model-training run, the system should additionally record:

- Model-training run identifier
- Model-training Cutoff
- Training-window boundaries
- Feature-definition version
- Feature-selection procedure or version
- Hyperparameter or configuration version
- Model-selection or promotion rule
- Retraining policy
- Source-data version
- Code or model-artifact version

For each training dataset, the system should additionally record:

- Sample As-of Times
- Labels with their Valid Times and Issue Times
- Dataset Cutoff
- Training, validation, and test periods
- Target-buffer or label-completeness rule

---

# What VIA Is Not

VIA is not:
- a forecasting algorithm;
- a temporal database theory;
- a leakage detection theory;
- a replacement for feature stores;
- a replacement for forecasting methodology.

VIA is the temporal governance and auditing convention centred on Valid Time, Sample As-of Time, and Issue Time.

The wider document describes how VIA interacts with adjacent production controls such as Cutoff, forecasting strategy, inference organisation, Forecast Period coverage, and I/O schema.

---

# References

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

# License

Licensed under the [Apache License 2.0](LICENSE).

---

# Citation

```bibtex
@misc{VIA-framework,
  author       = {Fan Zhang},
  title        = {VIA: A Governance and Auditing Convention for Production Forecasting},
  year         = {2026},
  howpublished = {\url{https://github.com/roguewatt/via-framework}}}
```
