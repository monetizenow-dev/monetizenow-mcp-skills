# MonetizeNow entity reference

Read this when you need to recognize an id, or when a request names an entity type and you need to
know what it actually is and how it connects to the others.

## Id prefixes

Ids are prefixed by type, so an id alone tells you what it points at. Several write tools validate
the prefix and reject a mismatch, which makes this a fast way to catch a wrong id before a call.

| Prefix | Entity |
|---|---|
| `acct_` | Account |
| `quot_` | Quote |
| `quof_` | Quote offering |
| `subs_` | Subscription |
| `offr_` | Offering |
| `rate_` | Rate |
| `prod_` | Product |
| `disc_` | Discount |
| `ctrct_` | Contract |
| `bilg_` | Bill group |
| `invc_` | Invoice |
| `pymt_` | Payment |
| `crdt_` | Credit |
| `crnt_` | Credit note |
| `cont_` | Contact |
| `usr_` | User |

Most of these are stated in the retrieval tool's own description, including product, discount,
contract, and subscription. The exceptions are offering, rate, and user, which come from search-parameter schema
examples — treat them as reliable but confirm against a real id if a call is rejected.

## What each entity is

**Account** — the customer. Quotes, invoices, credits, and bill groups all hang off an account.

**Quote** — a proposed commercial arrangement with a contract span. Retrieving a quote returns its
quote offerings inline, so there is no separate quote-offering retrieval.

**Offering** — a catalog construct: a sellable bundle of products. Products within it are flagged
mandatory or optional through `offeringProducts[*].isMandatory`.

**Rate** — the pricing behavior attached to an offering. Two properties matter most: the billing
frequency, which sets the interval at which the calculated price is charged, and the pricing model,
which determines how the price is calculated at all. The pricing model lives in `priceModel` inside
the rate's `prices` array and is only present on the full retrieved rate.

**Quote offering** — an offering placed on a quote, at a rate, for a span. Quote offerings belong to
quote offering groups: one root plus any number of ramps, each ramp pointing at the root through its
schedule. Every offering in a group keeps the same billing frequency.

**Product** — an individual item within an offering. Searchable in its own right, by `sku`,
`productType`, `bucketId`, `locked`, and `prorationPolicy`. That last one decides whether a partial
billing period is charged pro rata (`PRORATED`) or in full (`NON_PRORATED`), which makes the
product the place to look when a prorated total does not reconcile.

**Contract** — the durable commitment an account holds; quotes sit on it. A retrieved contract
carries every quote on the contract — the original plus each amendment and renewal — along with its
term in months, its status, whether it has been renewed, and its renewal terms. For any question
about history, amendments, or what a customer is currently on, the contract is the right object;
searching quotes gives you an unordered pile with no indication of which superseded which.

**Subscription** — what the customer actually has running, as opposed to what was quoted. A
retrieved subscription carries its currently-active items (each with its product and negotiated
price), its offering, and its discounts. Items removed by a prior amendment are not included, so a
subscription shows the present state rather than a history — use the contract for that. Searchable
by account, bill group, offering, rate, and by billing and provisioning status.

A quote is the proposal, a contract is the commitment, and a subscription is the live result. When
a question is about what someone is being billed for right now, the subscription is usually the
right object.

**Discount** — a reduction applied to a quote offering or item. `category` separates a reusable
catalog discount (`CATALOG`) from one created for a single quote (`CUSTOM_QUOTE_DISCOUNT`), and an
unfiltered search returns both. `discountType` is `PERCENTAGE` or `FLAT`; `durationType` is
`ONE_TIME`, `LIMITED` (with `durationMonths`), or `UNLIMITED`. Scope lives on `scopeOfferingId`,
`scopeProductIds`, and `scopeRateIds` — a discount can match on value and still be scoped to
something other than what you are pricing.

**Bill group** — a billing grouping under an account, determining how charges are collected
together. Has its own stats endpoint.

**Invoice** — a bill issued against an account. Its date fields (bill date, due date, paid date) are
plain dates rather than zoned timestamps, which matters when filtering.

**Payment** — a payment recorded against an invoice.

**Credit** — a monetary value applied against an account's invoices. Optionally scoped to a bill
group, and optionally linked to a credit note.

**Credit note** — the document a credit may be linked to.

**Contact** — a person associated with an account. Note that quote contacts specifically are
readable but not writable by any available tool.

**User** — a person with access to the tenant. When the user says "my" or refers to their own
profile, resolve it from the current user's context rather than searching by name.
