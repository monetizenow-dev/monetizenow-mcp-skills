# Install — MonetizeNow sync migration skill

A self-contained Claude Code skill for migrating Salesforce Accounts and Contacts into
MonetizeNow. Everything it needs is in this folder; it does not depend on any repository.

## What you need

- **Claude Code**, with either:
  - a **Salesforce MCP** connection to the target org, or
  - the **`sf` CLI** authenticated to it (`sf org login web`)
- The **MonetizeNow managed package** installed in that org (namespace `monetize`)
- A **MonetizeNow MCP** connection — optional, used to verify records actually landed
- A Salesforce user with: read/write on Account and Contact and the package's
  `monetize__*` fields, and **Author Apex** (the skill runs bounded anonymous Apex; it
  never deploys code or changes metadata)

## Install

Unzip this folder into one of:

```
~/.claude/skills/monetizenow-sync-migration/          # available in every project
<your-project>/.claude/skills/monetizenow-sync-migration/   # shared with your team via git
```

Restart Claude Code, or run `/doctor` to confirm the skill is picked up.

## Use it

Just ask, in a session connected to the org:

> Migrate my Salesforce accounts and contacts to MonetizeNow — all accounts with a
> closed-won opportunity.

> Enable MonetizeNow sync for accounts owned by the EMEA team, and their contacts.

> Some of my accounts show a MonetizeNow sync error. Find out why and retry the ones
> that can be retried.

The skill will confirm the org, translate your criteria into a query, show you the counts
and what it intends to do, and **wait for your approval before writing anything**.

### Optional: a slash command

For a fixed entry point, save this as `~/.claude/commands/mn-sync.md`:

```markdown
---
description: Migrate Salesforce accounts and contacts into MonetizeNow
argument-hint: [org alias] [which accounts to sync]
---

Use the `monetizenow-sync-migration` skill.

$ARGUMENTS — first token is the target org alias if given, the rest describes which
accounts to sync.

If no org was named, list the connected orgs and ask. If no criteria were given, ask what
selects the accounts to migrate. Follow the skill's phases in order and do not write to the
org before the plan is approved.
```

Then run `/mn-sync my-prod-org accounts with a closed-won opportunity`.

## What it does to your org

- **Reads** Account, Contact, `monetize__MonetizeNowSettings__c`, `monetize__FieldMapping__c`,
  and `AsyncApexJob`.
- **Writes** exactly one field: `monetize__EnableSync__c` on the Accounts and Contacts you
  selected. That is what tells the managed package to sync the record to MonetizeNow.
- **Deploys nothing.** No classes, no metadata, no configuration changes.
- **Batches at 200 records per transaction** and paces itself against the async queue, so a
  large migration will not blow governor limits or silently drop records.
- **Asks before every write phase**, and produces a report of what reached MonetizeNow,
  what failed and why, and what was skipped.

Enabling sync sends your account and contact data to the MonetizeNow tenant configured in
`monetize__MonetizeNowSettings__c`. The skill shows you which tenant that is during
preflight — confirm it before approving.

## Contents

| File | What it is |
| --- | --- |
| `SKILL.md` | The procedure Claude follows |
| `references/how-sync-works.md` | How the package syncs, eligibility rules, real limits |
| `references/api-selection.md` | Which Salesforce API to use at which data volume |
| `references/soql-cohorts.md` | Assessment queries |
| `references/apex-templates.md` | Bounded, resumable execution templates |
| `references/troubleshooting.md` | Error → cause → recovery |

## Maintainers

`references/how-sync-works.md` encodes package internals: `MAX_EVENTS_PER_JOB`, the
eligibility predicates in `AccountSelector`/`ContactSelector`, the account→contact fan-out
delay, and the enqueue guard in `ObjectSyncService` that drops events silently. If those
change in the package, update that file — the batch sizes in this skill are derived from
them.
