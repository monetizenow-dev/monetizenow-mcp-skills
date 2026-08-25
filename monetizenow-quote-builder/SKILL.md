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
2. **Create the quote.** The contract span is set here. Note that create and update express the
   span differently — one takes a length, the other an end date — so read the tool description for
   whichever you are calling rather than carrying fields across from the other.
3. **Add the quote offerings**, one call per offering. Ramps are a special case; see below.
4. **Price them.** Pricing is its own ordered procedure — see `monetizenow-pricing-strategy`. Do
   not reach for a discount just because it is the quickest way to move a total.
5. **Verify** by retrieving the finished quote and reading its actual state. Describe what the
   quote *is*, not what you were trying to make it be — if a step did not land, that belongs in the
   summary.
6. **Signal the redirect once**, after the quote is successfully created, so the user can navigate
   to it. It is for newly created entities only, not for updates.

## Defaults and judgment calls

**A quote starts today** unless the user says otherwise. Do not invent a future start date to make
a term land on a round number.

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

Quote contacts appear on a retrieved quote, but no available tool writes them. When a user asks to
change them, say plainly that it is not something you can do. Do not describe a manual workaround
or a path outside the platform — that reads as a capability and it is not one.

## Related skills

`monetizenow-data-retrieval` for resolving accounts, offerings, and rates and handling multiple
matches. `monetizenow-pricing-strategy` for setting or changing any price on the quote.
