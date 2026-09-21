# AIOps Monitoring and Event Processing Assessment

## Overview
This repository simulates a lightweight AIOps workflow for a payment service. The service emits telemetry such as response time, CPU usage, memory usage, and log-level data. The goal is to detect abnormal conditions, convert them into structured anomaly events, and confirm that those events move through a producer → topic → consumer flow before being reported as an operational issue.

The scenario mirrors a realistic operations problem: a service begins to slow down and become unstable under load. AIOps is used to recognize the issue early and route the signal into an event-processing pipeline for investigation.

## Operational data
The dataset lives in `data/service_data.json` and contains 10 records. Each record represents a single telemetry observation with:

- `timestamp`
- `service`
- `response_time_ms`
- `cpu_percent`
- `memory_percent`
- `log_level`
- `message`

The log and metric data together show how the service behaves over time. The healthy records are the early entries, while the abnormal records appear around `2026-09-20T10:05:00` and `2026-09-20T10:06:00`.

### Observations from the metrics and logs
The unhealthy events show:

- response time rising above the 500 ms threshold
- CPU usage climbing above 80%
- memory usage climbing above 80%
- log severity changing to `ERROR`
- log messages such as `Payment service timeout` and `Database connection timeout`

These conditions indicate degraded service health and potential system instability.

## Anomaly-detection findings
The detector in `src/anomaly_detector.py` is threshold-based. It flags a record as anomalous when one or more operational metrics exceed the expected limits. In this dataset, the clear anomaly events are:

1. `2026-09-20T10:05:00` — `Payment service timeout`
   - response time: 610 ms
   - CPU: 75%
   - memory: 70%
   - log level: `ERROR`

2. `2026-09-20T10:06:00` — `Database connection timeout`
   - response time: 640 ms
   - CPU: 94%
   - memory: 91%
   - log level: `ERROR`

These records represent the operational issue that the workflow is meant to highlight.

## Event-processing flow
The event flow is intentionally simple and in-memory:

1. The pipeline loads telemetry from `data/service_data.json`.
2. Each record is checked by `AnomalyDetector.detect()`.
3. If the record is anomalous, an `ANOMALY` event is created.
4. `EventProducer` publishes the anomaly event to an in-memory topic.
5. `EventConsumer` reads the same topic.
6. The final output reports the processed anomaly information.

This mirrors the pattern of a real operational pipeline: detect an issue, convert it to a structured event, and pass it downstream for handling.

## Final workflow result
The final execution of the end-to-end workflow produced the expected result:

- `Records processed: 10`
- `Anomalies detected: 2`
- `Events consumed: 2`

The output displayed the two real operational issues and their reasons, confirming that the full AIOps path functioned correctly.

## Issues identified and corrected
Two issues were present and corrected during the investigation:

1. Wrong log-level condition
   - The detector was checking for `WARNING` instead of `ERROR`.
   - Cause: the workflow did not match the actual anomaly pattern in the telemetry.
   - Correction: the detector now evaluates `record["log_level"] == "ERROR"`.

2. Topic mismatch in the pipeline
   - The producer and consumer were using different topic instances.
   - Cause: anomaly events were written to one stream but read from another, so events appeared lost.
   - Correction: both producer and consumer now use the same `EventTopic("service-events")` object.

These corrections are consistent with the existing architecture and keep the project lightweight while restoring the intended data path.

## Limitation and possible improvement
This implementation uses fixed thresholds for response time, CPU, and memory. That is useful for a simple demo, but it has limitations:

- it may miss gradual degradation that does not cross a hard threshold
- it may not adapt to normal variance across services or time windows
- it does not include richer correlation across logs, traces, and multiple services

A natural improvement would be to move from static thresholds to rolling baselines or adaptive anomaly scoring that learns the service’s normal behaviour before raising alerts.

## Reproduce the demonstration
From the repository root, run these commands:

```bash
cd /workspaces/github-skills-challenge
source .venv/bin/activate
python -m pytest -q
python src/aiops_pipeline.py
```

Expected outcome:

- the test suite passes
- the operational dataset is processed
- the anomaly detector identifies the degraded records
- the anomaly event is published and consumed
- the output shows the service issues with their reasons

## Summary
This repository demonstrates the core AIOps workflow in a compact form:

Operational Data → Anomaly Detection → Event Generation → Producer → Topic → Consumer → AIOps Output

The final executed pipeline confirms that the service health issue is correctly identified and surfaced as an operational anomaly event.

---

&copy; 2025 GitHub

