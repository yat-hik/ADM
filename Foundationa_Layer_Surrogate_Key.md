# Foundation Layer Surrogate Key Technical Architecture

## 1. Purpose
This document explains the surrogate-key step implemented in the Foundation Layer physical-modeling flow inside `ADM_SILVER_LAYER`.

The goal of this feature is to let the platform assess, plan, confirm, and persist surrogate-key decisions before downstream physical-modeling steps such as Standard Naming, STTM generation, and DDL/DML generation.

This document is intentionally detailed so a developer, architect, or delivery team can understand:

- why the feature exists
- where it lives
- what repos and files participate
- how UI, API, Azure Functions, metastore persistence, and agent execution work together
- how confirmed decisions are propagated downstream

---

## 2. Business Requirement Summary
The implemented feature is designed around these core rules:

- `Generate Surrogate Keys` is the first sub-step in Foundation Layer Stage 4 Physical Modeling.
- Users can assess whether surrogate keys are needed before generating a detailed plan.
- SCD Type 2 entities must receive surrogate keys and cannot be skipped.
- Non-SCD2 entities remain optional and can be selected by the user.
- Surrogate key datatype is configurable as `INT` or `BIGINT`, defaulting to `INT`.
- Business keys remain present in the entity even when a surrogate key is introduced.
- Confirmed surrogate keys must flow into Standard Naming, STTM, and DDL/DML.

---

## 3. Repo Layout And Ownership
The surrogate-key feature spans three repos in this workspace.

### 3.1 `apps-azure`
Frontend Foundation Layer UI.

Primary files:

- `apps-azure/frontend/components/stages/Foundation_Layer/Stage4_PhysicalModeling.tsx`
- `apps-azure/frontend/components/stages/Foundation_Layer/stage4/GenerateSurrogateKeysView.tsx`

Responsibilities:

- render the Stage 4 surrogate-key step
- call the backend assessment and planning APIs
- stream progress logs through Web PubSub
- enforce the HITL flow
- collect user choices
- confirm the plan
- use confirmed surrogate-enriched entities as the source for the next Stage 4 sub-steps

### 3.2 `apis-azure`
Backend orchestration and metastore persistence layer.

Primary files:

- `apis-azure/backend/api/routes/agents.py`
- `apis-azure/backend/api/routes/surrogate_key_helpers.py`
- `apis-azure/backend/api/routes/dependency_agents.py`

Responsibilities:

- identify dependency sessions required by the surrogate-key agent
- consolidate upstream outputs into a normalized agent input payload
- write `input_json` into the metastore
- trigger the Azure Function activity
- negotiate Web PubSub access for frontend streaming
- confirm and persist user selections
- restructure output for UI consumption
- expose confirmed surrogate-key state to downstream agents

### 3.3 `adm-foundation-layer-agents-azure`
Azure Functions and Python agent execution layer.

Primary files:

- `adm-foundation-layer-agents-azure/function_app.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/SurrogateKeyGeneratorAgent.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/master/agent.py`

Responsibilities:

- receive the execution request from the backend API
- load normalized agent input from the metastore
- run the deterministic surrogate-key workflow using LangGraph
- persist the output back to the metastore
- publish execution lifecycle updates to Web PubSub

---

## 4. Why This Step Lives In Stage 4
The surrogate-key step is intentionally placed as the first sub-step of Foundation Stage 4 Physical Modeling.

Current Stage 4 order:

1. Generate Surrogate Keys
2. Standardize Names
3. Apply Transformations
4. Generate STTM
5. Generate DDL & DML
6. Download Artifacts

Reason:

- surrogate keys change the physical-table structure
- surrogate keys affect physical primary-key behavior
- surrogate keys may add new foreign-key columns to related child entities
- the naming step must run after surrogate-key attributes exist
- STTM and DDL generation must use the confirmed surrogate-aware entity model

If this step ran later, downstream outputs would be inconsistent.

---

## 5. Technology Choices

### 5.1 Frontend Stack
Used in `apps-azure`:

- React
- TypeScript
- Vite
- existing Stage 4 component pattern
- browser `fetch`
- WebSocket client for Azure Web PubSub

Why:

- fits the current Foundation Layer UI architecture
- allows clear HITL interaction
- easy state management for assessment, plan, confirmation, and downstream propagation

