# 🏥 SD-WAN Network Health Score (NHS)

> Built a network performance scoring model for Versa & Viptela SD-WAN — scores health at every level from individual links up to the full customer network

---

## 🔍 Problem

Enterprise network operations teams managing SD-WAN infrastructure across thousands of customers had no single, defensible metric to answer the question:

**"How healthy is this customer's network today?"**

Before NHS, operators had to manually review raw metric tables across multiple systems to assess network health. There was no single number that told a customer *"your network is performing at 87% today."* NHS created that number — and made it defensible with a full audit trail of contributing metrics.

---

## ⚙️ How It Works

### Data Ingestion
- Ingests one day of telemetry per customer per model run
- Data arrives pre-aggregated at the link level (`dnsEntityName` + `circuitName` + `remoteDnsEntityName` + `remoteCircuitName`)
- Sources: Versa logs (`intfUtilLog`, `sdwanB2BSlamLog`, `systemLoadLog`, `monStatsLog`, `alarmLog`) and Viptela objects (`device-health`, `device-interface`, `device-tunnel`)

---

### 📊 Metric Categories

| Category | Metrics | Notes |
|---|---|---|
| Bandwidth utilization | ulBwUtil, dlBwUtil | Upload and download utilization % |
| Latency | delay, delayIncrease | Today's delay vs historical EMA baseline |
| Jitter | fwdDelayVar, revDelayVar (Versa) / jitter (Viptela) | Forward and reverse delay variation |
| Packet loss | fwdLossRatio, revLossRatio (Versa) / loss (Viptela) | Forward and reverse loss ratios |
| Downtime | pduLossRatio (Versa) / downtime (Viptela) | PDU loss or availability subtracted from 100 |
| System load | cpuLoad, memLoad | Device CPU and memory utilization |
| Alarms | alarmType, alarmKey, alarmSeverity | Converted to alarm duration scores per 5-min block |

---

### 🔄 Metric Transformation
- Each raw metric passed through a **logistic (sigmoid) transformation** with tunable slope and midpoint parameters
- Converts raw values into a **0–1 probability score** — higher = worse performance
- Example: delay of 100ms → mid-range probability · 300ms → close to 1.0 (severe)
- Parameters (slope, midpoint, weight, limit) configurable per customer and per metric via YAML config

---

### 🏗️ Scoring Hierarchy — 5 Levels
Link score        (tunnel: local circuit + remote circuit)
↓           weighted average by traffic volume
Interface score   (circuit level)
↓           weighted average by traffic volume
Device score
↓           weighted average by traffic volume
Site score
↓           weighted average
Network score     (single customer-level daily score)


- Weights calculated from `slamCount` (communication frequency) and `totalOctets` (traffic volume) — busier links carry more weight
- Both a **single-day score** and a **7-day rolling score** produced at every hierarchy level
- 7-day score uses a rolling window — if run on Wednesday, covers previous Wednesday through Tuesday

---

### 📉 EMA Smoothing
- Exponential moving average applied to delay, utilization, and system load metrics
- EMA weights configurable per metric: `slam: 0.0328` · `util: 0.0145`
- Intermediate EMA state stored as opaque DataFrames — model is **stateful across daily calls**

---

### 🛡️ Failure Handling
- ✅ Validates all input fields and numeric types — **fails loudly** on schema violations
- ⚠️ Logs WARNING if missing values present in input data
- ❌ Logs ERROR if device weights do not sum to 1
- 🔄 New devices automatically incorporated — single-day scores only until 7 days accumulated

---

## 📤 Output

- **10 DataFrames per run:** `linkScoresDay`, `interfaceScoresDay`, `deviceScoresDay`, `siteScoresDay`, `networkScoresDay` + weekly equivalents for each
- Each row contains: raw metric values · transformed probability scores · weight · score · alarm data · EMA values · version · customer identifiers (`cleId`, `billingAccountNumber`, `locationId`)

---

## 📈 Scale

| Dimension | Value |
|---|---|
| Devices monitored | 11K+ |
| Vendors | Versa, Viptela |
| Telemetry volume | Millions of records/day |
| Hierarchy levels | 5 (link → interface → device → site → network) |
| Output DataFrames per run | 10 (day + week × 5 levels) |
| Model cadence | Daily |

---

## 🧰 Tech Stack

| Component | Tool |
|---|---|
| Language | Python |
| Data processing | Pandas, NumPy |
| Statistical transforms | Scipy |
| Pipeline orchestration | Apache Airflow |
| Data warehouse | GCP BigQuery |
| Reporting | Looker Studio |
| Config management | YAML |

---

## ✅ Results

- Improved network visibility and end-user experience scores by **15%**
- Enabled customer-facing at-a-glance health scoring across Verizon's enterprise SD-WAN base
- Unified scoring model across Versa and Viptela vendors
- Full audit trail of contributing metrics at every hierarchy level for operator diagnostics

---

*Built and owned at Verizon Business Group — VBG Product AI/ML Team*
