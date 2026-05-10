---
layout: post
title: "Fingerprinting 600,000 customers in 20 characters each"
categories: [ai]
tags: [recommendations, personalisation, snowflake, jollyes, retail]
---
*How a tiny per-customer encoding turns weekly basket analysis into real-time personalised recommendations — and now also powers voucher targeting.*

<div class="tldr" markdown="1">
**TL;DR**

- Every active Jollyes loyalty member is compressed into a ~20-character fingerprint that captures their pet's species, lifestage, and top‑5 spend categories. 600k members fit into a single VM in memory.
- We build one-to-one personalisation by mixing signals from the current SKU page, the basket, and the fingerprint — every customer sees a personalised set of recommendations at request time.
- The same fingerprint repurposes cleanly: the marketing team reads it sideways to pick audiences like *"cat customers who buy food but not litter"* — five lines of code, zero new data pipelines.
- Validation runs through Claude over our [MCP server](/stateful-retail-analyst-mcp/) — it samples real customers, real baskets, and real product pages, and reports in natural language whether the experience feels coherent. AI checking AI, with the customer's eyes.
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

One always‑on container, two CSVs, one heartbeat. The infrastructure cost is roughly the same as the [MCP server](/stateful-retail-analyst-mcp/): ~$10/month. There is no database in the request path.

## Part 1 — A customer in 20 characters

A customer fingerprint is a variable-length string of base62 characters, 5 + 3·N where N is the number of category entries (0–5):

```
S J E W  X1Y1T1  X2Y2T2  X3Y3T3  X4Y4T4  X5Y5T5  K
└┬┘ │ │  └──┬─┘                                  │
 │  │ │     │                                    └─ checksum
 │  │ │     └─ "I spend tier T on category XY"  (×5)
 │  │ └─ breadth: distinct (dept, cat) over 180d
 │  └─ senior species mask (Senior/Mature spend last few months)
 │  junior species mask  (Puppy/Kitten/Junior spend last few months)
 species mask  (any spend over 180d, by species)
```

A new cold-start customer who has only ever bought a Christmas dog toy looks like `b0021` (5 chars: Dog species, no junior/senior signal, breadth 2, checksum 1). A multi-pet senior‑cat household with diversified spend looks like `nA4Ax7M9aB7Bc6Cb5gC2y`. That's it. That's a whole customer.

Within this string we capture **>95% of spend for >90% of customers** — meaning that in 20 characters we keep every touchpoint tailored to a customer's pet and spend preferences.

**Asymmetric trust between product and customer.** When we tag a *product* with a lifestage, we trust a stack of upstream signals — the catalogue tag, the category path, and a word‑boundary regex over the product description — and let the most restrictive win, so anything reading "junior" beats "adult". When we tag a *customer* with a lifestage, we deliberately throw the catalogue signal away and use only their transactions. A team labelling a generic adult treat as "Junior" is making a marketing perception call, and a marketing perception call shouldn't flip a real household's profile.

Both halves of the asymmetry point at the same outcome: **a non-puppy household should never see puppy recs**. We err inclusively on the product side (any "junior" signal wins, so nothing puppy-shaped slips out as a generic adult product) and strictly on the customer side (we only believe a household has a puppy when their *own* transactions say so). The two errors compose — puppy SKUs only ever reach customers we've actually confirmed are puppy owners. The asymmetry exists to protect the customer experience, not the model's tidy logic.

## Part 2 — Personalisation on the fly

This is the part I find most fun.

At Sainsbury's — where I previously led marketing data science — we never cracked this in production. Product pages were static (no per‑customer reranking), and the personalised "before you go" interstitial was customer-aware but couldn't react to a live basket. What runs at Jollyes does all three at once: anchor SKU, customer fingerprint, and live basket, blended at request time in well under a millisecond.

