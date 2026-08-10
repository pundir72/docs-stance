# Stance Clinical Summarizer and Phase Intelligence Platform

## System Architecture, Module Reference, and Optimization Assessment

**Reviewed branch:** `v27.5.3-prod`  
**Reviewed commit:** `cb25742`  
**Document version:** 1.0  
**Technical baseline:** Current production branch, runtime entry points, API routes, database access, AI integrations, containers, routing, and deployment workflow.

---

## 1. Executive summary

This project is an AI-assisted rehabilitation decision-support backend. It reads a patient's longitudinal clinical records and VALD measurements, builds a chronological timeline, and generates two independent clinical outputs:

1. A structured clinical summary describing the patient's condition, evidence, progress, risks, adherence, and gaps.
2. A rehabilitation phase analysis comparing expected rehabilitation stages with actual activity and progress.

A third service, the Audit Orchestrator, watches MongoDB activity and decides when either AI pipeline should run. Nginx is the public HTTPS and WebSocket gateway.

### Solution overview

> Stance converts fragmented clinical notes and objective VALD measurements into one chronological patient view. It then uses specialized AI pipelines to create a concise clinical summary and a phase-wise rehabilitation assessment. A central orchestration service controls when generation occurs, while the final structured results are stored in MongoDB for clinician review.

The product should be described as clinician-reviewed decision support, not autonomous diagnosis or treatment.

### Current technical assessment

The core clinical workflow is implemented and the three-service separation is sensible. However, the current branch has material reliability, security, deployment, and cost-efficiency gaps:

- A successful HTTP dispatch is treated as a completed AI job even though both downstream endpoints only accept background work.
- Shared queue fields are cleared after one pipeline runs, which can incorrectly mark or suppress the other pipeline.
- A combined automatic run performs approximately four clinical-history reads, three VALD reads, and about eight normal AI generations per patient.
- The concise-summary stage repeats AI processing section by section and is the largest avoidable AI-call multiplier.
- Full source records are nested repeatedly in the timeline sent to the model, increasing prompt size and patient-data exposure.
- Several disconnected functions, old CLI paths, stale deployment files, and unused dependencies remain in the repository.
- Production credentials and a private key are tracked in Git, and one source file contains a hardcoded database fallback credential. All affected credentials must be rotated.
- APIs have no application authentication and broad CORS configuration.
- There are no automated tests.
- The CI workflow does not actually select the dev Compose definition for dev branches.

The priority-zero and priority-one items in this document should be addressed before treating the platform as fully optimized and production-hardened.

---

## 2. The three production parts

| Service | Port | Primary responsibility | Uses AI | Persistent output |
|---|---:|---|---|---|
| Clinical Auditor | 5001 | Builds an evidence-informed clinical assessment and concise summary | Yes | Configured summary collection, currently `new-summary` in root Compose |
| Phase Analysis | 5002 | Determines ideal-versus-actual rehabilitation phases | Yes | Configured phase collection, normally `patient-phases` |
| Audit Orchestrator | 5003 | Watches activity, maintains a queue, applies thresholds, and dispatches pipelines | No | `pending_audits`, resume tokens, and estimated cost reports |

Nginx is supporting infrastructure rather than a fourth business service. It terminates TLS and routes public summary, phase, orchestrator, and WebSocket traffic.

### Difference between the three

- The Clinical Auditor answers: **What is the patient's overall clinical state and rehabilitation story?**
- Phase Analysis answers: **Where is the patient in the rehabilitation pathway, and what phase work is missing or delayed?**
- The Orchestrator answers: **Should either analysis run now, and how should that work be queued?**

The first two services interpret clinical data. The orchestrator does not interpret clinical data; it controls execution.

---

## 3. Service implementation and integration reference

This section provides the direct answer to which service uses each technology or integration and why it is used.

### 3.1 Integration coverage by service

| Capability or integration | Clinical Auditor | Phase Analysis | Audit Orchestrator | Purpose |
|---|---|---|---|---|
| FastAPI | Yes | Yes | Yes | REST and WebSocket application framework |
| Uvicorn | Yes | Yes | Yes | Runs each FastAPI application |
| MongoDB/PyMongo | Yes | Yes | Yes | Reads source data, stores generated results, and maintains orchestration state |
| `structured-user-reports` | Reads | Reads | No | Supplies the consolidated patient clinical history to both AI pipelines |
| `vald-exercise-data` | Reads | Reads | Watches changes | Supplies objective VALD measurements and triggers regeneration |
| `reports` | Indirectly through `structured-user-reports` | Indirectly through `structured-user-reports` | Watches changes | Raw clinical activity source used for trigger detection |
| Google Vertex AI through `google-genai` | Yes | Yes | No | Hosts the Gemini model calls |
| Gemini generation | EBA and concise stages | Phase intelligence stage | No | Produces the clinical and phase interpretations |
| Google Search grounding | **Yes, EBA stage only** | **No** | **No** | Adds external evidence context and grounding sources to the comprehensive clinical assessment |
| HTTPX | Calls orchestrator | Calls orchestrator | Calls both AI services | Internal service-to-service communication |
| WebSockets | Summary trigger and status channel | Phase trigger and status channel | No direct frontend WebSocket | Frontend manual requests and progress/status messages |
| Google Search directly from application code | No standalone HTTP search | No | No | Search is invoked only as a Gemini tool inside the EBA generation configuration |
| Nginx | Routed through Nginx | Routed through Nginx | Routed under the summary domain | TLS termination and reverse proxying |

### 3.2 Google Search grounding: exact current usage

Google Search grounding is **not used by all three services**.

| Processing stage | Gemini used | Google Search grounding used | Current reason |
|---|---:|---:|---|
| Evidence-Based Assessment in Clinical Auditor | Yes | **Yes** | Supports evidence-informed clinical discussion and source references |
| Concise Summary in Clinical Auditor | Yes | **No** | Condenses the already generated EBA content; it does not perform a new web evidence search |
| Phase Intelligence | Yes | **No** | Determines rehabilitation phases using the patient timeline and the phase instructions embedded in its prompt |
| Audit Orchestrator | No | **No** | Contains scheduling and queue logic only; it makes no model call |
| Timeline and VALD transformers | No | **No** | Perform deterministic MongoDB loading and Python data transformation |

In the active code, grounding is enabled by passing `GoogleSearch` as a tool in the EBA Gemini generation configuration. The concise and phase configurations do not pass a search tool. Therefore, the correct answer is:

> Google Search grounding is used only for the comprehensive Evidence-Based Assessment within the Clinical Auditor. It is not used by the concise-summary stage, Phase Analysis, or the Audit Orchestrator.

