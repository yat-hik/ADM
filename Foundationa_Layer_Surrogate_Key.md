# Foundation Layer Surrogate Key Technical Architecture

> **Platform Scope**
> Foundation Layer -> Stage 4 Physical Modeling -> `Generate Surrogate Keys`

> **Execution Style**
> Deterministic rules engine wrapped in an agentic orchestration shell

> **Repos Involved**
> `apps-azure` + `apis-azure` + `adm-foundation-layer-agents-azure`

---

## Executive View

The surrogate-key step is the first sub-step of Foundation Layer Stage 4. It is intentionally placed before Standard Naming, STTM, and DDL/DML generation because it changes the physical structure of the model and therefore must become part of the authoritative physical-design state before downstream artifacts are generated.

This implementation is not just another backend agent call. It is a structured HITL workflow that:

- assesses whether surrogate keys are required
- uses confirmed SCD behavior, not stale recommendation state
- distinguishes mandatory vs optional surrogate-key behavior
- lets the user review and confirm table-level decisions
- persists the confirmed physical-design intent back into session state
- turns the confirmed result into the downstream source of truth

### Plain-Language Summary

In simple terms, this is what we built:

1. the user first asks the system to assess whether surrogate keys are needed
2. the system checks business keys, SCD type, entity role, and logical relationships
3. if SCD Type 2 exists, those entities are marked as mandatory for surrogate keys
4. the user then reviews a generated plan instead of blindly accepting a result
5. once the user confirms, that confirmed surrogate-key design becomes the new active physical-design input for later Stage 4 steps

So this step is not only “showing a recommendation.” It is changing the authoritative design state used by the rest of the physical-modeling flow.

---

## 1. Design Goals

### 1.1 Business Goals

- enforce surrogate keys for SCD Type 2 entities
- keep surrogate keys optional for non-SCD2 entities
- preserve business keys alongside surrogate keys
- make datatype selectable as `INT` or `BIGINT`, default `INT`
- ensure surrogate decisions flow into Standard Naming, STTM, and DDL/DML

### 1.2 Engineering Goals

- deterministic outputs for repeatability
- HITL control without fragile client-side-only state
- minimal API surface area
- compatibility with existing metastore/session patterns
- reuse of existing ADM orchestration patterns

### 1.3 Why This Agent Is Stronger Than Most Other Agents In This Workflow

Compared with many other Foundation Layer agents, this surrogate-key agent has several advantages:

- it is **deterministic**, not LLM-dependent for the final business rule
- it is **stateful**, but still easy to reason about
- it supports **assessment-first HITL**, not just generate-and-show
- it merges **confirmed SCD state** instead of trusting only the original recommendation payload
- it affects **multiple downstream contracts** in a controlled way
- it persists both **draft planning context** and **confirmed structural decisions**

This makes it one of the most operationally reliable Stage 4 agents because:

- the business rules are explicit
- the review path is explicit
- the metastore state transition is explicit
- downstream propagation is explicit

---

## 2. Stage Placement

### 2.1 Current Stage 4 Order

1. Generate Surrogate Keys
2. Standardize Names
3. Apply Transformations
4. Generate STTM
5. Generate DDL & DML
6. Download Artifacts

### 2.2 Why It Must Be First

Surrogate-key generation must run first because it can:

- add new physical primary-key columns
- change the effective physical primary key of a table
- preserve business keys as non-primary business columns
- add propagated surrogate foreign keys into child entities

If this step ran after naming or STTM generation, those downstream artifacts would be built on stale physical structure.

---

## 3. Architecture Topology

```text
User
  |
  v
apps-azure (React UI)
  |
  |  POST /api/v1/agents/surrogate-key/execute   [phase=assess | plan]
  |  POST /api/v1/agents/surrogate-key/confirm
  v
apis-azure (FastAPI orchestration layer)
  |
  |  build_agent_input(...)
  |  write_agent_input_to_db(...)
  |  trigger Azure Function
  v
adm-foundation-layer-agents-azure (Azure Function + Python Agent)
  |
  |  fetch_input() from dfl_agent_input
  |  run LangGraph nodes
  |  push_data() to dfl_adm_session
  v
Metastore (Postgres)
  |
  v
apis-azure restructure_output_by_agent(...)
  |
  v
apps-azure UI review and confirm
```