### 5.2 Backend/API Stack
Used in `apis-azure`:

- FastAPI route handlers
- SQLAlchemy DB access
- existing metastore helper methods
- existing output restructuring path

Why:

- the rest of the ADM workflow already uses these conventions
- surrogate-key orchestration needed to behave like the other agents
- session persistence and downstream dependency handling were already centralized here

### 5.3 Agent/Execution Stack
Used in `adm-foundation-layer-agents-azure`:

- Azure Functions
- Python
- LangGraph
- PostgreSQL metastore access through the existing DB connector abstraction

Why LangGraph:

- the requirement called for an agentic execution shell
- LangGraph gives explicit state transitions
- it keeps the workflow easy to read and extend
- the decision logic can remain deterministic instead of LLM-driven

Important note:

- LangGraph is used here as an orchestration shell, not as an open-ended reasoning engine
- the surrogate-key decision itself is rules-first and deterministic

---

## 6. Data Sources Used By This Step
The surrogate-key step depends on outputs from upstream Foundation Layer agents.

Dependency registration lives in:

- `apis-azure/backend/api/routes/dependency_agents.py`

For `SurrogateKeyGeneratorAgent`, the dependency list is:

- `MapAttributesToConceptsAgent`
- `PrimaryKeyAgent-logical`
- `EntityRoleRecommenderAgent`
- `SCDTypeRecommenderAgent`
- `RelationshipIdentifierLogicalAgent`

These provide:

- logical entities and attributes
- business keys
- entity role classification
- SCD type recommendations and confirmed SCD2 tracked attributes
- logical parent-child relationships

---

## 7. End-To-End User Flow

### 7.1 User Opens Stage 4
The user enters Foundation Layer Stage 4 in the frontend.

File:

- `Stage4_PhysicalModeling.tsx`

The first visible sub-step is `Generate Surrogate Keys`.

### 7.2 User Clicks Assessment Action
The frontend calls:

- `POST /api/v1/agents/surrogate-key/execute`

Form fields:

- `session_id`
- `phase=assess`

The frontend resets surrogate-key local state, clears logs, and starts progress streaming.

### 7.3 Backend Prepares Agent Input
In `agents.py`, the route:

- validates the phase
- loads active dependency sessions from `dfl_adm_session`
- uses `build_agent_input(agent_name="SurrogateKeyGeneratorAgent", ...)`
- calls `consolidate_for_surrogate_key_generator(...)`
- writes the resulting payload to `dfl_agent_input`

Metastore write method:

- `write_agent_input_to_db(...)`

Target table:

- `{ADM_SCHEMA}.dfl_agent_input`

Inserted columns:

- `session_id`
- `run_id`
- `agent_name`
- `input_json`

### 7.4 Backend Triggers Azure Function
The backend calls the Azure Function endpoint:

- `/agent/surrogate_key_generator`

The backend sends parameters including:

- `session_id`
- `agent_input_id`
- `phase`
- vault settings
- agent name

Then it negotiates a Web PubSub URL and returns that to the frontend.

### 7.5 Azure Function Executes Agent
In `function_app.py`, the activity trigger:

- validates `instance_id`
- initializes `SurrogateKeyGeneratorAgent`
- publishes a running event to Web PubSub
- invokes `agent.invoke()`
- persists output using `UpdateJobLogTable.push_data(...)`
- publishes a completion event

### 7.6 Agent Loads Input From Metastore
In `SurrogateKeyGeneratorAgent.py`, `_load_state(...)`:

- reads `input_json` from `dfl_agent_input`
- normalizes `logical_entities`
- normalizes `primary_key_data`
- normalizes `entity_roles`
- normalizes `relationship_identifier_data`
- normalizes SCD recommendations

Important confirmed-SCD behavior:

- if `scd_type2_tracked_attributes` exists from the confirmed SCD step, those entities are upgraded to effective `Type 2`
- this prevents stale pre-confirm `scd_type_1` values from suppressing mandatory surrogate-key behavior

### 7.7 Assessment Phase Output
The agent builds entity plans in memory and returns only assessment data when `phase=assess`.

The assessment contains:

- `scd2_entities`
- `scd1_entities`
- `non_scd_entities`
- `allow_skip`
- `allow_generate`
- `entity_count`
- `message_for_modeler`

