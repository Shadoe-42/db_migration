# Snowflake Ingestion & Landing Patterns
*Meridian Analytics — how it actually gets in the door | July 2026*

---

## Purpose

`source_pattern_playbooks.md` covers getting data out of the source. This doc covers the other half: once data is sitting in cloud storage or arriving as a change stream, how it actually lands in Snowflake, and — just as importantly — where it lands before it's trusted enough to reach a governed schema.

---

## Ingestion Mechanism, Chosen by Latency Requirement

Three mechanisms, not one default, matched to what the assessment's write-pattern rating actually requires:

**Batch — external stage + `COPY INTO`.** The right default for one-time migrations and scheduled bulk loads. Files land in S3 or GCS, an external stage points at that location, and `COPY INTO` loads on a schedule or on demand. No continuous infrastructure to maintain, and the simplest mechanism to validate — which matters directly for `validation_and_cutover.md`'s reconciliation step, since a discrete batch load has a clean start and end point to check against.

**Snowpipe — continuous, file-based.** For sources producing a steady trickle of new files (a CDC tool writing change files to cloud storage, for instance) where near-real-time matters but sub-10-second latency doesn't. Snowpipe auto-ingests as files land, without the operational overhead of a scheduled task polling for new data.

**Snowpipe Streaming — row-level, no staging.** For genuinely latency-sensitive ingestion where data arrives as a stream rather than files — a CDC tool writing directly via the Snowflake ingestion SDK rather than dropping files. The current high-performance architecture ingests through a `PIPE` object rather than a table object, and Snowflake auto-creates a managed pipe (`<table_name>-streaming`) the first time data streams into a table — no `CREATE PIPE` step required. Verified current benchmarks put this as low as 5-second end-to-end latency, which is the mechanism to reach for when a migration's CDC requirement is closer to "operational replication" than "periodic sync." This project doesn't have a workload that currently justifies it — Meridian's migration sources are batch or moderate-CDC, not sub-10-second — but it's the right answer if that requirement shows up.

---

## Ongoing Sync: Streams + Tasks, or Dynamic Tables

Once data is landing, keeping a downstream table current is a second decision, and the two current mechanisms solve different problems — this project defaults to naming which one applies rather than picking one as a blanket standard:

**Streams + Tasks** — procedural, and the correct choice whenever the pipeline needs conditional logic, upsert behavior on compound keys, or row-level modification after the fact. That last point matters specifically for this project: a right-to-erasure request under `privacy_consent_management.md` needs to directly delete or crypto-shred a specific row, and that kind of targeted modification requires a standard table with Streams/Tasks logic — not a Dynamic Table, whose contents are owned entirely by its defining query and aren't meant to be directly edited.

```sql
CREATE OR REPLACE STREAM RAW.BILLING_CHANGES ON TABLE RAW.BILLING_STAGING;

CREATE OR REPLACE TASK ANALYTICS.APPLY_BILLING_CHANGES
    WAREHOUSE = MIGRATION_WH
    SCHEDULE = '5 MINUTE'
    WHEN SYSTEM$STREAM_HAS_DATA('RAW.BILLING_CHANGES')
AS
    MERGE INTO ANALYTICS.BILLING AS target
    USING RAW.BILLING_CHANGES AS source
    ON target.billing_id = source.billing_id
    WHEN MATCHED AND source.METADATA$ACTION = 'DELETE' THEN DELETE
    WHEN MATCHED AND source.METADATA$ACTION = 'INSERT' AND source.METADATA$ISUPDATE THEN
        UPDATE SET target.amount = source.amount, target.updated_at = source.updated_at
    WHEN NOT MATCHED AND source.METADATA$ACTION = 'INSERT' THEN
        INSERT (billing_id, amount, updated_at) VALUES (source.billing_id, source.amount, source.updated_at);
```

**Dynamic Tables** — declarative, and the better fit for pure transformation layers: joining staged tables into a denormalized reporting shape, building a star-schema dimension or fact table, or any multi-table refresh where the output is defined entirely by a `SELECT` and doesn't need row-level intervention. Migration-relevant use case: once source data has landed and been validated, a Dynamic Table is frequently the right way to build the analyst-facing shape on top of it, with `TARGET_LAG` tuned to how fresh that shape actually needs to be — without hand-writing the MERGE logic Streams + Tasks would require for the same join.

The rule this project uses going forward: if the pipeline needs to directly delete or modify specific rows after landing — erasure requests being the concrete example — it's Streams + Tasks on a standard table. If it's a pure read-and-transform layer, Dynamic Tables are less code and less to maintain.

---

## Staging Before Governance

Nothing lands directly in a `GOVERNANCE`-tiered or classified production schema. Every source lands first in a `RAW` schema — VARIANT for semi-structured sources per `source_pattern_playbooks.md`, typed columns for well-formed relational sources — and only moves into a classified schema after the classification identified during assessment is actually applied.

```sql
-- Landing: raw, unclassified, isolated from anything governed
CREATE SCHEMA IF NOT EXISTS RAW;

CREATE TABLE RAW.CRM_CONTACTS_RAW (
    contact_id      STRING,
    email           STRING,
    phone           STRING,
    _source_system  STRING DEFAULT 'legacy_mysql_crm',
    _ingested_at    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Promotion: classification applied on the way into the governed schema,
-- consistent with the tiers defined in snowflake_data_security_guardrails.md
CREATE OR REPLACE TABLE GOVERNANCE.CRM_CONTACTS AS
SELECT contact_id, email, phone, _source_system, _ingested_at
FROM RAW.CRM_CONTACTS_RAW;
-- Row access policies and masking policies applied to GOVERNANCE.CRM_CONTACTS
-- follow the same pattern already established in the sibling data_sec project,
-- not a new policy design invented for migrated data specifically.
```

This isn't new reasoning specific to migration — it's the same classify-at-the-boundary logic `snowflake_data_security_guardrails.md` already established for data entering the system generally. The reason it's worth restating here: migrated data has a natural pull toward skipping this step, because it already "worked" in the source system without this kind of gate. That history isn't a reason to land it ungoverned.

---

## Sources

- Internal: `source_pattern_playbooks.md`, `snowflake_data_security_guardrails.md`, `privacy_consent_management.md`, `secrets_management.md` (this project's sibling repo `data_sec`)
- [Snowpipe Streaming high-performance architecture](https://docs.snowflake.com/en/user-guide/snowpipe-streaming/snowpipe-streaming-high-performance-overview)
- [Introduction to streams](https://docs.snowflake.com/en/user-guide/streams-intro)
- [Dynamic tables compared to streams and tasks](https://docs.snowflake.com/en/user-guide/dynamic-tables-comparison)
- [Decision guide for dynamic tables](https://docs.snowflake.com/en/user-guide/dynamic-tables/decision-guide)