### 3.3 Clinical Auditor service reference

| Item | Current implementation |
|---|---|
| Service entry point | `api/summary_api.py` |
| Internal port | 5001 |
| Main REST trigger | `POST /api/audits` |
| Manual frontend channel | `/ws/report-gen` |
| Main coordinator | `LLM/report-gen/run_audit_by_stance_id.py` |
| Data preparation | `load_reports_from_mongo.py` -> `chronological_data_processor.py` -> `agent_data_pipeline.py` |
| AI stage 1 | `eba_agent.py`: comprehensive evidence-based assessment with Google Search grounding |
| AI stage 2 | `concise_agent.py`: section-wise condensation without Google Search grounding |
| Parser/persistence | `push_report_to_mongo.py` |
| Input collections | `structured-user-reports`, `vald-exercise-data` |
| Output collection | `SUMMARY_COLLECTION`; root production Compose defaults it to `new-summary` |
| Normal AI calls | Approximately 1 EBA call plus up to 6 concise calls |
| Output purpose | Overall patient status, clinical evidence, progress, concerns, adherence, information gaps, and references |

Clinical Auditor processing sequence:

```text
Patient ID
  -> load clinical history
  -> load and normalize VALD measurements
  -> build chronological/hierarchical timeline
  -> generate grounded EBA report with Gemini
  -> condense selected report sections with Gemini
  -> parse final Markdown into structured fields
  -> upsert one summary document by patient ObjectId
```

The grounding step belongs to the first EBA call. The later concise calls operate on the EBA report and do not search the web.

### 3.4 Phase Analysis service reference

| Item | Current implementation |
|---|---|
| Service entry point | `api/phase_analysis_ws.py` |
| Internal port | 5002 |
| Main REST trigger | `POST /phase-analysis` |
| Manual frontend channel | `/ws/phase-analysis` |
| Main engine | `LLM/phase-analysis/phase_intelligence_engine.py` |
| Data preparation | Reuses `LLM/shared/agent_data_pipeline.py` and its clinical/VALD loaders |
| AI stage | One phase-intelligence Gemini generation under normal conditions |
| Google Search grounding | Not enabled |
| Parser/persistence | `LLM/phase-analysis/push_phases_to_mongo.py` |
| Input collections | `structured-user-reports`, `vald-exercise-data` |
| Output collection | `PHASES_COLLECTION`, normally `patient-phases` |
| Output purpose | Expected and actual phases, session coverage, phase status, exit criteria, missing work, immediate priorities, future phases, and mismatch alerts |

Phase Analysis processing sequence:

```text
Patient ID
  -> build the same clinical and VALD timeline
  -> construct the rehabilitation-phase prompt
  -> generate phase report with Gemini without Google Search
  -> save Markdown/CSV artifacts
  -> parse and validate the report
  -> upsert one patient-phases document by patient ObjectId
```

### 3.5 Audit Orchestrator service reference

| Item | Current implementation |
|---|---|
| Service entry point | `orchestrator/main.py` |
| Internal port | 5003 |
| AI or Gemini usage | None |
| Google Search grounding | None |
| MongoDB watcher | `orchestrator/watcher.py` |
| Queue/state management | `orchestrator/queue.py` |
| Dispatch and cost estimation | `orchestrator/scheduler.py` |
| Operational API | `orchestrator/api.py` |
| Watched collections | `reports`, `vald-exercise-data` |
| Supporting collections | `appointments`, `pending_audits`, `orchestrator_resume_tokens`, `orchestrator_cost_reports` |
| Downstream calls | Clinical Auditor `/api/audits`; Phase Analysis `/phase-analysis` |
| Current automatic rules | Five distinct report edits, five distinct VALD edits, or the configured 15-day fallback with pending activity |

The orchestrator prevents a model call from being made after every small database save. It deduplicates source record identifiers, checks configured thresholds, and limits concurrent dispatches. It does not read the complete patient timeline and does not perform clinical reasoning.

### 3.6 Supporting infrastructure

| Component | Used for | Not used for |
|---|---|---|
| Nginx | HTTPS, certificate termination, REST routing, WebSocket upgrades | Clinical logic, AI calls, or queue management |
| Docker Compose | Starts and networks the three services and Nginx | Application-level scheduling |
| GitHub Actions | Copies code to EC2 and rebuilds/restarts containers | Clinical processing |
| AWS EC2 | Hosts the containers | AI model hosting; Gemini is accessed through Google Vertex AI |
| MongoDB Atlas | Source data, generated outputs, queue and operational metadata | Model inference |
| Google Vertex AI | Gemini inference and EBA search grounding | Application hosting or MongoDB storage |

### 3.7 Gemini models and fallback behavior

| Stage | Preferred model | Configured fallbacks | Attempts per model |
|---|---|---|---:|
| EBA | `gemini-3.1-pro-preview` | `gemini-3-pro-preview`, `gemini-3-flash-preview`, `gemini-3.1-flash-lite-preview`, `gemini-2.5-flash` | 2 |
| Concise Summary | `gemini-3.1-pro-preview` | `gemini-3-pro-preview`, `gemini-3-flash-preview`, `gemini-2.5-flash` | 2 |
| Phase Intelligence | `gemini-3.1-pro-preview` | `gemini-3-flash-preview`, `gemini-3.1-flash-lite-preview`, `gemini-2.5-flash` | 3 |

Fallback occurs for configured rate-limit and server-error conditions. Phase also skips to the next model when a configured model is not found. Because preview model availability can change, production should use an approved model-version policy and verify availability during startup or deployment.

### 3.8 Module ownership and call map