---

## 4. Repo Responsibilities

## 4.1 `apps-azure`

### Key Files

- `apps-azure/frontend/components/stages/Foundation_Layer/Stage4_PhysicalModeling.tsx`
- `apps-azure/frontend/components/stages/Foundation_Layer/stage4/GenerateSurrogateKeysView.tsx`

### Responsibilities

- open the Stage 4 surrogate-key modal
- trigger the assessment phase
- trigger the plan phase
- render assessment summary groups
- render the table-level plan
- enforce locked mandatory SCD Type 2 behavior
- allow optional row selection
- allow datatype changes
- confirm reviewed selections
- hold the confirmed surrogate-enhanced logical entities in frontend state for downstream Stage 4 use

---

## 4.2 `apis-azure`

### Key Files

- `apis-azure/backend/api/routes/agents.py`
- `apis-azure/backend/api/routes/surrogate_key_helpers.py`
- `apis-azure/backend/api/routes/dependency_agents.py`

### Responsibilities

- resolve dependent sessions from metastore
- build normalized surrogate-agent input
- write `input_json` to `dfl_agent_input`
- call the Azure Function endpoint
- negotiate Web PubSub
- confirm surrogate selections
- persist the new active session version to `dfl_adm_session`
- provide surrogate-enriched output to downstream agents

---

## 4.3 `adm-foundation-layer-agents-azure`

### Key Files

- `adm-foundation-layer-agents-azure/function_app.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/SurrogateKeyGeneratorAgent.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/master/agent.py`

### Responsibilities

- receive execution request from API layer
- fetch stored input from metastore
- run deterministic surrogate-key flow using LangGraph
- persist output to metastore
- publish status lifecycle events

### One-Line Responsibility Split

| Repo | Short Responsibility |
|---|---|
| `apps-azure` | user interaction, HITL review, selection, confirmation |
| `apis-azure` | orchestration, metastore coordination, downstream session shaping |
| `adm-foundation-layer-agents-azure` | actual agent execution and result persistence |

---

## 5. Tech Stack And Why It Was Chosen

| Layer | Technology | Why It Fits |
|---|---|---|
| UI | React + TypeScript + Vite | Fits the existing Foundation Layer app and supports responsive HITL state updates |
| API | FastAPI + SQLAlchemy | Matches the rest of the ADM orchestration pattern and metastore access layer |
| Execution | Azure Functions | Already used for agent runtime integration and async activity execution |
| Agent Shell | LangGraph | Gives explicit graph state and clean deterministic node orchestration |
| Persistence | PostgreSQL metastore | Existing ADM session and agent-input persistence model |
| Realtime Feedback | Azure Web PubSub | Streams progress back to the UI without polling loops |

### Why LangGraph Was Used Here

LangGraph is not used here to generate free-form reasoning. It is used to provide an explicit execution graph:

- `load_state`
- `assess_surrogate_keys`
- conditional route
- `plan_surrogate_keys`

This is valuable because:

- the workflow is inspectable
- the execution order is rigid
- future nodes can be added safely
- the business rules remain deterministic

---

## 6. Dependency Model

Dependency registration lives in:

- `apis-azure/backend/api/routes/dependency_agents.py`

For `SurrogateKeyGeneratorAgent`, dependencies are:

```python
"SurrogateKeyGeneratorAgent": [
    "MapAttributesToConceptsAgent",
    "PrimaryKeyAgent-logical",
    "EntityRoleRecommenderAgent",
    "SCDTypeRecommenderAgent",
    "RelationshipIdentifierLogicalAgent"
]
```

### What Each Dependency Contributes

