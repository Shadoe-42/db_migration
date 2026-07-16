# Database Migration to Snowflake — Reference Architecture
*Meridian Analytics migrates off a legacy patchwork onto Snowflake*

A migration reasoning framework for moving heterogeneous legacy sources — Oracle, MySQL, MongoDB, and the inevitable Access databases and Excel workbooks holding real business logic — onto Snowflake. Covers discovery and assessment before any plan gets written, source-specific extraction patterns, how data actually lands and earns governance, and where migrations actually fail: validation and cutover.

## What this is, and isn't

This is a **sibling project to [`data_sec`](https://github.com/Shadoe-42/data_sec)**, reusing the same fictional company (Meridian Analytics) and the same landing zone, security, and governance model already built there, rather than duplicating it. This repo is deliberately scoped to migration-specific reasoning — it cross-references `data_sec`'s existing docs for anything already covered (classification tiers, RAP/masking patterns, retention discipline) instead of rebuilding them.

As with `data_sec`: real technology, fictional company. Every tool and syntax referenced here (Snowflake, AWS DMS, GCP DMS) is real and current as of this writing; Meridian Analytics and its legacy systems are not.

## Structure

| Path | What's there |
|---|---|
| `migration_pipeline_hld.svg` / `.png` | Architecture diagram — the full pipeline from heterogeneous sources through readiness scoring, extraction, RAW landing, validation, to governed schema |
| `migration_assessment_methodology.md` | Discovery and profiling before any plan is written — schema inventory, data profiling, downstream dependency mapping, and a scored migration readiness matrix |
| `source_pattern_playbooks.md` | Extraction patterns by source type — relational CDC vs. bulk load, MongoDB's schema-flattening problem, and the Access/Excel shadow-IT case |
| `snowflake_ingestion_landing_patterns.md` | How data actually lands — Snowpipe vs. Snowpipe Streaming vs. batch COPY INTO, Streams/Tasks vs. Dynamic Tables, and staging before governance |
| `validation_and_cutover.md` | Reconciliation depth by complexity rating, parallel run vs. big bang, the cutover runbook, and rollback/retirement discipline |

## Related

Sibling repos in the same portfolio, all reusing or extending `data_sec`'s architecture rather than duplicating it:

- [`data_sec`](https://github.com/Shadoe-42/data_sec) — the landing zone, Snowflake security model, and compliance crosswalk this project builds on top of
- [`finops_remediation`](https://github.com/Shadoe-42/finops_remediation) — Snowflake cost-incident diagnosis and permanent remediation
- [`cloud_retrofit`](https://github.com/Shadoe-42/cloud_retrofit) — risk-tiered retrofit of a live environment that grew without a plan
- [`dc_decomm`](https://github.com/Shadoe-42/dc_decomm) — full on-prem-to-cloud exit, reusing `data_sec`'s landing zone Terraform directly
- [`ma_consolidation`](https://github.com/Shadoe-42/ma_consolidation) — post-merger data platform consolidation, extraction mechanics for Oracle/SQL Server borrowed from this repo

## License

MIT — see `LICENSE`.
