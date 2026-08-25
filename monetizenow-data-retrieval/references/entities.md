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
| `offr_` | Offering |
| `rate_` | Rate |
| `prod_` | Product |
| `bilg_` | Bill group |
| `invc_` | Invoice |
| `pymt_` | Payment |
| `crdt_` | Credit |
| `crnt_` | Credit note |
| `cont_` | Contact |
| `usr_` | User |

The prefixes for account, quote, quote offering, bill group, invoice, payment, credit, credit note,
and contact are stated in the tool descriptions themselves. Those for offering, rate, product, and
user are taken from the search-parameter schema examples — treat them as reliable but confirm
against a real id if a call is rejected.

## What each entity is

**Account** — the customer. Quotes, invoices, credits, and bill groups all hang off an account.

**Quote** — a proposed commercial arrangement with a contract span. Retrieving a quote returns its
quote offerings inline, so there is no separate quote-offering retrieval.

**Offering** — a catalog construct: a sellable bundle of products. Products within it are flagged
mandatory or optional through `offeringProducts[*].isMandatory`.

**Rate** — the pricing behavior attached to an offering. Two properties matter most: the billing
frequency, which sets the interval at which the calculated price is charged, and the pricing model,
which determines how the price is calculated at all. The pricing model lives in `priceModel` inside
the rate's `pricing` array and is only present on the full retrieved rate.

**Quote offering** — an offering placed on a quote, at a rate, for a span. Quote offerings belong to
quote offering groups: one root plus any number of ramps, each ramp pointing at the root through its
schedule. Every offering in a group keeps the same billing frequency.

**Product** — an individual item within an offering.

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