| Dependency | What It Provides | Why Surrogate Step Needs It |
|---|---|---|
| `MapAttributesToConceptsAgent` | logical entities and attributes | base logical model to enrich |
| `PrimaryKeyAgent-logical` | business keys | preserve natural/business key identity |
| `EntityRoleRecommenderAgent` | fact vs dimension context | entity role display and context |
| `SCDTypeRecommenderAgent` | SCD type + confirmed tracked attrs | mandatory-vs-optional rule evaluation |
| `RelationshipIdentifierLogicalAgent` | parent-child relationships | propagated surrogate foreign-key planning |

---

## 7. HITL Design

## 7.1 Why This Is HITL

This step is HITL because the user is not forced to accept the first generated structural plan. Instead the system:

1. assesses
2. explains
3. generates a reviewable plan
4. accepts user edits
5. persists confirmed structural choices

That is not just “user sees result.” It is user-guided structural confirmation.

## 7.2 How HITL Works Without Multiple Execute Endpoints

This design is elegant because the API does not need separate execution endpoints for:

- assessment
- plan generation

Instead, one execution endpoint is reused:

```http
POST /api/v1/agents/surrogate-key/execute
```

and behavior is controlled by a single key:

```json
{
  "phase": "assess"
}
```

or

```json
{
  "phase": "plan"
}
```

This is one of the strongest parts of the design:

- same route
- same dependency preparation logic
- same metastore input pattern
- different graph behavior driven by state

The confirm step is intentionally separate because confirmation is not execution. It is a **state-commit action**:

```http
POST /api/v1/agents/surrogate-key/confirm
```

That distinction is important.

### Why This Pattern Is Better Than Adding Many New Endpoints

This implementation keeps the API easier to maintain because:

- assessment and plan use the same execution route
- only one control key, `phase`, changes the behavior
- all dependency preparation logic stays in one place
- the confirm route is reserved only for persisting reviewed design decisions

This is cleaner than building separate routes like:

- `/surrogate-key/assess`
- `/surrogate-key/plan`
- `/surrogate-key/replan`

because those would duplicate orchestration behavior and make the session flow harder to reason about.

---

## 8. End-To-End Flow

## 8.1 Step A: User Clicks `Assess Surrogate Key Need`

Frontend sends:

```http
POST /api/v1/agents/surrogate-key/execute
Content-Type: multipart/form-data
```

Form payload:

```text
session_id=<current_session_id>
phase=assess
```

### What Frontend Does Before Call

- clears surrogate logs
- clears existing assessment/plan state
- resets confirmation state
- opens progress experience

### Why This Matters

This reset behavior ensures the user always sees a clean, current surrogate-key review flow rather than stale plan state from a previous run.

---

## 8.2 Step B: API Builds Input

Route:

- `surrogate_key_generator_agent_server(...)` in `agents.py`

### What Happens

1. validate phase
2. load dependent agent outputs from metastore
3. call `build_agent_input(agent_name="SurrogateKeyGeneratorAgent", ...)`
4. write input to `dfl_agent_input`
5. trigger Azure Function

### Relevant Code Pattern

```python
surrogate_input = build_agent_input(
    agent_name=target_agent_name,
    agent_outputs=agent_outputs,
)
surrogate_input["phase"] = normalized_phase

write_agent_input_to_db(
    session_id=session_id,
    run_id=agent_input_id,
    agent_name=target_agent_name,
    input_json=surrogate_input,
)
```

---

## 8.3 Step C: Metastore Input Persistence

Target table:

- `{ADM_SCHEMA}.dfl_agent_input`

Inserted fields:

- `session_id`
- `run_id`
- `agent_name`
- `input_json`

### Example Stored Input JSON

```json
{
  "logical_entities": {
    "Coverage": [
      {
        "attribute_name": "Coverage_Code",
        "attribute_data_type": "string",
        "attribute_description": "Coverage type code"
      }
    ]
  },
  "primary_key_data": {
    "Coverage": {
      "primary_key": ["Coverage_Code", "Coverage_Group"],
      "justification": "Business key identified from logical modeling"
    }
  },
  "entity_roles": {
    "Coverage": "DIMENSION"
  },
  "scd_recommendations": {
    "Coverage": {
      "scd_type": "Type 2",
      "tracked_attributes": ["Coverage_Name"],
      "reasoning": "SCD Type 2 confirmed from saved tracked attributes."
    }
  },
  "relationship_identifier_data": [],
  "phase": "assess"
}
```

