# AIOps Monitoring and Event Processing Assessment

## Overview
This repository simulates a basic AIOps workflow for a payment service. The service emits operational telemetry in the form of timing metrics, CPU and memory usage, and log messages. The objective is to detect abnormal behaviour, turn those anomalies into events, and verify that the event information moves through a simple producer → topic → consumer pipeline before being reported as an operational issue.

The scenario reflects a real operations problem: a service is experiencing slow responses and potential system stress. AIOps helps detect that behaviour early and route it to downstream monitoring or incident workflows.

## Service being monitored
The monitored application is a `payment-service` that processes payment requests. It produces telemetry such as response time, resource consumption, and log entries that reveal whether traffic is healthy or degraded.

## Operational data
The operational dataset is stored in `data/service_data.json`. Each record represents one observation at a timestamp and includes:

- `timestamp`: when the event was recorded
- `service`: the service name
- `response_time_ms`: service latency in milliseconds
- `cpu_percent`: CPU usage
- `memory_percent`: memory usage
- `log_level`: severity such as `INFO`, `ERROR`, or `WARNING`
- `message`: the log message text

### Metrics vs. logs
- Metrics: `response_time_ms`, `cpu_percent`, `memory_percent`
- Log information: `log_level` and `message`
- Time usage: the `timestamp` field is used to order records and show when the service became unhealthy

### Normal vs. abnormal observations
Normal observations are the records where response time stays low and resource levels remain near baseline, such as the first five entries in the dataset. These show successful payment processing and stable system behaviour.

The abnormal records are the later entries around `2026-09-20T10:05:00` and `2026-09-20T10:06:00`:

- `response_time_ms` spikes above the threshold of 500 ms
- `cpu_percent` rises above 80%
- `memory_percent` rises above 80%
- `log_level` indicates an error condition
- messages such as `Payment service timeout` and `Database connection timeout` indicate service degradation

## Repository components
The architectural flow is intentionally lightweight and in-memory:

- `data/service_data.json` — operational data
- `src/anomaly_detector.py` — checks telemetry thresholds and builds anomaly events
- `src/event_topic.py` — in-memory topic for event storage
- `src/event_producer.py` — publishes anomaly events to the topic
- `src/event_consumer.py` — consumes events from the topic
- `src/aiops_pipeline.py` — orchestrates the end-to-end workflow from data load to output

## Anomaly detection findings
The detection logic is threshold-based. It raises an anomaly when a record exceeds the expected operating limits for response time, CPU use, or memory use. The detector also surfaces a relevant log-based concern when a record contains an error message or error-level event.

Observed anomalies in the sample dataset:

1. `2026-09-20T10:05:00` — Payment service timeout
   - response time: 610 ms
   - CPU: 75%
   - memory: 70%
   - log level: `ERROR`

2. `2026-09-20T10:06:00` — Database connection timeout
   - response time: 640 ms
   - CPU: 94%
   - memory: 91%
   - log level: `ERROR`

These are the clearest signals that the payment service is experiencing severe latency and resource pressure.

## Event-processing flow
The workflow follows this sequence:

1. Operational data is loaded.
2. Each record is analysed by the anomaly detector.
3. Anomalous records are converted into anomaly events.
4. The event producer publishes the anomaly event to an in-memory topic.
5. The event consumer reads the messages from the same topic.
6. The downstream AIOps pipeline reports the final operational issue.

This is the simulated equivalent of a production message pipeline and shows how a monitoring signal can propagate from detection into an operational response.

## Workflow result
The final pipeline output is a summary showing the number of records processed and the number of anomaly events detected and consumed. In this dataset, the expected result is that the pipeline processes all 10 records and detects the two clear anomalies from the degraded service period.

## Issues identified and corrected
The assessment includes a few workflow issues that prevent the pipeline from behaving cleanly in its initial form. The main problems are:

- the anomaly logic should treat error-level logs as a relevant anomaly signal, not a generic warning-only condition
- the producer and consumer should operate on the same event topic so the event flow is consistent and observable

Once those items are aligned, the anomaly signal can travel correctly through the event pipeline instead of being lost between components.

## Limitation / improvement
This implementation is intentionally simple and rule-based. It uses fixed thresholds for response time, CPU, and memory, which means it can miss more subtle performance degradation or flag normal traffic if the workload changes substantially. A future improvement would be to add adaptive baselines, time-window analysis, and more advanced alert correlation.

## Reproduction steps
From a terminal in the repository root:

```bash
cd /workspaces/github-skills-challenge
pytest -q
python src/aiops_pipeline.py
```

Expected result:

- the tests validate the detection and event-processing flow
- the pipeline prints the processed records and detected anomalies
- the final output shows the service degradation as an anomaly event flow

## Summary
This project demonstrates the core AIOps flow:

Operational Data → Anomaly Detection → Event Generation → Producer → Topic → Consumer → AIOps Output

It is a minimal but realistic simulation of how monitoring systems can surface operational problems and route them to downstream processing.

---

&copy; 2025 GitHub

