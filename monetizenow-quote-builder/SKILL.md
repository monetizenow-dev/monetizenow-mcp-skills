---
name: monetizenow-quote-builder
description: How to assemble a MonetizeNow quote end to end — the order of operations, what to include by default, how to build multi-period ramped pricing as a quote offering group, and how to recognize one when reading a quote back. Use this skill whenever the user wants to create, build, populate, extend, or inspect the structure of a MonetizeNow quote: "quote Acme for 1500 seats", "add a second year at a higher price", "set up a 3-year ramp", "build a quote from this order form", "what's on this quote", or any multi-period, ramped, or escalating pricing request. Use it before calling any quote or quote-offering tool, because the individual tool descriptions cover single calls and not the sequence they belong in.
---

# Building a MonetizeNow quote

The individual tools describe their own mechanics well — required fields, id prefixes, which
parameters belong on a create versus an update. Read them; this skill deliberately does not restate
them, because a second copy of a field list is a copy that goes stale.

What the tool descriptions cannot tell you is the **order**, the **defaults**, and the handful of
judgment calls that decide whether the resulting quote is structurally correct.

## Order of operations

1. **Resolve the account** first. A quote is created against an account, so an ambiguous account
   name has to be settled before anything is written. See `monetizenow-data-retrieval`.
2. **Create the quote.** The contract terms all have documented defaults, and the tool
   description says plainly not to ask the user for them. Take it at its word: asking a rep to
   confirm a contract length that already defaults to twelve months is friction, not diligence.
   Ask only about what the request actually leaves open.

   Create and update express the span differently — one takes a length, the other an end date —
   so read the description of whichever you are calling instead of carrying fields across.
   Converting a length into an end date is not arithmetic you should do from memory: span dates
   are inclusive, and adding months clamps at short months. The tool description gives the exact
   rule and a worked example; follow it rather than reasoning it out.

   One term does not behave like the others: **auto-renew is ignored on create.** The platform
   takes it from the tenant's quote settings, so a value passed here is silently dropped. If the
   user asked for a specific auto-renew setting, read it off the returned quote and follow up
   with an update when it differs. This is the one default worth verifying every time.
3. **Add the quote offerings**, one call per offering. Ramps are a special case; see below.

   Searching offerings by name or attribute returns only `ACTIVE` ones unless you say otherwise,
   which is what you want while quoting — an inactive offering is not sellable. Two consequences:
   a name that finds nothing may exist in another state, and a search targeting one specific
   offering by id is not filtered this way, so it can return an inactive one. Check the status
   before building on a result you reached by id.
4. **Price them.** Pricing is its own ordered procedure — see `monetizenow-pricing-strategy`. Do
   not reach for a discount just because it is the quickest way to move a total.
5. **Verify** by retrieving the finished quote and reading its actual state. Describe what the
   quote *is*, not what you were trying to make it be — if a step did not land, that belongs in the
   summary. Auto-renew above is the concrete case: the create call reports success while quietly
   substituting the tenant's value, so only the returned quote tells you what was set.
6. **Signal the redirect once**, after the quote is successfully created, so the user can navigate
   to it. It is for newly created entities only, not for updates.

## Defaults and judgment calls

**Do not invent a start date** to make a term land on a round number. Leaving it unset starts the
contract today, which the tool description covers.

**Exclude optional products the user did not mention.** Offering products are flagged mandatory or
optional through `offeringProducts[*].isMandatory`. When quoting, an optional product the user
never brought up should be left off — adding it silently inflates the quote.

This is worth holding carefully, because the neighbouring rule is the exact opposite: *creating a
rate* requires a pricing configuration for every product on the offering, optional ones included.
Quoting excludes unmentioned optional products; rate creation prices all of them. The two rules sit
one tool apart and are easy to transpose.

## Ramps are groups, not repeated offerings

Any multi-period or ramped offering has to be built as a **quote offering group**: one root
offering plus any number of ramp offerings that point back at it through their schedule. Every
offering in the group keeps the same billing frequency.

Creating a standalone offering per period instead is not a stylistic variation — it is a
fundamental design error. The periods end up unrelated, so the platform cannot treat them as one
commercial arrangement, and downstream amendment and billing behavior is wrong even though the
quote may total correctly.

The `create_quote_offering` tool description carries the exact construction rules — which field
identifies the root, which is mandatory on a ramp, what the ramp's span defaults to. Follow it.

### Recognizing a ramp group when reading

Construction is documented; **reading** is not, and an agent can build a group correctly and still
misread one. When you retrieve a quote, its offerings arrive as a flat list. To reconstruct the
groups: an offering carrying a parent reference in its schedule is a ramp, and the offering it
points at is the root. Offerings with no parent reference are roots — either of a group or
standalone.

So a quote showing three offerings may be one root with two ramps, three independent offerings, or
a root-plus-ramp alongside a standalone. Report the structure, not the count. "Three offerings" is
misleading when the answer is "one product ramping across three years".

## What cannot be done

Some things a quote shows are not writable by any tool — quote contacts are the case you will hit,
and `update_quote` says so directly. When you meet one, say plainly that it is not something you
can do. Do not offer a manual workaround or a path outside the platform: that reads as a
capability, and it is not one.

## Quotes sit on contracts

A quote is not the whole story. Contracts are retrievable in their own right, and a retrieved
contract carries **every quote on it** — the original plus each amendment and renewal — along with
its renewal terms and whether it has been renewed.

That makes the contract the right object for any question about history or the current shape of a
customer's commitment: what changed at the last amendment, what renews and when, which quote is
live. Reconstructing that by searching quotes account-by-account gets you an unordered pile with no
indication of which superseded which.

So when a request says "extend", "amend", "renew", or asks what the customer is currently on, start
from the contract rather than the newest quote you can find. Search `object_type` "contract" to get
there — not the account.

## Changing an existing commitment: amend or renew

A change to something the customer already has is not a net new quote. Two dedicated tools exist,
and picking between them is the decision this skill can help with:

- **`amend_contract`** changes the commitment in place. It opens a DRAFT AMENDMENT quote on the
  **same** contract, pre-populated with the contract's current offerings and items.
- **`renew_contract`** continues the commitment into a new term. It opens a DRAFT RENEWAL quote
  which, once processed, creates a **new** contract pointing back at this one.

Three things follow that are easy to get wrong:

**Neither tool changes anything by itself.** Both leave a draft for review, and the contract is
untouched until that draft is processed. Renewal in particular never happens outright — if a user
asks you to just renew something, say that leaving a draft is as far as it goes.

**A net new quote cannot be either of these.** `create_net_new_quote` has no contract linkage, so
labelling a quote as a renewal there produces a quote attached to nothing. These tools are the only
route.

**Edit the draft, do not rebuild it.** What comes back is an ordinary quote — adjust it with
`update_quote`, `create_quote_offering` and `update_quote_offering`, exactly as in the assembly
steps above. The offerings that arrive pre-populated are the existing commitment; recreating them
produces duplicates rather than changes. When describing what an amendment does, read it off the
per-item amendment status rather than diffing by hand.

Each tool documents its own preconditions, and they differ in ways worth reading rather than
assuming — the states that block an amendment are not the states that block a renewal.

## Related skills

`monetizenow-data-retrieval` for resolving accounts, offerings, and rates and handling multiple
matches. `monetizenow-pricing-strategy` for setting or changing any price on the quote.