### Critical HITL Control Keys

| Key | Why It Matters |
|---|---|
| `phase` | decides whether graph ends at assessment or proceeds to full planning |
| `scd_recommendations` | drives mandatory vs optional logic |
| `tracked_attributes` | proves why an entity is Type 2 |
| `primary_key_data` | defines business keys that must remain persisted |

### Additional Important Runtime Keys

| Key | Layer | Usage |
|---|---|---|
| `session_id` | UI -> API -> metastore | ties all activity to the active Foundation session |
| `agent_input_id` | API -> Azure Function -> metastore | identifies the exact input row in `dfl_agent_input` |
| `run_id` / `instance_id` | Azure Function / PubSub | used for execution tracking and websocket group streaming |
| `confirmed_entities` | UI -> confirm API | carries user-reviewed table-level selections |

---

## 8.4 Step D: Azure Function Trigger

The API calls the Azure Function endpoint:

```text
/agent/surrogate_key_generator
```

Payload sent to Azure Function includes values such as:

```json
{
  "session_id": "<session_id>",
  "user_id": "1",
  "agent": "SurrogateKeyGeneratorAgent",
  "agent_input_id": "<uuid>",
  "phase": "assess",
  "vault_type": "AzureKeyVault",
  "vault_url": "https://llm-accelerator-adm.vault.azure.net/"
}
```

The API then negotiates a Web PubSub URL and returns that to the frontend so progress logs can stream in real time.

---

## 8.5 Step E: Azure Function Executes Agent

File:

- `adm-foundation-layer-agents-azure/function_app.py`

Key behavior:

```python
agent = SurrogateKeyGeneratorAgent(params)
agent_result = asyncio.run(agent.invoke())
UpdateJobLogTable(params).push_data(result)
```

This layer also publishes Web PubSub lifecycle events such as:

- execution started
- execution completed

### Why This Agent Feels Different In The UI

Most other agents in the workflow return a result and stop there. This agent behaves more like a review-oriented design service:

- phase 1 gives assessment
- phase 2 gives a draft plan
- phase 3 commits the reviewed decision

That is why the user experience includes:

- progress logs
- summary chips
- locked mandatory selections
- confirmation animation

---

## 8.6 Step F: Agent Loads State

In `SurrogateKeyGeneratorAgent.py`, `_load_state(...)` does the following:

- fetches `input_json`
- normalizes entity names
- normalizes PK payloads
- normalizes roles
- normalizes relationships
- normalizes effective SCD state

### Critical Confirmed-SCD Override

This is a major engineering safeguard.

Sometimes the confirmed SCD state is stored separately as:

```json
{
  "scd_type2_tracked_attributes": {
    "Coverage": ["Coverage_Name"],
    "Policyholder": ["Address_State", "Credit_Score_Tier"]
  }
}
```

while the original `scd_recommendations` block may still say:

```json
{
  "Coverage": {
    "scd_type": "scd_type_1"
  }
}
```

The surrogate-key normalization logic merges these and upgrades the effective SCD type to `Type 2`.

Without this, the surrogate-key step would make the wrong mandatory/optional decision.

---

## 8.7 Step G: Assessment Output

When `phase=assess`, the graph returns only assessment data.

### Example Assessment JSON

```json
{
  "assessment": {
    "allow_skip": false,
    "allow_generate": true,
    "entity_count": 6,
    "scd2_entities": ["Coverage", "Insurance Policy", "Policyholder"],
    "scd1_entities": ["Claim", "Insurance Agency", "Insurance Agent"],
    "non_scd_entities": [],
    "message_for_modeler": "[Surrogatekeyagent] : I found 3 SCD Type 2 entities in this model: Coverage, Insurance Policy, Policyholder. Surrogate keys are mandatory for those entities, so skip is disabled. Please proceed to generate the detailed surrogate-key plan."
  }
}
```

