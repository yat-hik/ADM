# Foundation Layer Surrogate Key Technical Architecture

## 1. Purpose

This document describes the enterprise implementation of the Foundation Layer surrogate-key workflow used in Stage 4 Physical Modeling.

The workflow intelligently manages surrogate key generation during Logical-to-Silver Physical modeling. It enforces surrogate keys where required, keeps them optional where allowed, derives the key structure from logical business keys, supports configurable surrogate key datatype, and propagates confirmed decisions into Standard Naming, STTM, and DDL/DML generation.

## 2. Scope

| Area | Value |
|---|---|
| Platform stage | Foundation Layer, Stage 4 Physical Modeling |
| UI step | Generate Surrogate Keys |
| Primary repos | `apps-azure`, `apis-azure`, `adm-foundation-layer-agents-azure` |
| Runtime model | React UI + FastAPI orchestration + Azure Function agent |
| Rule engine | Deterministic Python logic inside a LangGraph shell |
| Realtime logs | Azure Web PubSub |
| Persistence | ADM PostgreSQL metastore |

Out of scope:

- Standard Naming output remains lowercase physical naming, for example `product_sk`.
- The surrogate-key step does not perform SCD recommendation itself; it consumes SCD output.
- The surrogate-key step does not create source-system data values. It defines physical model structure and mapping intent.

## 3. Requirement Mapping

| Requirement | Implementation |
|---|---|
| Add Generate Surrogate Keys as first LDM physical step | Stage 4 first step is `Generate Surrogate Keys`. |
| Provide generate or skip option | Assessment returns `allow_skip`; UI shows skip only when allowed. |
| SCD2 mandatory and cannot be skipped | Agent marks Type 2 entities mandatory; UI locks them; backend forces generation for required rows. |
| If SCD2 exists, disable skip | Assessment returns `allow_skip=false` for any Type 2 entity. |
| User can choose INT/BIGINT, default INT | Plan defaults to `INT`; UI allows `INT` or `BIGINT`; API validates only these values. |
| Default selected tables are all SCD2 tables | SCD2 rows are preselected; optional rows are opt-in. |
| Business keys remain persisted | Confirmed output keeps `business_key` beside surrogate physical PK metadata. |
| STTM target fields include surrogate key | STTM generator reads surrogate-aware standard names and emits surrogate target fields. |
| STTM relationships use selected SK PK/FK | Confirmed surrogate relationships are passed to STTM and override prior business-key relationship columns only for selected surrogate-key parents. |
| Naming standard `<Entity_SK>` | Draft logical surrogate names use `Entity_SK`; Standard Naming preserves surrogate semantics as lowercase physical names. |
| Optional fact/table surrogate keys | Non-SCD2 entities are optional and can be selected or cleared by modeler. |
| SCD review skipped | Surrogate assessment returns a skip-only result; no generation option is shown. |

## 4. Stage Placement

Current Stage 4 order:

1. Generate Surrogate Keys
2. Standardize Names
3. Apply Transformations
4. Generate STTM
5. Generate DDL & DML
6. Download Artifacts

The surrogate-key step must run before naming and STTM because it can add columns, change physical primary keys, and add surrogate foreign keys. Downstream artifacts must be generated from the surrogate-aware model, not from stale logical entities.

## 5. Components

### 5.1 UI: `apps-azure`

Key files:

- `apps-azure/frontend/components/stages/Foundation_Layer/Stage4_PhysicalModeling.tsx`
- `apps-azure/frontend/components/stages/Foundation_Layer/stage4/GenerateSurrogateKeysView.tsx`

Responsibilities:

- Render `Generate Surrogate Keys` as first Stage 4 step.
- Submit assessment and plan requests.
- Show Web PubSub logs in the console.
- Type surrogate console logs letter by letter.
- Show assessment and generated plan.
- Lock SCD2 rows as mandatory.
- Let optional rows be selected, cleared, or left skipped.
- Let modeler choose `INT` or `BIGINT`.
- Confirm selected surrogate-key decisions.
- Store confirmed surrogate-enhanced logical entities for downstream Stage 4 UI flow.

### 5.2 API: `apis-azure`

Key files:

- `apis-azure/backend/api/routes/agents.py`
- `apis-azure/backend/api/routes/surrogate_key_helpers.py`
- `apis-azure/backend/api/routes/dependency_agents.py`

Responsibilities:

- Resolve upstream dependency sessions.
- Build normalized agent input.
- Persist request payload to `dfl_agent_input`.
- Trigger Azure Function activity.
- Negotiate Web PubSub URL.
- Confirm user selections.
- Validate surrogate key datatype.
- Merge confirmed surrogate output into a new active `dfl_adm_session` row.
- Shape surrogate-aware data for Standard Naming, STTM, and DDL agents.

### 5.3 Agent Runtime: `adm-foundation-layer-agents-azure`

Key files:

- `adm-foundation-layer-agents-azure/function_app.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/SurrogateKeyGeneratorAgent.py`
- `adm-foundation-layer-agents-azure/agents/SurrogateKeyGenerator/master/agent.py`

Responsibilities:

- Fetch stored agent input.
- Normalize logical entities, primary keys, roles, SCD recommendations, and relationships.
- Assess mandatory vs optional surrogate-key behavior.
- Auto-generate the plan when SCD2 exists.
- Persist output.
- Publish Web PubSub lifecycle messages.

## 6. Dependency Contract

`SurrogateKeyGeneratorAgent` depends on:

| Upstream agent | Purpose |
|---|---|
| `MapAttributesToConceptsAgent` | Logical entities and attributes. |
| `PrimaryKeyAgent-logical` | Business keys. |
| `EntityRoleRecommenderAgent` | Dimension/fact/other classification. |
| `SCDTypeRecommenderAgent` | SCD Type 1/2 decisions and confirmed tracked attributes. |
| `RelationshipIdentifierLogicalAgent` | Parent-child logical relationships for FK propagation. |

Dependency behavior:

- `SCDTypeRecommenderAgent` is required for generation decisions, but if the user skipped the SCD review step, the surrogate API returns a direct skip-only assessment instead of failing with `424`.
- `RelationshipIdentifierLogicalAgent` is optional for assessment. If it is missing, surrogate assessment can still run, but FK propagation cannot be produced.
- `StandardNamingGeneratorAgent` and `SttmGeneratorAgent` can run when `SurrogateKeyGeneratorAgent` is absent because the user skipped the surrogate step. In that case they use the pre-surrogate logical model, primary keys, and relationships.

Downstream consumers:

| Downstream agent | Consumption |
|---|---|
| `StandardNamingGeneratorAgent` | Uses surrogate-enhanced logical entities. |
| `SttmGeneratorAgent` | Uses surrogate-aware entities and primary-key metadata. |
| `STTMtoDDLAgent` | Uses surrogate-aware PK metadata for DDL PK generation. |

## 7. Execution Flow

### 7.1 Assessment Request

UI sends:

```http
POST /api/v1/agents/surrogate-key/execute
Content-Type: multipart/form-data

session_id=<session-id>
phase=assess
```

API performs:

1. Fetch dependency sessions.
2. If SCD review output is missing, return a direct skip-only assessment.
3. Otherwise build normalized agent input.
4. Save input into `dfl_agent_input`.
5. Trigger Azure Function `/agent/surrogate_key_generator`.
6. Return `instance_id` and negotiated Web PubSub URL.

UI then joins the Web PubSub group and renders streamed messages.

When SCD review was skipped, no Azure Function is started and no Web PubSub connection is required. The API response already contains the assessment payload.

### 7.2 Assessment Behavior

The agent builds an entity plan from the current upstream state.

Rules:

- `scd_type == Type 2` means surrogate key is mandatory.
- `scd_type != Type 2` means surrogate key is optional.
- Default surrogate key datatype is `INT`.
- Draft name is `<Entity>_SK`, with spaces converted to underscores.
- Business keys come from logical primary-key output.

If any Type 2 entity exists:

- `allow_skip=false`
- Type 2 entities are mandatory
- plan generation runs automatically in the same agent execution

If no Type 2 entity exists:

- `allow_skip=true`
- the modeler may skip or explicitly generate a plan

If SCD review was skipped or not completed:

- `allow_skip=true`
- `allow_generate=false`
- UI shows only `Skip and Continue`
- no surrogate session is persisted
- downstream steps run from the existing logical model and existing relationships

