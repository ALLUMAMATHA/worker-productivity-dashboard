# worker-productivity-dashboard

A production-style full-stack web application that ingests AI-generated CCTV events, stores them, computes productivity metrics, and displays them in a dashboard for a sample factory setup.


 # Project Overview

This application simulates how AI/computer-vision systems in a manufacturing factory can monitor worker activity and production in real time.

The system:

Ingests structured AI events

Persists them in a database

Computes productivity metrics

Displays insights via a web dashboard

# Architecture 

AI / CCTV System
   │
   │  (JSON events)
   ▼
Spring Boot REST API
   │
   │  (persisted events)
   ▼
Relational Database
   │
   │  (query + aggregate)
   ▼
Metrics Engine
   │
   │  (computed metrics)
   ▼
Frontend Dashboard (HTML +  CSS + JS)


# Key Components

Event Ingestion API
Accepts AI-generated events via REST (POST /events, POST /events/bulk).

Persistence Layer
Stores workers, workstations, and AI events using JPA.

Metrics Service
Computes worker-level, workstation-level, and factory-level metrics on demand.

Frontend Dashboard
Fetches metrics and events to display productivity insights and timelines.

This separation allows the system to scale ingestion, storage, and analytics independently.

# Database Schema 

workers
- id (PK)
- worker_id (unique)
- name

workstations
- id (PK)
- workstation_id (unique)
- name

ai_events
- id (PK)
- timestamp
- worker_id
- workstation_id
- event_type
- confidence
- count

# Metric Definitions

Worker-Level Metrics
--------------------
* Total Active Time	Sum of time intervals where event type is working (and selected product_count cases)
* Total Idle Time	Sum of time intervals where event type is idle
* Utilization %	(Active Time / Observed Time) × 100
* Total Units Produced	Sum of all product_count.count values
* Units per Hour	Total Units / Observed Hours

Observed Time = Active + Idle (+ Absent if present)

Workstation-Level Metrics
-------------------------
* Occupancy Time	Time when a worker is present (working or idle)
* Utilization %	(Active Time / Occupancy Time) × 100
* Total Units Produced	Sum of product_count events at that station
* Throughput per Hour	Total Units / Occupancy Hours

Factory-Level Metrics
---------------------
* Total Productive Time	Sum of all workers’ active time
* Total Production Count	Sum of all product units
* Avg Production Rate	Average units/hour across workers
* Avg Utilization	Average utilization % across workers

# Assumptions and Tradeoffs
1. Event-Driven State Model

Assumption:
AI events represent state transitions, not continuous signals.

Tradeoff:
* Simple, scalable, realistic.

2. Last Known State Persists

Assumption:
A worker’s last event state (working, idle, absent) continues until a new event arrives.

Tradeoff:
* Prevents overestimating productivity

3. Out-of-Order Events

Assumption:
Events may arrive out of order.

Tradeoff:
*  Correct metrics
*  Slight compute overhead (acceptable at this scale)

4. Duplicate Events

Assumption:
AI systems may resend the same event.

Tradeoff:
*  Strong consistency

# Intermittent connectivity (how we handle it)

Cameras and edge devices can lose network connectivity or have flaky links. The system is designed to be resilient to these real-world conditions:

1. Local buffering and batch retry (edge-friendly)
Cameras / edge agents should buffer events locally for short intervals (e.g. 1–10s) and send them in batches using POST /events/bulk. If the network is down, the edge retries with exponential backoff and preserves the batch order. Batching reduces HTTP overhead and makes retries efficient.

2. Idempotency and deduplication
Events are de-duplicated on ingest using a deterministic event key and a database unique constraint on (worker_id, workstation_id, timestamp, event_type, count). The service checks for existence before saving and treats DataIntegrityViolationException as a benign duplicate. This allows safe retries from the edge without risk of double-counting.

3. Last-known-state persists (conservative model)
If events are delayed, the backend treats the last received state as continuing until a newer event arrives. This conservative approach avoids overstating activity during network gaps. A rolling window option (configurable) can be used to avoid very old states dominating UI visuals.

4. Ordering & out-of-order handling
Events are sorted by timestamp at metric-computation time so minor out-of-order arrivals do not corrupt metrics. For large reordering risks, include a sequence number or camera timestamp in the event payload so the edge can preserve ordering before sending.