### Assessment Decision Rules

```text
If any entity is Type 2:
  allow_skip = false
Else:
  allow_skip = true
```

### Requirement Mapping For Assessment

| Requirement Area | How Assessment Supports It |
|---|---|
| generate or skip option | returns `allow_skip` and `allow_generate` |
| SCD2 must not be skippable | sets `allow_skip=false` when any Type 2 exists |
| user clarity | returns grouped `scd2_entities` and `scd1_entities` |

---

## 8.8 Step H: User Clicks `Generate Surrogate Key Plan`

Frontend again uses the same execute endpoint:

```http
POST /api/v1/agents/surrogate-key/execute
```

but now sends:

```text
phase=plan
```

This is a clean HITL reuse pattern:

- same API route
- same orchestration
- different result depth

---

## 8.9 Step I: Draft Plan Output

When `phase=plan`, the graph executes the planning node and returns `surrogate_key_generation`.

### Example Plan JSON

```json
{
  "surrogate_key_generation": {
    "default_surrogate_key_type": "INT",
    "relationship_plans": [],
    "entity_plans": [
      {
        "entity_name": "Coverage",
        "entity_type": "Dimension",
        "scd_type": "Type 2",
        "business_keys": ["Coverage_Code", "Coverage_Group"],
        "surrogate_key_required": true,
        "generate_surrogate_key": true,
        "skip_allowed": false,
        "surrogate_key_type": "INT",
        "surrogate_key_name": "Coverage_SK",
        "reasoning": "SCD Type 2 entities require surrogate keys to preserve historical versions."
      },
      {
        "entity_name": "Insurance Agency",
        "entity_type": "Dimension",
        "scd_type": "Type 1",
        "business_keys": ["Agency_Name"],
        "surrogate_key_required": false,
        "generate_surrogate_key": false,
        "skip_allowed": true,
        "surrogate_key_type": "INT",
        "surrogate_key_name": "Insurance_Agency_SK",
        "reasoning": "Non-SCD2 entities may skip surrogate keys unless the modeler chooses otherwise."
      }
    ]
  }
}
```

### Why This Matters

The plan is not yet the source of truth. It is a reviewable draft.

That is the heart of the HITL design.

---

## 8.10 Step J: Frontend Review Experience

The UI renders:

- mandatory SCD2 summary chips
- optional candidate chips
- per-entity business keys
- per-row surrogate-key selection
- per-row datatype selector
- in-button confirm animation
- autoscrolling logs

### Current UI Rules

- SCD Type 2 rows are preselected and locked on
- non-SCD2 rows are visible but unchecked by default
- locked required rows cannot be disabled
- `Select All` can opt in every row
- `Clear Optional` removes only optional selections

### Confirm Button Behavior

When the user clicks `Confirm Selections`:

- the button itself becomes the progress indicator
- a loading spinner is shown inside the button
- the button is temporarily disabled
- once the confirm API succeeds, the confirmed result is committed into session state and the workflow advances

---

## 8.11 Step K: User Clicks `Confirm Selections`

Frontend sends:

```http
POST /api/v1/agents/surrogate-key/confirm
Content-Type: application/json
```

### Example Confirm Request JSON

```json
{
  "session_id": "abc-session-id",
  "session_name": "Insurance Foundation Session",
  "user_name": "data.modeler@company.com",
  "confirmed_entities": [
    {
      "entity_name": "Coverage",
      "entity_type": "Dimension",
      "scd_type": "Type 2",
      "business_keys": ["Coverage_Code", "Coverage_Group"],
      "surrogate_key_required": true,
      "generate_surrogate_key": true,
      "skip_allowed": false,
      "surrogate_key_type": "INT",
      "surrogate_key_name": "Coverage_SK",
      "reasoning": "SCD Type 2 entities require surrogate keys to preserve historical versions."
    }
  ]
}
```

### UI Behavior During Confirm

- the `Confirm Selections` button switches to in-button loading state
- the spinner indicates state commit is happening
- this is intentionally different from idle state to reassure the user that persistence is in progress

