# PS21 — Interactive Virtual Lab: Distributed Tracing & Telemetry Collector

Course: Advanced Cloud System Software Design · Primitives: Cloud Compute + Cold Object Storage

A **browser-based simulator** (React + TypeScript + Vite + Tailwind + React Flow + Recharts). It does **not** run a real
OpenTelemetry Collector, OpenSearch or S3 — it models their behaviour with a deterministic, seeded discrete-event engine.
No Docker, AWS credentials, backend or paid API is needed.

## Run it

Requires Node.js 18+ (20 or 22 recommended).

```bash
npm install
npm run dev        # then open the printed URL (default http://localhost:5173)
npm test           # runs the 14 simulation-engine tests (Vitest)
npm run build      # type-check + production build (optional)
```

## First simulation (30 seconds)

1. Open **Virtual Lab** (opens by default) and press **Start Simulation** in the top bar.
2. Watch requests flow in the canvas counters and traces appear in the **Live trace feed** (bottom). Raise **Speed** to go faster.
3. Click a trace row → span **waterfall** (slowest span amber, failure origin red). Full table with filters: **Live Traces**.
4. Click any node on the canvas (e.g. *Batch Processor*) to change its settings → **Run again (same seed)**.

## Experiments & scenarios

| Goal | How |
|---|---|
| Baseline / High traffic / Tail sampling / Batch size / Service failure / Collector bottleneck / Index outage / Cold outage / Recovery | **Experiments** page → *Run Experiment* (runs all variants on the same seed, shows Expected vs *Actually observed* vs *Why*). *Watch live* replays a variant in the Lab. |
| Verify error retention (C) | Pipeline Inspector → Tail Sampling → keep *Retain error traces* on, set *Ordinary* to 0%, run: only Error traces are Sampled. |
| Batch size comparison (D) | Pipeline Inspector → Batch Processing → change size → *Run again with same seed*; or Experiment 4. |
| Service B outage (E) | Fault Injection → *Service unavailable*, Service B, duration 3 s → Inject while running; see failures, then recovery. |
| Searchable storage outage (F) | Fault Injection → *Searchable storage unavailable*; traces stay **Pending**, become **Indexed** after removal. |
| Save & compare (G, H) | Experiments → *Save these runs* (or Results → *Save run*); tick 2+ runs under **Results & Comparison**. Refresh the page: saved runs persist (localStorage). |

## How the simulation works (viva cheat-sheet)

* **Engine** (`src/engine/`, no UI code): `simulationEngine.ts` (virtual clock + event queue), `metrics.ts`, `experiments.ts`, `config.ts`, `rng.ts`, `types.ts`.
* **Request path:** A → B → C. Each service adds work time (base latency × multiplier, ±20% jitter), may fail with the error probability, or be unavailable. A failing service stops the chain; parent spans are marked as errored (*Downstream … failed*); the failing span is the *origin*.
* **Trace context:** every span carries the same 128-bit-style trace ID (32 hex chars) and its parent span ID.
* **Telemetry path:** finished trace → *Collector* (single server, processing time per trace, bounded backlog → telemetry lost when full) → *Tail sampling* (waits `waitMs`; priority Error → Slow → Ordinary %; spans that waited longer than the window ⇒ *Incomplete*; pending buffer limit) → *Batch processor* (flush at size or timeout; bounded queue drops when full) → *Exporter* (per-batch delay + per-record cost; 3 retries with back-off) → *Indexing* (delay; 12 retries; only then **Indexed**) → *Archival worker* (every interval, bundles traces older than the eligibility age into an object; retried until cold storage is back).
* **Determinism:** every request/trace draws from its own stream derived from `(seed, index, purpose)`, so changing batch/sampling settings never changes the workload.
* **Metric definitions:** avg/p95/throughput/error-rate use *finished requests*; p95 is nearest-rank (`sorted[ceil(0.95·n)-1]`); retention % = retained ÷ evaluated (evaluated = retained + dropped + incomplete; *lost* traces never reached a decision); export success = exported ÷ (exported + failed + queue-dropped).
* **Simulation limits (be honest in the viva):** the request path has no retries/timeouts; no data is persisted outside localStorage; numbers illustrate behaviour, not production performance.

## Project layout

```
src/engine/      simulation engine (tested, UI-independent)
src/components/  canvas (React Flow), trace table + waterfall, charts, shared UI
src/pages/       Lab, Live Traces, Pipeline Inspector, Fault Injection, Experiments, Results, Learn
tests/           engine.test.ts (14 tests)
```