Rules:

- if any entity is `Type 2`, `allow_skip = false`
- otherwise, `allow_skip = true`

### 7.8 User Clicks Generate Plan
The frontend again calls:

- `POST /api/v1/agents/surrogate-key/execute`

Form fields:

- `session_id`
- `phase=plan`

The same orchestration path runs again, but the agent now returns a detailed draft plan.

### 7.9 Draft Plan Produced
The plan contains:

- `default_surrogate_key_type`
- `entity_plans`
- `relationship_plans`

Each `entity_plan` includes:

- `entity_name`
- `entity_type`
- `scd_type`
- `business_keys`
- `surrogate_key_required`
- `generate_surrogate_key`
- `skip_allowed`
- `surrogate_key_type`
- `surrogate_key_name`
- `reasoning`

Key rule:

- SCD Type 2 entities are mandatory
- non-SCD2 entities remain optional

### 7.10 Frontend HITL Review
The UI then lets the user:

- see mandatory SCD2 entities
- see optional candidates
- review business keys
- keep or remove optional rows
- select datatype per row
- bulk `Select All`
- bulk `Clear Optional`

Current UI behavior aligned to the requirement:

- SCD Type 2 rows are selected and locked on by default
- non-SCD2 rows are shown but not selected by default
- the confirm button shows an in-button confirmation animation while the save request is running
- surrogate-step logs auto-scroll

### 7.11 User Confirms Selections
The frontend calls:

- `POST /api/v1/agents/surrogate-key/confirm`

JSON body:

- `session_id`
- `session_name`
- `user_name`
- `confirmed_entities`

### 7.12 Backend Persists Confirmed Session
In `agents.py`, `confirm_surrogate_keys(...)`:

- loads the current active `SurrogateKeyGeneratorAgent` session from `dfl_adm_session`
- calls `merge_confirmed_surrogate_session(...)`
- writes a new active version to `dfl_adm_session` with `append_session_to_db(...)`
- restructures the result for UI consumption

Target table:

- `{ADM_SCHEMA}.dfl_adm_session`

Persistence pattern:

- the existing active row for the same `session_id + agent_name` is deactivated
- a new row is inserted with `is_active=true`
- this preserves session history while keeping a single current version

### 7.13 Merge Logic Applied During Confirmation
`merge_confirmed_surrogate_session(...)` and `apply_confirmed_surrogate_plan(...)` perform the core merge.

They:

- remove prior surrogate artifacts before reapplying
- keep business keys persisted
- insert surrogate-key columns into selected entities
- set the surrogate key as the physical primary key for selected entities
- preserve business keys as `business_key`
- propagate parent surrogate foreign keys into child entities when relationships exist

The persisted confirmed output includes:

- `confirmed_surrogate_keys`
- `logical_entities_with_surrogate_keys`
- `primary_key_data_with_surrogate_keys`
- `propagated_foreign_keys`

---

## 8. Detailed Flow By Layer

### 8.1 Frontend Layer
Files:

- `Stage4_PhysicalModeling.tsx`
- `GenerateSurrogateKeysView.tsx`

Key responsibilities:

- local modal state
- progress logs
- WebSocket lifecycle
- plan editing
- confirm/skip actions
- downstream state handoff through `surrogateEnhancedLogicalEntities`

Notable UX choices:

- assessment-first workflow
- explicit review before generating the plan
- visible locked mandatory rows
- per-row datatype control
- in-button confirmation loading state

### 8.2 API Layer
Files:

- `agents.py`
- `surrogate_key_helpers.py`

Key responsibilities:

- dependency resolution
- payload normalization
- metastore input writes
- agent execution trigger
- session versioning
- downstream compatibility shaping

Why helper extraction was used:

- route handlers stay focused on orchestration
- surrogate business rules stay reusable and testable
- downstream agents can reuse the same normalized confirmed output

### 8.3 Azure Function Layer
Files:

- `function_app.py`
- `SurrogateKeyGenerator/master/agent.py`

Key responsibilities:

- invoke the Python agent
- load `input_json` from `dfl_agent_input`
- push `output_json` to the metastore
- publish running/completed events

### 8.4 Agent Layer
File:

- `SurrogateKeyGeneratorAgent.py`

State carried through LangGraph:

- `logical_entities`
- `primary_key_data`
- `entity_roles`
- `scd_recommendations`
- `relationship_identifier_data`
- `phase`
- `assessment`
- `surrogate_key_generation`

Graph shape:

1. `load_state`
2. `assess_surrogate_keys`
3. conditional route
4. `plan_surrogate_keys` if `phase=plan`
5. end

Why this is deterministic:

- SCD type drives mandatory vs optional behavior
- default datatype is fixed
- naming rule is fixed
- the same input session produces the same plan

---

## 9. Confirmed SCD2 Override Behavior
This is a critical detail.

The SCD confirmation step may save user-confirmed Type 2 tracked attributes under:

- `scd_type2_tracked_attributes`

while the older `scd_recommendations` block may still contain `scd_type_1`.

To avoid incorrect surrogate-key assessment:

- the surrogate-key normalization path merges confirmed tracked attributes into the effective SCD view
- any entity with confirmed tracked attributes is treated as `Type 2`

This merge now happens in:

- `surrogate_key_helpers.py`
- `SurrogateKeyGeneratorAgent.py`

Without this merge, surrogate-key assessment can incorrectly report that no SCD2 entities exist.

---

## 10. Downstream Propagation

### 10.1 Standard Naming
The Standard Naming input builder checks for confirmed surrogate-enriched logical entities from the surrogate-key session first.

Result:

- naming runs against the surrogate-aware logical model, not the old one

### 10.2 STTM
The STTM input builder also consumes surrogate-aware primary-key data and confirmed logical entities.

Result:

- target fields include surrogate-key columns
- physical PK behavior reflects confirmed surrogate decisions

### 10.3 DDL/DML
The DDL input builder reads surrogate-aware PK metadata from the surrogate session if available.

Result:

- DDL generation reflects surrogate-aware physical primary keys

---

## 11. Data Modeling Interpretation
One dimension gets one surrogate key column, not one surrogate key per tracked attribute.

Example:

- `Coverage` may have multiple business-key columns
- `Coverage` may also have one or more Type 2 tracked attributes
- the physical dimension still gets a single surrogate key such as `Coverage_SK`
- when any tracked Type 2 attribute changes, a new row version is created with a new surrogate-key value

That matches standard dimensional modeling behavior.

---

## 12. Logging And Observability

### Frontend
- assessment logs
- plan-generation logs
- confirmation completion/error logs
- auto-scroll behavior in the surrogate-step console

### API
- input-preparation logs
- execution error logs
- session persistence logs

### Azure Function
- activity start logs
- success/failure logs
- PubSub lifecycle events

### Agent
- input-load logs
- assessment logs
- plan-generation logs
- workflow completion logs

---

## 13. Why This Design Was Chosen
This design was selected because it balances:

- deterministic physical-modeling rules
- agentic orchestration requirements
- existing session persistence conventions
- user confirmation needs
- downstream compatibility

It avoids a brittle design where:

- surrogate keys are generated too late
- confirmed SCD choices are ignored
- Standard Naming or STTM work from stale models
- physical PK behavior diverges from the reviewed plan

---

## 14. Key Files

### Frontend
- `apps-azure/frontend/components/stages/Foundation_Layer/Stage4_PhysicalModeling.tsx`
- `apps-azure/frontend/components/stages/Foundation_Layer/stage4/GenerateSurrogateKeysView.tsx`

### API
- `apis-azure/backend/api/routes/agents.py`
- `apis-azure/backend/api/routes/surrogate_key_helpers.py`
- `apis-azure/backend/api/routes/dependency_agents.py`

### Agent/Azure Functions
- `adm-foundation-layer-agents-azure/function_app.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/SurrogateKeyGeneratorAgent.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/master/agent.py`

---

## 15. Current Functional Summary
As implemented today, the surrogate-key step does the following:

- assesses whether the current model has SCD Type 2 entities
- respects confirmed SCD Type 2 tracked attributes
- makes SCD2 surrogate keys mandatory
- lets users opt into surrogate keys for non-SCD2 entities
- defaults selection to all SCD2 entities only
- keeps business keys alongside surrogate keys
- persists the confirmed surrogate-aware logical model
- pushes that confirmed model into Standard Naming, STTM, and DDL/DML

That is the current source of truth for how this step works inside `ADM_SILVER_LAYER`.
