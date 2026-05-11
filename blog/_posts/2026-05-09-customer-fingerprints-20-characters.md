---
layout: post
title: "Fingerprinting every loyalty customer in 20 characters"
categories: [ai]
tags: [recommendations, personalisation, snowflake, jollyes, retail]
---
*How a tiny per-customer encoding turns weekly basket analysis into real-time personalised recommendations - and now also powers voucher targeting.*

<div class="tldr" markdown="1">
**TL;DR**

- Every active Jollyes loyalty member is compressed into a ~20-character fingerprint that captures their pet's species, lifestage, and top‑5 spend categories. The entire loyalty base fits onto a single VM in memory.
- We build one-to-one personalisation by mixing signals from the current product page, the basket, and the fingerprint - every customer sees a personalised set of recommendations at request time.
- The same fingerprint repurposes cleanly: the marketing team reads it sideways to pick audiences like *"cat customers who buy food but not litter"* - five lines of code, zero new data pipelines.
- Validation runs through Claude over our [MCP server](/stateful-retail-analyst-mcp/) - it samples real customers with real baskets, and understands in natural language whether the recommendations feel 'right'.
</div>

## The stack

```
Snowflake (weekly Sunday 02:00 London)
   │  3 stored procedures: category codes → recs → loyalty
   ▼
S3   (~1 MB recs CSV  +  ~120 MB loyalty CSV)
   │
   ▼
Express service on Fargate (1 vCPU / 2 GB)
   │  loads both CSVs at boot, refreshes daily 03:00 London
   ▼
In‑memory Maps  (recsByAnchor, loyaltyTopN, fingerprintByCustomer, …)
   │
   ▼
Public recs API   +   Private segments API
```

One always‑on container, two CSVs, one heartbeat. The infrastructure cost is roughly the same as my [stateful MCP server for retail analytics](/stateful-retail-analyst-mcp/): ~$10/month. There is no database in the request path.

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

A new cold-start customer who has only ever bought a Christmas dog toy looks like `b0021` (5 chars: Dog species, no junior/senior signal, breadth 2, checksum 1). A multi-pet senior‑cat household with diversified spend looks like `nA4Ax7M9aB7Bc6Cb5gC2y`. That's a whole customer.

Within this string we capture **>95% of spend for >90% of customers** - meaning that in 20 characters we keep every touchpoint tailored to a customer's pet and spend preferences.

**Asymmetric trust between product and customer.** When we tag a *product* with a lifestage, we trust a stack of upstream signals - the catalogue tag, the category path, and a word‑boundary regex over the product description - and let the most restrictive win, so anything reading "junior" beats "adult". When we tag a *customer* with a lifestage, we deliberately throw the catalogue signal away and use only their transactions. A team labelling a generic adult treat as "Junior" is making a marketing perception call, and a marketing perception call shouldn't flip a real household's profile.

Both halves of the asymmetry point at the same outcome: **a non-puppy household should never see puppy recs**. We err inclusively on the product side (any "junior" signal wins, so nothing puppy-shaped slips out as a generic adult product) and strictly on the customer side (we only believe a household has a puppy when their *own* transactions say so). The two errors compose - puppy SKUs only ever reach customers we've actually confirmed are puppy owners. The asymmetry exists to protect the customer experience, not the model's tidy logic.

## Part 2 - Personalisation on the fly

This is the part I find most fun.

At Sainsbury's - where I previously led marketing data science - we never cracked this in production. Product pages were static (no per‑customer reranking), and the personalised "before you go" interstitial was customer-aware but couldn't react to a live basket. What runs at Jollyes does all three at once: anchor SKU, customer fingerprint, and live basket, blended at request time in well under a millisecond.

When a customer hits a product page, we don't have a precomputed "recommendations for this customer × this product". We have a per‑anchor list (from Sunday's Snowflake export) and a per‑customer fingerprint (also from Sunday's export). Everything specific to this customer × this anchor × this basket happens at request time, in memory.

### The blend

At request time, every candidate SKU starts with the anchor product’s pre-ranked also-bought score - the anonymous baseline.

Two additional signals can boost it:

* basket co-occurrence: candidates frequently bought alongside items already in the basket
* loyalty affinity: candidates appearing in the customer’s collaborative-filter top-N

All three signals decay naturally by rank position: appearing first matters much more than appearing twentieth.

The combined score is then multiplied by a fingerprint factor between 1.0 and 1.9, depending on whether the candidate’s category matches one of the customer’s top-5 spend categories and how concentrated their spend is in that category. A department allowlist and lifestage filter sit in front as binary vetoes.

Two properties fall out cleanly. First, the fingerprint multiplier is deliberately multiplicative, not additive: a strong "Dog Adult Wet Food" fingerprint cannot drag Cat Litter onto a Dog Food page, because the candidate must already exist in the anchor’s recall set to begin with. The fingerprint amplifies plausible candidates; it does not create them.

Second, anonymous traffic runs through the exact same arithmetic. With no basket and no loyalty data, the extra terms reduce to zero and the multiplier collapses to ×1, leaving the original ranking unchanged. There is no separate "personalised" code path - same function, different inputs.

### Dogfooding with Claude

You can't unit-test *"does this feel personalised"*. So we validate by using Claude Code with our [stateful MCP server](/stateful-retail-analyst-mcp/) and letting it browse: pick a real loyalty number, pick a real basket, walk through what that customer would see on a handful of product pages, and judge whether the recommendations feel coherent. It surfaces things a deterministic test never would - a senior‑cat household whose second rec is a puppy chew, a basket where the co-occurrence pass tilts the page in a direction that feels wrong with the context of the customer's prior transactions.

## Part 3 - Same fingerprints, sideways: voucher targeting

This is where the encoding pays back a second time.

The marketing team wants to send a voucher to *cat customers who buy food but not litter*. That's a real campaign: customers using us as a top-up for food but going to the supermarket for litter. The hypothesis is that a litter voucher converts them into a full-basket cat customer. Previously, we'd have spent analyst time on Snowflake queries to build the audience.

Instead, the segments API just queries the in‑memory fingerprints sideways:

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

The same fingerprint that says *"this customer's recommendations should lean cat-food‑heavy"* also says *"this customer is a cat-food shopper who has never bought cat litter"* - to a marketing system that has no idea what the recommendations service is.

## Closing

The pattern, in one sentence:

> Do the expensive analytical work once a week, encode the result so tightly that the whole population fits in RAM, and make the runtime a thin layer over `Map` lookups.

The runtime composes fingerprints, static product relationships, and live basket state into request-time personalisation with no online inference.

Each surface of the system - recommendations, segmentation, voucher targeting - is just a different way of reading the same encoded artefact. New use cases have already surfaced by querying the same data sideways.
