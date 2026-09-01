---
name: monetizenow-pricing-strategy
description: How to change the price of a MonetizeNow quote offering correctly — the ordered ladder of pricing options, how to read a rate's pricing model, and how to judge whether an unexpected total is a real problem. Use this skill whenever the user wants to change, set, match, or explain a price on a MonetizeNow quote: "get this to $X", "apply a discount", "match the price they had last year", "why is this line $Y", "make the second year more expensive", or any request touching rates, discounts, custom pricing, proration, or a total that looks wrong. Use it even when the request sounds like a simple field edit, because the correct move is usually not the most obvious one.
---

# Pricing a MonetizeNow quote offering

There are four different ways to change what a quote offering costs, and they are not
interchangeable alternatives — they form a ladder that must be walked in order. Reaching for the
wrong rung produces a quote that shows the right number and is structurally wrong: the total
matches what the user asked for, but the commercial story on the document is not what they meant.

The most common failure is jumping straight to a discount because it is the easiest lever that
moves the total. Work the ladder instead.

## Before you can price anything: find the pricing model

The right rung depends entirely on how the offering's rate calculates price, so establish that
first. The pricing model lives in `priceModel`, inside the rate's `pricing` array — and that array
is only present on the **full rate entity**. Search results do not include it.

So: retrieve the rate. An agent that searches for the rate and reads the summary cannot tell which
rung applies and will guess.

| Pricing model | How price is calculated |
|---|---|
| `VOLUME` | `unit_price × quantity`. May itself include tiered volume-based pricing. |
| `TIERED` | Incremental volume-based pricing determined by tier thresholds. |
| `FLAT` | A fixed price, independent of quantity. |
| `PERCENT` (of total) | A percentage of other products on the quote — e.g. support at 10% of license. |
| `CUSTOM` | A per-unit price set by hand during quoting. |

## The ladder

Walk these **in order**, and **for each quote offering individually**. Stop at the first rung that
achieves the desired price — once one works, do not attempt the later ones.

1. **Update custom pricing** — available only when the rate uses the `CUSTOM` model. Custom
   pricing fields on a quote offering are inert for every other model, so setting them on a
   `VOLUME` rate silently does nothing.
2. **Find an existing rate that already matches** the desired configuration. Review the offering's
   available rates. If more than one matches, stop and present them for the user to choose — this
   is a genuine multiple-match situation, not a judgment call you can make for them. If the user
   says none of them fit, continue to rung 3 for this offering.
3. **Create an account-based rate.** Note that this requires a pricing configuration for *every*
   product on the offering, optional ones included — see the `create_account_based_rate` tool
   description for the exact requirement.
4. **Apply a discount**, at the quote offering or the item level. Rung 4 has the same shape as
   rung 2: look for an existing discount before inventing one. Discounts are searchable, and
   `category` separates a reusable catalog discount (`CATALOG`) from one created for a single
   quote (`CUSTOM_QUOTE_DISCOUNT`) — an unfiltered search returns both, so filter when you mean
   one. Prefer an existing `CATALOG` discount that fits; fall back to a custom one only when
   none does. Check `discountType` (`PERCENTAGE` vs `FLAT`), `durationType` (`ONE_TIME`,
   `LIMITED`, `UNLIMITED`) and the scope fields, since a discount that matches on value can still
   be scoped to the wrong offering, product, or rate.

The order reflects how invasive each option is. Rung 1 edits a field on this one offering. Rung 2
reuses something the catalog already has. Rung 3 adds a durable new artifact scoped to the account.
Rung 4 changes what the customer sees as the negotiated concession. Stopping early keeps the quote
clean and keeps later rungs available if the number changes again.

## Two constraints that hold at every rung

- **Do not adjust quantity to influence price.** Quantity describes what the customer is buying.
  Changing it to hit a number misstates the deal, even when the total comes out right. Only change
  quantity when the user explicitly asks you to.
- **Discounts can only reduce a price, never increase it.** If the target is *above* the calculated
  price, rung 4 cannot get you there — the answer is on rung 2 or 3.

## Reading the result

Once priced, the total may not match a hand calculation. Two mechanisms usually explain the gap
before you go looking for a bug.

**Proration** reflects how long an offering is active within the contract term. The term is split
into billing segments by the rate's billing frequency, and proration is based on the number of
active days in each segment.

Do not treat proration as an unverifiable assumption — it is observable, in two places, and
checking beats guessing:

- **Whether a partial period prorates at all** is the product's `prorationPolicy`: `PRORATED`
  charges pro rata, `NON_PRORATED` charges the full period. It is both a field on the product and
  a filter you can search on, so a mixed quote may well contain products with different policies.
- **What was actually applied** is on the records themselves — a quote offering carries
  `debitProrationMultiplier` and `creditProrationMultiplier`, and an invoice item carries
  `prorationMultiplier`. Read the multiplier rather than reconstructing it from dates.

So when a total does not reconcile, retrieve the product to see its policy and read the multiplier
off the offering or invoice item. Only after both have been checked, and still do not explain the
number, is it fair to tell the user something tenant-level may be responsible — and say precisely
what you did check.

**Rounding**: small per-segment variances, on the order of a cent, are rounding artifacts and not
worth raising. Larger discrepancies are worth flagging as potential issues rather than explaining
away.

## Related skills

Identifying the account, offering, or rate to price against — and handling a search that returns
several candidates — is covered by `monetizenow-data-retrieval`. Assembling a whole quote, including
ramped multi-period pricing, is covered by `monetizenow-quote-builder`.
