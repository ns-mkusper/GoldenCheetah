# Fitness Data Integration & Scoring Blueprint

## Objectives
- Centralize nutrition, training, sleep, and recovery data from MyFitnessPal and Samsung Health into the app.
- Normalize the data so downstream analytics can compute actionable guidance on fueling, training load, sleep sufficiency, and fitness progress.
- Surface the guidance both inside the app and within GoldenCheetah via custom metrics and dashboards.

## Data Access Landscape

### MyFitnessPal
- **Partner API (private access):** MyFitnessPal offers an OAuth2 partner API, but access remains private and requires approval via their developer program and submission of a business case (API@myfitnesspal.com).citeturn1search0
- **App Gallery links:** End users can still connect MyFitnessPal to approved partners through the in-app App Gallery, which can help for interim manual sync workflows.citeturn1search1
- **Aggregation platforms:** Middleware such as Terra already normalizes MyFitnessPal nutrition data and exposes unified webhooks plus historical fetch endpoints, providing a faster path to ingestion without direct API approval.citeturn1search2turn1search3

### Samsung Health / Galaxy Watch
- **Health Connect (Android hub):** Health Connect acts as a common data exchange on Android and is replacing Google Fit support starting June 30 2025, making it the preferred on-device interoperability layer.citeturn2search5
- **Samsung Health Data SDK (2025):** Samsung released a public v1.0 SDK (July 31 2025) that lets approved Android apps read, write, and aggregate data types such as sleep, exercise, heart rate, and nutrition, with device source metadata. Requires Samsung Health ≥ 6.30.2.citeturn2search0
- **Samsung Health Sensor & Accessory SDKs:** Updated 2025 releases expose live sensor streams (e.g., heart rate, power, cadence) and accessory integrations for BLE cycling sensors—useful for high-frequency training data not routed through Health Connect.citeturn2search1turn2search4
- **Aggregation platforms:** Terra also supports Samsung Health via its unified API, delivering normalized sleep and activity payloads through the same webhook infrastructure as MyFitnessPal.citeturn1search2

### GoldenCheetah Touchpoints
- GoldenCheetah allows athlete-specific custom metrics and charts via `Tools → Options → Metrics → Custom`, plus downloadable metric packs through CloudDB.citeturn0search0turn0search8
- Custom metric definitions compile into runtime formulas (see `src/Metrics/UserMetricSettings.h`) and can be shared through curated collections. External wellness data can be ingested as daily Measures or activity-linked XData via CSV/JSON imports when mapped to GoldenCheetah formats.citeturn3search3

## Target Architecture

```
Sources (MyFitnessPal, Samsung Health, Terra/Webhooks)
        ↓
Ingress Layer (Webhook receiver + OAuth data pull jobs)
        ↓
Normalization Service (schema mapping → unified wellness/training tables)
        ↓
Data Lake / Warehouse (time-series store + document cache)
        ↓
Analytics & Scoring Engine (batch + streaming jobs)
        ↓                     ↓
App APIs / UI Widgets   GoldenCheetah Exporter (metrics & dashboards)
```

### Source Connectors
- **Webhook Receiver:** Host a dedicated endpoint (e.g., AWS API Gateway + Lambda or FastAPI) to consume Terra webhook payloads; persist raw JSON for auditing before transformation.
- **Polling Jobs:** For providers without push (e.g., initial historical pull from Terra HTTP APIs or partner exports), schedule jobs with backoff and delta tracking.
- **Direct SDK Integrations:** Mobile app can embed Health Connect and Samsung Health Data SDK clients to push consented data to your backend using secure batch uploads when the user is online.

