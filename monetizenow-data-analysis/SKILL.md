---
name: monetizenow-data-analysis
description: Use this skill whenever the user asks for data analysis, reporting, metrics, trends, or ad-hoc querying of MonetizeNow system data — e.g. "break down our invoices by month," "how many active quotes per offering," "what's our total credit balance by account," or any request to aggregate, summarize, join, or filter MonetizeNow records (invoices, quotes, offerings, rates, credits, accounts, bill groups, etc.) in ways the single-purpose MonetizeNow tools (get_invoices, get_quotes, etc.) can't answer directly. Trigger this even if the user doesn't say "SQL," "database," or name this skill explicitly — any request that requires computing something across MonetizeNow records rather than just looking one up should use this skill's discover-then-query workflow instead of guessing at table names or fabricating numbers.
---

# MonetizeNow Data Analysis

## Why this workflow

The MonetizeNow MCP server has purpose-built tools for single-record lookups (`get_invoice`, `get_quote`, `get_accounts`, etc.), but those don't cover analysis — aggregations, joins across entities, or filtering on combinations of fields. For that, the server also exposes raw database access: `list_databases`, `list_tables`, `get_table_fields`, and `execute_query`. This skill is about using those four tools in the right order so queries are built on real schema instead of guesses.

Guessing table or column names wastes a round trip on an error at best, and silently returns wrong numbers at worst (e.g. `amount` vs `amount_cents` vs `total_amount` are all plausible-sounding fields that mean different things). Always confirm the schema before writing a query.

## Workflow

1. **Identify the database** — call `list_databases`.
   - If it returns exactly one database, use it and move on.
   - If it returns more than one, stop and ask the user which one to use before doing anything else. Don't guess from naming conventions (e.g. "prod" vs "staging," or a multi-tenant setup) — get it confirmed.

2. **Explore the schema** — call `list_tables` on the chosen database, then `get_table_fields` for each table that looks relevant to the question.
   - Read the field names, types, and any relationships before writing a query. If the analysis spans multiple entities (e.g. "revenue by account"), identify the join keys from the schemas rather than assuming standard names like `account_id` will match up.
   - If nothing in the schema obviously matches what's being asked, say so and ask the user for clarification rather than querying the closest-sounding table.

3. **Write and run the query** — once the tables and fields are confirmed, write a standard SQL query and run it with `execute_query`.
   - Prefer explicit column selection over `SELECT *` so results are easy to reason about and present.
   - For anything exploratory, it's fine to run a small `SELECT ... LIMIT 5` first to sanity-check data shape (e.g. date formats, null patterns, enum values) before writing the full aggregation.
   - If a query errors, read the error and re-check the schema rather than retrying blindly with a guessed fix.

4. **Present the results** — summarize findings in plain language rather than dumping the raw query output. Note anything that shapes interpretation: date ranges applied, rows excluded (e.g. nulls, cancelled records), currency or unit assumptions, etc.

## What not to do

- Don't skip straight to `execute_query` with table or column names inferred from the question's wording — always confirm via `list_tables`/`get_table_fields` first.
- Don't silently pick a database when `list_databases` returns more than one — ask.
- Don't present raw query results as the final answer without translating them into an actual finding.
