---
layout: post
title: "Democratising data access with Claude.ai: how we are replacing BI"
categories: [ai]
tags: [claude, mcp, llm, bi, jollyes]
---
*Notes from building a stateful personal analyst at Jollyes Pets on a $10/month VM*

> "Is Uber performing well?"

That's the kind of question Jollyes managers now drop into Claude.ai and get a defensible answer to in under a minute - without an analyst, nor a dashboard, yet with the right business logic attached.

Our Claude.ai tennant can SSO into a read-only MCP server I deployed alongside a daily Snowflake export. It has been shockingly effective at drawing together disparate data sources into coherent analyses across our retail domain, and has let colleagues at every level interact with our data in real time without waiting for BI or analyst capacity to come free.

The solution is secure, cheap to run, and, most importantly, accurate from both a SQL and a business perspective.

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

Total infrastructure cost: roughly $10/month - one always-on Fargate task, an S3 bucket, and one daily ECS export task, with enough capacity for our enterprise and effectively no cost to scale usage.

The data choices were deliberate: small enough to fit in memory on a cheap VM, PII-clean, and well-described. But the real power doesn't come from the infrastructure. It comes from using the **stateful nature of MCP** to gate access to context, and to manage tokens and attention carefully.

## The trick: about → describe_table → query

Naïve "natural language to SQL" works in demos, but cannot work in an enterprise context. The reason isn't model capability - it's that production data is full of traps that aren't in the schema:

- Our fiscal calendar sometimes has 53-week years; YoY needs `dim_date.last_year_date`, not `today - 365`.
- We always use LFL in our analysis, but what does that mean at Jollyes?
- `fact_stock` double-counts inventory across parent/child SKU pairs unless you join through `dim_sku_links`.
- We switched to Every Day Low Pricing in January 2025; any YoY revenue or discount comparison crossing that date is a trap, but only for one more month.

So the **MCP server enforces a curriculum**:

1. **`about()`** - must be called first. Returns current fiscal state, the table catalogue, the canonical join graph, and the business context (EDLP, loyalty, livestock) that frames every YoY question. ~6k tokens, paid once per session.
2. **`describe_table(name)`** - must be called for every table the model wants to query. Returns the schema, sample rows, and - crucially - the per-table column notes, useful patterns, and warnings I've accumulated from working with Jollyes data.
3. **`query(sql)`** - only runs if the referenced tables have been described in this session. Otherwise it returns: *"Tables not yet described: X, Y. Call describe_table('X,Y') first."*

The gating is enforced inside the tool itself - the SQL is parsed for `FROM`/`JOIN` clauses and checked against a per-session set. The model can't skip steps even if it wants to. When it tries, the error message tells it exactly what to do next, and it self-corrects within the same turn.

## Context, not corpus

The gating system is not merely a guardrail. It is a context management strategy.

Instead of building the gating, we could expose the entire schema and business rules in the connector prompt. At small scale this is workable. However, a warehouse with 100 tables, sample rows, join guidance, and operational caveats becomes tens of thousands of tokens of mostly irrelevant context attached to every interaction, and I want our users to leave this connector 'always on' with no detriment to them. 

Therefore, this MCP design inverts that model. There are three distinct context tiers:

| Tier | Context loaded | Approx size | Loaded when |
|---|---|---|---|
| Connector registration | Tool names + descriptions | ~300 tokens | Once, when connector enabled |
| `about()` | Fiscal calendar, join graph, business-wide context | ~6k tokens | Once per session |
| `describe_table(X)` | One table's schema, examples, warnings, patterns | ~1–2k tokens | Only when that table is needed |

The connector tier is the one that matters most for adoption. Because the registration is almost empty, an analyst can leave the connector permanently enabled in Claude.ai alongside Calendar or Drive without paying any meaningful context cost until a retail question is asked. There's no "should I turn this on?" friction - the answer is always yes.

The describe tier is the one that matters most for correctness. A voucher ROAS query may require `fact_voucher_campaign_week`, `fact_vouchers`, `dim_date` - and nothing else. The model never sees the unrelated stock tables, the unrelated labour metrics, the unrelated footfall caveats, the hundreds of irrelevant columns, the dozens of unrelated warnings. At 100 tables the system behaves almost identically to 10, because retrieval is conditional rather than global.

**This matters for more than token cost, but attention**. The scarce resource isn't tokens or context - though sensible management is prudent to maximise ROI. Every irrelevant warning loaded into context is a distractor competing for model reasoning capacity:

- does parent/child SKU deduplication matter for voucher analysis?
- does the EDLP pricing transition affect today's stock query?

Usually the answer is no. Large prompts don't simply cost more - they dilute signal. The describe-table notes are therefore intentionally terse and highly specific, usually under 50 words each. The objective is not maximum context; it is maximum signal density - context is distraction in an attention-focussed model.

### A short example: parent/child SKUs

Beef Single Tin showed ~40k units in stock, but only a handul sold last quarter. Thus, it looked grossly overstocked. Cover calculation: 5,791 weeks 1000x fold where we should be.

But the tin is a parent SKU. Real demand sits in the child 12-pack record. True cover: 3.7 weeks. The model would have confidently misdiagnosed inventory health based on entirely correct SQL. (A mistake, I did make in my first month at Jollyes!)

The fix can't be SQL guardrails - it has to be context. It was a short note in `dim_sku_links` describing parent/child semantics, loaded only when the model touches that table.

## Why not Skills?

Anthropic's Skills feature is the obvious alternative for some of this. We didn't go that route, for three reasons:

- **Skills are user-side; MCP is server-side.** Anyone who connects gets the right context, regardless of whether they've enabled a skill. Not all our MCP users are coming from Claude.ai.
- **Updates are centralised.** When EDLP context changes or a new structural break happens, I update the MCP server once and not need to distribute a new skill file.
- **`describe_table` doesn't fit in a Skill anyway.** Per-table notes, sample rows, and warnings are too large to ship as static skill content, and they should only load when relevant.

## Closing

Context window size can be a red herring. The real question isn't how much data you can fit into the model. It is:

> What is the smallest set of high-signal tokens required at the exact moment the model needs them?

Most MCP servers I've seen are effecmderused in the MCP ecosystem so far.

## Open questions we’re talking about:

- How do we make analyses reproducible across weeks and between users?
- How do we make sure table notes get updated when users find errors?
- When connecting a new data source, how do we add the right notes?
- Who owns the `about()` context?