| Service or stage | Module called | Why it is called | Calls next |
|---|---|---|---|
| Clinical Auditor API | `api/summary_api.py` | Receives REST/WebSocket requests, reports job state, and delegates manual requests to the orchestrator | `run_audit_by_stance_id.py` for orchestrator-triggered execution |
| Summary coordinator | `LLM/report-gen/run_audit_by_stance_id.py` | Coordinates data loading, EBA, concise output, artifacts, parsing, and persistence | Shared data pipeline, EBA agent, concise agent, summary pusher |
| Shared patient-data loader | `LLM/shared/load_reports_from_mongo.py` | Reads and normalizes clinical reports from `structured-user-reports` | Chronological processor |
| Shared VALD loader | `LLM/shared/fetch_and_transform_vald.py` | Reads, filters, and normalizes objective measurements | Chronological processor |
| Shared timeline merger | `LLM/shared/chronological_data_processor.py` | Sorts clinical and VALD events and creates the hierarchical patient timeline | Agent data pipeline |
| Shared agent payload builder | `LLM/shared/agent_data_pipeline.py` | Packages the timeline in the structure expected by both AI engines | EBA agent or Phase Intelligence engine |
| EBA stage | `LLM/report-gen/eba_agent.py` | Creates the comprehensive clinical assessment and obtains grounded evidence | Concise agent |
| Concise stage | `LLM/report-gen/concise_agent.py` | Reduces verbose EBA sections while retaining required clinical fields | Summary parser/pusher |
| Summary persistence | `LLM/report-gen/push_report_to_mongo.py` | Parses Markdown, validates data, maps frontend fields, and upserts the summary | MongoDB summary collection |
| Phase API | `api/phase_analysis_ws.py` | Receives phase requests, manages in-memory phase jobs, runs analysis, validates, and pushes results | Phase engine and phase pusher |
| Phase engine | `LLM/phase-analysis/phase_intelligence_engine.py` | Uses the shared timeline to generate ideal-versus-actual rehabilitation phases | Phase parser/pusher |
| Phase persistence | `LLM/phase-analysis/push_phases_to_mongo.py` | Parses and validates phase tables/sections and upserts structured phase data | MongoDB phase collection |
| Orchestrator bootstrap | `orchestrator/main.py` | Starts queue, watcher, scheduler, API, and shutdown handling | All orchestrator modules |
| Trigger watcher | `orchestrator/watcher.py` | Detects relevant MongoDB changes and center eligibility | Audit queue |
| Queue state | `orchestrator/queue.py` | Deduplicates edits and maintains patient processing state | Scheduler |
| Scheduler | `orchestrator/scheduler.py` | Checks eligibility, dispatches downstream requests, retries HTTP handoff, and estimates cost | Summary and phase REST APIs |
| Orchestrator API | `orchestrator/api.py` | Provides queue inspection, manual trigger, forced dispatch, stats, and cost endpoints | Scheduler and queue |

`rehabilitation_phases.py`, the two export scripts, old direct WebSocket runners, and the obsolete root Dockerfile are not part of this active call map. Their disposition is documented in the legacy-code section.

### 3.9 API and WebSocket reference

#### Clinical Auditor

| Route | Method/type | Current purpose |
|---|---|---|
| `/health` | GET | Process-level health response |
| `/status` | GET | Service configuration and status information |
| `/api/patients` | GET | Lists available patients from the structured report source |
| `/api/patients/{patient_id}` | GET | Returns patient metadata and clinical-record count |
| `/api/audits` | POST | Accepts an orchestrator-triggered summary background job |
| `/api/up-to-date` | POST | Broadcasts that a summary requires no regeneration |
| `/api/jobs`, `/api/jobs/{job_id}` | GET | Reads in-memory summary job state |
| `/api/patients/{patient_id}/jobs` | GET | Reads in-memory jobs for one patient |
| `/api/jobs/{job_id}` | DELETE | Marks an in-memory job cancelled; it does not terminate running inference |
| `/api/jobs/reset` | POST | Resets in-memory job tracking |
| `/api/queue`, `/api/instance/status` | GET | Operational/debug information |
| `/api/audits/{patient_id}` | GET | Retrieves audit/job information for a patient |
| `/api/reports`, `/api/reports/{filename}` | GET | Lists or reads generated local report artifacts |
| `/api/summaries`, `/api/summaries/{patient_id}` | GET | Reads persisted summary documents |
| `/ws/report-gen` | WebSocket | Main frontend summary trigger; delegates manual requests to the orchestrator |
| `/ws/audit/{patient_id}` | WebSocket | Older patient-specific stream; current implementation has an await/connect defect |

#### Phase Analysis

| Route | Method/type | Current purpose |
|---|---|---|
| `/health` | GET | Process-level health response |
| `/phase-analysis` | POST | Accepts an orchestrator-triggered phase background job |
| `/up-to-date` | POST | Intended phase up-to-date notification; currently broken because `broadcast` is undefined |
| `/api/phase-jobs` | GET | Lists in-memory phase jobs |
| `/api/phase-jobs/{job_id}` | GET | Reads one in-memory phase job |
| `/ws/phase-analysis` | WebSocket | Main frontend phase trigger; delegates manual requests to the orchestrator |

#### Audit Orchestrator

| Route | Method | Current purpose |
|---|---|---|
| `/health` | GET | Orchestrator process-level health response |
| `/stats` | GET | Scheduler and queue statistics |
| `/queue` | GET | Lists queue records |
| `/queue/{patient_id}` | GET | Reads one patient's queue state |
| `/force-dispatch` | POST | Forces an eligible pending row to dispatch |
| `/manual-trigger` | POST | Handles frontend summary/phase generation requests and staleness checks |
| `/cost-report` | GET | Returns stored static daily cost estimates |

### 3.10 Internal service calls

| Caller | Destination | Request | Reason |
|---|---|---|---|
| Summary WebSocket API | Orchestrator | `POST /manual-trigger`, source `summariser` | Central staleness decision and controlled execution |
| Phase WebSocket API | Orchestrator | `POST /manual-trigger`, source `phase` | Central staleness decision and controlled execution |
| Orchestrator | Clinical Auditor | `POST /api/audits` | Starts summary generation |
| Orchestrator | Phase Analysis | `POST /phase-analysis` | Starts phase generation |
| Orchestrator | Clinical Auditor | `POST /api/up-to-date` | Notifies summary clients that no generation is required |
| Orchestrator | Phase Analysis | `POST /up-to-date` | Intended equivalent phase notification; currently returns an error |

### 3.11 Principal configuration by service

| Service | Important settings | Why they exist |
|---|---|---|
| Clinical Auditor | `MONGO_URI`, `MONGO_DB`, `SUMMARY_COLLECTION`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_APPLICATION_CREDENTIALS`, `MAX_CONCURRENT_AUDITS`, `ORCHESTRATOR_URL` | Database access, output destination, Vertex AI authentication, concurrency, and orchestration address |
| Phase Analysis | `MONGO_URI`, `MONGO_DB`, `PHASES_COLLECTION`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_APPLICATION_CREDENTIALS`, `PHASE_WS_PORT`, `ORCHESTRATOR_URL` | Database access, output destination, Vertex AI authentication, port, and orchestration address |
| Audit Orchestrator | `MONGO_URI`, collection names, thresholds, fallback interval, pipeline list, downstream URLs, dispatch concurrency/timeouts/attempts, allowed center IDs, static unit-cost values | Change detection, eligibility policy, routing, resilience, access filtering, and cost estimation |