---

## 8.12 Step L: Backend Session Commit

Route:

- `confirm_surrogate_keys(...)` in `agents.py`

### What Happens

1. fetch current active surrogate-key session from `dfl_adm_session`
2. load current session JSON
3. merge confirmed selections
4. write a new active session version
5. restructure result back to UI shape

### Core Code Pattern

```python
updated_session_json = merge_confirmed_surrogate_session(
    current_session_json,
    [selection.model_dump() for selection in payload.confirmed_entities],
)

append_session_to_db(
    session_id=payload.session_id,
    run_id=str(uuid.uuid4()),
    agent_name="SurrogateKeyGeneratorAgent",
    session_json=updated_session_json,
    user_id="1",
    username=payload.user_name,
    session_name=payload.session_name,
)
```

---

## 8.13 Step M: Metastore Session Versioning

Target table:

- `{ADM_SCHEMA}.dfl_adm_session`

### Versioning Pattern

Before insert:

- old active row for the same `session_id + agent_name` is marked inactive

Then:

- a new row is inserted as active

This gives:

- historical lineage
- reproducibility
- clear current session state

### Example Confirmed Session Output JSON

```json
{
  "result": {
    "logical_entities": {
      "Coverage": [
        {
          "attribute_name": "Coverage_SK",
          "attribute_description": "surrogate key auto increment",
          "attribute_data_type": "INT",
          "attribute_domain": "Identifier",
          "is_surrogate_key": true,
          "surrogate_key_type": "INT"
        },
        {
          "attribute_name": "Coverage_Code",
          "attribute_data_type": "string"
        },
        {
          "attribute_name": "Coverage_Group",
          "attribute_data_type": "string"
        }
      ]
    },
    "primary_key_data": {
      "Coverage": {
        "primary_key": ["Coverage_SK"],
        "business_key": ["Coverage_Code", "Coverage_Group"],
        "justification": "Confirmed surrogate key replaces the physical primary key while business keys remain persisted."
      }
    },
    "surrogate_key_generation": {
      "confirmed_surrogate_keys": [
        {
          "entity_name": "Coverage",
          "generate_surrogate_key": true,
          "surrogate_key_name": "Coverage_SK",
          "surrogate_key_type": "INT"
        }
      ],
      "logical_entities_with_surrogate_keys": {
        "Coverage": [
          {
            "attribute_name": "Coverage_SK",
            "is_surrogate_key": true
          }
        ]
      },
      "primary_key_data_with_surrogate_keys": {
        "Coverage": {
          "primary_key": ["Coverage_SK"],
          "business_key": ["Coverage_Code", "Coverage_Group"]
        }
      },
      "propagated_foreign_keys": []
    }
  }
}
```

---

## 9. Merge Semantics At Confirmation

Confirmation logic lives mainly in:

- `merge_confirmed_surrogate_session(...)`
- `apply_confirmed_surrogate_plan(...)`

### What The Merge Does

- strips old surrogate artifacts before rebuilding
- preserves base logical entities
- preserves base business keys
- inserts surrogate-key attributes into selected entities
- replaces physical PK with surrogate PK where selected
- preserves business keys separately
- propagates surrogate foreign keys to children when relationships exist

### Why This Is Better Than Mutating Rows Blindly

Because it:

- avoids duplicate surrogate columns
- keeps a base logical model reference
- allows reruns without compounding artifacts
- keeps confirmed state reproducible

### Important Business Effect

After confirmation:

- business keys are still retained in the entity
- the surrogate key becomes the physical primary key for selected entities
- the confirmed structure is what downstream agents now consume

---

## 10. Downstream Propagation

### Impact Matrix For Subsequent Agents

| Subsequent Agent | Is It Impacted? | How |
|---|---|---|
| `StandardNamingGeneratorAgent` | Yes | receives surrogate-enriched logical entities |
| `SttmGeneratorAgent` | Yes | receives surrogate-aware entity structure and primary-key metadata |
| `STTMtoDDLAgent` | Yes | uses surrogate-aware primary-key metadata for physical PK generation |
| `RelationshipIdentifierLogicalAgent` | Used as upstream input | existing parent-child relationships are read for surrogate planning, but the surrogate-key step does not re-run or rewrite the relationship model itself |

