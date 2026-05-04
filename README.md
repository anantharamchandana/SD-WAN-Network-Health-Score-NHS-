# SD-WAN-Network-Health-Score-NHS-
Built a network performance scoring model for Versa and Viptela SD-WAN scores health at every level from individual links up to the full customer network.

Enterprise network operations teams managing SD-WAN infrastruture across thousand of customers had no single, defensible metric to answer the question: 

"How healthy is this customer's network today?"

Software defined - Network Health Score (NHS)

What it does:
Produces a daily composite health score for every customer network — computed bottom-up from individual tunnel links all the way to the overall customer network score. Gives network operations teams and customers an at-a-glance view of how well their network is performing, with enough supplemental data to drill down and diagnose what is driving score changes.

The business problem it solves:
Before NHS, operators had to manually review raw metric tables across multiple systems to assess network health. There was no single number that told a customer "your network is performing at 87% today." NHS created that number — and made it defensible with a full audit trail of contributing metrics.
How it works — step by step:
Data ingestion:

Ingests one day of telemetry per customer per model run
Data arrives pre-aggregated at the link level (dnsEntityName + circuitName + remoteDnsEntityName + remoteCircuitName)
Sources: Versa logs (intfUtilLog, sdwanB2BSlamLog, systemLoadLog, monStatsLog, alarmLog) and Viptela objects (device-health, device-interface, device-tunnel)

Metric categories and raw inputs:

Bandwidth utilization: ulBwUtil, dlBwUtil — upload and download utilization percentages
Latency: delay, delayIncrease (today's delay vs historical EMA baseline)
Jitter: fwdDelayVar, revDelayVar (Versa) / jitter (Viptela) — forward and reverse delay variation
Packet loss: fwdLossRatio, revLossRatio (Versa) / loss (Viptela) — forward and reverse loss ratios
Downtime: pduLossRatio (Versa) / downtime (Viptela) — PDU loss or availability subtracted from 100
System load: cpuLoad, memLoad — device CPU and memory utilization
Alarms: alarmType, alarmKey, alarmSeverity — converted to alarm duration scores per 5-minute block

Metric transformation:

Each raw metric is passed through a logistic (sigmoid) transformation with tunable slope and midpoint parameters per metric
Transformation converts raw values into a 0–1 probability score where higher = worse performance
Example: a delay of 100ms maps to a mid-range probability; 300ms maps close to 1.0 (severe)
Parameters (slope, midpoint, weight, limit) are configurable per customer and per metric via YAML config

Scoring hierarchy — 5 levels:
Link score (tunnel: local circuit + remote circuit)
        ↓ weighted average by traffic volume
Interface score (circuit level)
        ↓ weighted average by traffic volume
Device score
        ↓ weighted average by traffic volume
Site score
        ↓ weighted average
Network score (single customer-level daily score)

Weights are calculated from slamCount (communication frequency) and totalOctets (traffic volume) — busier links carry more weight
Both a single-day score and a 7-day rolling score are produced at every hierarchy level
7-day score uses a rolling window (not calendar week) — if run on Wednesday, covers previous Wednesday through Tuesday

EMA smoothing:

EMA (exponential moving average) applied to delay, utilization, and system load metrics
EMA weights configurable per metric category (e.g., slam: 0.0328, util: 0.0145)
Intermediate EMA state stored as opaque DataFrames and passed back on each daily call — model is stateful across days

Failure handling:

Model validates all input fields and numeric types before processing — throws exceptions on schema violations (fails loudly)
Logs WARNING if missing values present in input data
Logs ERROR if device weights do not sum to 1 (score calculation integrity check)
New devices automatically incorporated — single-day scores only until 7 days of data accumulated

Output:

10 DataFrames per run: linkScoresDay, interfaceScoresDay, deviceScoresDay, siteScoresDay, networkScoresDay + weekly equivalents for each
Each row contains: raw metric values, transformed probability scores, weight, score, alarm data, EMA values, version, customer identifiers (cleId, billingAccountNumber, locationId)

Key outcome: Improved network visibility and end-user experience scores by 15%

