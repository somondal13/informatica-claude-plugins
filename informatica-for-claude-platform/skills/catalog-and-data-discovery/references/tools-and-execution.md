# Tools & Execution Model — Full Reference

> Canonical source for SKILL.md §4 (Tools & Segments). Read when the agent is about to issue tool calls (especially multi-round scenarios or data-exploration hand-off).

Two read-only discovery tools. This skill does NOT enrich, edit, classify, or certify assets — those are steward workflows outside its scope. When a gap is found, the skill reports it and points to the steward; it never proposes to write to the catalog.

## `search_assets` — find assets by query + filters

Search the catalog by natural language (NL) or keyword. Returns a page of asset summaries — identity, `classType`, name, description, stakeholders, assetGroups — plus optional aggregations for narrowing and `suggested_next_steps` for iterative refinement. Always call this **first** to discover asset identities before drilling into `get_asset_details`.

**Parameters:** `query` (required; `*` = whole catalog), `mode` (`KEYWORD` default, `NL` fallback), `filterSpec` (typed filters — see [asset-types.md](asset-types.md) §3.4, [search-rules.md](search-rules.md)), `aggregationSpec` (max 1 aggregation, ≤20 buckets), `sortSpec` (max 2), `from`, `size` (default 20).

## `get_asset_details` — fetch typed asset by identity

Fetch full details of one asset by identity. Always returns a `resolvedSummary` (identity, name, type, location, description, stakeholders resolved to names/emails, assetGroups, timestamps) plus raw `systemAttributes`. Additional aspects are opt-in via `segments[]` — pick the minimum set for the intent (batching rules below).

**Parameters:** `assetIdentity` (required), `identityType` (`INTERNAL` = UUID default, `EXTERNAL` = path), `segments[]` (see below; default is `summary` only).

**Always use internal UUID (the `identity` field from search results) with the default `INTERNAL` identity type.** Do not use `identityType: EXTERNAL` — the `externalId` field includes a `~classType` suffix that the API rejects with 400. Do not attempt to construct external IDs from child asset paths. External IDs are only needed for the `data_explore` hand-off ("Reading actual data" below), where you pass the `externalId` value exactly as returned from search/details — never constructed.

## Segments

- **`summary`** (default) — core identity + lifecycle: name, `classType`, description, `certified`, rating, `assetLifecycle`, timestamps, stakeholders, path.
- **`selfAttributes`** — self-declared attributes: `sourceStatementText` (view SQL), `NumberOfRows`, custom attributes, `businessName`.
- **`stakeholdership`** — full stakeholder list with governance roles (Data Owner, Data Steward, etc.), beyond the truncated `summary` view.
- **`glossary`** — linked business terms with `curationStatus` (`ACCEPTED` | `INFERRED` | `REJECTED`) — the load-bearing signal for meaning verdicts (§2 rule 2 in SKILL.md).
- **`hierarchy`** — declared children along the containment chain: columns under a Table, PK/FK/indexes, SubDomains under a Domain.
- **`neighborhood`** — associations to other assets: declared PK/FK table-to-table joins (`core.PKFK`), glossary links, DQ rules, `RelatedTo`. Excludes `ParentChild`, `DataFlow`, `ClassifiedAs`.
- **`dataClassification`** — PII, PCI, and other sensitivity labels (technical assets only).

## Batching rules

- TRUST_ASSESSMENT → `[selfAttributes, stakeholdership, glossary]` (`summary` is always returned and provides `modifiedOn` for Freshness scoring)
- RELATIONSHIPS → `[hierarchy, neighborhood]`
- SENSITIVITY → `[hierarchy, dataClassification]`
- FIELD_MEANING → `[hierarchy]` on parent, then `[glossary, selfAttributes]` per child

## Parallel execution model

Issue independent tool calls in the **same turn** — do not wait for one to return before sending the next. Two calls are independent when neither needs a value from the other's response.

**Round model for BROAD_DISCOVERY (target: 2–3 rounds depending on path):**

Column UUIDs can arrive from two sources — Round 1 search results or Round 2 hierarchy. Which path applies determines whether glossary wiring runs in Round 2 or Round 3.

