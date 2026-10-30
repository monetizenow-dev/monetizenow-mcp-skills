# Choosing the right API for the volume

Two separate decisions. Reads scale with volume; the **sync-enabling write does not** — it
is 200 records per transaction at every scale, because the failure a larger batch causes is
silent (see `how-sync-works.md`).

## Decision table

| In-scope records | Sizing | Listing records | Enabling sync (the write that fires sync) | Writes that fire *no* sync | Driving the run |
| --- | --- | --- | --- | --- | --- |
| **< 2,000** | `SELECT COUNT()` | plain SOQL via MCP / `sf data query` | anonymous Apex, 200/batch | same | inline; minutes |
| **2,000 – 50,000** | `SELECT COUNT()` | SOQL + keyset pagination (`Id > :lastId ORDER BY Id LIMIT n`) | anonymous Apex, 200/batch, gated on queue depth | Bulk API 2.0 or large Apex batches | ledger file, resumable, progress reported per batch |
| **> 50,000** | `SELECT COUNT()` only — a plain query cannot return this many rows | Bulk API 2.0 query (`sf data query --bulk --wait 10`) | anonymous Apex, 200/batch | Bulk API 2.0 CSV | background job, ledger, expect multiple sessions and a 24h API-limit ceiling |

## Reads

- **Always size with `SELECT COUNT()`.** It is not subject to the 50,000-row cap and costs
  one call. Never pull rows to count them.
- **Under 50,000 rows**, ordinary SOQL with keyset pagination is simplest and gives you the
  cursor you need for the write loop anyway. Select `Id` only — the trigger re-queries
  whatever it needs, and a wide select is what blows heap and CPU.
- **Over 50,000 rows**, a plain query cannot return the set. Use Bulk API 2.0 query, which
  streams to CSV:
  `sf data query --query "<soql>" --bulk --wait 10 --result-format csv --target-org <alias>`
  Then drive the write loop from the CSV. This is also the right tool for producing the
  failure listings in the final report.

## Writes

The package fires sync from a trigger, so **any** write path — Apex, REST, Bulk API, Data
Loader — goes through the same trigger and the same limits. What differs is how many
records land in one transaction, and that is the only thing that matters.

### The pass that enables sync: 200 per transaction, always

200 records = one trigger invocation = one queueable = well inside every limit. At 2,000
you are at 10 queueables, which still works but leaves no headroom for other automation in
the org. Past ~10,000 in a single transaction the package exceeds 50 queueables and
**silently stops enqueuing** — the update succeeds, the records look enabled, and they
never sync.

**Do not use Bulk API for this pass.** Bulk API 2.0 chooses its own server-side batch
sizes; you cannot pin them to 200, and a large batch lands you in exactly that silent-drop
zone. Anonymous Apex with an explicit `LIMIT 200` is the only write path where you control
the transaction size.

### Passes that publish no sync events: bulk away

Setting `EnableSync__c = false` makes a record ineligible, so no events are published and
no queueables are enqueued. That pass is limited only by DML rows, so use Bulk API 2.0 or
2,000-record Apex batches:

```bash
sf data update bulk --sobject Account --file ./disable.csv --target-org <alias> --wait 10
```

The same is true of any pure-data cleanup you do first (fixing billing country, currency,
contact emails) — as long as you are not touching a field the package tracks for sync. Check
`monetize__FieldMapping__c` before assuming a field is untracked.

## Pacing

Volume determines pacing, not batch size:

- Gate each batch on queue depth:
  `SELECT COUNT() FROM AsyncApexJob WHERE JobType = 'Queueable' AND Status IN ('Queued','Processing')`
  Pause while it is above ~100.
- Every account batch produces a second, delayed wave of contact syncs about a minute
  later. The queue looking empty once does not mean the run is done — require zero twice,
  90 seconds apart.
- Watch the org's 24h API limit on large runs. Every synced record is one outbound callout
  plus the write-back. At six figures, plan the run across a window rather than hammering.

## When to run it in the background

Past roughly 10,000 records the run is long enough that it should be a background job with
a ledger, reporting progress as it goes, rather than something held open in the
conversation. Keep the ledger current after **every** batch so an interruption costs one
batch, not the run.
