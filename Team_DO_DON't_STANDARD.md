# ADM Silver Layer Team Do / Don't Standard

Last updated: 2026-05-05
Scope: `apps-azure`, `apis-azure`, `adm-foundation-layer-agents-azure`

This is the short team version of the enterprise standard.

Primary reference: [ENTERPRISE_CODING_STANDARD.md](/home/yathikvanka/ADM_SILVER_LAYER/ENTERPRISE_CODING_STANDARD.md:1)

## Non-Negotiables

### Secrets

Do:

- use per-service `.env.example`
- use Azure Key Vault or platform secret injection in deployed environments
- keep frontend, API, and agent config separate

Don't:

- hardcode passwords, keys, tokens, URLs, or connection strings
- commit real `.env` files
- put secrets in comments, samples, or Dockerfiles

Why:

- hardcoded secrets are a direct compromise risk and hard to rotate safely

### Frontend Config

Do:

- use `VITE_*` variables for frontend runtime configuration
- keep API base URLs environment-specific
- keep Vite proxy usage limited to local development

Don't:

- hardcode backend URLs in React source
- use `vite.config.ts` as the real application service registry
- rely on localhost fallback URLs in production code

Why:

- hardcoded UI URLs create environment drift and cause unsafe cross-environment coupling

### API Contracts

Do:

- create explicit API methods per feature or agent
- use typed request and response models
- make UI calls map to real backend business endpoints

Don't:

- use one generic UI fetch pattern that only passes `agent_name`
- route all new agent behavior through one shared execution endpoint
- hide business behavior behind generic `executeAgent()` calls

Why:

- explicit contracts improve isolation, testing, logging, auth, and code review quality

### Browser Storage

Do:

- keep active workflow state in memory
- persist recoverable state in backend storage
- use `sessionStorage` only for minimal non-sensitive UI state if absolutely needed

Don't:

- store workflow outputs, session payloads, tokens, or model artifacts in `localStorage`
- treat browser storage as trusted

Why:

- one XSS issue can expose or poison all `localStorage` content

### Database Safety

Do:

- parameterize all query values
- allowlist dynamic table names and column names
- centralize shared DB access logic

Don't:

- write f-string SQL in runtime paths
- duplicate DB helpers across routes
- mix query-building logic everywhere

Why:

- this is the cleanest way to prevent SQL injection and reduce query drift

### Network Security

Do:

- keep TLS validation enabled
- use explicit CORS allowlists by environment
- restrict hosts and origins to known values

Don't:

- use `verify=False`
- use wildcard-like host or origin settings outside local dev

Why:

- disabling certificate validation allows service impersonation and trust-boundary failure

### Agent Architecture

Do:

- expose per-agent or per-capability endpoints
- give each agent its own config, timeout, logging, and auth scope
- share common connector code from one package

Don't:

- dispatch all agents through one generic route forever
- copy connector implementations into every agent folder
- create duplicate variables and near-duplicate config names

Why:

- enterprise systems need isolation, low drift, and predictable operations

### Storage Architecture

Do:

- keep relational metastore data in Postgres
- use MongoDB for session documents, workflow state, and agent input/output history if the platform approves it
- use managed vector capability aligned to the chosen platform

Don't:

- force document-heavy session state into versioned relational JSON blobs long term
- depend on local filesystem vector storage in production architecture

Why:

- document-centric workflow state fits MongoDB better than relational session-row versioning

## PR Gate Checklist

- no hardcoded secrets
- no hardcoded frontend URLs
- no `verify=False`
- no banned `localStorage` usage
- no f-string SQL
- no new generic agent routing
- no duplicated connectors
- correct env ownership for the service being changed

## Priority Order

### P0

- remove hardcoded secrets
- remove hardcoded frontend URLs
- remove `verify=False`
- remove sensitive `localStorage`
- tighten CORS and allowed hosts

### P1

- replace generic UI fetch patterns with explicit API methods
- split generic backend and Function routes into explicit agent endpoints
- centralize shared connectors and DB helpers

### P2

- move session workflow documents to MongoDB
- replace local LanceDB architecture with managed vector capability if approved

## Team Rule

If a new implementation makes it harder to answer these three questions, do not merge it:

1. Which UI screen calls which exact backend endpoint?
2. Which service owns this config value?
3. Where is this sensitive state stored, and why is that storage safe?