`DISABLE_CHANGE_STREAMS` appears in root Compose for the two AI services, but no active Python module reads it. Change streams are owned only by the orchestrator in the current architecture.

---

## 4. End-to-end runtime flow

### Automatic generation

```text
MongoDB change in reports or vald-exercise-data
  -> ChangeStreamWatcher validates patient/center and deduplicates the source ID
  -> AuditQueue updates one pending_audits row per patient
  -> Scheduler checks 5-report, 5-VALD, or 15-day fallback eligibility
  -> Scheduler calls Clinical Auditor and Phase Analysis concurrently
  -> Each service independently reloads and transforms the patient history
  -> Gemini generates its output
  -> Parser validates and upserts structured output into MongoDB
```

### Manual generation

```text
Frontend opens the summary or phase WebSocket
  -> API sends /manual-trigger to the orchestrator with a pipeline name
  -> Orchestrator checks its queue timestamps and pending edit arrays
  -> It either sends an up-to-date response or calls the selected REST endpoint
  -> REST endpoint accepts a background task and immediately returns 2xx
```

### Source and destination collections

| Collection/view | Usage |
|---|---|
| `reports` | Raw clinical-record changes watched by the orchestrator |
| `structured-user-reports` | Consolidated clinical history loaded by both AI pipelines |
| `vald-exercise-data` | VALD source changes and objective measurements |
| `appointments` | Center eligibility lookup for the orchestrator |
| `pending_audits` | One orchestration state row per patient |
| `new-summary` or configured summary collection | Parsed clinical summaries |
| `patient-phases` or configured phase collection | Parsed phase analysis |
| `orchestrator_resume_tokens` | MongoDB change-stream recovery tokens |
| `orchestrator_cost_reports` | Static estimated daily generation cost |

The watcher observes `reports`, while the AI loaders read `structured-user-reports`. This appears to rely on an external aggregation/view process. That dependency is not created or managed by this repository.

---

## 5. Module-by-module inventory

Status meanings:

- **Active:** Called by the deployed runtime.
- **Supporting:** Used for deployment, routing, configuration, or operations.
- **CLI only:** Usable manually but not part of the deployed request path.
- **Legacy/disconnected:** Present but not reached by the current deployed flow.
- **Broken/obsolete:** References missing or inconsistent runtime components.

### 5.1 API modules

| Module | Status | Responsibility | Review notes |
|---|---|---|---|
| `api/summary_api.py` | Active, with legacy sections | FastAPI service for summary REST routes, WebSockets, background jobs, patient/report queries, and orchestration delegation | Very large mixed-responsibility module. Contains unused old direct WebSocket runners, unused patient locks, in-memory jobs, and a failed import in its preliminary data-preparation step. |
| `api/phase_analysis_ws.py` | Active, with legacy sections | FastAPI service for phase REST trigger, WebSocket trigger delegation, background execution, validation, persistence, and jobs | The REST background job does not broadcast progress to delegated WebSocket clients. `/up-to-date` calls a nonexistent `broadcast` method. Old direct WebSocket runner is disconnected. |

Important API behavior:

- `POST /api/audits` and `POST /phase-analysis` are fire-and-forget endpoints. Their 2xx responses mean **accepted**, not **completed**.
- Jobs live only in process memory. Restarting a container loses job history and state.
- The cancel endpoint changes a job status but does not stop an already running thread/model request.
- Health checks prove that the web process responds; they do not prove MongoDB or Vertex AI readiness.
- Summary and phase WebSocket protocols use ad hoc colon-delimited text rather than a versioned JSON event schema.
- `GET /api/reports/{filename}` exposes server-side generated report artifacts by filename. It should be removed or strictly authorized if artifacts are retained.

### 5.2 Clinical-summary modules

| Module | Status | Responsibility | Review notes |
|---|---|---|---|
| `LLM/report-gen/run_audit_by_stance_id.py` | Active and CLI | Main summary workflow coordinator | Loads patient data more than once, invokes the shared pipeline, writes Markdown artifacts, rereads a file for parsing, and contains a hardcoded MongoDB fallback credential. |
| `LLM/report-gen/eba_agent.py` | Active, with legacy methods | Builds the evidence-based assessment prompt and calls Gemini with Google Search grounding | One normal grounded generation, but full-report validation can repeat it. Large prompt, layered fallbacks, stale 2020–2024 research instruction, and unused earlier generation/helper methods. |
| `LLM/report-gen/concise_agent.py` | Active | Parses the EBA Markdown into sections and asks Gemini to condense verbose sections | Processes multiple sections independently, including parent sections and extracted subsections. This is the main AI-call and latency multiplier. Per-section output ceiling is 65,536 tokens. |
| `LLM/report-gen/push_report_to_mongo.py` | Active and CLI | Parses concise Markdown, normalizes fields, validates patient ID/sections, and upserts the summary | More than 1,000 lines and tightly coupled to exact Markdown headings. A schema-driven model response would remove much of this parsing risk. |
| `LLM/report-gen/export_patient_raw_data.py` | CLI only | Exports patient clinical data to JSON | Not imported by production. Uses path assumptions intended for manual execution. |
| `LLM/report-gen/export_vald_data.py` | CLI only / duplicate | Independently fetches and exports VALD data | Duplicates functionality available in the active shared VALD transformer. Candidate for removal or relocation to a tools directory. |

### 5.3 Shared data-processing modules

| Module | Status | Responsibility | Review notes |
|---|---|---|---|
| `LLM/shared/load_reports_from_mongo.py` | Active | Reads `structured-user-reports` and transforms embedded reports into clinical records | Stores extracted fields and the entire original report under `raw_data`, duplicating content. TLS certificate and hostname verification are disabled. |
| `LLM/shared/fetch_and_transform_vald.py` | Active | Reads `vald-exercise-data`, removes invalid zero-force metrics, and normalizes exercises/sessions | Creates repeated exercise/session structures and prints extensive debug information. TLS verification is disabled. |
| `LLM/shared/chronological_data_processor.py` | Active, with unused methods | Combines clinical and VALD events into chronological and hierarchical timelines | Active timeline creation is useful; `generate_chronological_context` and `save_chronological_report` are not called by production. Creates another loader/connection per instance. |
| `LLM/shared/agent_data_pipeline.py` | Active, with CLI-only helpers | Converts the merged patient timeline into the agent payload used by EBA and Phase Intelligence | Saves JSON artifacts on every production run. `get_agent_ready_data` is disconnected; `stream_to_agent` and `save_agent_payload` are only used by its manual entry point. |
| `LLM/shared/rehabilitation_phases.py` | Legacy/disconnected | Defines deterministic rehabilitation phase data structures and matching utilities | No current module imports or calls it. The live phase pipeline instead embeds its rehabilitation logic in the Gemini prompt. |

