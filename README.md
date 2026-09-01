# monetizenow-mcp-skills

Agent Skills for working with the MonetizeNow MCP server from Claude Code or Claude Desktop.

## Why these exist

The hosted MonetizeNow AI client carries a large body of operating knowledge in its system prompt.
An MCP client receives the server's tool descriptions and nothing else, so anything that lives only
in that prompt is knowledge it does not have — the ordered pricing ladder, where a rate's
`priceModel` actually lives, what a quote offering group is and how to recognize one.

These skills close that gap. They deliberately **do not** restate what the tool descriptions already
publish, because a second copy of a field list is a copy that goes stale. Where per-call mechanics
matter, the skills point at the tool's own description as the authority.

## Skills

| Skill | Covers |
|---|---|
| [`monetizenow-pricing-strategy`](monetizenow-pricing-strategy/) | The ordered ladder for changing a price, the five pricing models, catalog vs custom discounts, verifying proration |
| [`monetizenow-quote-builder`](monetizenow-quote-builder/) | Quote assembly order, defaults worth trusting, the auto-renew trap, ramp groups, contracts and amendments |
| [`monetizenow-data-retrieval`](monetizenow-data-retrieval/) | Which schema tool answers which question, which entities support which operations, reading empty and rejected searches, the multiple-match protocol |
| [`monetizenow-data-analysis`](monetizenow-data-analysis/) | Aggregation and ad-hoc querying of MonetizeNow data through Metabase |

`monetizenow-data-retrieval` underpins the others — identifying the right record is a prerequisite
for pricing it, quoting it, or reporting on it — but each skill installs independently.

`monetizenow-data-analysis` predates the other three and is maintained separately. It is tracked
here as source, unpacked from the `.skill` archive it previously shipped as, with its content
unchanged — including a known gap: it names `get_accounts`, `get_invoice`, and `get_quote`, which
the server does not publish, and attributes the Metabase query tools to the MonetizeNow server.
Its discover-then-query workflow is sound; the tool references around it need a refresh.

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

## Keeping them current

These are *derived* from the AI client's prompts rather than generated from them, so the two will
drift. Before extending a skill, confirm the content is not already published by a tool description
— otherwise the skill becomes a stale duplicate of the server's own documentation. The analysis
behind the split, including the procedure for checking it, lives in the `ai-assistant` repo at
`docs/mcp-skills-candidates.md`.

The pricing ladder in `monetizenow-pricing-strategy` is the piece most worth keeping in sync: it is
the only content here with no redundant copy anywhere in the tool descriptions, and the most costly
to have wrong.