When a customer hits a product page, we don't have a precomputed "recommendations for this customer × this product". We have a per‑anchor list (from Sunday's Snowflake export) and a per‑customer fingerprint (also from Sunday's export). Everything specific to this customer × this anchor × this basket happens at request time, in memory.

We deliberately didn't precompute the cross product. ~10k anchors × ~600k customers × every realistic basket state is a number with no useful upper bound; baskets change by the click while fingerprints refresh weekly; and welding the layers together in Snowflake would couple them — the voucher team couldn't query the customer side without dragging the recs side along.

### The blend

For each candidate SKU in the anchor's pre-ranked list, the runtime sums three position-decayed signals — the SKU's rank in the anchor's also‑bought list (the anonymous baseline), its rank in the also‑bought of each item already in the basket (the basket boost), and its rank in the customer's collaborative-filter top‑N (the loyalty boost) — and multiplies that sum by a fingerprint factor between 1.0 and 1.9 driven by whether the candidate's category matches a top‑5 entry and how concentrated the customer's spend is in that category. A department allowlist and a lifestage filter sit in front as binary vetoes.

Two properties fall out cleanly. The multiplier is *multiplicative* on purpose: a strong "Dog Adult Wet Food" fingerprint won't drag Cat Litter onto a Dog Food page, because the candidate has to be in the anchor's also‑bought list to start with — the fingerprint amplifies recall, it doesn't create it. And anonymous calls go through the same arithmetic: both extra terms are zero, the multiplier reduces to ×1, and the input order falls out unchanged. There is no "personalised" code path forking off — same function, different inputs.

The whole personalisation pipeline is ~50 lines of TypeScript. No model inference, no embedding lookup, no remote call at request time. The expensive intelligence — the basket co‑occurrence matrix from >1M baskets, the CF aggregation, the fingerprint encoding — was paid for once, last Sunday, in Snowflake. The runtime just composes precomputed signals with arithmetic.

### Dogfooding with Claude

You can't unit-test *"does this feel personalised"*. So we validate by pointing Claude (via API) at our [stateful MCP server](/stateful-retail-analyst-mcp/) and letting it browse: pick a real loyalty number, pick a real basket, walk through what that customer would see on a handful of product pages, and judge in plain English whether the recommendations feel coherent. It surfaces things a deterministic test never would — a senior‑cat household whose second rec is a puppy chew, a basket where the co-occurrence pass tilts the page in a direction that reads wrong to a human. AI checking AI, on real customers, with the customer's eyes.

## Part 3 — Same fingerprints, sideways: voucher targeting

This is where the encoding pays back a second time.

The marketing team wants to send a voucher to *cat customers who buy food but not litter*. That's a real campaign: customers using us as a top-up for food but going to the supermarket for litter. The hypothesis is that a litter voucher converts them into a full-basket cat customer.

A naïve build would mean:

- A new pipeline against Snowflake to materialise this segment.
- A handshake on freshness ("when did this audience snapshot last run?").
- A new join key, new caching strategy, new ownership boundary.

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

600k iterations of a few bitwise ops and a short loop over ≤5 fingerprint entries. Returns in <100ms. Results are cached per criteria-key per generation; a new fingerprint generation = automatic cache invalidation.

The same fingerprint that says *"this customer's recommendations should lean cat-food‑heavy"* also says *"this customer is a cat-food shopper who has never bought cat litter"* — to a marketing system that has no idea what the recommendations service is.

## Why not a customer embedding?

A 32-dim float32 embedding is 128 bytes per customer; 600k customers is 73 MB before any of the masks, lifestage flags, or category structure that downstream code actually needs. The fingerprint is ~20 bytes and carries the semantic structure on its face. We can read one and know what we're looking at; embeddings are opaque without their model. The marketing team can read a fingerprint. Auditors can read a fingerprint. Neither can read an embedding.

## Closing

The pattern, in one sentence:

> Do the expensive analytical work once a week, encode the result so tightly that the whole population fits in RAM, and make the runtime a thin layer over `Map` lookups.

Each surface of the system — recommendations, segmentation, voucher targeting — is just a different way of reading the same encoded artefact. New use cases land by querying sideways, not by building another pipeline.

The fingerprint isn't clever because it's small. It's clever because smallness is a forcing function on what counts as a signal. Every bit had to argue for its place. Everything that survived genuinely earns its keep.