### 5.4 Phase-analysis modules

| Module | Status | Responsibility | Review notes |
|---|---|---|---|
| `LLM/phase-analysis/phase_intelligence_engine.py` | Active and CLI | Builds the shared patient timeline, prompts Gemini for phase intelligence, and writes Markdown/CSV reports | One normal model generation. Production API writes both Markdown and CSV even though persistence parses the in-memory text. |
| `LLM/phase-analysis/push_phases_to_mongo.py` | Active, with CLI-only paths | Parses, validates, and upserts phase output | API uses parsing and push functions directly. File-processing entry point is CLI only; text-processing wrapper is used by the engine's CLI path, not the API path. |

### 5.5 Orchestrator modules

| Module | Status | Responsibility | Review notes |
|---|---|---|---|
| `orchestrator/main.py` | Active | Initializes config, queue indexes, recovery, watcher threads, scheduler, and FastAPI/Uvicorn | Correct runtime entry point for the orchestrator container. |
| `orchestrator/config.py` | Active | Reads thresholds, URLs, collections, concurrency, centers, ports, and static cost values | Defaults are operational policy embedded in code; production should validate all required configuration at startup. |
| `orchestrator/watcher.py` | Active | Watches `reports` and `vald-exercise-data`, filters by center, extracts patient/source IDs, and persists resume tokens | Center lookup cache has no invalidation. A patient whose appointment/center changes can retain a stale authorization result until restart. |
| `orchestrator/queue.py` | Active | Maintains per-patient pending/in-flight state, edit deduplication, threshold queries, retry state, and timestamps | One shared pair of edit arrays is used for two pipelines. This cannot accurately represent separate summary and phase checkpoints. |
| `orchestrator/scheduler.py` | Active | Dispatches ready work, handles manual triggers, performs external-summary supersession, and writes cost estimates | Marks dispatch complete on downstream acceptance; creates a new HTTP client per call; cost logic assumes phase count equals summary count. |
| `orchestrator/api.py` | Active | Exposes health, stats, queue inspection, force dispatch, manual trigger, and cost report routes | Operational and mutation routes have no authentication. Queue documents are returned with minimal access control or redaction. |
| `orchestrator/log.py` | Active | Central logging setup | Small and appropriate, but application code still uses many direct `print` calls. |
| `orchestrator/__init__.py` | Active package marker | Package metadata/documentation | No material runtime logic. |

### 5.6 Deployment and operational files

| File | Status | Responsibility | Review notes |
|---|---|---|---|
| `docker-compose.yml` | Active deployment definition | Runs the three prod-named services plus Nginx | Exposes all three service ports publicly in addition to Nginx. Uses source-code bind mounts in production, reducing image immutability. |
| `services/clinical-auditor/Dockerfile` | Active | Python 3.13 clinical service image | Installs a broad, unpinned dependency set; source under `LLM` is supplied only by a runtime bind mount. |
| `services/phase-analysis/Dockerfile` | Active | Python 3.13 phase service image | Same dependency and image-reproducibility concerns. |
| `services/audit-orchestrator/Dockerfile` | Active | Python 3.11 orchestrator image | Dependencies are pinned and health check is meaningful at HTTP-process level. |
| `nginx/nginx.conf` | Active | TLS termination, REST proxying, and WebSocket upgrades | Only prod hostnames are configured. CI attempts certificates for additional dev names that Nginx does not serve. |
| `.github/workflows/deploy.yml` | Active, defective environment selection | Syncs repository to EC2, writes environment values, and rebuilds Compose | Both dev and prod branches select `docker-compose.yml`; environment detection does not change the deployed stack. Every deploy runs `docker compose down`, causing full-stack downtime. |
| `deploy.sh` | Supporting/manual | Interactive branch creation/deployment helper and optional EC2 security-group changes | Hardcodes a host/user and expects a local private key. It derives version-bump type from changed paths; it is not the CI deployment entry point. |
| `promote.sh` | Likely legacy/inconsistent | Intended to tag dev images as prod and provide image backups | Refers to dev containers/images not created by active root Compose. Rollback loads an image twice and restarts both services without independently restoring/tagging both images correctly. |
| `cleanup_ec2.sh` | Supporting/manual | EC2 cleanup utility | Operational script; should be reviewed and run only with explicit production procedure. |
| `docker-compose.dev.yml` | Legacy/incomplete | Defines only dev summary and phase services | Not selected by CI, contains no orchestrator or Nginx, and uses port mappings different from the active root stack. |
| `docker-compose.prod.yml` | Legacy/incomplete | Defines only prod summary and phase services | Not selected by CI and omits orchestrator and Nginx. |
| Root `Dockerfile` | Broken/obsolete | Earlier single-container phase cron image | Its command references missing `LLM/phase-analysis/cron_phase_analysis.py`. It is not used by active root Compose. |
| `bulk_ec2_executions/simulate_change_stream_load.py` | Legacy test utility | Simulates an older change-stream/load design | Documentation and assumptions reference the removed durable queue and old architecture. Not an automated current-stack test. |
| `ORCHESTRATOR_ARCHITECTURE.md` | Stale documentation | Describes earlier orchestration behavior | Contains earlier thresholds/sources and should not be used as the current operational specification. |

---

## 6. Active AI calls and cost structure

### 6.1 Normal model-generation count per patient

| Pipeline stage | Normal generations | Notes |
|---|---:|---|
| Evidence-Based Assessment | 1 | Uses Google Search grounding |
| Concise Summary | Up to approximately 6 | One generation for each selected verbose section |
| Phase Intelligence | 1 | Separate phase prompt and response |
| **Combined automatic run** | **Approximately 8** | This is the normal code-path estimate, before retries/fallbacks |

The code therefore does not make only one summary call and one phase call. The clinical summary is a multi-generation pipeline.

### 6.2 Retry amplification

- The EBA completeness loop may regenerate the entire report up to three times.
- EBA model fallback can try five model names with two attempts per model for qualifying failures.
- The concise workflow can retry its entire section pipeline up to three times when final validation fails.
- Each concise section also has its own retry and model-fallback behavior.
- Phase analysis can try four model names with three attempts per model for qualifying failures.

