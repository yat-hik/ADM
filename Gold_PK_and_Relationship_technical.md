# Gold Agent Design

## Scope

This document describes the current design of the GOLD-layer:

- `SurrogateKeyAgent`
- `GoldRelationshipAgent`

It covers:

- purpose
- input contracts
- output contracts
- rule coverage
- current implementation behavior
- supported review modes
- known limitations

## Components

Relevant files:

- [SurrogateKeyAgent.py](/home/yathikvanka/ADM_GOLD_LAYER/adm-product-layer-agents-azure/agents/SurrogateKeyAgent/SurrogateKeyAgent.py)
- [GoldRelationshipAgent.py](/home/yathikvanka/ADM_GOLD_LAYER/adm-product-layer-agents-azure/agents/GoldRelationshipAgent/GoldRelationshipAgent.py)
- [prompt_library.py](/home/yathikvanka/ADM_GOLD_LAYER/adm-product-layer-agents-azure/agents/GoldPromptLibrary/prompt_library.py)
- [agents.py](/home/yathikvanka/ADM_GOLD_LAYER/adm-product-layer-apis-azure/backend/api/routes/agents.py)

## Purpose

### SurrogateKeyAgent

Determines the final primary key strategy for each GOLD table.

Expected outcomes:

- choose `natural-composite` when a valid business-key-based key can represent the declared grain
- choose `surrogate_key` when a valid natural key cannot be confirmed from the available candidates

### GoldRelationshipAgent

Determines valid analytical joins between GOLD facts/aggregates and dimensions.

Expected outcomes:

- return only valid `dimension_to_fact` relationships
- enforce `one-to-many`
- avoid fact-to-fact joins
- avoid surrogate-key-based analytical joins

## Review Modes

The upstream "Review Entities and Attributes" step can produce two effective shapes:

### Yes

Source tables are mapped largely as-is.

Examples:

- raw source aliases like `customer`, `sales`, `web_engagement`
- KPI attribute sources like `sales.customer_key, customer.customer_key`

### No

Dimension/fact-style modeled tables are created.

Examples:

- `dim_customer`
- `fact_sales`
- `fact_web_engagement`

## Supported Input Behavior

Both agents support both review modes through the same payload contract.

This works because the implementation resolves:

- modeled table names like `dim_customer`
- raw aliases stored under `source_table` like `customer`
- multi-source references like `sales.customer_key, customer.customer_key`

## API Flow

### SurrogateKey API

Endpoint:

- `POST /surrogate-key`
- `POST /surrogate-key/execute`

The API reads the latest `LogicalModalAgent` session output and builds:

```json
{
  "KPI_Mappings": { "...": "..." },
  "SourceTables": { "...": "..." }
}
```

### GoldRelationship API

Endpoint:

- `POST /gold-relationship/execute`

The API reads:

- `LogicalModalAgent`
- `SurrogateKeyAgent`

And builds:

```json
{
  "KPI_Mappings": { "...": "..." },
  "SourceTables": { "...": "..." },
  "SurrogateKeyDecisions": { "...": "..." }
}
```

## Tech Stack

- Python
- Pydantic
- Azure OpenAI through `LLMFactory`
- JSON session payloads
- DB-backed session handoff through API
- Shared prompt templates from `GoldPromptLibrary`

## Architect Rules Covered

## SurrogateKey Rules

Source rules:

- primary key must fully represent the declared grain
- prefer natural composite keys derived from business identifiers
- use surrogate keys only when:
  - natural keys are excessively large
  - required by BI or semantic tooling
- time must be included in the key when KPIs are time-variant
- no nullable attributes in primary keys

Current implementation coverage:

- constructs allowed key candidates from GOLD-side mapped key attributes
- treats time-variant grain as requiring a time key in a natural-composite result
- validates that chosen natural keys come from allowed candidates only
- normalizes surrogate key naming to `<table_name>_sk`

Important note:

The implementation does not directly infer BI-tooling requirements. It relies on the LLM decision plus the available candidates and validation rules.

## Relationship Rules

Source rules:

- facts join to dimensions using foreign keys
- relationship cardinality must be one-to-many
- direction must be dimension to fact
- many-to-many must be handled through bridge tables
- no fact-to-fact joins
- dimensions must not filter facts beyond declared grain

Current implementation coverage:

- builds candidate relationships only from dimension-like sources
- validates output as `dimension_to_fact`
- validates `one-to-many`
- rejects fact-to-fact joins
- excludes surrogate-key analytical joins
- only accepts gold-side mapped join columns

## Input Contract

### SurrogateKeyAgent Input

```json
{
  "KPI_Mappings": {
    "<gold_table_name>": {
      "attributes": [
        {
          "desc": "...",
          "source": "...",
          "attr_name": "...",
          "data_type": "..."
        }
      ],
      "included_kpis": ["..."]
    }
  },
  "SourceTables": {
    "<table_name>": {
      "attributes": [...],
      "source_table": "...",
      "dim_name": "...",
      "fact_name": "...",
      "bridge_name": "..."
    }
  }
}
```