### 7.3 Plan Request

For optional models or manual replan:

```http
POST /api/v1/agents/surrogate-key/execute
Content-Type: multipart/form-data

session_id=<session-id>
phase=plan
```

Plan output includes:

```json
{
  "surrogate_key_generation": {
    "default_surrogate_key_type": "INT",
    "entity_plans": [
      {
        "entity_name": "Product",
        "entity_type": "Dimension",
        "scd_type": "Type 2",
        "business_keys": ["ProductCode"],
        "surrogate_key_required": true,
        "generate_surrogate_key": true,
        "skip_allowed": false,
        "surrogate_key_type": "INT",
        "surrogate_key_name": "Product_SK"
      }
    ],
    "relationship_plans": [
      {
        "parent_entity_name": "Product",
        "child_entity_name": "Sales",
        "parent_matching_column": "ProductCode",
        "child_matching_column": "ProductCode",
        "generated_foreign_key_name": "Product_SK"
      }
    ]
  }
}
```

## 8. Confirmation Flow

UI sends:

```http
POST /api/v1/agents/surrogate-key/confirm
Content-Type: application/json

{
  "session_id": "<session-id>",
  "session_name": "<session-name>",
  "user_name": "<user-name>",
  "confirmed_entities": [
    {
      "entity_name": "Product",
      "business_keys": ["ProductCode"],
      "surrogate_key_name": "Product_SK",
      "surrogate_key_type": "INT",
      "surrogate_key_required": true,
      "generate_surrogate_key": true
    }
  ]
}
```

API confirmation rules:

- `surrogate_key_type` must be `INT` or `BIGINT`.
- SCD2 required rows are forced to `generate_surrogate_key=true`.
- Existing generated surrogate artifacts are stripped before reapplying the confirmed plan.
- Base business keys remain persisted.
- Confirmed surrogate key becomes physical PK metadata.
- Parent surrogate keys are propagated to child entities when relationship plans exist.
- A new active `SurrogateKeyGeneratorAgent` session row is appended.

Confirmed output shape:

```json
{
  "surrogate_key_generation": {
    "logical_entities_with_surrogate_keys": {},
    "primary_key_data_with_surrogate_keys": {},
    "propagated_foreign_keys": [],
    "relationship_identifier_data_with_surrogate_keys": {
      "relationships": []
    },
    "confirmed_surrogate_keys": []
  }
}
```

`relationship_identifier_data_with_surrogate_keys.relationships` rewrites relationship columns only for parents where `generate_surrogate_key=true`.

For each rewritten relationship:

- `parent_matching_column` becomes the parent surrogate key
- `child_matching_column` becomes the propagated child surrogate FK
- `original_parent_matching_column` keeps the prior business-key column
- `original_child_matching_column` keeps the prior child business-key column
- `relationship_key_strategy` is set to `surrogate_key`
- existing relationship metadata is preserved, including relationship cardinality/type, confidence score, and `describe_relationship`

The confirmed surrogate relationship payload is a full relationship set:

- relationships whose parent has a selected surrogate key are rewritten to SK PK/FK columns
- relationships whose parent was not selected for surrogate key generation are preserved with their original business-key columns
- preserved relationships are marked with `relationship_key_strategy=business_key`
- if an upstream relationship is reversed but the selected SK entity appears on the child side and the other table carries that entity's business key, confirmation corrects the direction and marks `relationship_direction_corrected=true`

## 9. Reassessment Semantics

Reassessment must always rebuild from current upstream SCD and logical-model state.

When the user runs assessment again:

1. UI clears local surrogate plans.
2. UI clears confirmed surrogate-enhanced entity state.
3. API rebuilds input from upstream dependency sessions.
4. Agent reassesses SCD Type 2 from current SCD output.
5. Existing confirmed surrogate entities are not used as the assessment source.

This allows model changes or SCD edits to affect the next assessment correctly.

## 10. UI Behavior

### 10.1 Plan Review