These are theoretical failure-path maxima, not normal billing counts. Their significance is that layered retries are not governed by one total attempt or cost budget. A provider incident can create a much larger burst than the orchestrator's dispatch count indicates.

### 6.3 Cost-report scope and limitations

The orchestrator uses fixed configured estimates of `$0.67` per summary and `$0.45` per phase, and records `$1.12` when both are assumed to run. It does not read model usage metadata, prompt tokens, output tokens, grounding charges, retries, model fallback choice, or failed-call consumption.

It also:

- Counts completed queue rows, although the queue is marked complete at request acceptance.
- Assumes phase runs equal summary runs whenever phase is globally configured.
- Cannot correctly represent manual single-pipeline runs.
- Reuses one patient queue row, so daily counts can miss or misclassify executions depending on timestamps.

The cost report is therefore an operational estimate only, not a billing reconciliation.

---

## 7. Unnecessary and duplicated work

### 7.1 Repeated MongoDB loading

For the active summary path:

1. `run_audit_by_stance_id.py` loads clinical history for patient metadata.
2. It constructs a `ChronologicalDataProcessor`, which loads clinical history and VALD data.
3. It then creates `AgentDataPipeline`, which constructs another processor and loads clinical history and VALD data again.

The phase service independently creates another `AgentDataPipeline` and reloads both datasets.

For an automatic combined summary-and-phase run, the code path therefore performs approximately:

- Four clinical-history reads.
- Three VALD reads.

This estimate excludes retry-related reruns. A single immutable patient snapshot could reduce these to one clinical read and one VALD read: about a 75% reduction in clinical-read count and a 67% reduction in VALD-read count for the combined path.

`summary_api.py` also tries a preliminary `from LLM.agent_data_pipeline import AgentDataPipeline`. The real file is under `LLM/shared`, so the import fails and is swallowed. It currently causes a predictable exception and misleading log entry on every summary. Fixing the import without removing duplication would add yet another complete data load.

### 7.2 Prompt and memory duplication

The clinical loader extracts useful fields but also attaches the complete source report as `raw_data`. The chronological processor nests that record under an event `data` field, and the agent transformer nests it again under `full_data`. The resulting structure is serialized for AI prompts.

Consequences:

- Larger prompt token usage and model cost.
- More application memory and serialization time.
- More patient information sent to the model than may be required.
- Harder debugging because the same facts exist at multiple nesting levels.

VALD processing also emits repeated exercise data across session-date events. The model should receive a compact, canonical measurement series rather than repeated full exercise objects.

### 7.3 Unnecessary filesystem work

- Both summary and phase pipelines save agent-ready JSON artifacts during production processing.
- Summary writes the full EBA Markdown and concise Markdown, then reparses the concise output from disk rather than using the in-memory string.
- Phase writes Markdown and CSV before parsing and saving the in-memory report to MongoDB.
- Summary and phase can write patient-named artifacts concurrently, creating overwrite/race and retention concerns.

Production should persist only explicitly required audit artifacts, use unique job IDs if retained, enforce encryption/retention, and otherwise parse responses in memory.

### 7.4 Unnecessary AI work

The concise agent extracts both main sections and subsections, then processes several verbose entries independently. Parent content such as the clinical audit and rehabilitation journey can overlap their extracted subsections, so the same source text is summarized more than once.

Best option: ask the initial EBA generation to return the final concise structured schema and eliminate the per-section agent. This changes the normal summary call count from roughly seven generations to one, an approximately 86% call-count reduction. Actual financial savings must be verified using token telemetry because model, grounding, and token prices differ.

If a long human-readable EBA is still a requirement, retain it as an optional artifact and create the final structured summary with one additional model call, not six section calls.

Google Search grounding is enabled for every EBA run. Evidence that is protocol-level rather than patient-specific should be refreshed on a controlled schedule and cached by diagnosis/protocol/version. Patient data should not be included in search-oriented context unless required and approved.

---

## 8. Reliability and correctness findings

### P0 — Dispatch acceptance is incorrectly treated as job completion

The orchestrator calls the two REST endpoints. Both immediately schedule background work and return 2xx. The orchestrator then calls `mark_done`, clears pending edits, and records completion before Gemini generation or MongoDB persistence has finished.

Impact:

- Failed background work can appear successful.
- Pending changes can be cleared without a result.
- Cost reports count acceptance, not completed generation.
- Retry logic protects only the HTTP handoff, not the actual AI work.

Required design: use a durable job ID and a completion callback/status poll, or let the downstream request remain open until persistence succeeds. Queue completion must occur only after verified persistence.

### P0 — Shared queue checkpoint corrupts single-pipeline state

`mark_done` always stamps `last_summary_generated` and clears both clinical and VALD edit arrays. It only stamps the phase timestamp conditionally.

Consequences:

- A phase-only run falsely records summary completion.
- A summary-only run clears edits that phase analysis still needs.
- Edits arriving while work is in flight can be erased.

Required design: maintain separate per-pipeline state, watermarks, attempts, and completion timestamps. A job should capture a source-version watermark and clear only events at or before that watermark after that specific pipeline succeeds.

### P0 — First manual request can be rejected as up to date

When a patient queue row is first inserted, both edit arrays are empty and both generation timestamps are `None`. `_is_up_to_date` returns true solely because the arrays are empty. The first manual generation can therefore return `up_to_date` even when no result exists.

Required design: the relevant result document or successful generation timestamp must exist before a pipeline can be considered current.

### P0 — Credentials in source control

The repository tracks service-account JSON files and a PEM private key. The audit runner also includes a hardcoded MongoDB URI fallback containing credentials.

Required action:

1. Revoke and rotate all affected Google, MongoDB, and SSH credentials.
2. Remove them from Git history, not only the latest commit.
3. Use a managed secret store or protected CI secrets with instance roles where possible.
4. Add secret scanning and pre-commit/CI prevention.

### P1 — Phase up-to-date notification is broken

The phase `ConnectionManager` implements `send` and `send_json`, but `/up-to-date` calls `manager.broadcast`. That route returns 500. The orchestrator does not call `raise_for_status` or otherwise inspect non-2xx responses in its notification method, so the failure is effectively hidden.

### P1 — Phase progress does not reach the triggering WebSocket

Manual WebSocket requests delegate to the orchestrator. The orchestrator then calls the phase REST endpoint, whose background job does not associate itself with or broadcast to the original connection. The frontend receives acceptance but not the documented phase progress sequence.

### P1 — Manual trigger can create duplicate execution

`manual_trigger` unconditionally writes queue status back to `pending`, including for a row already in flight. A second manual click can therefore interfere with or duplicate an active job.