### Normalization & Storage
- **Schema:** Adopt an entity schema grouped into `daily_summary`, `meal_log`, `workout_session`, `sleep_session`, `recovery_marker`, and `device_metadata` tables. Ensure ingestion stamps include `source_system`, `source_device`, and `ingested_at` for traceability.
- **Processing:** Use a streaming ETL (e.g., Kafka → Flink/Spark Structured Streaming) to fan out normalized records and trigger recalculations. Lightweight transformations can run serverlessly (AWS Lambda / GCP Cloud Functions) if event volume is modest.
- **Storage:** Combine an append-only time-series store (e.g., TimescaleDB) for metrics with object storage (S3/Cloud Storage) for raw payload retention. Index data by user + UTC date to simplify daily scoring.

### Analytics & Scoring
- Expose the scoring engine as a stateless microservice that consumes the latest normalized data and computes:
  - Daily component scores (fuel, load, sleep, plateau)
  - Rolling aggregates (7-day, 28-day) for trend analysis
  - Threshold-triggered recommendations
- Persist scores per day and per training block to support audits and GoldenCheetah exports.

### GoldenCheetah Integration Layer
- **Metric Definitions:** Ship `.gmetric` files that map to exported score fields (e.g., `Fuel_Balance`, `Training_Load_Risk`, `Sleep_Sufficiency`, `Plateau_Index`). Users import them via the Custom Metrics dialog or download from a hosted CloudDB feed.citeturn0search0turn0search8
- **Data Delivery Options:**
  1. **Daily Measures CSV/JSON:** Generate files matching GoldenCheetah’s Measures or XData formats and sync them into the athlete cache; suitable for wellness metrics unrelated to specific rides.citeturn3search3
  2. **Activity Annotations:** Attach per-ride metrics (e.g., training load risk) as XData series so charts align with the workout timeline.
  3. **CloudDB Sync:** Optionally publish metrics to a curated CloudDB collection for one-click install across devices once governance is in place.

### Security & Compliance
- Enforce user-level OAuth scopes, align retention with provider terms (MyFitnessPal prohibits commercial re-use without approval).citeturn1search0
- Maintain auditable consent logs (timestamp, scope, revocation) and support user-initiated data deletion.
- Encrypt data at rest and in transit; segregate raw vs. normalized stores for breach isolation.

### Resilience & Monitoring
- Implement idempotent upserts keyed by `(user_id, source_id, timestamp)` to avoid duplication.
- Track webhook latency, error rates, and source freshness; alert on ingestion stalls > 6 hours.

## Fitness Score Methodology

### Inputs & Baselines
- **Energy Intake/Expenditure:** Calories, macros from MyFitnessPal; exercise energy, steps, HR from Samsung Health.
- **Training Load:** Use TRIMP, HR TSS, or power-based load pulled from Samsung workouts or GoldenCheetah sessions.
- **Recovery Markers:** Sleep stages/duration, resting HR, HRV (if available) from Samsung Health; optional subjective readiness captured in-app.
- **Performance KPIs:** Critical Power, VO₂max estimates, pace/power PBs, or FTP trends from GoldenCheetah analytics.

### Component Scores
1. **Fuel Balance Score (0–100)**
   - Compute individualized Total Daily Energy Expenditure (TDEE) using basal metabolic rate + activity multipliers.
   - Score daily calorie delta: full marks when 3-day moving average is within ±5% of the period goal; degrade to 0 when > 20% deficit/surplus sustained over 7 days.
   - Flag macronutrient issues (e.g., < 20% calories from protein when in deficit) as modifiers for coaching recommendations.
2. **Training Load Risk Score (0–100)**
   - Calculate Acute:Chronic Workload Ratio (ACWR) or exponentially weighted TRIMP using the normalized training load stream.
   - Score 80–100 for ACWR 0.8–1.3; warn at > 1.5 (drop score) and < 0.7 (detraining risk). Complement with monotony (daily load / SD) and strain (weekly load × monotony) for deeper context.
   - Incorporate Samsung sensor data (HRV, resting HR) as recovery modifiers when available.