- Table list shows all planned entities.
- SCD2 rows display mandatory state and cannot be toggled off.
- Non-SCD2 rows are optional and can be included or excluded.
- `Select All` includes every table.
- `Clear Optional` leaves mandatory SCD2 selected and clears only optional rows.
- Detail panel is collapsed by default.
- Clicking a table opens its detailed view.
- Clicking the same table again closes the detailed view and expands the table list to full width.

### 10.2 Logs

Surrogate logs are rendered in the modal console.

Log sources:

- UI request submission status.
- Web PubSub connection status.
- Web PubSub messages published by the Azure Function.
- Error messages from UI request/response failures.

Rendering behavior:

- Each log line is typed letter by letter.
- Completed latest log keeps a cursor/pulse while waiting for the next message.
- Agent text avoids first-person phrasing such as `I found` or `I am`.

## 11. Naming Rules

Draft logical surrogate key name:

```text
<Entity>_SK
```

Examples:

| Entity | Draft surrogate key |
|---|---|
| Product | `Product_SK` |
| Insurance Policy | `Insurance_Policy_SK` |

Physical standardized name remains lowercase because Standard Naming uses the platform physical naming convention:

| Draft name | Standardized physical name |
|---|---|
| `Product_SK` | `product_sk` |
| `Insurance_Policy_SK` | `insurance_policy_sk` |

The important invariant is that surrogate keys remain surrogate-key names ending in `_sk`; they are not converted to `_id`.

## 12. Physical Modeling Example

Logical:

```text
Product(ProductCode, ProductName)
Sales(ProductCode, DateId, SalesValue)
```

If `Product` is SCD2 and `Sales` is non-SCD2:

```text
Product(
  Product_SK,
  ProductCode,
  ProductName
)

Sales(
  Product_SK,
  ProductCode,
  DateId,
  SalesValue
)
```

If the modeler also selects optional surrogate key for `Sales`:

```text
Sales(
  Sales_SK,
  Product_SK,
  ProductCode,
  DateId,
  SalesValue
)
```

`Sales_SK` is optional because `Sales` is not SCD2.

## 13. Data Persistence

### 13.1 `dfl_agent_input`

Stores normalized input for the surrogate run:

- logical entities
- primary key data
- entity roles
- SCD recommendations
- relationship identifier data
- phase

### 13.2 `dfl_adm_session`

Stores the active and historical outputs for `SurrogateKeyGeneratorAgent`.

On confirmation, a new active row is appended with:

- base logical entities
- base primary-key data
- confirmed surrogate entities
- surrogate-aware primary-key metadata
- propagated foreign-key metadata

## 14. Downstream Integration

### 14.1 Standard Naming

The API consolidation for `StandardNamingGeneratorAgent` checks the surrogate session first. If confirmed surrogate entities exist, those are passed to Standard Naming.

Required effect:

- surrogate key attributes are included in naming input
- surrogate key names remain `_sk`
- output remains lowercase physical naming

### 14.2 STTM

The STTM generator receives:

- surrogate-aware standard names
- surrogate-aware primary-key metadata
- relationship data

Required effect:

- `Target_Field` contains surrogate keys
- physical PK flags follow confirmed surrogate primary keys
- propagated FK fields appear when parent surrogate keys are selected
- `physical_relationships` use standardized SK columns for selected surrogate-key parents
- child SK rows are marked `FK`

Relationship source priority:

1. `surrogate_key_generation.relationship_identifier_data_with_surrogate_keys`
2. top-level `relationship_identifier_data_with_surrogate_keys`
3. original `RelationshipIdentifierLogicalAgent` output

The fallback path is intentional. If the user skips surrogate keys or no surrogate relationships were confirmed, STTM keeps the original logical relationship columns.

### 14.3 DDL/DML

DDL generation receives surrogate-aware primary-key metadata. Physical table DDL must use the confirmed surrogate key as PK where selected.

## 15. Build Checklist

To rebuild this feature from scratch:

1. Add Stage 4 UI step `Generate Surrogate Keys` before Standard Naming.
2. Add modal with assessment, plan list, datatype selection, confirm, skip, and logs.
3. Add API route `POST /surrogate-key/execute`.
4. Add API route `POST /surrogate-key/confirm`.
5. Add dependency mapping for `SurrogateKeyGeneratorAgent`.
6. Normalize upstream dependency outputs.
7. Implement deterministic assessment:
   - SCD2 mandatory
   - non-SCD2 optional
   - INT default
   - `<Entity>_SK` naming
