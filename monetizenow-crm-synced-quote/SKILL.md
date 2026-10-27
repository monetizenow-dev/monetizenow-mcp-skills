---
name: monetizenow-crm-synced-quote
description: How to create a MonetizeNow quote for a customer that lives in Salesforce so the quote and everything under it lands on the right CRM records — pulling the account and its contacts across first, creating the base quote, and stamping the opportunity that falls out of it with the Salesforce opportunity id. Use this skill whenever a quoting request names Salesforce or carries CRM ids: "create a quote for this Salesforce opportunity", "quote the account behind 006Ab000001XyZ", "this customer is in SFDC but not in MonetizeNow", "the quote I made isn't showing on the opportunity", "sync this account over so I can quote it", or any request where the deal already exists in a CRM. Use it before calling create_net_new_quote, because creating the quote first is the natural move and it is the one that strands the deal: the account may not exist yet, and the opportunity the create call produces is MonetizeNow-native until you link it back.
---

# Creating a MonetizeNow quote that lands in Salesforce

A quote built through the MCP server is correct in MonetizeNow and invisible in Salesforce unless
three things happen in order. Each tool documents its own call well — this skill is the sequence
they belong in, and the two places where a call reports success while leaving the CRM side wrong.

## The order is the whole point

1. **Pull the account** from the CRM, which brings its contacts with it.
2. **Create the base quote** against the account that pull returned.
3. **Stamp the opportunity** that quote creation produced with the Salesforce opportunity id.

Taken out of order, each step fails in a way that is not obvious at the time:

- Quote first, and there is no account to quote against — or worse, a near-match MonetizeNow
  account that has no CRM link, so the quote attaches to a customer record Salesforce has never
  heard of.
- Contacts before the account lands, and there is nothing to put on the quote: a contact cannot
  exist in MonetizeNow ahead of its account.
- Skip the stamp, and the quote sits on a MonetizeNow-native opportunity. Nothing errors. The
  deal simply never appears on the Salesforce opportunity the rep is working.

## Step 1 — Pull the account from the CRM

`monetizenow_pull_crm_entity` takes the **Salesforce** account id (`001…`), not an `acct_` id, and
returns the MonetizeNow account the CRM record maps to. Its contacts come across with it; there is
no separate call for them.

Two properties make this the right first call rather than a search:

- It is **safe to repeat**. An account already linked comes back with `created` false and its
  existing id, so this doubles as the way to resolve a Salesforce account id to a MonetizeNow one.
  Searching by name does not — a name match proves nothing about whether the record is linked to
  the CRM.
- It will not create a second account for the same CRM record, so there is no duplicate to clean
  up if you call it when you did not need to.

**Read `summary` before reporting success.** Partial failures are reported there, not raised: the
account can be created while some of its contacts are not, or while the CRM record fails to be
stamped with the MonetizeNow id. The call returns an account either way.

Then confirm what actually arrived — `monetizenow_search_objects` on `contact`, filtered by
`account_id`. When a contact the user expects is missing, the usual cause is that it has no email
address: contact sync requires one, and a contact without it will never come across. That is a data
fix in Salesforce, not something to retry.

## When the pull is not enough

`monetizenow_pull_crm_entity` handles one account at a time and only accounts. Switch to
`monetizenow-sync-migration` when:

- there are many accounts to bring over, not one;
- the Salesforce record is already sync-enabled but stuck, or carries a sync error — the pull does
  not re-drive a record the package has already taken ownership of;
- the user wants ongoing sync turned on rather than a one-off import.

That path is heavier and worth naming as such: it writes to Salesforce and needs either a
Salesforce MCP connection or the `sf` CLI. If neither is available in the session, say so rather
than looking for a way around it.

## Step 2 — Create the base quote

Ordinary quote creation against the account id from step 1. `monetizenow-quote-builder` owns the
defaults, the contract-span rule, and the auto-renew trap; do not re-derive them here.

One thing is specific to this flow: **leave `opportunityId` unset.** That is what produces the new
opportunity this skill then links to the CRM. Pass an `opp_` id only when the user is adding a quote
to a deal that already exists in MonetizeNow — in which case that opportunity probably already
carries a `customId`, and step 3 is a check rather than a write.

Offerings and pricing come after, and belong to `monetizenow-quote-builder` and
`monetizenow-pricing-strategy`. Nothing about them changes because the account came from a CRM.