### GoldRelationshipAgent Input

```json
{
  "KPI_Mappings": { "...": "..." },
  "SourceTables": { "...": "..." },
  "SurrogateKeyDecisions": {
    "<gold_table_name>": {
      "primary_key_strategy": "...",
      "primary_key": ["..."],
      "surrogate_key": ["..."],
      "final_primary_key": ["..."]
    }
  }
}
```

## Output Contract

### SurrogateKeyAgent Output

```json
{
  "SurrogateKeyDecisions": {
    "<gold_table_name>": {
      "table_name": "...",
      "table_role": "fact|aggregate|dimension",
      "all_key_candidates": ["..."],
      "primary_key_strategy": "natural-composite|surrogate_key",
      "primary_key": ["..."] or null,
      "surrogate_key": ["..."] or null,
      "final_primary_key": ["..."],
      "final_primary_key_strategy": "natural-composite|surrogate_key",
      "justification": "...",
      "warnings": [],
      "dimension_tables": [...],
      "gold_table_options": {
        "table_name": "...",
        "candidate_keys": ["..."]
      },
      "silver_table_options": [...]
    }
  }
}
```

### GoldRelationshipAgent Output

```json
{
  "GoldRelationshipDecisions": {
    "<gold_table_name>": {
      "gold_table_name": "...",
      "table_role": "fact|aggregate",
      "declared_grain": "...",
      "grain_contributors": [...],
      "bridge_required": false,
      "bridge_tables": [],
      "dimension_tables": ["..."],
      "warnings": [],
      "relationships": [
        {
          "from_table": "...",
          "from_column": "...",
          "to_table": "...",
          "to_column": "...",
          "relationship_type": "dimension_to_fact",
          "cardinality": "one-to-many",
          "confidence": 1.0,
          "justification": "..."
        }
      ]
    }
  }
}
```

## Current Functional Coverage

### SurrogateKeyAgent

Implemented behavior:

- parses KPI attributes and source references
- resolves raw aliases to modeled tables
- collects allowed key candidates from GOLD-side mapped attributes
- identifies time-like attributes such as `month_key`, `week_key`, `date_key`
- lets the LLM choose strategy
- validates the LLM output against allowed candidates
- auto-adds a time key only when:
  - the LLM selected natural-composite
  - a valid natural key exists
  - the grain is time-variant
  - and an allowed time candidate exists
- keeps LLM justification as-is whenever possible

### GoldRelationshipAgent

Implemented behavior:

- parses one or more source references per attribute
- resolves raw aliases to modeled dimension tables
- builds valid candidate joins from mapped source metadata
- validates LLM relationship output against those candidates
- filters out generated surrogate key artifacts from `grain_contributors`
- falls back to deterministic candidate relationships only if LLM validation fails

## Example Behavior

### Good SurrogateKey Outcomes

Examples that align well:

- `gold_fct_customer_monthly`
  - `["customer_key", "month_key"]`
- `gold_fct_customer_unknown`
  - `["customer_key"]`

These are strong because they:

- use business identifiers
- include time when needed
- come directly from allowed candidates

### Acceptable SurrogateKey Outcome

- `gold_fct_engagement_weekly`
  - surrogate key

This is acceptable when only a time key is available and no business identifier is present in the allowed candidates.

### Higher-Risk SurrogateKey Outcome

- `gold_fct_performance_monthly`
  - `["month_key", "date_key"]`

This is structurally valid by current rules because:

- both are allowed candidates
- time is included

But it is still a modeling risk because it lacks a non-time business identifier.

## Known Limitations

- The agents depend heavily on the quality of upstream `KPI_Mappings`.
- The surrogate-key logic cannot prove BI-tooling requirements from metadata alone.
- `declared_grain` is generated text and may be broader than a human architect would write.
- `grain_contributors` is a UI enrichment field, not part of the architect’s minimal output requirement.
- Time-only fact tables may still be ambiguous from a strict modeling perspective.

## Current Assessment

Based on the reviewed examples:

- `SurrogateKeyAgent`: approximately `85%` aligned
- `GoldRelationshipAgent`: approximately `88%` aligned
- overall: approximately `86-87%` aligned

Primary residual risk:

- some time-based fact tables may still produce technically valid but semantically weak natural-composite keys when no non-time business identifier is available

## Recommended Validation

Before further code changes, validate with more real sessions covering:

- time-only KPI tables
- mixed campaign/customer/date tables
- bridge-table scenarios
- fact-like sources that should not become dimensions
- sparse metadata cases

## Summary

The current design is stable for both Yes and No review modes.

Strengths:

- shared payload contract
- alias-aware source resolution
- business-key-first surrogate strategy
- strict dimension-to-fact relationship validation

Main caution:

- outputs are good, but some edge-case modeling semantics still depend on LLM judgment and the quality of upstream metadata.
