---
name: monetizenow-data-retrieval
description: How to find the right MonetizeNow record and how to behave when a lookup is ambiguous — which schema tool answers which question, why a field you can read is often not a field you can filter on, what to do when a filter value is rejected, and the protocol for presenting multiple matches instead of picking one. Use this skill whenever a MonetizeNow request depends on identifying a record — looking up an account, quote, invoice, credit, rate, or offering by name, filtering or listing records, resolving "the latest one" or "the biggest one", or interpreting an empty or rejected search. Use it especially when a search returns nothing or returns several candidates, and before reporting that a record does not exist.
---

# Finding MonetizeNow records

The search tool's own description covers the filter syntax thoroughly — operator prefixes,
substring matching on names, the two kinds of date field, whole-day ranges. Read it and follow it;
this skill does not duplicate it.

What follows is the surrounding discipline: which tool to ask, what a failure actually means, and
what to do when the answer is not unique.

## The two schema tools answer different questions

They are not interchangeable, and the distinction is the single most common source of failed
searches.

- **`monetizenow_describe_object`** gives the *data model*: which fields an entity has, their
  types, allowed values, and how the entity relates to others. Narrow it with its field parameter
  rather than pulling everything.
- **`monetizenow_get_object_search_attributes`** gives the *filter keys*: which fields a search can
  actually filter on.

**The filterable set is substantially smaller than the field set.** Seeing a field on a described
or retrieved entity tells you nothing about whether you can search on it. Confirm the filter key
before building a search rather than after the call fails.

Consult these before guessing, not after an error — but do not re-derive what the conversation has
already established. When all you need is a filter key, the search-attributes tool is the cheaper
question.

Prefer lightweight lookups before detailed ones: search to identify the entity, then retrieve it
only when you actually need its full details. The exception is pricing, where the model only exists
on the retrieved rate — see `monetizenow-pricing-strategy`.

## Superlatives require an explicit sort field

Any ordering or superlative in a request — "latest", "most recent", "newest", "first", "last",
"top", "highest", "biggest" — depends on a sort field. If the user has not named one, stop and ask.

This holds even when the rest of the request is precise. "The most recent invoice for Acme" is
specific about the account and silent about whether recency means invoice date, due date, or
creation date, and those can disagree. Answering by taking whichever record came back first
produces a confident, unsourced, and frequently wrong answer.

## Reading failures correctly

**An empty result means your filter matched nothing. It does not mean the record is absent.** Report
that nothing matched what you searched for and ask the user to confirm the name. Never state that
the entity does not exist — you have not established that, and the user cannot tell the difference
between "absent" and "you searched wrong".

**A rejected filter value is a formatting problem on your side** until proven otherwise. Read the
error; it generally names the accepted format. Correct the value rather than retrying
near-identical variations.

Never tell the user that a field, filter, or date format is a "platform limitation" or "not
supported". A rejected value does not support that conclusion. Say what you were unable to
construct and ask for help, or offer a different angle — filtering on a related id and comparing
values from the results, for instance.

**Do not broaden a search blindly.** Widening filters until something comes back produces a match
that may have nothing to do with the request. Ask the user to clarify instead.

**Do not assume a reference id points where you want.** An id field on one entity names *a*
related record, not necessarily the one relevant to the request. Verify it before relying on it.

## When a lookup returns more than one candidate

This cannot be pre-empted by asking better questions up front: a perfectly clear request can still
match three accounts. Whenever a lookup returns more than one viable record, pause and let the user
choose — however confident you are that one is the obvious match.

Present the matches as a table with an option number, the id, the name, and whatever fields
actually distinguish them, then close with a direct prompt to select one.

| Option | ID          | Name            | Description |
|--------|-------------|-----------------|-------------|
| 1      | entity_id_1 | option_one_name | ...         |
| 2      | entity_id_2 | option_two_name | ...         |

**Every option must come from something you actually observed** — a tool result, or a tool's own
schema. Never invent an option, recall one from general knowledge, or carry one over from an earlier
conversation. In practice the options come from a lookup that returned several matches, an entity's
schema when the question is a field's allowed values, or a tool's input schema when it is what a
parameter accepts.

Fill the id, name, and description from what was returned rather than paraphrasing or embellishing,
and only include a column you can actually populate. If the results were paginated or truncated,
say so rather than presenting the page you have as the complete set. If you cannot assemble a full
set of options, present what you found and ask the user to clarify — do not fill the gap yourself.

Take no further action until the user selects. If they say none of the matches are valid, move to
the next logical step. Re-check after every tool result: a new set of matches means a new pause. One
match means proceed normally.

The reason to hold this line even when a match looks obvious is that the cost is asymmetric. A
needless question costs one turn. Silently operating on the wrong account can mean a quote, an
invoice, or a credit landing against the wrong customer.

## Entity reference

For id prefixes and short definitions of each entity type, read `references/entities.md`.

## Related skills

`monetizenow-quote-builder` for assembling a quote once the records are identified.
`monetizenow-pricing-strategy` for anything that changes a price. `monetizenow-data-analysis` for
questions that need aggregation or cross-entity correlation rather than identifying one record.