## Step 3 — Stamp the opportunity with the Salesforce id

The created quote carries its opportunity nested inside it; read the `opp_` id from there, or
retrieve the quote if you no longer have the create response. Quote search has no opportunity
filter, so there is no way back to it other than through the quote itself.

**Check the id is free first.** `monetizenow_search_objects` on `opportunity` with
`filters: {"custom_id": "006…"}`. Note the filter is named `custom_id` — snake_case — while
the field on the opportunity and the parameter on the update call are both `customId`.

- No match: proceed.
- It matches the opportunity you are about to stamp: already done, nothing to write.
- It matches a **different** opportunity: stop and report which one. Do not clear the other
  opportunity's `customId` to make room. Two opportunities pointing at one Salesforce deal is a
  situation someone has to decide about, and the most likely explanation is that the quote should
  have been created on that existing opportunity instead.

Then `update_opportunity` with `customId` only. Leave `primaryQuoteId` out unless the user asked
for a specific primary quote; sending both means `customId` is written first, so a later failure on
the primary quote leaves the stamp already landed.

**A rejection here means nothing was written.** Every rule is checked before anything is committed,
so the same call will fail the same way — report the reason instead of retrying.

Verify by retrieving the opportunity: `customId` set, and a `url` pointing at the CRM record. That
url is an authenticated, human-facing Salesforce page — surface it to the user, do not fetch it.
Its absence is not proof of failure; the connector may simply be unreachable.

## Where the Salesforce opportunity id comes from

It is not something quote creation can produce — only the user or the CRM has it. In order of
preference: ask the user, read it out of a Salesforce link they pasted, or look it up through a
Salesforce connection if the session has one. A Salesforce opportunity id starts `006` and is 15 or
18 characters.

Never infer one from an account or a quote name, and never pass it as `opportunity_id` — that
parameter is the MonetizeNow `opp_` id and the platform rejects a CRM id outright.

## Step 4 — Put the synced contacts on the quote

Having pulled the contacts across, the quote can carry them. `set_quote_contacts` **replaces** the
quote's whole contact set rather than patching it, so retrieve the quote first and resend the
contacts that should survive alongside the ones you are changing — anything omitted is cleared.

Two rules are easy to trip over on a freshly created quote: `primaryId` and `billingId` must be
sent together, so setting one means resending the other; and the three signer lists are silently
ignored when the quote has `requiresEsign` false, so pass `[]` for them rather than guessing.

The contacts must already belong to the quote's account, which is exactly what step 1 arranged. A
contact that is in Salesforce but not on the MonetizeNow account is rejected — go back to step 1
and read the pull's `summary`.

## What this skill cannot do

An opportunity's name, account and owner are read-only through every tool, and its status is
derived from the linked quotes rather than stored. `customId` and `primaryQuoteId` are the only two
things about it that can be changed. When a user asks to rename an opportunity, move it to another
account or set it to won, say that it is not something the platform exposes — the change belongs
in Salesforce, where it will flow back.

A WON opportunity's primary quote is frozen, so the window for changing it closes once a quote is
accepted.

## Recovery

| What you see | What it means | What to do |
|---|---|---|
| `customId` conflict on update | Another opportunity already points at that Salesforce deal | Report both opportunities. Do not clear the other one's `customId` unless the user asks for exactly that |
| Pull succeeded, contacts missing from `summary` | Some contacts were rejected; the account still landed | Check for missing email addresses first — those never sync. Report the rest verbatim |
| Pull succeeded, CRM record not stamped | MonetizeNow has the account; Salesforce does not know its MonetizeNow id | The quote can still be built. Flag it — the link back is incomplete, and `monetizenow-sync-migration` is the repair |
| `set_quote_contacts` rejects a contact | It does not belong to the quote's account, or it is canceled | Re-check the account's contacts by search; do not create a duplicate contact to get past it |
| Update rejected for any reason | Validated up front, so nothing was written | Report the reason. A retry fails identically |
| Opportunity is WON | Primary quote is frozen | Say so; this is not a transient state |

## Related skills

`monetizenow-quote-builder` for assembling the quote itself — offerings, ramps, amendments and
renewals. `monetizenow-pricing-strategy` for any price on it. `monetizenow-data-retrieval` when the
account has to be resolved by name rather than by CRM id. `monetizenow-sync-migration` for the
Salesforce-side path: many accounts, stuck records, or turning ongoing sync on.