3. **Sleep Sufficiency Score (0–100)**
   - Base on nightly duration vs. personalized target (e.g., 7.5 h default, adjustable). Score 100 when 7-day average ≥ target and no single night < 70% target.
   - Deduct for low efficiency (< 85%), high wake-after-sleep-onset, or circadian irregularity (> 2 h bedtime variance) gleaned from Samsung sleep stages.
4. **Fitness Progress / Plateau Index (−50 to +50)**
   - Track rolling improvements in key performance indicators (CP curve, best 20 min power, 5 km run time) relative to historical variance via GoldenCheetah analytics.
   - Plateau signal when 28-day trend < 1 standard deviation gain despite adequate training load and recovery (scores near 0). Positive trend yields positive score; regression yields negative score prompting load or recovery adjustments.

### Composite Guidance
```
Overall Fitness Score = 0.3 * Fuel + 0.3 * Load + 0.25 * Sleep + 0.15 * PlateauNormalized
```
- Thresholds:
  - **90–100:** Maintain current plan; monitor for marginal gains.
  - **70–89:** Watch list; highlight lowest component for targeted tweaks.
  - **50–69:** Prompt actionable recommendations (calorie tweak, deload, sleep hygiene).
  - **< 50:** Escalate with coach-notification or automated deload suggestions.

### Decision Outputs Aligned to User Questions
1. **Calories:** Compare Fuel Score and 3-day energy balance; recommend increase/decrease when sustained deviation > 10% with low Fuel Score.
2. **Training Load:** Use Load Risk score plus plateau index; advise deload when Load < 60 and ACWR > 1.5 or monotony > 2.0.
3. **Sleep:** Trigger alerts when Sleep Score < 70 for 3 consecutive nights and duration deficit > 45 minutes.
4. **Plateaus:** When Plateau Index ≤ −10 for 2+ weeks with Load 70–90, recommend block change (e.g., intensity focus, technique work) and highlight relevant GoldenCheetah charts.

## GoldenCheetah Delivery
- Export daily component scores and recommendations as Measures CSV (date, metric, value) for standalone wellness tracking, plus XData aligned to workouts for training-load context.citeturn3search3
- Provide curated `.gmetric` files so users can display the composite score, component breakdowns, and decision flags inside Trends/Overview panels.citeturn0search0turn0search8
- Offer optional Python/R charts (leveraging GoldenCheetah’s scripting) for rolling score timelines and component contributions; distribute via CloudDB once stable.citeturn0search6

## Implementation Roadmap
1. **Foundation (Weeks 0–4)**
   - Secure provider access (Terra contract, MyFitnessPal partner request, Samsung SDK enrollment).
   - Build webhook receiver, raw payload store, and consent UX in mobile/web apps.
2. **Normalization (Weeks 4–10)**
   - Implement schema mapping for nutrition (meals, macros), activity (sessions, load), sleep, and recovery data.
   - Stand up scoring microservice with Fuel and Sleep components (lowest integration complexity).
3. **Analytics Expansion (Weeks 10–16)**
   - Add training load and plateau logic, calibrate thresholds using historical GoldenCheetah data exports.
   - Integrate sensor SDK data for intra-session metrics if required.
4. **GoldenCheetah Export (Weeks 12–18)**
   - Generate automated Measures/XData export jobs; author `.gmetric` definitions and supporting charts.
   - Pilot with internal athletes, iterate on metric naming and tagging conventions.
5. **Production Hardening (Weeks 18–24)**
   - Add monitoring, alerting, and SLA dashboards.
   - Conduct security review, finalize data retention and deletion workflows.
   - Document partner compliance (MyFitnessPal ToS, Samsung data-sharing policies).

## Open Questions
- Confirm MyFitnessPal partner approval timeline; if delayed, rely on Terra + user-triggered CSV exports as interim.
- Decide whether to build native Android ingestion (Health Connect SDK) vs. defer fully to Terra for Samsung data in MVP.
- Establish governance for distributing `.gmetric` and chart files (self-hosted vs. contribution to GoldenCheetah CloudDB).