```
Round 1 (parallel):  search(topic) + search(dimension terms) + search(fiscal terms)
                     ↓ identities resolve — check: did search return column UUIDs?

─── Fast path (columns in Round 1 results — 2 rounds total) ─────────────────
Round 2 (parallel):  get_asset_details(top asset, [hierarchy, selfAttributes, dataClassification, stakeholdership])
                     + get_asset_details(date column, [glossary])      — fiscal wiring
                     + get_asset_details(region column, [glossary])    — dimensional wiring
                     Column UUIDs already known → glossary runs alongside hierarchy, not after it.

─── Slow path (columns NOT in Round 1 results — 3 rounds) ───────────────────
Round 2 (parallel):  get_asset_details(top asset, [hierarchy, selfAttributes, dataClassification, stakeholdership])
                     + get_asset_details(related dimension table, [hierarchy])
                     ↓ column identities resolve from hierarchy
Round 3 (parallel):  get_asset_details(date column, [glossary])      — fiscal wiring
                     + get_asset_details(region column, [glossary])   — dimensional wiring
```

**Rules:**
- All searches in step 1–3 of the BROAD_DISCOVERY strategy are independent → **always parallel in Round 1**.
- **Path selection after Round 1:** inspect search results for column/DataElement identities. If the columns you need for glossary wiring (date column, region column, etc.) appeared in Round 1 results, their UUIDs are already known — take the fast path. If search returned only the parent table (common when column names don't match the keyword), take the slow path.
- **Fast path:** all `get_asset_details` calls — hierarchy on the parent, glossary on known columns — are independent → **all parallel in Round 2**. Do NOT wait for hierarchy to "discover" column UUIDs you already have from search. This eliminates Round 3 entirely.
- **Slow path:** hierarchy fetch in Round 2 → column UUIDs resolve → glossary wiring checks on those columns in Round 3.
- **Do not re-search for columns the hierarchy already gave you.** After Round 2's hierarchy fetch returns the full column list with UUIDs, use those UUIDs directly for Round 3 glossary calls. A separate `search_assets(query="ORDER_DATE")` to find a column UUID that hierarchy already returned is a wasted call.
- Escalation-on-zero (BROAD_DISCOVERY step 3) is inherently sequential — it retries only when the prior call returned zero. This is the one case where waiting is correct.
- Segment batching (above) still applies — combine segments into one call per asset rather than one call per segment.

## Limitations (state when relevant)

1. No lineage — `DataFlow` is excluded from `neighborhood`.
2. FK/PK objects appear as `hierarchy` children; the join they define ALSO appears in `neighborhood` as a `core.PKFK` table-to-table edge — both are valid evidence of a declared relationship.
3. `neighborhood` excludes `DataFlow` lineage, `ParentChild`, and `ClassifiedAs` associations.
4. No reverse lookup — search by name to find dependents.
5. `data_explore` / `master_data_explore` expose **no structured field for resolved catalog facts** — resolved measures, filters, and date ranges must ride inside the free-text `prompt` (see "Enriching the hand-off prompt" below). This is a documented workaround; the durable fix is a structured resolved-filters field on the tool contract. Until then, the pre-hand-off verification gate is required because prompt enrichment is not otherwise enforced.

## Reading actual data — hand off to `informatica-data-exploration`

This skill is **metadata-only** — it decides *which* asset is right, trusted, and compliant, and never reads row values. When the user's question needs the **actual data** (row values, counts, aggregates, ranked lists, anti-joins, sampling the contents) — not just picking the asset — hand off to the **`informatica-data-exploration`** MCP *after* discovery has resolved the trusted asset and its identity.

**Sequence (catalog-first, always):** run discovery to pick the right/trusted/compliant asset and resolve its identity → then call `informatica-data-exploration` to read the data → carry the governance verdict forward (do not read data from an asset you flagged FORBIDDEN/CAUTION in the SAFE_USAGE strategy without stating the gate).

**How to call it:**
- Authenticate first: `authenticate` → `complete_authentication`. (The data-explore token is a flat 60-minute TTL with no refresh — re-authenticate when it expires.)
- Then call **`data_explore`** with the request payload.

**MANDATORY — always send `external_id` (single asset) or `external_ids` (multiple assets) in the `data_explore` payload.** This is the asset's **EXTERNAL** catalog identity — the same value passed as `assetIdentity` with `identityType: EXTERNAL` — carried straight through from the `search_assets` / `get_asset_details` response. If `external_id`/`external_ids` is omitted, `data_explore` returns **empty result frames (zero rows) silently — no error is raised**. An empty frame therefore means "the external_id was missing," not "the asset has no data." Never call `data_explore` without it. The `prompt` you send is equally mandatory — it MUST be enriched with the resolved facts (see "Enriching the hand-off prompt" below), never the raw user question. Re-confirm the exact field spelling against the tool schema if a call unexpectedly returns empty.

For sources the `informatica-data-exploration` does not cover, pass the resolved path into the direct source MCP instead (Snowflake, Postgres, S3, BI). If the relevant connector is installed but not enabled in this chat, its tools will not appear — say so and ask the user to enable it, rather than reporting the question as unanswerable.

## Enriching the hand-off prompt (MANDATORY)

`external_id` binds the *asset* (see "Reading actual data" above). It does NOT carry anything else discovery resolved. Whenever discovery resolved a fact the query engine needs — a Metric's business-logic formula, a `BusinessTerm`'s `AliasNames`, a fiscal/period term's date range, a dimension term's resolved member list, the specific measure column, or column wiring — the `prompt` argument passed to `data_explore` / `master_data_explore` **MUST be a rewritten, fully-resolved instruction carrying those facts. It MUST NOT be the user's raw question.** Include the resolved table name and external_id, the column mapping (which columns serve the user's business terms), any filters resolved from catalog terms (fiscal period boundaries, region lists), and any governance gates (PII columns to mask, RTBF/consent filters to apply).