## 10.1 Standard Naming

The Standard Naming consolidation checks surrogate-confirmed logical entities first.

Why:

- naming must run on the surrogate-aware model

### Effective Effect

- surrogate key attributes appear in the naming input
- the downstream standardized model includes those physical attributes

## 10.2 STTM

The STTM consolidation uses:

- surrogate-aware primary-key metadata
- surrogate-aware logical entities

Why:

- STTM target fields must include the surrogate keys
- STTM physical PK treatment must follow the confirmed plan

### Current Relationship Handling Limitation

This is important for architects to know.

Current implementation behavior is:

- logical relationships are already available to the surrogate-key step as upstream input
- the surrogate-key step can plan propagated foreign-key behavior
- however, physical relationship/STTM realization still effectively happens when the user clicks `Generate STTM`

So in current implementation:

- the surrogate-key step prepares and persists surrogate-aware structural intent early
- relationship materialization into the STTM output still happens in the STTM generation step

This means the current implementation is structurally aligned, but the visible flow of LDM relationships into physical mapping still becomes fully evident only after STTM generation.

## 10.3 DDL/DML

The DDL builder uses surrogate-aware primary-key data when the surrogate session exists.

Why:

- final DDL should reflect the reviewed physical primary-key design, not just the original business-key model

---

## 11. Data Modeling Interpretation

One dimension gets one surrogate key column, not one surrogate key per tracked attribute.

Example:

- `Coverage` can have business key columns:
  - `Coverage_Code`
  - `Coverage_Group`
- `Coverage` can have one or more SCD Type 2 tracked attributes:
  - `Coverage_Name`
- the physical table still gets one surrogate PK:
  - `Coverage_SK`

When any tracked Type 2 attribute changes:

- a new dimension row version is created
- that new row receives a new surrogate-key value

That is correct dimensional modeling behavior.

---

## 12. Logging, Streaming, And UX Observability

## 12.1 Frontend

- progress logs render inside the surrogate-step modal
- logs auto-scroll
- most recent item pulses during active progress
- confirm button animates while commit is running

## 12.2 API

- logs input preparation
- logs execution failures
- logs session commit failures

## 12.3 Azure Function

- publishes execution lifecycle events to Web PubSub
- logs activity trigger start and completion

## 12.4 Agent

- logs normalized input load
- logs assessment phase
- logs plan generation phase
- logs workflow completion

---

## 13. Why This Agent Is Architecturally Better Than Typical Generate-Only Agents

This agent is stronger than a typical generate-only agent in the following ways:

### 13.1 It Is Review-Driven
Most agents return a generated result immediately. This one returns:

- assessment
- then plan
- then confirmation commit

That is a more mature workflow for structural modeling decisions.

### 13.2 It Is Structurally Deterministic
Many agents depend more heavily on inference or LLM output. This agent’s core decision logic is rules-based:

- SCD2 -> mandatory
- non-SCD2 -> optional
- default datatype -> `INT`

### 13.3 It Handles Confirmed State Correctly
It merges:

- original SCD recommendation output
- confirmed SCD Type 2 tracked attributes

This is a critical robustness improvement.

### 13.4 It Persists Authoritative Downstream State
It does not stop at UI review. It writes the confirmed result back so later agents consume the corrected structural model.

That makes it not just a recommendation step, but a structural-state transition step.

### 13.5 It Reduces Hidden Coupling

Because this agent persists its confirmed output into active session state, downstream agents do not need custom UI hacks to remember user choices. They simply read the current active session contract.

That is a major architectural strength.

---

## 14. Key Files

### Frontend

- `apps-azure/frontend/components/stages/Foundation_Layer/Stage4_PhysicalModeling.tsx`
- `apps-azure/frontend/components/stages/Foundation_Layer/stage4/GenerateSurrogateKeysView.tsx`

### API

