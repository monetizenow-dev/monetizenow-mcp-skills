---
name: monetizenow-crm-synced-quote
description: The setup order for a MonetizeNow quote that has to sync back to Salesforce — bring the Salesforce account and its contacts into MonetizeNow, create (or reuse) a MonetizeNow opportunity whose customId is the Salesforce opportunity id, then create the quote on that opportunity. Use this skill whenever a quote is being created for a customer who lives in Salesforce, or the user says the quote must show up in / sync to SFDC, mentions a Salesforce opportunity or account id (006…, 001…), or asks to "quote this Salesforce opportunity". Use it before calling create_net_new_quote for such a customer, because the account and opportunity the quote is created on are what tie it to Salesforce.
---

# Creating a quote that syncs to Salesforce

A MonetizeNow quote reaches Salesforce through the records it hangs off: the **account** must be
the one linked to the Salesforce account, and the **opportunity** must carry the Salesforce
opportunity id as its `customId`. Get either wrong and the quote is created successfully but lands
nowhere in Salesforce — or on the wrong record.

So the order is fixed: **account and contacts → opportunity → quote**. Each step consumes an id
the previous one produced. Do not create the quote first and patch the links on afterwards.

This skill covers only that setup. Once the quote exists, building it out — offerings, ramps,
pricing — follows `monetizenow-quote-builder` and `monetizenow-pricing-strategy` as usual.

## What you need before starting

- The **Salesforce account id** (`001…`).
- The **Salesforce opportunity id** (`006…`) the quote belongs to.

If the user gives names instead of ids and a Salesforce connection is available, look the ids up
there (an Opportunity query filtered on the account). Otherwise ask. Do not guess an opportunity
from its name — two open opportunities on one account is common, and the quote will sync to
whichever one you pick.

Use the ids exactly as the Salesforce API returns them (18 characters). The `customId` lookups
below are exact matches, so a 15-character id copied from a browser URL will not find an
opportunity stored under its 18-character form — and will let you create a duplicate.

## Step 1 — Sync the account and its contacts

**One account for this quote:** call `monetizenow_pull_crm_entity` with the Salesforce account id.
It is safe to call whether or not the account is already in MonetizeNow — an already-linked account
comes back with `created: false` — so use it as the resolver from Salesforce id to `acct_` id
rather than searching MonetizeNow by name. Contacts come across with the account; there is no
separate call.

Before moving on:

- **A validation error means nothing was created.** The pull checks the Salesforce account before
  writing, and rejects one it cannot map — for example an empty MonetizeNow Billing Currency
  (`monetize__BillingCurrency__c`). That is a data fix in Salesforce, not something a retry will
  change. The tenant's default currency in the package settings is not filled in on this path, so
  an account that would sync fine from Salesforce can still fail here. Stop before step 2, tell the
  user which field to fix, and only write to Salesforce yourself if they ask you to.
- **Read `summary`.** The account can be created while some contacts are not, or while the
  Salesforce record fails to be stamped. Both are reported, not raised. A failed stamp means the
  link back to Salesforce is incomplete — stop and tell the user rather than quoting on it.
- **Watch for a native duplicate.** If a name search already showed a MonetizeNow account for this
  customer and the pull returned a *different* `acct_` id, the existing one was never linked to
  Salesforce. Do not pick one silently: tell the user there are now two, and which is linked.
- **Check the contacts you will need.** A Salesforce contact with no email never syncs, so a
  missing contact is often that rather than a failure. Search contacts on the account and confirm
  the people who will be primary and billing contact on the quote are there.

**Many accounts, or records stuck with a sync error:** that is a migration, not a quote setup. Use
the `monetizenow-sync-migration` skill, which drives the managed package's own sync from the
Salesforce side in governor-safe batches and reports what landed. Come back here per quote once the
accounts are in.

## Step 2 — Create the opportunity with the Salesforce id

Search opportunities with the `custom_id` filter set to the Salesforce opportunity id **first**.

- **Found** — that is the opportunity. Reuse it; do not create another. Check it is on the account
  from step 1. If it is on a different account, stop and tell the user — the account on an
  opportunity cannot be changed, and creating a second opportunity is not a fix.
- **Not found** — `manage_opportunity` with `action: CREATE`, the `acct_` id from step 1,
  `customId` set to the Salesforce opportunity id, and the Salesforce opportunity's name as `name`.
  Name is fixed at creation, so take it from Salesforce rather than inventing one.

Leave `primaryQuoteId` off the create — there is no quote yet.

A conflict error on `customId` means another opportunity already holds it, which the search should
have found. Re-search and use that one; do not retry the create.

## Step 3 — Create the quote on that opportunity

`create_net_new_quote` with `accountId` from step 1 and `opportunityId` (the `opp_` id, never the
Salesforce id) from step 2. Everything else follows the tool's defaults and
`monetizenow-quote-builder`.

Then retrieve the opportunity and read it back:

- The new quote should appear in its `quotes`.
- The first quote on an opportunity becomes its primary quote automatically, so on a new
  opportunity `primaryQuoteId` will already be the new quote. On a reused opportunity that already
  had a primary quote it will not change. If the new quote is meant to replace it, ask first, then
  set it with `manage_opportunity` `action: UPDATE`. The opportunity reports its amount and currency
  from the primary quote, so this is what Salesforce will show as the deal value.

Set the quote's primary and billing contacts from the contacts synced in step 1 with
`set_quote_contacts`; they cannot be set on create. If the account has no contacts in MonetizeNow,
skip this and say so: the quote exists and syncs without them, but it cannot be sent until contacts
are added.

Getting those contacts across later is not automatic. A newly pulled account is linked in
Salesforce (`monetize__Id__c` is stamped) but its `monetize__EnableSync__c` stays false, and a
contact only syncs from Salesforce while its account has sync enabled. Contacts added in Salesforce
afterwards therefore stay there until someone enables sync on the account — the
`monetizenow-sync-migration` skill covers that. Say this when reporting an account that was pulled
with no contacts.

## Confirming it reached Salesforce

The opportunity's `url` can be empty even when the link to Salesforce is working, so it does not
show whether the sync worked. When a Salesforce connection is available, check the Salesforce side
directly: the quote syncs as a `monetize__Quote__c` record whose `monetize__Id__c` is the `quot_`
id, and its `monetize__Opportunity__c` and `monetize__Account__c` should be the Salesforce ids you
started from. It normally appears within a few seconds of the create. Without a Salesforce
connection, say the sync was not checked rather than implying it was.

## Reporting

Say which records were **created** and which were **reused** — an existing account or opportunity
picked up silently looks, to the user, exactly like a new one. Give the `acct_`, `opp_` and `quot_`
ids alongside the Salesforce ids they map to, the opportunity's CRM `url` when it has one, and
whether the Salesforce quote record was confirmed.
Report anything from step 1's `summary` that did not land, rather than only the happy path.

## Related skills

`monetizenow-sync-migration` for bulk or stuck account and contact syncs from the Salesforce side.
`monetizenow-quote-builder` for everything after the quote exists. `monetizenow-data-retrieval` for
resolving records and handling multiple matches.
