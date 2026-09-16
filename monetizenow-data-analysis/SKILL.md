---
name: monetizenow-data-analysis
description: Use this skill whenever the user asks for data analysis, reporting, metrics, trends, or ad-hoc querying of MonetizeNow system data — e.g. "break down our invoices by month," "how many active quotes per offering," "what's our total credit balance by account," or any request to aggregate, summarize, join, or filter MonetizeNow records (invoices, quotes, offerings, rates, credits, accounts, bill groups, etc.) in ways the MonetizeNow API tools (which identify and retrieve individual records) can't answer directly. Trigger this even if the user doesn't say "SQL," "database," or name this skill explicitly — any request that requires computing something across MonetizeNow records rather than just looking one up should use this skill's discover-then-query workflow instead of guessing at table names or fabricating numbers.
---

# MonetizeNow Data Analysis

## Why this workflow

Two separate MCP servers are involved, and knowing which is which keeps you from looking for a tool where it cannot be.

The **MonetizeNow server** works at the level of records. `monetizenow_search_objects` finds them, `monetizenow_retrieve_object` fetches one, and `monetizenow_describe_object` and `monetizenow_get_object_search_attributes` explain the data model and what is filterable. Search is explicitly for identification rather than aggregation, but it is more capable than it first looks: it covers roughly a dozen object types and supports operator prefixes, pagination, and sort. A question that only needs filtering, counting, or the top few records by some field is often answerable there — try it before reaching for SQL.

The **Metabase server** is the one with raw database access: `list_databases`, `list_tables`, `get_table_fields`, `get_field_values`, and `execute_query`. Reach for it when the question genuinely needs what the API cannot express in a single call — an aggregation across many records, or a join correlating different entity types.

This skill is about using the Metabase tools in the right order, so queries are built on real schema and real values instead of guesses.

Guessing costs more than a round trip. A wrong table or column name usually errors, which is the good case. A wrong *value* does not: filtering `status = 'active'` when the column stores `ACTIVE` returns zero rows and no error, and zero rows reads exactly like a true finding. The same goes for picking `amount` when the number you want is in `amount_cents`. Confirm the schema, and confirm the literals, before trusting a result.

## Workflow

1. **Identify the database** — call `list_databases`.
   - If it returns exactly one database, use it and move on.
   - If it returns more than one, stop and ask the user which one to use before doing anything else. Don't guess from naming conventions (e.g. "prod" vs "staging," or a multi-tenant setup) — get it confirmed.

2. **Explore the schema** — call `list_tables` on the chosen database, then `get_table_fields` for each table that looks relevant to the question.
   - Read the field names, types, and any relationships before writing a query. If the analysis spans multiple entities (e.g. "revenue by account"), identify the join keys from the schemas rather than assuming standard names like `account_id` will match up.
   - Note each field's `has_field_values`. When it is `list` or `auto-list`, Metabase already holds that column's distinct values and step 3 should read them.
   - A field's fingerprint describes the *shape* of the data — value lengths, distinct counts — never the values themselves. Do not infer a filter literal from it, nor from the column's name or description.
   - If nothing in the schema obviously matches what's being asked, say so and ask the user for clarification rather than querying the closest-sounding table.

3. **Read the real values before filtering on a fixed set** — for any column whose values come from a known set (status, type, state, category), call `get_field_values` with the field id from `get_table_fields` and use what it returns **verbatim**.

   This exists because casing is not guessable. `ACTIVE`, `Active`, and `active` are all plausible, only one is stored, and the wrong one returns an empty result rather than an error — so the failure looks like a finding. Reading the value costs one call and removes the guess.

   Two results need care:
   - **An empty list** means the column's values are not cached — expected for anything `get_table_fields` did not mark `list` or `auto-list`. Fall back to `SELECT DISTINCT <column> FROM <table>` through `execute_query`.
   - **`truncated: true`** means you are seeing a partial view, capped at 100 values. A literal taken from it is still safe to filter on. The list is *not* safe to treat as the column's complete value set, so do not use it to enumerate every status, build an `IN` list you describe as exhaustive, or conclude a value does not exist.

4. **Write and run the query** — once the tables, fields, and any value literals are confirmed, write a standard SQL query and run it with `execute_query`.
   - Prefer explicit column selection over `SELECT *` so results are easy to reason about and present.
   - For anything exploratory, it's fine to run a small `SELECT ... LIMIT 5` first to sanity-check data shape (e.g. date formats, null patterns, enum values) before writing the full aggregation.
   - If a query errors, read the error and re-check the schema rather than retrying blindly with a guessed fix.

5. **Present the results** — summarize findings in plain language rather than dumping the raw query output. Note anything that shapes interpretation: date ranges applied, rows excluded (e.g. nulls, cancelled records), currency or unit assumptions, etc.

## What not to do

- Don't skip straight to `execute_query` with table or column names inferred from the question's wording — always confirm via `list_tables`/`get_table_fields` first.
- Don't guess an enum literal or its casing. Read it with `get_field_values`, or `SELECT DISTINCT` when the column has no cached values. An empty result from a guessed literal is indistinguishable from a real zero.
- Don't present a truncated value list as the full set of possibilities.
- Don't silently pick a database when `list_databases` returns more than one — ask.
- Don't present raw query results as the final answer without translating them into an actual finding.