- `apis-azure/backend/api/routes/agents.py`
- `apis-azure/backend/api/routes/surrogate_key_helpers.py`
- `apis-azure/backend/api/routes/dependency_agents.py`

### Agent / Azure Functions

- `adm-foundation-layer-agents-azure/function_app.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/SurrogateKeyGeneratorAgent.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/master/agent.py`

---

## 15. Reference JSON Contracts

## 15.1 Assess Execute Request

```json
{
  "session_id": "<session_id>",
  "phase": "assess"
}
```

## 15.2 Plan Execute Request

```json
{
  "session_id": "<session_id>",
  "phase": "plan"
}
```

## 15.3 Assessment Output

```json
{
  "assessment": {
    "allow_skip": false,
    "allow_generate": true,
    "entity_count": 6,
    "scd2_entities": ["Coverage", "Insurance Policy", "Policyholder"],
    "scd1_entities": ["Claim", "Insurance Agency", "Insurance Agent"],
    "non_scd_entities": []
  }
}
```

## 15.4 Draft Plan Output

```json
{
  "surrogate_key_generation": {
    "default_surrogate_key_type": "INT",
    "entity_plans": [
      {
        "entity_name": "Coverage",
        "surrogate_key_required": true,
        "generate_surrogate_key": true,
        "surrogate_key_type": "INT",
        "surrogate_key_name": "Coverage_SK"
      }
    ]
  }
}
```

## 15.5 Confirm Request

```json
{
  "session_id": "<session_id>",
  "session_name": "<session_name>",
  "user_name": "<user_name>",
  "confirmed_entities": [
    {
      "entity_name": "Coverage",
      "generate_surrogate_key": true,
      "surrogate_key_type": "INT",
      "surrogate_key_name": "Coverage_SK"
    }
  ]
}
```

## 15.6 Confirmed Output

```json
{
  "surrogate_key_generation": {
    "confirmed_surrogate_keys": [
      {
        "entity_name": "Coverage",
        "generate_surrogate_key": true
      }
    ],
    "logical_entities_with_surrogate_keys": {
      "Coverage": [
        {
          "attribute_name": "Coverage_SK",
          "is_surrogate_key": true
        }
      ]
    }
  }
}
```

---

## 16. Current Functional Summary

As implemented today, this step:

- evaluates effective SCD state using confirmed tracked attributes where present
- forces surrogate keys for SCD Type 2 entities
- keeps surrogate keys optional for non-SCD2 entities
- defaults selected rows to all SCD2 entities only
- preserves business keys in the entity model
- allows datatype review before commit
- uses a clean assessment-first HITL pattern
- persists confirmed surrogate-aware logical structure as the new active session state
- propagates that state to Standard Naming, STTM, and DDL/DML

This document is the current deep technical source of truth for the Foundation Layer surrogate-key feature in `ADM_SILVER_LAYER`.

---

## 17. What We Changed In Practical Terms

For someone reading this without prior context, the practical implementation changes are:

- introduced a new first sub-step in Foundation Stage 4 called `Generate Surrogate Keys`
- added an assessment-first HITL flow before plan generation
- introduced deterministic surrogate-key planning logic through `SurrogateKeyGeneratorAgent`
- added confirm-time merge logic so surrogate choices become the new active structural state
- ensured confirmed SCD Type 2 tracked attributes override stale recommendation values
- updated downstream Stage 4 input consolidation so later agents consume surrogate-aware session data
- improved the UI review flow with locked mandatory rows, optional row selection, autoscrolling logs, and in-button confirmation progress

---

## 18. Recommended Reader Interpretation

If you are:

- a **data architect**: focus on Sections 2, 7, 9, 10, and 11
- a **frontend engineer**: focus on Sections 4.1, 7, 8.1, 8.10, and 8.11
- an **API engineer**: focus on Sections 4.2, 7.2, 8.2, 8.12, and 8.13
- an **agent/backend engineer**: focus on Sections 4.3, 5, 8.5, 8.6, and 9

That should help anyone understand both the business reason and the technical execution path of this feature quickly.
