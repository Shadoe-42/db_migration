# Validation & Cutover
*Meridian Analytics — where migrations actually fail | July 2026*

---

## Purpose

Extraction and ingestion are the parts of a migration that get planned in detail. Validation and cutover are the parts that get a bullet point — and they're also where migrations actually go wrong, because "the data moved" and "the data is correct and the business can safely rely on it now" are different claims, and only the second one matters on cutover day.

---

## Reconciliation, Not Spot-Checking

Every migrated table gets checked against its source, at a depth proportional to what the assessment rated it — see the readiness matrix in `migration_assessment_methodology.md`.

**Row counts, always.** Cheapest possible check, and it catches an entire class of failure (a filtered extraction, a failed batch, a silently dropped file) for almost no cost. Run it, but don't stop there — matching row counts don't prove matching data.

**Checksum/hash comparison for anything rated Medium complexity or above.** An aggregate hash per table (or per meaningful partition — per tenant, per date range) computed on both source and target and compared, catching value-level corruption that a row count can't. This is where the shadow-IT sources from `source_pattern_playbooks.md` actually get their manual, row-for-row reconciliation — affordable at their scale, and specifically suited to catching the "hidden column with a manual override" class of error that an automated hash comparison would miss if the override wasn't part of what got extracted in the first place.

**Business-rule validation for anything with logic in flight.** A migrated pricing table isn't validated by confirming the numbers moved — it's validated by confirming a sample of actual price calculations produce the same result in Snowflake that they did against the source. This is the validation step that shadow-IT sources need most, since their "business logic" was often a spreadsheet formula with no equivalent anywhere else to check against, per the discovery work `migration_assessment_methodology.md` already did on that source.

---

## Parallel Run vs. Big Bang

Not a universal choice — sized to what the source actually is:

**Parallel run** — source and Snowflake both stay live and in sync (via the CDC pattern already chosen in `source_pattern_playbooks.md`) for a defined window, with downstream consumers gradually cut over and validated against both systems before the source is retired. Justified for high-dependency, high-complexity sources — Billing in the readiness matrix example is the clear case, given six known downstream dependencies and Regulated-tier data. The cost is real (running two systems, maintaining sync, more elapsed calendar time) and is spent deliberately where the risk of an undetected discrepancy is highest.

**Big bang** — a defined cutover window, final sync, validation gate, and a hard switch. Appropriate for low-dependency, well-understood sources — the regional pricing workbooks in the same example, where the entire dataset is small enough to fully reconcile by hand and the "downstream dependency" count is genuinely low once discovery is complete. Running a parallel-sync architecture around four Excel workbooks would be effort spent for safety margin the source doesn't need.

The assessment matrix's complexity/priority rating is what decides this, not source size — restating the same point `migration_assessment_methodology.md` makes, because it's the point most likely to get overridden by instinct on a real project (the "big" databases feel like they need the careful treatment; the readiness matrix is what keeps that instinct from silently deciding the plan).

---

## The Cutover Runbook

For a big-bang cutover, the shape is the same regardless of source:

1. **Freeze window announced and enforced** — writes to the source stop, or are captured and applied as a final delta if a brief freeze isn't tolerable.
2. **Final delta sync** — whatever changed since the last full reconciliation lands, via the same CDC or batch mechanism already in place.
3. **Validation gate** — the reconciliation checks from this doc's first section run against the post-freeze state, not the pre-freeze snapshot. A validation pass that ran before the final delta isn't a validation of what's actually about to go live.
4. **Cutover** — connection strings, service configuration, or downstream job definitions repoint to Snowflake. This is a configuration change, not a data change, and should be the single fastest, most reversible step in the whole sequence.
5. **Rollback trigger criteria defined in advance** — what specific validation failure or business signal triggers a rollback, decided before cutover starts, not improvised in the moment under pressure.

---

## Rollback and Retirement

The source system stays available, read-only, for a defined window after cutover — not indefinitely. An open-ended "just in case" retention of the old system is the same instinct this project has already rejected elsewhere: infrastructure that isn't the data itself is cattle, not pets, per the standing design philosophy in `network_security_foundations.md`, and a legacy database kept alive out of caution past its useful window is exactly that — an unretired pet, still costing money and still a second copy of data that now needs its own security posture maintained.

Retention window length follows the same tiering logic as everything else classification-driven in this project — a Regulated-tier source stays available longer, with more deliberate sign-off before final decommission, than an Internal-tier one. That tiering is defined in `resilience_disaster_recovery.md` and isn't reinvented here; migration retirement uses the same retention discipline already established for data lifecycle generally, rather than a separate ad hoc rule for "how long do we keep the old database around."

---

## Sources

- Internal: `migration_assessment_methodology.md`, `source_pattern_playbooks.md`, `network_security_foundations.md`, `resilience_disaster_recovery.md` (this project's sibling repo `data_sec`)
