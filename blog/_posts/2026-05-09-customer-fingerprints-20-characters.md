---
layout: post
title: "Fingerprinting every loyalty customer in 20 characters"
categories: [ai]
tags: [recommendations, personalisation, snowflake, jollyes, retail]
---
*How a tiny per-customer encoding turns our weekly basket analysis into real-time personalised recommendations - and now also powers voucher targeting.*

<div class="tldr" markdown="1">
**TL;DR**

- Every active Jollyes loyalty member is compressed into a ~20-character fingerprint that captures their pet's species and lifestage, as well as their top-5 spend categories. The entire loyalty base fits onto a single VM in memory.
- We built one-to-one personalisation by mixing signals from the current product page, the basket, and the fingerprint - every customer sees a personalised set of recommendations which changes with their basket.
- The same fingerprint has already found a second role: the marketing team reads it sideways to pick audiences like *"cat customers who buy food but not litter"*.
- Validation was run through Claude over our [MCP server](/stateful-retail-analyst-mcp/) - it samples real customers with real baskets, and understands in natural language whether the recommendations feel 'right'.
</div>

## The stack

```
Snowflake
   │  3 stored procedures: category codes → recs → loyalty
   ▼
S3
   │
   ▼
Express service on Fargate
   │  loads both CSVs into memory
   ▼
In-memory Maps  (recsByAnchor, loyaltyTopN, fingerprintByCustomer, …)
   │
   ▼
Public recs API   +   Private segments API
```

The infrastructure cost is roughly the same as my [stateful MCP server for retail analytics](/stateful-retail-analyst-mcp/): ~$10/month, with no database in the request path.

## Part 1 - A customer in 20 characters

A customer fingerprint is a variable-length string of base62 characters, 5 + 3·N where N is the number of category entries (0–5):

```
S J E W  X1Y1T1  X2Y2T2  X3Y3T3  X4Y4T4  X5Y5T5  K
│ └┬┘ │  └──┬─┘                                  │
│  │  │     │                                    └─ checksum
│  │  │     └─ "I spend tier T on (dept, cat, subcat1) XY"  (×5)
│  │  └─ breadth: distinct (dept, cat) over 180d
│  └─ lifestage mask: junior (J) / senior (E) spend last few months
└─ species mask  (any spend over 180d, by species)
```

A new cold-start customer who has only ever bought a Christmas dog toy looks like `b0021` (5 chars: Dog species, no junior/senior signal, breadth 2, checksum 1). A multi-pet senior-cat household with diversified spend looks like `nA4Ax7M9aB7Bc6Cb5gC2y`: that's the whole customer!

Within this string we capture **>95% of spend for >90% of customers** - meaning that in 20 characters we keep every touchpoint tailored to a customer's pet and spend preferences.

**Asymmetric trust between product and customer.** When we tag a *product* with a lifestage, we trust a stack of upstream signals - the catalogue tag, the category path, and a word‑boundary regex over the product description - and let the most restrictive win, so anything reading "junior" beats "adult". When we tag a *customer* with a lifestage, we deliberately are conservative and only tag the most certain customers with "Puppy" - e.g. buying-curated puppy food lists.

Both halves of the asymmetry point at the same outcome: **a non-puppy household should never see puppy recs**. We err inclusively on the product side (any "junior" signal wins, so nothing puppy-shaped slips out as a generic adult product) and strictly on the customer side. The asymmetry exists to protect the customer experience, not the model's tidy logic.

## Part 2 - Personalisation on the fly

This is the part I find most fun.

At Sainsbury's - where I previously led marketing data science - we never cracked this in production. Product pages were static (no per-customer reranking), and the personalised "before you go" page was 1-2-1 personalised but didn't react to a live basket. What runs at Jollyes does all three at once: anchor SKU, customer fingerprint, and live basket, blended at request time.

### The blend

At request time, every candidate SKU starts with the anchor product’s pre-ranked also-bought score - the anonymous baseline from a classic basket analysis.

Two additional signals can boost it:

* basket co-occurrence: candidates frequently bought alongside items already in the basket
* loyalty affinity: candidates appearing in the customer’s collaborative-filter top-N

All three signals decay naturally by rank position: appearing first matters much more than appearing twentieth.

The combined score is then multiplied by a fingerprint factor, depending on whether the candidate’s category matches one of the customer’s top-5 spend categories and how concentrated their spend is in that category. A department allowlist and lifestage filter sit in front as binary vetoes.

Two properties fall out cleanly. First, the fingerprint multiplier is deliberately multiplicative, not additive: a strong "Dog Adult Wet Food" fingerprint cannot drag Cat Litter onto a Dog Food page, because the candidate must already exist in the anchor’s recall set to begin with. The fingerprint amplifies plausible candidates; it does not create them.

Second, anonymous traffic runs through the exact same arithmetic. With no basket and no loyalty data, the extra terms reduce to zero and the multiplier collapses to ×1, leaving the original ranking unchanged. There is no separate "personalised" code path - same function, different inputs.

### Dogfooding with Claude

You can't unit-test *"does this feel personalised"*. So, we validate by using Claude Code with our [stateful MCP server](/stateful-retail-analyst-mcp/) and letting it browse: pick a real loyalty number, pick a real basket, walk through what that customer would see on a handful of product pages, and judge whether the recommendations feel coherent. It surfaces things a deterministic test never would - a senior-cat household whose second rec is a very active toy, a basket where the co-occurrence pass tilts the page in a direction that feels wrong with the context of the customer's prior transactions.

## Part 3 - Same fingerprints, sideways: voucher targeting

We realised we could read the same map to size customer groups for marketing, for example the marketing team wants to send a voucher to *cat customers who buy food but not litter*. The marketing team now have a self-serve GUI to get instant audience sizes, and a press to deliver button. Previously, we'd have spent analyst time on Snowflake queries to build the audience.

Simply, the segments API just queries the in-memory fingerprints sideways:

```json
POST /api/private/segments/count
{
  "speciesAny": ["cat"],
  "categories": [
    { "code": "<cat-food code>",   "op": "CONTAINS",     "minTier": 1 },
    { "code": "<cat-litter code>", "op": "NOT_CONTAINS"                }
  ]
}
```

Under the hood:

```ts
for (const [loyaltyNumber, mask] of speciesMaskMap) {
  if (speciesAnyMask !== 0 && (mask & speciesAnyMask) === 0) continue;
  if (!fingerprintMatchesFilters(fp.get(loyaltyNumber), filters)) continue;
  result.push(loyaltyNumber);
}
```

One iteration per customer, each a few bitwise ops and a short loop over ≤5 fingerprint entries. Returns in <100ms.

The same fingerprint that says *"this customer's recommendations should lean cat-food-heavy"* also says *"this customer is a cat-food shopper who has never bought cat litter"*.

## Closing

We were trying to solve: how do you personalise every customer interaction quickly and cheaply. By tightly encoding each customer - a summary of an expensive, slow analytics query - we can store everything in RAM and reduce the runtime to a few multiplications and `Map` lookups. We've avoided runtime inference, and big customer embeddings, and used simple static lists instead: and yet it feels more personalised.
