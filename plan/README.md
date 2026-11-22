# RimbaGuard

## Adaptive AIoT Early Warning for Forest-Edge Fire Risk

Proposed project for the BINUS Robotics Cup 2026 AIoT Innovation Competition.

Chosen subtheme:

> AIoT untuk Ketahanan Lingkungan & Mitigasi Iklim

Project status: concept and validation design. No performance results are claimed yet.

## 1. Core idea

RimbaGuard estimates whether a defined forest-edge area will experience a satellite-detected active fire during the next 24 hours. It combines local environmental sensors, weather, terrain, vegetation, and historical fire observations.

The prediction supports a concrete operational decision:

- prioritize ranger or community patrols;
- increase sensor sampling in high-risk areas;
- activate a local warning beacon after risk and local anomaly confirmation;
- record the ranger's confirmation as feedback for later model improvement.

The system must not claim to predict every forest fire. Its initial target is a measurable, bounded event:

> Fire detected by the selected satellite-derived label within the next 24 hours in the selected pilot grid.

## 2. Why this fits the competition

The guidebook requires functional integration between AI and IoT. AI must perform intelligent processing, prediction, analytics, or data-based decision-making. IoT must provide sensing, connectivity, and/or actuation.[1]

RimbaGuard provides the complete loop:

```text
Environmental sensors
        |
        v
Connectivity and gateway
        |
        v
Feature processing and AI risk model
        |
        v
Risk probability and explanation
        |
        v
Patrol priority / warning beacon / alert
        |
        v
Ranger confirmation and system feedback
```

This is stronger than a dashboard because the model changes sensing and field actions.

## 3. Target user and decision

### Primary user

A forest ranger, village fire-response team, land manager, or conservation officer.

### Operational problem

Patrol resources are limited. Teams need to decide which areas deserve attention before a fire becomes difficult to control.

### Decision produced by RimbaGuard

```text
Which grid should be checked first during the next patrol cycle?
```

### Proposed prediction contract

| Item | Initial proposal |
|---|---|
| Pilot area | One defined forest-edge or peatland region; TBD |
| Spatial unit | 1 km² grid; proposed starting point |
| Forecast horizon | Next 24 hours |
| Prediction event | Satellite-detected active fire in the grid |
| Update frequency | Hourly or when new sensor data arrives |
| Output | Probability, risk level, explanation, data-health status |
| Action | Patrol priority, increased sampling, local alert |
| Human authority | Ranger confirms the event; system does not replace emergency services |

The team must freeze these values before model training.

## 4. Data design

Use a small, defensible feature set first. Add data only when it improves validation or the field decision.

| Data | Role | Type |
|---|---|---|
| Temperature | Detect heat and drying trend | Local IoT sensor |
| Relative humidity | Estimate atmospheric dryness | Local IoT sensor |
| Soil moisture | Estimate local ground dryness | Capacitive soil-moisture sensor |
| Rainfall accumulation | Capture recent drought or wetness | Weather data |
| Wind speed and direction | Estimate spread-supporting conditions | Weather data |
| Elevation and slope | Represent terrain context | Static geospatial data |
| Land cover and vegetation dryness | Represent available fuel and vegetation condition | Satellite/geospatial data |
| Nearby historical fire activity | Represent spatial and temporal fire pattern | NASA FIRMS archive |
| Ranger confirmation | Create operational feedback labels | Human-in-the-loop record |

### Important sensor limitation

Soil moisture is a proxy for local ground dryness. It is not automatically equivalent to forest-fuel moisture. The prototype must document sensor placement, calibration, sampling interval, missing values, and environmental limitations.

## 5. NASA FIRMS and VIIRS usage

NASA's Fire Information for Resource Management System (FIRMS) provides access to active-fire products from MODIS, VIIRS, and Landsat.[3][4]

Use FIRMS in three ways:

1. Historical label: determine whether a fire was detected during the future prediction window.
2. Historical feature: count nearby earlier detections using only data available before the prediction time.
3. Demonstration replay: replay a historical event through the complete AIoT pipeline.

### Prevent target leakage

Never use a future fire detection as an input for predicting that same event.

Correct temporal structure:

```text
Feature cutoff: 10:00
        |
        v
Prediction: fire risk from 10:00 today to 10:00 tomorrow
        |
        v
Label: whether FIRMS detects fire during that future window
```

FIRMS detects satellite-observed active fires. It does not represent every ground fire. Near-real-time products are suitable for a live-style demonstration; research validation should document the selected product and data version. NASA also advises users performing scientific analysis to use standard science-quality data when available.[3]

## 6. AI approach

### Baselines

The model must beat simple alternatives, not only produce a high standalone accuracy number.

- B0: historical fire-rate baseline.
- B1: transparent dryness rule using temperature, humidity, rainfall, and soil moisture.
- B2: logistic regression using weather, terrain, and land-cover features.
- B3: gradient-boosted trees using the same features.
- B4: best model plus local IoT sensors and historical FIRMS features.

### Main candidate model

Start with logistic regression as the interpretable baseline. Test a gradient-boosted tree model only if it improves the user-relevant metrics.

The system should display:

- risk probability;
- risk category;
- leading contributing factors;
- confidence or uncertainty status;
- freshness and health of each sensor/data source.

### Model decision rule

Do not optimize only for accuracy. Fire events will likely be imbalanced, and false alarms consume patrol resources.

Primary evaluation should include:

- precision-recall AUC;
- recall at a declared false-alarm rate;
- Brier score or calibration error;
- warning lead time;
- alert latency;
- sensor uptime and packet-loss tolerance.

The team must select one primary metric before reporting results.

## 7. Validation plan

### Data split

Use a chronological holdout. Add a geographic holdout if enough regions are available.

Do not rely only on a random split. Random splitting can place observations from the same event or nearby time period in both training and test sets.

### Required comparisons

| Question | Experiment |
|---|---|
| Does AI improve the simple rule? | B1 vs. best learned model |
| Do local sensors add value? | B2/B3 vs. B4 |
| Do historical fire features add value? | Ablation with and without FIRMS history |
| Does the model generalize spatially? | Geographic holdout |
| Does the system tolerate field problems? | Missing sensor, stale data, packet loss, network outage |
| Is the output operationally useful? | Recall and false alarms at the patrol capacity limit |

### Pass condition

The selected fusion model must outperform the strongest simple baseline at the same declared false-alarm budget. If it does not, use the simpler model and report the result honestly.

## 8. IoT prototype

### Minimum hardware

- microcontroller;
- temperature and humidity sensor;
- capacitive soil-moisture sensor;
- connectivity module or gateway;
- LED or buzzer for local warning;
- optional smoke, PM, or CO sensor for local confirmation.

### Required telemetry

Each message should include:

```json
{
  "device_id": "node-01",
  "timestamp": "ISO-8601 timestamp",
  "latitude": "pilot coordinate",
  "longitude": "pilot coordinate",
  "temperature_c": 0.0,
  "humidity_percent": 0.0,
  "soil_moisture_percent": 0.0,
  "battery_voltage": 0.0,
  "sequence": 0
}
```

Use actual measured values in the final prototype. The structure above is an interface proposal, not experimental data.

### Fault handling

| Failure | Required behavior |
|---|---|
| Network loss | Buffer data locally; show connection failure |
| Stale sensor data | Mark the feature stale; lower confidence or use fallback |
| Sensor disconnected | Show sensor fault; do not silently substitute zero |
| Model unavailable | Use the declared baseline rule |
| Low battery | Show maintenance warning |
| High-risk prediction | Alert a human; do not trigger dangerous autonomous firefighting |

## 9. Golden-path demonstration

The video and final demonstration should show one complete reliable scenario:

1. Display normal sensor readings.
2. Replay rising temperature, falling humidity, and declining soil moisture.
3. Send telemetry through the connectivity layer.
4. Run the AI model.
5. Show the risk probability rising.
6. Display the affected grid and leading factors.
7. Update patrol priority.
8. Activate the LED, buzzer, or local warning indicator after the declared decision condition.
9. Replay a historical fire event.
10. Simulate a network or sensor failure.
11. Show stale-data status and safe fallback.
12. Display the logged outcome and ranger feedback.

Build an offline deterministic replay as a backup. A live hardware failure must not destroy the demonstrable AIoT loop.

## 10. Innovation position

Avoid presenting the project as:

> Sensor data + Random Forest + dashboard.

Present it as:

> An uncertainty-aware, multi-scale early-warning system that combines local microclimate sensing with satellite-derived fire history and dynamically changes sensing and patrol priority.

The defensible innovation is the operational loop:

- local sensors provide conditions at field level;
- satellite data provides broader spatial context;
- AI combines both at a declared forecast horizon;
- the prediction changes sensing and patrol behavior;
- human confirmation closes the feedback loop.

## 11. Scope cuts

Cut these unless validation proves they matter:

- automatic water spraying or autonomous firefighting;
- excessive external datasets;
- nationwide prediction claims;
- computer vision without a clear decision role;
- a complex digital twin that does not improve the demo;
- a custom deep-learning model without sufficient labeled data;
- a large dashboard with no operational action.

One reliable golden path is worth more than many fragile features.

## 12. Limitations and responsible deployment

RimbaGuard may produce false alarms, miss fires, fail under cloud cover or sensor faults, and generalize poorly outside the pilot region. Satellite-derived labels have their own detection limitations. The prototype must communicate uncertainty and never imply that a low predicted risk guarantees safety.

Deployment requirements include:

- human confirmation before consequential action;
- documented sensor calibration and maintenance;
- local data ownership and access control;
- encrypted device-to-gateway communication where feasible;
- signed or authenticated telemetry in a real deployment;
- audit logs for predictions and interventions;
- model-drift monitoring;
- rollback to a tested baseline;
- coordination with local emergency-response procedures.

These are design controls and test targets, not claims of certification or legal compliance.

## 13. Competition deliverables

According to the guidebook:

- Full proposal: 15–20 pages, PDF maximum 10 MB.
- Citations: Harvard Style.
- Similarity result: maximum 20%.
- Prototype/simulation video: MP4, 3–7 minutes, minimum 720p, 16:9.
- Proposal and video deadline: 30 November 2026, 23:59 WIB.
- Final presentation and demonstration: 12 minutes plus 8 minutes of questions.
- Final scoring: research depth 30%, technical AIoT implementation 30%, innovation/originality and impact 20%, presentation/Q&A 20%.[1]

The official competition webpage confirms the 30 November registration endpoint but lists a slightly different final-announcement date from the guidebook. Confirm the discrepancy with the organizer before submission.[2]

## 14. Immediate next artifact

Create a one-page `Prediction and Validation Contract` containing:

- target user;
- pilot area;
- grid size;
- forecast horizon;
- exact label;
- feature cutoff time;
- data sources;
- baseline;
- train/validation/test split;
- primary metric;
- alert action;
- fallback behavior;
- limitations.

Pass condition:

> Every team member can state what the system predicts, for whom, when, from which data, against which baseline, and what action follows.

## 15. Sources

[1] BINUS Robotics Cup 2026, `AIoT Innovation Competition` Official Guidebook v4.1. Local file: `(v4.1) Guidebook AIoT.pdf`.

[2] BINUS Robotics Cup 2026 official page. https://makerspace.binus.edu/binus-robotics-cup-2026

[3] NASA FIRMS Active Fire Data. https://firms.modaps.eosdis.nasa.gov/content/active_fire

[4] NASA Earthdata FIRMS. https://www.earthdata.nasa.gov/data/tools/firms
