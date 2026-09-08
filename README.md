# monetizenow-mcp-skills

Agent Skills for working with the MonetizeNow MCP server from Claude Code or Claude Desktop.

## Why these exist

A tool description can explain one call. It cannot tell you that changing a price has a correct
order to try things in, that a multi-period offering has to be built as a group, that auto-renew is
silently ignored on create, or that a contract — not the newest quote — is what answers "what is
this customer on".

That is what these skills carry: the sequences, choices and traps that span several calls. They
deliberately **do not** restate what the tool descriptions already publish; where per-call mechanics
matter, they point at the tool's own description as the authority.

## Skills

| Skill | Covers |
|---|---|
| [`monetizenow-pricing-strategy`](monetizenow-pricing-strategy/) | The ordered ladder for changing a price, the five pricing models, catalog vs custom discounts, verifying proration |
| [`monetizenow-quote-builder`](monetizenow-quote-builder/) | Quote assembly order, defaults worth trusting, the auto-renew trap, ramp groups, contracts and amendments |
| [`monetizenow-data-retrieval`](monetizenow-data-retrieval/) | Which schema tool answers which question, which entities support which operations, reading empty and rejected searches, the multiple-match protocol |
| [`monetizenow-data-analysis`](monetizenow-data-analysis/) | Choosing the API or SQL, schema and enum-value discovery, aggregation through Metabase |

`monetizenow-data-retrieval` underpins the others — identifying the right record is a prerequisite
for pricing it, quoting it, or reporting on it — but each skill installs independently.

`monetizenow-data-analysis` predates the other three and is tracked here as source, unpacked from
the `.skill` archive it previously shipped as. Its tool references have since been corrected: it
had named several tools the server does not publish and attributed the Metabase query tools to the
MonetizeNow server. It now distinguishes the two servers and covers reading enum literals with
`get_field_values` before filtering on them.

## Installing

Copy the skill directories into your skills folder:

```bash
cp -r monetizenow-* ~/.claude/skills/
```

Use a project's `.claude/skills/` instead to scope them to that project. Claude Code and Claude
Desktop consult skills based on the `description` in each one's frontmatter, so no manual
invocation is needed.

## Layout

Each skill is a directory containing a `SKILL.md` with YAML frontmatter, plus optional
`references/` files that are read only when needed:

```
monetizenow-data-retrieval/
├── SKILL.md
└── references/
    └── entities.md      # id prefixes and entity definitions
```

Skills are kept as source rather than packaged `.skill` archives so changes are reviewable in a
diff. Package one for distribution when needed; the archive is a build artifact, not the source of
truth.

## Contributing

One rule matters more than the rest: **check whether the MCP server already says it.**

Tool descriptions are published to every client, so anything a tool already documents does not
belong in a skill. Repeating it creates two sources for the same fact, and the copy here is the one
that silently goes stale when the server changes. Where per-call mechanics matter, point at the
tool's own description instead of restating it.

That makes the useful shape of a skill fairly narrow — the things no single tool can tell you:

- **Order.** Which call comes first, and what to check before moving on.
- **Choice.** Which of several valid routes to take, and why one beats another.
- **Traps.** Where a call reports success but does not do what it looks like it did.

When the server changes, the skills need a pass. The pricing ladder in
`monetizenow-pricing-strategy` is the piece most worth re-checking: none of it is duplicated in any
tool description, so nothing else will catch it drifting.