**Why this is mandatory, not a nicety.** These tools route on the `prompt` string verbatim; there is **no separate structured field for resolved catalog facts** (see Limitations above). Anything discovery learned is invisible to the agent unless it is written into the prompt string. If the raw question is passed, the agent re-resolves the terms itself — off possibly-stale profiles — and silently returns a wrong number. (Observed live: the raw prompt `"Q2 Sales Revenue for EMEA"` made the agent filter `WHERE LOWER(SALES_REGION)='emea'`, which matched zero rows against country-valued data.)

**Prompt-rewrite template (ASCII only — no em-dashes, no angle brackets in the actual prompt string; both trip the MCP input validator):**

```
Sum MEASURE_COLUMN from TABLE where DIM_COLUMN in (resolved member list)
and DATE_COLUMN between resolved_fiscal_start and resolved_fiscal_end
[and status/scope filters]. Use these exact filter values; do not re-derive
the region or the period from the data.
```

**Worked example (the EMEA case):**

- Raw (WRONG — do not send): `Q2 Sales Revenue for EMEA`
- Enriched (CORRECT): `Sum NET_SALES from FACT_ORDER where SALES_REGION in ('France','Italy','Germany') and ORDER_DATE between 2026-05-01 and 2026-07-31. Use these exact filter values; do not re-derive the region or the period from the data.`

**Negative example (this is the live bug, not a valid call).** Emitting `data_explore` with `"prompt": "<the user's raw question>"` — even with `external_id` present — is a defect. Correctly decoding the terms in your visible reasoning is NOT a substitute for putting the resolved facts into the prompt; only the prompt string reaches the engine.

**Pre-hand-off verification gate (silent — run before emitting ANY `data_explore` / `master_data_explore` call; mirrors the Verdict Self-Check in SKILL.md):**

1. Does `prompt` contain the resolved **measure column**? If NO → rewrite before sending.
2. Does `prompt` contain the resolved **dimension member list** (the actual values, e.g. the countries) rather than the business-term label (e.g. "EMEA")? If NO → rewrite.
3. Does `prompt` contain the resolved **fiscal date range** (explicit start and end dates), not a bare period word ("Q2")? If NO → rewrite.
4. Is `prompt` byte-identical or near-identical to the user's **raw question**? If YES → STOP: it has not been enriched — rewrite before sending.
5. Is `external_id` / `external_ids` present (see "Reading actual data" above)? If NO → add it.

This gate is internal only — never print it. Its purpose is to catch an un-enriched hand-off before it reaches the engine, since prompt enrichment is not otherwise enforced.
