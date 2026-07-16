# Source Pattern Playbooks
*Meridian Analytics — the source dictates the pattern, not the other way around | July 2026*

---

## Purpose

A migration methodology that treats every source the same way is really a migration methodology for whichever source it was designed around. Oracle, MongoDB, and an Excel workbook fail differently, get assessed differently (see `migration_assessment_methodology.md`), and need genuinely different extraction patterns. This doc is organized by source type rather than as one generic playbook, because collapsing them into one flow is exactly where migration plans stop matching reality.

---

## Relational Sources — Oracle, MySQL, SQL Server

Two patterns, chosen by the write pattern the assessment already measured, not by habit:

**Log-based CDC** for sources with meaningful update/delete activity or a downtime budget too small for a bulk export-and-cutover. AWS DMS supports both homogeneous and heterogeneous replication (Oracle or SQL Server as a source, with Snowflake as the eventual target reached via S3 staging rather than a native DMS-to-Snowflake endpoint) and reads the database's transaction log rather than querying the live tables, keeping replication overhead low on a production system that's still serving traffic during migration. GCP's Database Migration Service is comparatively narrower today — its strongest story is near-zero-downtime Oracle-to-AlloyDB migration, not Snowflake-bound extraction — so on GCP the more common pattern is a self-managed log-based extraction (Oracle GoldenGate, Debezium) landing files in GCS rather than relying on GCP DMS itself. Worth stating plainly rather than implying parity: AWS's tooling is more directly applicable to a Snowflake-bound migration than GCP's is, as of this writing.

**Bulk extract-and-load** for append-heavy or low-update-frequency sources, and for one-time migrations where a maintenance window is acceptable. Native export tooling (Oracle Data Pump, `mysqldump`, SQL Server's `bcp`) to flat files, landed in cloud storage, ingested per `snowflake_ingestion_landing_patterns.md`. Simpler to build and verify than CDC, and the right default unless the assessment specifically justifies the added complexity of log-based replication.

Both patterns terminate the same way: files or change records land in cloud storage, and Snowflake ingestion (COPY INTO, Snowpipe, or Snowpipe Streaming, depending on latency requirement) picks up from there. DMS and GoldenGate solve *getting data out of the source*, not landing it in Snowflake — that's a separate, Snowflake-side decision covered in the next doc.

---

## MongoDB — The Schema Flattening Problem

MongoDB's core migration challenge isn't extraction, it's that the assessment methodology's "schema inventory" step doesn't fully apply — a collection doesn't have one schema, it has however many document shapes accumulated over the collection's lifetime. A `support_tickets` collection from 2022 and the same collection today may have added fields, renamed fields, or nested things differently, and none of that is enforced or declared anywhere.

Two things need to happen before extraction, not after:

**Document shape discovery.** Sample across the collection's full time range, not just recent documents, and build an inventory of field presence and type per era rather than assuming the newest shape represents the whole collection. A migration that only samples current documents will silently drop or mis-type older records that don't match.

**Landing strategy: VARIANT first, extract second.** The verified current best practice is a hybrid approach — land the full document as-is into a Snowflake `VARIANT` column on ingestion, preserving everything regardless of shape drift, then selectively extract the fields that are actually queried into typed, structured columns alongside it. Extracting everything up front either breaks on shape drift or requires maintaining a extraction schema that's perpetually out of date; extracting nothing leaves every downstream query doing expensive runtime JSON parsing. The hybrid keeps the raw document recoverable while giving frequently-accessed fields real column performance — measured in practice at roughly 40-45% faster query performance on extracted, typed columns versus querying the VARIANT directly for the same field.

```sql
-- Landing: preserve the full document, whatever shape it arrived in
CREATE TABLE RAW.SUPPORT_TICKETS_RAW (
    document        VARIANT,
    _source_id       STRING,
    _ingested_at     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Extraction: promote the fields that are actually queried, cast explicitly
CREATE OR REPLACE TABLE ANALYTICS.SUPPORT_TICKETS AS
SELECT
    document:_id::STRING                    AS ticket_id,
    document:tenant_id::STRING              AS tenant_id,
    document:created_at::TIMESTAMP_NTZ      AS created_at,
    document:status::STRING                 AS status,
    document:body::STRING                   AS ticket_text,
    document                                AS raw_document  -- kept for fields not yet promoted
FROM RAW.SUPPORT_TICKETS_RAW;
```

---

## Access and Excel — The Shadow IT Case

Treated as a one-time bulk load, not a repeatable pipeline, for a specific reason: the value of building CDC or scheduled sync infrastructure around a shadow-IT source is close to zero, because the actual goal is almost always to *retire* the spreadsheet, not keep syncing it indefinitely. Building durable pipeline infrastructure around something meant to be decommissioned is effort spent in the wrong place.

What actually takes the time, per the assessment methodology's shadow-IT discussion, is translating undocumented logic — merged cells, a macro that recalculates a discount tier, a lookup tab nobody's touched since 2019 but that a formula still references — into something a schema can express. That translation is a business-logic conversation with the workbook's owner, not a technical extraction problem, and it should be budgeted as such rather than estimated at the same rate as a well-formed database table with equivalent row count.

Practical sequence: extract to CSV (or read directly via a connector), land raw with minimal transformation, validate row-for-row against the source workbook by hand for anything under a few thousand rows — a full reconciliation pass is affordable at that scale and catches the "hidden column with a manual override" problem that automated validation alone would miss. See `validation_and_cutover.md` for where this validation step fits in the broader cutover sequence.

---

## Sources

- Internal: `migration_assessment_methodology.md`, `snowflake_ingestion_landing_patterns.md`, `tagging_standard.md` (workload dimension, for tracking migration pipelines as first-class tagged infrastructure)
- [AWS Database Migration Service](https://aws.amazon.com/dms/)
- [Running homogeneous data migrations in AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/dm-migrating-data.html)
- [Considerations for semi-structured data stored in VARIANT](https://docs.snowflake.com/en/user-guide/semistructured-considerations)
