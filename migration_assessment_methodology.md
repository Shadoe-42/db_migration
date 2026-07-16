# Migration Assessment Methodology
*Meridian Analytics — measure twice | July 2026*

---

## Purpose

Every migration failure mode in this project traces back to the same root cause: someone wrote a migration plan before they actually knew what was in the source system. A table nobody documented turns out to feed a finance report. A "small" MySQL database turns out to have a 40GB table with no primary key. A spreadsheet turns out to be the only record of a business rule that lives nowhere else. This doc is the discovery pass that has to happen before `source_pattern_playbooks.md` or `validation_and_cutover.md` mean anything — a migration plan built on an assumed schema instead of a discovered one is asserted, not designed, and that gap is exactly what shows up as a cutover-day surprise instead of a Tuesday-afternoon planning conversation.

---

## Discovery Scope

Four things get inventoried before a single row moves, regardless of source type:

**Schema inventory.** Tables, columns, data types, declared relationships (foreign keys) and — more importantly — *undeclared* ones. Most legacy systems have relationships enforced only in application code, never in the database itself. A column named `customer_ref` that's actually a foreign key with no constraint on it is a landmine for referential integrity once the data lands somewhere that expects it to be correct.

**Data profiling.** Null rates, cardinality, actual value ranges versus declared types (a `VARCHAR(50)` "date" column storing `03/14/25`, `2025-03-14`, and `March 14th` in the same table is a real and common finding, not a hypothetical). This pass is also where sensitive fields get flagged — profiling for anything that looks like PII, financial data, or health information *before* migration, not after, so the target Snowflake schema can be classified correctly on arrival rather than reclassified as an afterthought. Classification tiers carry over directly from `snowflake_data_security_guardrails.md` — if a source field profiles as Regulated-tier, it lands in a Regulated-tier Snowflake schema from the first load, not a default one that gets tightened later.

**Volume and velocity.** Row counts, table growth rate, and — critically — write pattern. A table that's 50GB but append-only and rarely updated is a straightforward bulk load. A table that's 5GB but has a 20% row-update rate every hour needs a CDC pattern, not a batch one. Sizing the migration approach to the actual write pattern, not just the table size, is what `source_pattern_playbooks.md` builds on.

**Downstream dependency mapping.** What reads from this table that isn't the application itself — scheduled reports, other services' ETL jobs, a BI tool with a saved connection, an analyst's ad hoc script that's quietly become load-bearing. This is consistently the most underestimated part of any migration assessment, because it requires asking people, not just querying `information_schema`. A dependency nobody mapped is a cutover-day outage, not a migration-day inconvenience.

---

## The Shadow IT Problem

Access databases and Excel workbooks fail this methodology differently than Oracle or MySQL do, and it's worth naming directly rather than treating them as smaller versions of the same problem.

There's no DBA to interview, no `information_schema` to query, and often no single owner who can describe the full scope — the "database" is frequently a spreadsheet that started as one person's tracking sheet and accumulated business logic (macros, VLOOKUPs standing in for joins, manually-maintained lookup tables) that nobody wrote down anywhere else. Discovery here starts before schema inventory: it starts with *finding out the thing exists at all*, usually by asking business users what they use to get their job done rather than scanning a server. The assessment output for a shadow-IT source isn't just a schema — it's a translation of undocumented business logic into something a real schema can express, which is a different and slower kind of work than profiling a well-formed relational table.

---

## Output: The Migration Readiness Matrix

Assessment concludes in a scored artifact, not a narrative — every source gets rated so sequencing and effort estimates are defensible rather than gut-feel.

| Source | Type | Est. rows | Write pattern | Sensitive data | Dependency count | Complexity | Priority |
|---|---|---|---|---|---|---|---|
| Billing (Oracle) | Relational | 40M | High-frequency update | Regulated (payment metadata) | 6 known | High | 1 |
| CRM (MySQL) | Relational | 2M | Low-frequency update | Confidential (contact PII) | 3 known | Medium | 2 |
| Support tickets (MongoDB) | Document | 800K | Append-heavy, schema drift across years | Confidential | 2 known | Medium-High | 3 |
| Regional pricing (Excel) | Shadow IT | ~12K rows across 4 workbooks | Manual, ad hoc | Internal | Unknown — under discovery | High (undocumented logic) | 4 |

"Complexity" here isn't a proxy for size — the smallest source in this table (the pricing workbooks) is rated highest-complexity because of undocumented logic and unknown dependencies, not row count. That's the entire point of running this assessment before writing a migration plan: complexity and volume are frequently uncorrelated, and a plan sequenced by table size alone gets the hardest problem last, when there's the least schedule slack left to absorb it.

---

## Sources

- Internal: `snowflake_data_security_guardrails.md`, `source_pattern_playbooks.md`, `snowflake_ingestion_landing_patterns.md` (this project's sibling repo `data_sec`)
