---
layout: post
title: "Building a Stateful Retail Analyst with Claude and MCP"
categories: [ai]
tags: [claude, mcp, llm, bi, jollyes]
---
*How we are changing BI at Jollyes Pets on a $10/month VM*

> "Is Uber performing well?"

That's the kind of question Jollyes managers now drop into Claude.ai and get a nuanced report back in under a minute - without an analyst, nor a dashboard, yet with real data framed within the real business context.

Our Claude.ai tenant can SSO into a read-only MCP server I deployed alongside a daily Snowflake export. It has been remarkably effective at drawing together disparate data sources into coherent analyses across our retail domain, and has let colleagues at every level interact with our data in real time without waiting for BI or analyst capacity.

I employed the stateful nature of MCPs to gate data access behind enforced data discovery. This leads to responses which are grounded in business truth and real data.

> Yes. Revenue is £XXk/week contributing X.X% of total revenue, up from ~£XXk seven months ago. ATV is climbing. Customer satisfaction is X.XX★.

The model correctly understands Uber means Jollyes' Zoomies last-mile channel, pulls sales data across channels and compares across time and the whole estate, as well as considering other meanings of performance such as ATV and customer reviews. Moreover, it knows to provide a Jollyes-ready response with a % as well as absolute revenue number.

## The stack

```
Claude.ai
   │ SSO (Entra ID)
   ▼
MCP Server (Node/Express)
   │
   ▼
DuckDB (in-memory)
   │
   ▼
Daily Snowflake exports → S3 (Airflow)
```

Total infrastructure cost: roughly $10/month - one always-on Fargate task, an S3 bucket, and one daily ECS export task. This set-up has enough capacity for our enterprise and has no variable cost with increased querying.

The data choices were deliberate: small enough to fit in memory on a cheap VM, PII-clean, and well-described. But the real power doesn't come from the infrastructure.

## The trick: about → describe_table → query

Naïve "natural language to SQL" works in demos, but cannot work in an enterprise context. The reason isn't model capability - it's that production data is full of traps that aren't in the schema:

- Our fiscal calendar sometimes has 53-week years; YoY needs `dim_date.last_year_date`, not `today - 365`.
- We always use LFL in our analysis, but what does that mean at Jollyes?
- `fact_stock` double-counts inventory across parent/child SKU pairs unless you join through `dim_sku_links`.
- We switched to Every Day Low Pricing in January 2025; any YoY revenue or discount comparison crossing that date is misleading.

So the **MCP server enforces a curriculum**:

1. **`about()`** - must be called first. Returns current fiscal state, the table catalogue, the canonical join graph, and the business context (e.g. EDLP, loyalty, livestock) that frames every YoY question. ~6k tokens, paid once per session.
2. **`describe_table(name)`** - must be called for every table the model wants to query. Returns the schema, sample rows, and - crucially - the per-table column notes, useful patterns, and warnings we've accumulated from working with Jollyes data.
3. **`query(sql)`** - only runs if the referenced tables have been described in this session. Otherwise it returns: *"Tables not yet described: X, Y. Call describe_table('X,Y') first."*

The gating is enforced inside the tool itself - the SQL is parsed for `FROM`/`JOIN` clauses and checked against a per-session set. The model can't skip steps even if it wants to. When it tries, the error message tells it exactly what to do next, and it self-corrects within the same turn.

## Context, not corpus

The gating system is not just a guardrail with SQL suggestions. It also serves as a context and token management strategy. 

Instead of building the gating, we could expose the entire schema and business rules in the connector prompt. At small scale this is workable. However, a warehouse with 100 tables, sample rows, join guidance, and operational caveats becomes tens of thousands of tokens of mostly irrelevant context attached to every interaction. I want our users to leave this connector 'always on' with no detriment to them, and so I've purposefully kept the connector registration small. 

Consequently, the gated commands introduce context on-demand and on the topics which are required:

| Tier | Context loaded | Approx size | Loaded when |
|---|---|---|---|
| Connector registration | Tool names + descriptions | ~300 tokens | Once, when connector enabled |
| `about()` | Fiscal calendar, join graph, business-wide context | ~6k tokens | Once per session |
| `describe_table(X)` | One table's schema, examples, warnings, patterns | ~1–2k tokens | Only when that table is needed |


**This matters for more than token cost, but attention**. The scarce resource isn't tokens or context (any more!) - though sensible management is prudent to maximise ROI. Every irrelevant warning loaded into context is a distractor competing for model reasoning capacity:

- does parent/child SKU deduplication matter for voucher analysis?
- does the EDLP pricing transition affect today's stock query?

Large prompts don't simply cost more - they dilute signal. Irrelevant context is distraction in an attention-focussed model.

### A short example: parent/child SKUs

Beef Single Tin showed ~40k units in stock, but only a handful sold last quarter. Thus, it looked grossly overstocked at 5,791 weeks cover: ~1,000x where we should be.

However, the tin is a parent SKU. Real demand sits in the child 12-pack record, which is what is sold in stores, making the real cover: 3.7 weeks. The model would have confidently misdiagnosed inventory health based on entirely correct SQL. (A mistake I have also made in my first month at Jollyes!)

## Why not Skills?

Anthropic's Skills feature is the obvious alternative for some of this. We didn't go that route, for three reasons:

- **Skills are user-side; MCP is server-side.** Anyone who connects gets the right context, regardless of whether they've enabled a skill. Not all our MCP users are coming from Claude.ai.
- **Updates are centralised.** When EDLP context changes or a new structural break happens, I update the MCP server once and not need to distribute a new skill file.
- **`describe_table` doesn't fit in a Skill anyway.** A large skill would equally dilute attention just as a large connector body would.

## Closing

Most MCP servers I’ve seen are effectively stateless - each tool call independent of the last. The small connector context and curriculum here only work because the server holds per-session state to ensure: progressive disclosure, and curriculum enforcement. I think this is an underused pattern in the MCP ecosystem so far.

Finally, context window size can be a red herring. The real question isn't how much data you can fit into the model. It is:

> What is the smallest set of high-signal tokens required at the exact moment the model needs them?

## Open questions we’re talking about:

- How do we make analyses reproducible across weeks and between users?
- How do we make sure table notes get updated when users find errors?
- When connecting a new data source, how do we add the right notes?
- Who owns the `about()` context?
- Should we move `tableNotes` from a markdown file into a database and allow agents to edit them on the fly?
- What are the next actions we should give to the MCP - direct access to update our PIM? What gating and enforcement would be needed?