8. Auto-plan when SCD2 exists during assessment.
9. Persist agent output to metastore.
10. Publish Web PubSub messages.
11. Confirm selections with backend validation.
12. Merge confirmed surrogate plan into session state.
13. Route surrogate-aware entities into Standard Naming.
14. Route surrogate-aware PK data into STTM and DDL.
15. Verify reassessment uses upstream SCD state, not confirmed surrogate entities.

## 16. Validation Checklist

| Scenario | Expected Result |
|---|---|
| No SCD2 entities | Skip is available; plan generation optional. |
| SCD review skipped | Skip is available; generation is unavailable; downstream uses existing relationships. |
| One or more SCD2 entities | Skip is disabled; plan auto-generated. |
| SCD2 row toggled off in UI | UI prevents it; backend also forces generation. |
| Optional table selected | Surrogate key added for that table on confirm. |
| Optional table cleared | No surrogate key added for that table. |
| Datatype BIGINT selected | Confirmed surrogate attribute uses `BIGINT`. |
| Invalid datatype sent to API | API returns 400. |
| Business keys exist | Business keys remain as table attributes and metadata. |
| Confirm then reassess | Assessment rebuilds from upstream SCD/logical state. |
| Standard Naming runs after confirm | Surrogate fields are present and standardized lowercase. |
| STTM runs after confirm | Target fields include surrogate keys. |
| STTM relationships after confirm | `physical_relationships` use selected SK PK/FK columns. |
| STTM after surrogate skip | `physical_relationships` remain on original logical/business-key columns. |

## 17. Operational Notes

- The surrogate-key decision is deterministic and should not depend on LLM output.
- The LLM-based agents upstream may influence the inputs, but the mandatory/optional rule is explicit.
- Confirmation is the state transition that makes surrogate output authoritative.
- Reassessment should clear local confirmed surrogate state before executing.
- Web PubSub is the source for runtime progress logs after request submission.
- UI should not invent agent reasoning logs; it should display request status, connection status, streamed events, and errors.
- Direct skip-only assessment is returned without Web PubSub when SCD review output is missing.
- Function/API logs use Python logging for operator diagnostics; UI shows only user-relevant messages.

## 18. Release Notes

### 18.1 Functional Changes

- Added `Generate Surrogate Keys` as the first Stage 4 physical-modeling step.
- Added SCD-aware assessment with mandatory SCD Type 2 surrogate keys.
- Added optional surrogate-key selection for non-SCD2 entities.
- Added `INT`/`BIGINT` datatype selection with `INT` default.
- Added skip-only behavior when Stage 3 SCD review was skipped.
- Added confirmation persistence for surrogate-enhanced logical entities, PK metadata, FK propagation, and surrogate-aware relationships.
- Added STTM integration so confirmed SK relationships override prior business-key relationships only when surrogate keys are selected.
- Preserved existing logical/business-key relationships when surrogate keys are skipped.

### 18.2 Reliability And Logging

- Added structured API exception handling for surrogate-key execution.
- Added function-app exception logging and Web PubSub error events.
- Added UI parsing for structured API errors.
- Added red/green log status rendering and typewriter log animation.
- Removed duplicate SCD tracked-attribute fetch during STTM generation.

### 18.3 Downstream Contract

- Standard Naming accepts absence of `SurrogateKeyGeneratorAgent` when the surrogate step was skipped.
- STTM accepts absence of `SurrogateKeyGeneratorAgent` when the surrogate step was skipped.
- STTM uses confirmed surrogate relationships from nested `surrogate_key_generation.relationship_identifier_data_with_surrogate_keys`.
- STTM marks generated child SK columns as `FK` by checking both logical and standardized target column names.
- Surrogate relationship plans preserve original relationship metadata and only replace PK/FK matching columns when the parent entity was selected for surrogate-key generation.
- Confirmed surrogate relationship output remains a full relationship set, so non-selected relationships are not dropped when selected SK relationships exist.
- Direction-corrected relationships use the selected parent entity's SK as both `parent_matching_column` and propagated child FK, while retaining original business-key column metadata.