### P1 — Partial pipeline retry can duplicate successful work

Summary and phase are dispatched concurrently. If one accepts and the other fails, the entire row returns to pending. A later retry can call both again because successful per-pipeline state is not persisted.

### P1 — External summary supersession likely uses the wrong type

The supersession query uses `{"patient": patient_id}` where `patient_id` is a string. The summary writer stores `patient` as a MongoDB `ObjectId`. This check will normally fail for ObjectId-backed documents.

### P1 — TLS verification is disabled in active loaders

Clinical and VALD Mongo clients set invalid-certificate and invalid-hostname allowances. This removes server identity protection and should not be used for MongoDB Atlas production traffic.

### P1 — API security boundary is absent in code

The REST and WebSocket endpoints contain no authentication or authorization. CORS permits all origins. Root Compose publishes ports 5001, 5002, and 5003 on the host, bypassing the expectation that Nginx is the only public entry point.

At minimum, bind internal ports only to the Docker network, authenticate at the gateway and/or services, authorize patient access, rate limit generation endpoints, restrict CORS, and record auditable actor identity.

### P1 — Deployment environment selection is incorrect

The workflow detects `-dev` versus `-prod`, but assigns `docker-compose.yml` in both cases and never uses its `compose_file` output. The root file defines prod-named containers and production collection variables. A dev-branch push therefore rebuilds the same root stack unless external conditions not represented in this repository intervene.

### P2 — Operational weaknesses

- Full `docker compose down` creates avoidable downtime on each deployment.
- Source bind mounts allow runtime code to differ from the built image.
- Mixed Python 3.11 and 3.13 versions increase compatibility variance.
- Jobs and connection state are in memory and grow without a retention policy.
- Per-patient lock objects are created but never used or pruned.
- Multiple `MongoClient` objects are created in one request instead of using process-level clients.
- A new `httpx.AsyncClient` is created for every scheduler and notification request.
- Extensive debug `print` output can expose patient metadata/content and increases log volume.
- Model prompts contain a fixed 2020–2024 evidence instruction, which is stale in 2026.
- Summary code logs the first portion of the MongoDB URI. Even partial credential-bearing URIs should never be logged.
- `/ws/audit/{patient_id}` awaits a synchronous `connect` method and is likely broken if used.

---

## 9. Dependency audit

The summary and phase requirements install the same broad dependency set. Static import review found no active application imports of:

- `langchain`
- `langchain-google-vertexai`
- `google-cloud-aiplatform`
- `google-api-python-client`
- direct `google-auth` APIs

The active AI implementation imports the `google-genai` SDK dynamically. `websockets` is directly used only by the old load-simulation script; FastAPI/Starlette handles the deployed WebSocket endpoints. These packages may include transitive/runtime needs, so removal must be confirmed in a clean container test, but they should not remain without an explicit reason.

Other dependency issues:

- Summary and phase directly import `httpx` but do not declare it. They currently rely on a transitive dependency.
- The load simulator imports `requests`, but no current service requirement explicitly declares it.
- Summary and phase dependencies use broad minimum versions instead of a lock file or hashes.
- `LLM/requirements.txt` duplicates service requirements, contains duplicate entries, and belongs to the obsolete root Dockerfile path.

Recommended action: create a small pinned requirement/lock set per service, declare all direct imports, build without runtime bind mounts, and run import plus endpoint smoke tests against the clean images.

---

## 10. Legacy and removal candidates

### Safe first candidates after a reference/test check

- Entire `LLM/shared/rehabilitation_phases.py` module.
- Unused `get_patient_lock` and associated lock globals.
- Old summary `run_audit_for_websocket` functions.
- Old phase `run_phase_analysis_ws` function.
- Unused EBA `_generate_comprehensive_web_assessment`, `_extract_key_metrics`, and `_build_vald_exercises_section` methods.
- Unused chronological `generate_chronological_context` and `save_chronological_report` methods.
- Unused production `AgentDataPipeline.get_agent_ready_data` method.
- Old `USE_FIRST_AGENT` and “first/second/third agent” branches when no first-agent implementation exists.
- `DISABLE_CHANGE_STREAMS` environment entries, which current Python code never reads.
- Obsolete root Dockerfile and duplicate root requirements.

### Move to a clearly separated `tools/` or `archive/` area if still needed

- `export_patient_raw_data.py`
- `export_vald_data.py`
- `simulate_change_stream_load.py`
- CLI-only report-file processing paths
- Old architecture documentation

### Do not delete without operational confirmation

- `deploy.sh`, `promote.sh`, and `cleanup_ec2.sh`, because external operators may still invoke them manually.
- Alternate Compose files, until the intended dev/prod deployment model is confirmed and replaced.

Legacy removal should follow characterization tests so that accidental external usage is not confused with internal imports.

---

## 11. Recommended target architecture

```text
Authenticated API / WebSocket gateway
  -> durable per-pipeline job record with idempotency key
  -> one patient snapshot service/load per source version
       -> compact validated clinical + VALD schema
       -> Summary worker: one structured Gemini generation
       -> Phase worker: one structured Gemini generation
  -> schema validation
  -> atomic MongoDB upsert
  -> worker completion event
  -> per-pipeline queue checkpoint
  -> JSON WebSocket status event
  -> actual token/model/grounding/cost telemetry
```

Key characteristics:

- One immutable snapshot is reused by both pipelines.
- Summary and phase have separate job IDs, status, retries, and source watermarks.
- Model output uses JSON schema/structured response rather than Markdown as an integration contract.
- Each output upsert includes source version, prompt version, model, usage, job ID, generated time, and validation status.
- Generation is idempotent for `(patient_id, pipeline, source_version, prompt_version)`.
- Completion means validated MongoDB persistence, not HTTP acceptance.
- Search grounding is conditional and protocol evidence is cached separately from patient context.

---

## 12. Prioritized optimization plan

### Phase 0 — Immediate security containment

1. Rotate exposed credentials and remove secrets from Git history.
2. Remove the hardcoded database URI fallback and fail startup when required secrets are absent.
3. Stop logging connection strings and patient-content debug data.
4. Restrict host port exposure; route external traffic only through the protected gateway.
5. Add authentication, patient-level authorization, CORS allowlists, rate limits, and audit identity.
6. Restore valid TLS certificate and hostname verification.

### Phase 1 — Correctness before cost work

