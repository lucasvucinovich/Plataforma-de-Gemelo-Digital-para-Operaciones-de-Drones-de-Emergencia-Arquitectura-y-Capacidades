# Klout — Implementation Reference

**Project:** Klout-semifinal  
**Last updated:** 2026-03-07  
**Status:** All 8 epics complete (48 stories shipped)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Tech Stack](#2-tech-stack)
3. [Application Pages](#3-application-pages)
4. [API Routes](#4-api-routes)
5. [Core Libraries](#5-core-libraries)
6. [UI Components](#6-ui-components)
7. [Data Files](#7-data-files)
8. [Real-time Architecture](#8-real-time-architecture)
9. [Epic & Story Completion Summary](#9-epic--story-completion-summary)
10. [Recent UI Enhancements](#10-recent-ui-enhancements)

---

## 1. System Overview

Klout is a real-time emergency drone fleet management dashboard for city-scale incident response, set in Madrid, Spain. It implements a **digital twin** of a drone fleet, with AI-driven decision support, anomaly detection, scenario simulation, and closed-loop command execution.

The system is entirely in-memory (no external database) with file-backed scenario run persistence. There are no microservices — everything runs inside a single Next.js App Router process.

### High-level architecture

```
Browser
  ├── HTTP polling (1500ms)  ──► GET /api/drones, /api/incidents, /api/fleet/status
  ├── SSE stream             ──► GET /api/websocket  (command events, sim ticks, incidents)
  └── Smooth drone animation  ──  requestAnimationFrame ease-in-out interpolation

Next.js App Router (single process)
  ├── DigitalTwinStore (singleton)   ──  in-memory drones + incidents + history
  ├── CommandProcessor               ──  command state machine, auto-lifecycle
  ├── TelemetrySimulator             ──  tick-based random walk + anomaly presets
  ├── IncidentGenerator              ──  LLM + template fallback
  ├── AssistantService (Jarvis)      ──  OpenAI / Mistral / Gemini / Anthropic + deterministic
  ├── Anomaly Detectors (×4)         ──  battery, comms, coverage gap, saturation
  ├── AlertAggregator                ──  time-windowed alert deduplication & severity bucketing
  ├── SimulationEngine               ──  deterministic heuristic single-plan simulation
  ├── ResourceCoordination           ──  request → asset assignment → delivery tracking
  └── File-backed run persistence    ──  data/scenarios/runs/*.json
```

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router) |
| UI | React 19, Tailwind CSS v4 |
| Language | TypeScript (strict) |
| Validation | Zod (all API boundaries) |
| Maps | Leaflet + react-leaflet (SSR-disabled via `dynamic`) |
| AI providers | OpenAI, Mistral, Google Gemini, Anthropic |
| Testing | Vitest + @testing-library/react |
| Data | In-memory Maps + JSON file persistence |

---

## 3. Application Pages

### `/` — Live Ops Dashboard

The primary command-and-control view. On first visit a **scenario picker** is shown (sessionStorage-gated); after selection the full Live Ops layout appears.

- **ScenarioPicker** — choose from Default / Light Load / Heavy Load / Pre-canned Outcome. Calls `POST /api/demo/load-scenario` for non-default options, reseeding the twin store.
- **StatusPanel** — fleet aggregate bar: drone count, mission breakdown (idle/patrol/pursuit/delivery/surveillance/returning/charging), coverage percentage, active incident count.
- **MapContainer** — full-screen Leaflet map with layer toggles (Drones / Incidents / Coverage heatmap / Sector grid). Drone markers animate smoothly on move commands.
- **ChatPanel** — Jarvis conversational AI panel with recommendation cards and one-click plan application.

### `/fleet` — Fleet Management

Sectioned layout combining four capabilities:

- **Map** — same `MapContainer` as the main dashboard
- **Fleet Control** — command dispatch panel (see §6)
- **Simulator** — `LiveOpsControls`: start / pause / reset telemetry simulation, adjust speed (0.25×–4×), apply anomaly presets (battery drain spike, comms loss, sensor degradation, coverage stress)
- **Incident Generator** — `IncidentGeneratorMonitor`: configure the LLM incident generator (interval, severity range, allowed types), trigger manually, view last generated incident

### `/incidents` — Incident Queue

Live incident lifecycle management table via `IncidentQueue`. Shows all active incidents with type, priority, sector, state (new → triaged → assigned → in-progress → resolved → closed), SLA status, and assigned asset. Supports state transitions.

### `/scenarios` — Scenario Studio

Full simulation and plan comparison workspace:

- Street-name → sector geocoding (`/api/geocode` → Nominatim → sector mapping)
- `TimeHorizonToggle` for simulation horizon selection
- `MapContainer` with coverage heatmap overlay and sector highlighting
- `PlanBuilderPanel` — build drone-to-incident/sector assignment plans
- `ScenarioComparePanel` — run A/B plan comparison and display delta metrics (coverage %, gaps, unserved incidents, avg response delay)

### `/evaluator` — Vehicle Twins Explorer

OOP class hierarchy browser for technical evaluators:

- `VehicleTwinsExplorer` — displays `VehicleTwinBase → DroneTwin / GroundUnitTwin` with state descriptors, incoming telemetry event contracts, outgoing mission_status event contracts, and command API documentation

### `/field/tracking` — Field Operator Tracking

Mobile-friendly resource request status and ETA view for field personnel. Takes a `requesterId` query parameter, shows resource request status, assigned asset, estimated delivery, and alternative plan when drone delivery fails.

### `/simulator` — Simulator Controls (standalone)

Dedicated page for the TelemetrySimulator configuration and controls.

### `/simulation/[id]` — Simulation Run Detail

View a specific persisted simulation run by ID, showing scenario input, plan assignments, and outcome metrics.

---

## 4. API Routes

### Drone & Telemetry

| Method | Path | Description |
|---|---|---|
| GET | `/api/drones` | List all current drone states |
| POST | `/api/drones` | Create a drone twin |
| GET | `/api/drones/[droneId]/history` | 50-sample telemetry ring buffer for a drone |
| POST | `/api/telemetry` | Ingest telemetry payload → upsert drone twin state |

### Incidents

| Method | Path | Description |
|---|---|---|
| GET | `/api/incidents` | List all active incidents |
| POST | `/api/incidents` | Create an incident |
| PATCH | `/api/incidents/[id]` | Update incident (lifecycle state transition, assignment, etc.) |
| DELETE | `/api/incidents/[id]` | Delete an incident |

### Fleet

| Method | Path | Description |
|---|---|---|
| GET | `/api/fleet/status` | Fleet connectivity, staleness, anomaly events, aggregated alerts |
| GET | `/api/coverage` | Coverage heatmap data points |

### Commands

| Method | Path | Description |
|---|---|---|
| POST | `/api/commands` | Create a command (`move` with `targetSectorId` or `targetPosition`) |
| GET | `/api/commands?droneId=` | List command history for a drone |

### Simulation & Scenarios

| Method | Path | Description |
|---|---|---|
| GET | `/api/scenarios` | List scenario definitions |
| POST | `/api/scenarios/apply` | Run simulation for a scenario + plan, persist result |
| POST | `/api/scenarios/compare` | A/B compare two plans for a scenario |
| GET | `/api/scenarios/runs` | List all persisted simulation runs |
| GET | `/api/scenarios/runs/[id]` | Get a specific persisted run |

### Jarvis AI

| Method | Path | Description |
|---|---|---|
| POST | `/api/chat` | Send message to Jarvis, get reply + recommendations |
| POST | `/api/chat/proactive` | Trigger a proactive (unsolicited) Jarvis alert |

### Live Ops

| Method | Path | Description |
|---|---|---|
| GET | `/api/simulator` | Get telemetry simulator state |
| POST | `/api/simulator` | Start / pause / reset / configure simulator |
| GET | `/api/incident-generator` | Get incident generator status and config |
| POST | `/api/incident-generator` | Configure or trigger incident generator |
| GET | `/api/anomalies` | List current anomaly events |
| GET | `/api/activity` | Activity feed (command lifecycle audit trail) |
| GET | `/api/websocket` | SSE stream: command events, incident events, generator events |

### Resource Coordination

| Method | Path | Description |
|---|---|---|
| GET | `/api/requests` | List resource requests |
| POST | `/api/requests` | Create a resource request |
| GET | `/api/requests/tracking?requesterId=` | Per-requester request status and ETAs |
| POST | `/api/deliveries/assign` | Assign drone or ground asset to a delivery |
| GET | `/api/deliveries` | List all deliveries |

### Demo & Evaluator

| Method | Path | Description |
|---|---|---|
| GET | `/api/demo/scenarios` | List demo scenario options from `demo-scenarios.json` |
| POST | `/api/demo/load-scenario` | Reset twin store and seed from selected demo scenario |
| GET | `/api/demo-readiness` | Run demo readiness self-test checklist |
| GET | `/api/twins/registry` | Twin class hierarchy descriptors for evaluators |
| GET | `/api/geocode?address=` | Street address → lat/lon → sector ID mapping |

---

## 5. Core Libraries

### `lib/twin/DigitalTwinStore.ts`

The central in-memory store. Holds:

- `Map<droneId, DroneState>` — current state of every drone
- `Map<incidentId, Incident>` — all incidents
- `Map<droneId, DroneState[]>` — per-drone telemetry history (50-sample ring buffer)

Key operations: `upsertDroneFromTelemetry` (Zod-validated, immutable copy-on-write), `createIncident`, `updateIncident`, `getFleetState`, `getAggregates`, `resetForDemo` (demo-only, clears all maps).

All mutations return `StoreResult<T>` — never throws. The singleton instance in `lib/twin/storeInstance.ts` is shared across all API route handlers in the same process.

---

### `lib/commands/CommandProcessor.ts`

Full command/acknowledgement state machine:

```
queued → sent → acknowledged → executing → completed
                                         → failed
                                         → cancelled
```

Before creating a command it validates: drone exists, battery level ≥ threshold, comms state is not `lost`, no conflicting concurrent assignment for the same drone. Supports idempotency keys.

When a command reaches `executing`, `applyCommandToTwin` is called, which moves the drone to the target position in the store (via `upsertDroneFromTelemetry`). For `targetSectorId`, the sector center coordinates are resolved from a lookup table.

For demo purposes, the happy-path lifecycle is automatically simulated via `setTimeout` chains:

| Transition | Delay from issue |
|---|---|
| queued → sent | 500ms |
| sent → acknowledged | 1300ms |
| acknowledged → executing | 2100ms |
| executing → completed | 2900ms |

Publishes `command_event` via a pub/sub event bus consumed by the SSE stream (`/api/websocket`).

---

### `lib/liveops/TelemetrySimulator.ts`

Tick-based telemetry simulator with a seeded LCG RNG for determinism. Each tick computes:

- Random-walk position delta (configurable `movementScale`)
- Battery drain (linear with configurable rate)
- Heading jitter
- Speed variation

**Anomaly presets** (applied to all drones for demo):

| Preset | Effect |
|---|---|
| `battery-drain-spike` | 10× battery drain rate |
| `comms-loss` | All drones → comms state `lost` |
| `sensor-degradation` | All drones → sensors state `degraded` |
| `coverage-stress` | Drones cluster in one area, leaving gaps |

Configurable `speedFactor` (0.25×–4×) and `refreshMs` (250–10,000ms). Results fed back into the `DigitalTwinStore`.

---

### `lib/liveops/IncidentGenerator.ts`

LLM-driven incident generator. On each configured interval:

1. Builds a context prompt (sector map, recent incidents, current fleet state)
2. Calls the configured LLM provider
3. Validates output with Zod (type, severity, sectorId, impactedObjective, rationale)
4. Clamps severity to configured `minSeverity`–`maxSeverity` range
5. Persists the incident via `IncidentService`

Falls back to deterministic template-based generation if the LLM call fails or no API key is configured. Template picks a random sector (1–9), incident type, severity, and objective from predefined lists.

---

### `lib/ai/AssistantService.ts`

The Jarvis AI service. Supported providers:

| Provider | Model |
|---|---|
| OpenAI | `gpt-4o-mini` |
| Mistral | `mistral-small-latest` |
| Google Gemini | `gemini-1.5-flash` |
| Anthropic | `claude-3-5-sonnet-20241022` |
| Deterministic fallback | No API key required |

On each request it builds a context bundle: fleet state, incidents, connectivity/staleness status, anomaly events, operator alerts, active scenario/plan. Context arrays are capped for token limits. Responses in degraded mode get a prefix warning.

Returns a `JarvisReplyResult` with optional `recommendations[]` — each recommendation can include a full `plan` + `scenarioInput` for one-click application via `POST /api/scenarios/apply`.

---

### `lib/simulation/SimulationEngine.ts`

Deterministic heuristic simulation engine. Given:
- `ScenarioInput` (incidents, time horizon, optional sector/drone constraints)
- `Plan` (drone-to-incident/sector assignments)

Computes:
- `coveragePercent` — fraction of incidents/sectors covered by assigned drones
- `gaps` — number of uncovered zones
- `unservedIncidents` — count of incidents with no drone assignment
- `avgResponseDelayMinutes` — average estimated response delay
- `riskIndicators` — under-protected zones

IDs are deterministically hashed from inputs (no randomness). No external calls.

---

### `lib/anomaly/` — Anomaly Detectors

Four independent detectors, each returning `AnomalyEvent[]`:

| Detector | What it detects |
|---|---|
| `BatteryAnomalyDetector` | Low battery (<20%), abnormal drain rate, projected critical charge |
| `CommsAnomalyDetector` | Degraded/lost link, high latency (>200ms), high packet loss (>15%) |
| `CoverageAnomalyDetector` | Coverage gap (no drone within response radius of incident), fleet saturation (incident count >> nearby drones) |
| `AlertAggregator` | Groups anomaly events by drone/sector/fleet scope into time-windowed `Alert` records with severity bucketing (low/medium/high/critical) and resolve tracking |

---

### `lib/coordination/` — Resource Coordination

Implements the field operator resource request flow:

1. Field operator POSTs a `ResourceRequest` (items, urgency, preferred delivery mode)
2. `resourceAssignmentService` selects the best available drone or ground unit based on proximity, battery, and urgency
3. A `Delivery` record is created and tracked
4. If the drone delivery fails, `AlternativePlanner` suggests: ground handoff, alternate drone, or reroute-self
5. Field operators poll `/api/requests/tracking` for their request status and ETA

---

### `lib/twin/` — OOP Twin Hierarchy

```
VehicleTwinBase (abstract)
├── DroneTwin        ← aerial vehicle, full telemetry, battery, comms, sensors
└── GroundUnitTwin   ← ground vehicle, different mobility model
```

`TwinRegistry` exposes class descriptors (state shape, event contracts, command documentation) via `/api/twins/registry` for the Vehicle Twins Explorer evaluator page.

---

## 6. UI Components

### `components/map/MapContainer.tsx`

The map host component. Responsibilities:

- Polls `/api/drones`, `/api/incidents`, `/api/fleet/status` every **1500ms**
- Opens an SSE connection to `/api/websocket`; when a `command_event` with `state: "executing"` arrives, starts a smooth **ease-in-out drone animation** (4 seconds) via `requestAnimationFrame`
- Supports 4 toggleable map layers: **Drones**, **Incidents**, **Coverage heatmap**, **Sector grid**
- Passes `projectedCoveragePoints` (from scenario runs) and `highlightedSectorIds` (from geocode lookup) to `MapView`

### `components/map/MapView.tsx`

Leaflet map rendered with `react-leaflet`. Drone markers are colored by battery tier (green >50%, yellow 20–50%, red <20%). Stale drones render at 50% opacity with a dashed border. Incident markers are color-coded by priority (red=critical, amber=medium, gray=low). Includes `SectorGridOverlay` (9-sector grid), `HeatMapOverlay` (coverage heatmap), and sector highlighting rectangles.

### `components/fleet-control/FleetControlPanel.tsx`

Command dispatch UI with two modes toggled by a button group:

- **Move to sector** — dropdown of pre-established sectors (sector-1 through sector-9). Sends `POST /api/commands` with `targetSectorId`.
- **Dispatch to incident** — fetches live incidents from `/api/incidents` and shows them as `[type] id — priority N`. Sends `POST /api/commands` with `targetPosition` (incident's GPS coordinates). Includes a "Refresh incidents" link.

Listens on SSE for live command state updates and renders the `CommandTimeline` showing the command lifecycle (queued → sent → acknowledged → executing → completed/failed) for the selected drone.

### `components/chat/ChatPanel.tsx`

Jarvis chat interface. Sends messages to `POST /api/chat`, displays `role: assistant` responses in a scrollable history, and renders recommendation cards below with one-click "Apply plan" hooks that call `POST /api/scenarios/apply`.

### `components/demo/ScenarioPicker.tsx`

Landing scenario picker. Fetches available demo scenarios from `GET /api/demo/scenarios`. For non-default selections, calls `POST /api/demo/load-scenario` to reset and reseed the twin store. Stores the chosen scenario in `sessionStorage` (key: `klout:demoScenarioId`) so the picker is skipped on subsequent visits.

### `components/live-ops/LiveOpsControls.tsx`

Telemetry simulator controls: start, pause, reset buttons. Speed slider (0.25×–4×). Anomaly preset buttons (battery drain spike, comms loss, sensor degradation, coverage stress). Shows current simulator state and tick count.

### `components/live-ops/IncidentGeneratorMonitor.tsx`

LLM incident generator monitor. Shows health (healthy/degraded/disabled), last generated incident summary, last error. Controls to enable/disable, adjust interval, set allowed types and severity range.

### `components/incidents/IncidentQueue.tsx`

Live incident table. Columns: ID, type, sector, priority, severity, state, SLA status, assigned asset. State transitions happen via `PATCH /api/incidents/[id]`. SLA countdown badges change colour based on time remaining.

### `components/ui/StatusPanel.tsx`

Fleet aggregate bar rendered at the top of the Live Ops dashboard. Shows: total drones, mission counts by type, coverage percentage, active incident count. Shows a degraded-mode warning banner when fleet connectivity drops below threshold.

### `components/twins-explorer/VehicleTwinsExplorer.tsx`

OOP class hierarchy browser. Fetches from `GET /api/twins/registry`. Renders a tree of twin classes with state field descriptors, incoming event schemas (telemetry), outgoing event schemas (mission_status), and available commands.

### `components/demo-readiness/DemoReadinessPanel.tsx`

Self-test checklist panel. Calls `GET /api/demo-readiness` which checks: twin store has drones and incidents, simulator is available, incident generator is configured, AI service responds, SSE stream is reachable. Each check is displayed with a pass/fail badge.

---

## 7. Data Files

### `data/scenarios/demo-scenarios.json`

Four demo scenarios for the landing picker:

| ID | Label | Setup |
|---|---|---|
| `default` | Default | 4 drones, 2 incidents (built-in seed, no POST call) |
| `light` | Light Load | 2 drones, 1 incident (inline data) |
| `heavy` | Heavy Load | 6 drones, 5 incidents, 1 drone with degraded comms |
| `precanned-run` | Pre-canned Outcome | Loads persisted run `run-33a177ee`, seeds from its plan + incidents |

### `data/scenarios/scenarios.json`

Scenario definitions (scenario ID + `scenarioInput`) used by the Scenario Studio and simulation engine.

### `data/scenarios/baseline.json`

Baseline (no digital twin) scenario used for the before/after comparison in Scenario Studio.

### `data/scenarios/runs/*.json`

Four persisted simulation runs (`run-33a177ee`, `run-41bd34bf`, `run-56c3d30d`, `run-e80d648f`). Each file contains the full `PersistedScenarioRun` envelope: `scenarioInput`, `plan`, `outcome` (coverage %, gaps, unserved incidents, risk indicators), and provenance metadata.

### `data/fixtures/`

Test fixtures for anomaly detectors: `anomalies.json`, `battery-telemetry.json`, `comms-telemetry.json`, `coverage-telemetry.json`.

---

## 8. Real-time Architecture

### Polling

`MapContainer` polls three endpoints every **1500ms** in parallel:

- `GET /api/drones` — drone states (position, battery, mission, comms, sensors)
- `GET /api/incidents` — all active incidents
- `GET /api/fleet/status` — connectivity health, staleness, anomaly events, operator alerts

### SSE Stream (`/api/websocket`)

All real-time push events flow through a single SSE endpoint. Event envelope:

```json
{ "type": "<event_type>", "payload": { ... } }
```

| Event type | Fired when | Consumer |
|---|---|---|
| `connected` | Client connects | — |
| `heartbeat` | Every 25 seconds | — |
| `command_event` | Command state transitions | `FleetControlPanel` (timeline), `MapContainer` (animation trigger) |
| `incident_event` | Incident created/updated | — |
| `incident_generator_event` | Generator status change | `IncidentGeneratorMonitor` |

### Drone Movement Animation

When a `command_event` with `state: "executing"` and `type: "move"` arrives in `MapContainer`:

1. The target position is resolved (`targetPosition` directly, or `targetSectorId` → sector center lookup)
2. The drone's current position is captured (or the current interpolated position if already animating)
3. A `requestAnimationFrame` loop starts, interpolating lat/lon with an **ease-in-out** curve over **4 seconds**
4. The drone marker on the Leaflet map moves smoothly to the destination
5. After the animation completes, the server's position (from the 1500ms poll) takes over — they coincide since the server already moved the drone to the same target

---

## 9. Epic & Story Completion Summary

All 8 epics are **done**. 48 stories shipped.

### Epic 1 — Live Fleet & Incident View (Digital Twin Foundation)
- `1-1` Define drone/incident twin schemas and in-memory store
- `1-2` Ingest telemetry and update digital twin state
- `1-3` Handle partial data loss and expose degraded-mode flags
- `1-4` Implement map container with incidents and drone markers
- `1-5` Coverage heatmap view, layer toggles, and status panel

### Epic 2 — Anomaly Detection & Resilient Operations
- `2-1` Define anomaly and risk-scoring schemas
- `2-2` Implement battery and energy anomaly detection
- `2-3` Implement comms and link health anomaly detection
- `2-4` Implement coverage gap and fleet saturation detection
- `2-5` Aggregate anomalies into operator-facing alerts
- `2-6` Expose degraded-mode and alert state to the UI

### Epic 3 — Scenario Simulation & Before/After Comparison
- `3-1` Define scenario and plan data models
- `3-2` Implement simulation engine for a single plan
- `3-3` Compare baseline vs. digital-twin-enhanced runs
- `3-4` Support multi-plan A/B comparison for a single scenario
- `3-5` Persist and retrieve scenario runs for replay

### Epic 4 — Jarvis Conversational Decision Support
- `4-1` Define chat message and recommendation schemas
- `4-2` Implement chat API endpoint and basic message history
- `4-3` Integrate AI service to generate contextual responses
- `4-4` Surface recommendations with explanations and apply hooks
- `4-5` Adapt Jarvis behavior in degraded mode

### Epic 5 — Resource Coordination & Field Operator Support
- `5-1` Define resource request and delivery schemas
- `5-2` Implement resource request API for field operators
- `5-3` Assign drones or ground assets to fulfil requests
- `5-4` Provide mobile-friendly tracking and ETA view
- `5-5` Suggest alternative solutions when drone delivery fails
- `5-BONUS` Jarvis prompt integration and street-to-sector mapping

### Epic 6 — Ecosystem Readiness & Evaluator Experience
- `6-1` Implement `VehicleTwinBase` and `DroneTwin` classes
- `6-2` Define and document incoming/outgoing event schemas
- `6-3` Provide architecture and API overview for evaluators
- `6-4` Create example flow showing ecosystem integration concept
- `6-5` Vehicle Twins Explorer — OOP and extensibility showcase

### Epic 7 — Live Ops Closed-Loop Experience
- `7-1` Implement telemetry simulator and controls
- `7-2` Implement command and acknowledgement loop
- `7-3` Implement incident queue and lifecycle
- `7-4` Implement LLM incident generator and monitor

### Epic 8 — UI Navigation Overhaul & Demo Confidence
- `8-0` UI navigation and information architecture overhaul
- `8-1` Verify Live Ops dashboard and map interactions
- `8-2` Verify incident queue, fleet control, and command loop UI
- `8-3` Verify Jarvis chat, Scenario Studio, and before/after comparison UI
- `8-4` Verify Vehicle Twins Explorer and ecosystem readiness UI
- `8-5` Run demo readiness self-test and end-to-end UI checklist

---

## 10. Recent UI Enhancements

These features were added outside the formal epic/story structure, as direct UX improvements:

### Fleet Control — Pre-established Sector Dropdown

**File:** `components/fleet-control/FleetControlPanel.tsx`

Replaced the free-text sector ID input with a toggle between two command modes:

- **Move to sector** — a `<select>` dropdown pre-populated with the 9 system sectors (sector-1 through sector-9), matching the incident generator and simulation engine's sector definitions
- **Dispatch to incident** — fetches active incidents from `/api/incidents` and presents them as `[type] id — priority N`. On submit, sends `targetPosition` (the incident's GPS coordinates) directly to the command API. Includes a "Refresh incidents" link to reload without leaving the form.

### Map — Smooth Drone Movement Animation

**File:** `components/map/MapContainer.tsx`

When a move command transitions to `executing`, the map now animates the drone marker smoothly toward its target instead of teleporting. Implementation:

- `MapContainer` listens on the SSE stream for `command_event` with `state: "executing"` and `type: "move"`
- The target position is resolved (from `targetPosition` or by looking up the sector center for `targetSectorId`)
- A `requestAnimationFrame` loop interpolates the drone's lat/lon using an **ease-in-out** curve over 4 seconds
- Chained commands are handled: if a second command arrives while a drone is already animating, the new animation starts from the current interpolated position
- After animation completes, the server-side position (identical to the animation target) seamlessly takes over

### Demo Landing — Scenario Picker

**File:** `components/demo/ScenarioPicker.tsx`, `app/page.tsx`

Added a landing experience for evaluators. On first visit, a scenario picker is shown with four options. Non-default scenarios call `POST /api/demo/load-scenario`, which resets the twin store and seeds it from the selected scenario's definition (inline data or a persisted simulation run). Session state is persisted in `sessionStorage` to avoid showing the picker on refresh.