1. Replace acceptance-based completion with durable downstream job completion.
2. Split queue state by pipeline and introduce source-version watermarks.
3. Fix first-generation staleness logic.
4. Make trigger and persistence operations idempotent.
5. Preserve edits that arrive during an in-flight job.
6. Fix phase up-to-date/progress delivery and ObjectId supersession query.
7. Prevent manual triggers from resetting an active job.

### Phase 2 — Highest-value cost reduction

1. Build the patient snapshot once and reuse it for both services.
2. Remove `raw_data`/`full_data` duplication and apply MongoDB projections.
3. Replace six section-level concise calls with one structured-summary generation.
4. Make grounding conditional and cache non-patient protocol evidence.
5. Remove default JSON/Markdown/CSV artifact writes from production.
6. Capture actual model usage metadata and establish per-job token/cost budgets.
7. Apply one bounded retry policy across the whole generation, with jitter and circuit breaking.

### Phase 3 — Maintainability and operations

1. Split large API, prompt, parser, persistence, and job-store responsibilities into typed modules.
2. Replace Markdown parsers with Pydantic-validated structured model responses.
3. Reuse MongoDB and HTTP clients per process.
4. Pin dependencies and build immutable service images.
5. Correct dev/prod Compose selection and use environment protection/approval for production.
6. Add rolling or service-by-service deployment and verified rollback.
7. Add structured logs, metrics, traces, queue dashboards, and alerting.
8. Prune completed job state and implement data/artifact retention.
9. Remove confirmed legacy code and stale documentation.

---

## 13. Test strategy currently missing

No automated test suite is present. A minimum production test portfolio should include:

### Unit tests

- Clinical and VALD transformation with missing, malformed, and zero values.
- Timeline ordering, date handling, first/further interaction logic, and duplicate-session rules.
- Summary and phase parser/schema validation.
- Queue threshold, deduplication, watermark, retry, and per-pipeline state transitions.

### Integration tests

- MongoDB source snapshot to structured persisted output using model stubs.
- Orchestrator dispatch through actual downstream completion.
- Concurrent summary and phase execution without artifact or state collision.
- Changes arriving before, during, and after an in-flight job.
- First manual generation, up-to-date response, retry, partial failure, and restart recovery.

### Contract and security tests

- REST and WebSocket event schemas.
- Authentication/authorization for every patient and operational route.
- Prompt/data redaction rules and logs containing no secrets or unnecessary patient data.
- Clean Docker builds with only declared direct dependencies.

### AI evaluation

- Clinician-approved representative cases and expected structured fields.
- Hallucination, missing-data acknowledgment, unsupported citation, temporal consistency, and phase-mismatch evaluation.
- Prompt/model regression suite before promotion.
- Token, latency, and cost comparison against the current baseline.

---

## 14. Metrics required for cost and performance management

Every generation job should record:

- Pipeline and job ID.
- Patient ID in protected/auditable form.
- Source snapshot/version and prompt version.
- Requested and actual model.
- Model attempt/fallback count.
- Input, cached, reasoning if applicable, and output token usage supplied by the provider.
- Grounding/search usage.
- Load, transform, queue, model, parse, and persistence latency.
- Final validation and persistence result.
- Estimated and reconciled actual cost.
- Trigger type: manual, report threshold, VALD threshold, or fallback.

Only after these measurements exist should the team claim a percentage financial saving. The call-count and database-read reductions in this document are code-path estimates, not billing claims.

---

## 15. Governance and operating decisions

The following decisions form the operating contract around the software and should be maintained alongside the technical implementation:

1. Approved clinical scope: supported diagnoses, surgeries, age groups, and rehabilitation protocols.
2. Clinical validation: representative validation set, reviewer ownership, and measurable acceptance thresholds for summary and phase output.
3. Artifact policy: whether the long EBA Markdown report is a required audit artifact or only the structured summary is retained.
4. Grounding policy: whether evidence search is required on every EBA generation and which patient information is approved for model/search processing.
5. Source-view ownership: the system responsible for creating `structured-user-reports` from `reports`, including its consistency and latency commitment.
6. Frontend contract: the REST endpoints and versioned WebSocket events actively consumed by supported clients.
7. Environment topology: separate dev and production resources, collections, hostnames, containers, and release controls.
8. Deployment operations: approved deployment scripts, backup method, rollback procedure, and responsible owner.
9. Data governance: consent, retention, residency, access auditing, redaction, and deletion requirements for inputs, prompts, logs, and generated artifacts.
10. Operational ownership: responsibility for clinical approval, production incidents, queue failures, model/cost monitoring, and security response.
11. Service objectives: required latency, availability, throughput, recovery, and monthly AI-cost limits.
12. Manual-generation behavior: whether Generate always forces a new analysis or returns the current result when its verified source version is unchanged.

---

## 16. Project summary

### Business overview

The platform consolidates a patient's clinical history and VALD measurements into a chronological timeline. A Clinical Auditor creates the overall clinical summary, a Phase Analysis service evaluates rehabilitation stage and gaps, and an Orchestrator controls when those pipelines run. Results are stored as structured MongoDB records for clinician review. The next engineering focus is production hardening, accurate job tracking, and reducing duplicated database and AI processing.

### Architecture overview

Production is separated into three FastAPI services behind Nginx: summary on port 5001, phase analysis on 5002, and orchestration on 5003. MongoDB change streams feed a per-patient queue. The AI services independently build the same patient timeline and call Gemini, after which Markdown is parsed into MongoDB documents. The architecture is functional, but current completion tracking is acceptance-based, pipeline checkpoints are shared, data loading is duplicated, and the summary path makes multiple model calls. The recommended target is a shared versioned patient snapshot, separate durable per-pipeline jobs, schema-constrained model output, verified completion, and provider-derived cost telemetry.

### Service responsibilities

- The Orchestrator decides when work runs.
- The Clinical Auditor explains the patient's overall clinical condition and journey.
- Phase Analysis explains where the patient is in rehabilitation and what remains.

---

## 17. Conclusion

The project has a clear business purpose and a workable service split. Its strongest components are longitudinal clinical/VALD consolidation, separate clinical and phase perspectives, and centralized trigger control. Its largest risks are not the core AI prompts; they are job-state correctness, credential exposure, missing access control, duplicate data/model processing, and deployment ambiguity.

The correct engineering order is:

1. Contain security exposure.
2. Make completion and per-pipeline queue state correct.
3. Remove duplicate data loading and section-level AI calls.
4. Add real usage/cost telemetry and automated tests.
5. Simplify modules, dependencies, artifacts, and deployment definitions.

Following that sequence will improve reliability first, then materially lower model calls, database work, latency, and operating cost without weakening the clinical output contract.
