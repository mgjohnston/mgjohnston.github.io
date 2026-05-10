---
layout: post
title: "Fingerprinting 600,000 customers in 20 characters each"
categories: [ai]
tags: [recommendations, personalisation, snowflake, jollyes, retail]
---
*How a tiny per-customer encoding turns weekly basket analysis into real-time personalised recommendations — and now also powers voucher targeting.*

<div class="tldr" markdown="1">
**TL;DR**

- Every active Jollyes loyalty member is compressed into a ~20-character fingerprint that captures their species, lifestage, and top‑5 spend categories. 600k members fit in ~12 MB on the wire and ~150 MB in memory.
- All the heavy lifting — 1M+ basket co‑occurrence, collaborative filtering, lifestage inference — happens once a week in Snowflake, not on a server. The runtime is a thin Express container doing in‑memory map lookups.
- The same fingerprint repurposes cleanly: the recommendations engine reads it to re-rank products; the voucher team reads it sideways to pick audiences like *"cat customers who buy food but not litter"* — five lines of code, zero new data pipelines.
</div>

> "Show me cat owners who buy food but not litter."

That request used to take a data analyst, an ad-hoc Snowflake query, a CSV handoff, and a day's wait. Today it returns in <100ms from an Express service running on a single Fargate task. The same service answers product recommendation calls from the website. Same data, same code path, same per‑customer encoding.

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

A real cold-start customer who has only ever bought a Christmas dog toy looks like `b0021` (5 chars: Dog species, no junior/senior signal, breadth 2, checksum 1). A multi-pet senior‑cat household with diversified spend looks like `nA4Ax7M9aB7Bc6Cb5gC2y`. That's it. That's the whole customer.

### What each field earns its bits

| Field | Width | Carries |
|---|---|---|
| `S` | 1 char | 6-bit species mask (Dog, Cat, Sm. Animal, Fish, Reptile, Bird) |
| `J` | 1 char | Same 6 bits, but only species the customer recently bought junior food/medical/grooming for |
| `E` | 1 char | Same 6 bits, but senior lifestage |
| `W` | 1 char | Breadth — how many distinct (dept, cat) pairs they touched in 180d |
| `XnYn` | 2 chars | Flat ordinal into the ~307 used (dept, cat, subcat1) triples — opaque on the wire |
| `Tn` | 1 char | Tier 0–9, where 9 = >90% of spend on this category |
| `K` | 1 char | Checksum — tampered rows fail and are dropped at refresh |

The encoding makes hard decisions about what counts. A few examples:

**Why a separate junior/senior mask instead of a per‑category lifestage bit?** Because lifestage is a property of the household, not the basket. A customer who once bought puppy pads in May 2024 shouldn't still be tagged puppy-Dog. The J/E masks use a sliding window — `max(today − 180d, min(today − 90d, date of 5th-most-recent transaction))` — so a weekly shopper gets a tight 3-month window and a quarterly shopper gets the full 6 months. Pups grow up fast.

**Why 2 chars (XY flat) instead of 4 chars (dept + cat + subcat hierarchically)?** Earlier versions wrote each part of the hierarchy as its own char. That saved a layer of indirection at lookup time, but cost 1.8 MB on the wire and ~150 MB in memory across all 600k members. v3 collapses to a flat 2-char ordinal, and the service holds a tiny `xy → (dept, cat)` map in RAM to reconstruct the hierarchy when it needs to do the half-credit fallback boost ("Adult Dry Dog Food" still partially nudges "Adult Wet Dog Food"). The lookup is one `Map.get`. The wire savings paid for the runtime trade.

**Asymmetric trust between candidate-side and customer-side.** When we tag a *product* with a lifestage, we trust three signals: the PIM tag, the subcategory path, and a word-boundary regex on the description. Whichever signal is most restrictive wins — anything saying "junior" beats "adult". When we tag a *customer* with a lifestage, we throw the PIM signal away. Why: a PIM team labeling "Training Treats For Dogs" as Junior is a marketing perception choice and it shouldn't flip the customer's J-mask. The product is conservative about going out; the customer model is conservative about going in. Same shape, deliberately asymmetric trust.

The fingerprint is a forcing function. Every signal we want — species, lifestage, breadth, spend tier per category — has to earn its bit-width. Anything we couldn't justify keeping in <20 chars got cut.

## Part 2 — 1M baskets → 1 MB of recommendations

The fingerprint is the customer side. The other half is the per-SKU side: for each of our ~10k live SKUs, what tends to get bought alongside it? What is similar? What's popular among customers who bought it?

That's a co-occurrence join against >1M historical baskets. It's the kind of analytical query that, run live on every product page, would need a warehouse always running. Instead it runs once a week in Snowflake, on a Sunday morning, and exports two CSV files:

```
File: recommendations_*.csv
Size: ~1 MB
Rows: ~10k
What: one row per anchor SKU — also_bought, similar, popular, dept, cat, subcat, lifestage byte
────────────────────────────────────────
File: loyalty_personalisation_*.csv
Size: ~120 MB
Rows: ~600k
What: one row per active member — top_skus, top_purchased_skus, fingerprint
```

The service streams the 120 MB CSV directly into its in-memory maps at boot — never materialised as an array, never held twice. A failed loyalty fetch is caught and logged; the previous maps stay in place rather than emptying the customer side. A failed recommendations fetch throws and aborts the refresh, leaving prior state untouched. The atomic swap is one tick — every map is either the prior generation or the fresh one, never half-loaded.

To stop everyone seeing the exact same rec for a week, the stored procedure adds a deterministic ~5% jitter on every `(anchor, sku, week)` score: `MOD(ABS(HASH(...)), 100) / 2000.0`. Stable within a week, fresh between weeks, and the same across the herd so caching still works.

## Part 3 — Personalisation on the fly

This is the part I find most fun.

When a customer hits a product page, we don't have a precomputed "recommendations for this customer × this product". We have a per-anchor list (from Sunday's Snowflake export) and a per-customer fingerprint (also from Sunday's export). Everything that is specific to this customer × this anchor × this basket happens at request time, in memory, in well under a millisecond.

We could have gone the other way — precompute the cross product nightly. We deliberately didn't:

- **The cross product is huge.** ~10k anchors × ~600k customers × every realistic basket state = a number with no useful upper bound. We'd compute most of it for customers who never visit the page.
- **Baskets are intra-session.** A customer can add an item, remove it, swap it. The fingerprint refreshes weekly; the basket changes by the click. Anything that responds to basket has to live at request time.
- **It would couple the layers.** The per-anchor list and the per-customer fingerprint are independently meaningful and independently usable. Welding them together in Snowflake would mean the voucher team can't query the customer side without dragging the recs side along.

So instead the runtime takes one anchor list, one fingerprint, the live basket, and blends them with arithmetic. It's the same code path whether the customer is logged in or not — the absent signals just contribute zero.

### The four cases

A customer landing on a Beef Dog Food Tin product page (anchor SKU 52810) hits the same code path. What's different is which signals are populated:

| Case | Returned |
|---|---|
| Anonymous, no basket | Items 1–5 of the anchor's pre-ranked also-bought list. No request-time work beyond a `Map.get` and a slice. |
| Anonymous + basket | Anchor list re-ranked by basket co-occurrence — items that frequently co-occur with what's already in the basket get boosted. |
| Logged in, no basket | Anchor list re-ranked by the customer's CF top‑N membership and fingerprint category match. |
| Logged in + basket | Both layers compose. |

### The blending equation

For each candidate `sku` in the anchor's list (positions 0…19, capped at 20):

```
score(sku) =  [ pos(rank_in_anchor)                                ← anonymous baseline
              + Σ_{b ∈ basket}  pos(rank_in_alsoBought(b, sku))    ← basket boost (basket-bearing only)
              + 𝟙[sku ∈ topN]  · pos(rank_in_topN(sku))            ← CF boost (loyalty-bearing only)
              ]
              × ( 1 + fpFactor(sku) )                              ← spend-share multiplier (loyalty-bearing only)

pos(r) = 1 / (1 + r)
```

where the fingerprint factor is the most opinionated piece:

```
fpFactor(sku) =
  ( T + 1 ) / 10           if sku.xy_code is an exact match in fingerprint        ← same (dept, cat, subcat1)
  ( T + 1 ) / 20           elif sku.dept_cat_code matches a fingerprint entry     ← same (dept, cat)
  0                        otherwise

then × 0.5  if sku.species ≠ customer's dominant species                          ← multi-species dampening
```

`T` is the fingerprint tier 0–9 of the matched entry (≈ how concentrated the customer's spend is in that category). All of `pos`, `xy_code`, `dept_cat_code`, dominant species — every one of these is a `Map.get` on a structure built at boot.

A few properties fall out cleanly:

- **Position is the natural decay.** A customer's #1 CF affinity boosts a candidate by 1.0; their #20 affinity by 0.05. We never had to choose a separate "topN weight schedule" — the inverse-rank is the schedule. Same for basket co-occurrence: being the first also-bought of a basket item matters; being the 17th does not.
- **The multiplier is multiplicative on purpose.** A high-tier fingerprint match doesn't *create* a candidate, it *amplifies* one that the basket and CF layers already think is plausible. A customer with a strong "Dog Adult Wet Food" fingerprint won't see Cat Litter on a Dog Food PDP, no matter how high the tier — the candidate has to be in the anchor's also-bought list to start with. The multiplier respects the recall set.
- **Anonymous calls go through the same arithmetic.** Both extra terms are zero, the multiplier reduces to ×1, and we get the input order back. There's no "personalised" code path forking off — it's the same function with default inputs.

A department allowlist and a lifestage filter sit in front of the scoring step. They don't add or subtract score — they binary-veto. A puppy pad on a Dog Food anchor will be dropped before scoring if the customer's J‑mask doesn't include Dog; an Adult Dog candidate with `S = Dog` will pass.

### Live example: SKU 52810 (Beef Dog Food Tin, 12‑pack)

The dev recs API returns this anonymous top‑10 for `also-bought/52810` (real call, real ordering):

| rank | SKU | item | pos(rank) |
|---|---|---|---|
| 1 | 68321 | Dog Biscuit Selection | 1.000 |
| 2 | 50075 | Jumbone Large Beef | 0.500 |
| 3 | 22376 | Bakers Joint Delicious Large | 0.333 |
| 4 | 14099 | J/S E/Dose Wormer | 0.250 |
| 5 | 53422 | Roast Whole Bone (Jurassic) | 0.200 |
| 6 | 66912 | Free Flow Duck | 0.143 |
| 7 | 68580 | Pet Pillow Mattress | 0.125 |
| 8 | 52606 | Rope Ring | 0.111 |
| 9 | 45811 | Monster Munch Dog Treat | 0.100 |
| 10 | 68587 | Puppy Pads | 0.091 |

A logged-out browser sees the first 5 of this list. End of story for them.

Now suppose the customer logs in. They are a one-dog adult-only household with a fingerprint that resolves to:

```
S = Dog                                         (species mask: Dog only)
J = 0,  E = 0                                   (no junior or senior spend)
top categories: { (Dog, Treat, Biscuit, T=4),
                  (Dog, Medical, Wormer, T=2) }
CF top-N includes: 14099 at rank #2, 50075 at rank #4
```

Re-ranking proceeds in three steps:

**1. Filter.** 68587 (Puppy Pads, `lifestage_byte = junior`, species = Dog) is suppressed — customer's J‑mask has no Dog bit. Knocked out before scoring.

**2. Score.** Each surviving candidate gets `pos(rank) + topN boost`, then multiplied by `(1 + fpFactor)`:

| rank | SKU | item | pos | + topN | (1 + fpFactor) | final |
|---|---|---|---|---|---|---|
| 1 | 68321 | Dog Biscuit Selection | 1.000 | — | ×1.50 (Treat exact, T=4) | **1.500** |
| 2 | 50075 | Jumbone Large Beef | 0.500 | +0.250 (topN #4) | ×1.50 (Treat exact, T=4) | **1.125** |
| 3 | 22376 | Bakers Joint Delicious | 0.333 | — | ×1.10 (dept‑cat fallback, T=1) | 0.367 |
| 4 | 14099 | J/S E/Dose Wormer | 0.250 | +0.500 (topN #2) | ×1.30 (Medical exact, T=2) | **0.975** |
| 5 | 53422 | Roast Whole Bone | 0.200 | — | ×1.50 (Treat exact, T=4) | 0.300 |
| 6 | 66912 | Free Flow Duck | 0.143 | — | ×1.50 (Treat exact, T=4) | 0.214 |
| 7 | 68580 | Pet Pillow Mattress | 0.125 | — | ×1.00 (no fp match) | 0.125 |
| 8 | 52606 | Rope Ring | 0.111 | — | ×1.00 (no fp match) | 0.111 |
| 9 | 45811 | Monster Munch | 0.100 | — | ×1.50 (Treat exact, T=4) | 0.150 |
| 10 | 68587 | Puppy Pads | — | — | *filtered* | — |

**3. Sort.** New top 5:

1. **68321 Dog Biscuit Selection** (1.500) — was #1, holds. Strong fingerprint match.
2. **50075 Jumbone Large Beef** (1.125) — was #2, holds. CF + fingerprint stack.
3. **14099 J/S E/Dose Wormer** (0.975) — was #4, jumped to #3. Strong CF affinity (their personal #2) outweighs the rank gap.
4. **22376 Bakers Joint Delicious** (0.367) — was #3, drops to #4. Only a half-credit (dept, cat) fingerprint match.
5. **53422 Roast Whole Bone** (0.300) — was #5, holds.

The shape of the list is the same — the customer is still seeing dog accessories that go with adult dog food — but the ordering reflects what this customer actually buys. The wormer, which the anonymous list ranks middle-of-the-pack, climbs because it's something this customer reorders frequently.

Now suppose they also have salmon oil (62482) in their basket. The basket co-occurrence pass adds, for every item in 62482's also-bought list, `pos(rank_in_that_list)` to its score. The dev API confirms what falls out — the candidate pool expands from 10 items to 16, picking up things like 65661 Tennis Balls and 63147 Flashing Safety Light that weren't in 52810's list at all but co-occur with salmon oil. The top of the list stays stable (the baseline co-occurrence with the anchor is too strong to be flipped by one basket item) but the long tail genuinely changes shape.

That's the whole personalisation pipeline. Three additive layers, one multiplicative layer, two binary filters. No model inference, no embedding lookup, no remote call. The expensive intelligence — the basket co-occurrence matrix from >1M baskets, the CF aggregation, the fingerprint encoding — was paid for once, last Sunday, in Snowflake. The runtime just composes precomputed signals with arithmetic.

### Why this composition works

- **Each layer is independently meaningful and independently switchable.** Anonymous = baseline only. Add basket = + co-occurrence. Add loyalty = + CF + fingerprint. Each layer's contribution is auditable on its own; together they compose into a coherent re-ranking without any of them dominating.
- **The output is a stable transform of the input.** Empty basket, empty loyalty → exact input order. Same code path whether the customer is logged in or not. There is no "personalised" branch to maintain in parallel.
- **Cost is bounded by the anchor list size, not the catalogue.** We score 20 candidates, not 10k. Fingerprint multiplier is one `Map.get` per candidate. Basket boost is `O(|basket| × 20)` map lookups. Even a fully-populated request is hundreds of microseconds.
- **Fingerprint stays the bottleneck on personalisation strength.** A customer with a 5-char header-only fingerprint (cold start) gets zero multiplier, and the runtime gracefully falls back to anonymous + basket. We never need a "what if we know nothing" code path — we are always one degenerate fingerprint away from the anonymous baseline.

The whole thing is ~50 lines of TypeScript in `personaliseList`. The conceptual budget — "what is request-time allowed to do?" — was the real constraint. Once we agreed it could only do arithmetic over precomputed maps, the implementation wrote itself.

## Part 4 — Same fingerprints, sideways: voucher targeting

This is where the encoding pays back a second time.

The marketing team wants to send a voucher to *cat customers who buy food but not litter*. That's a real campaign: customers using us as a top-up for food but going to the supermarket for litter. The hypothesis is that a litter voucher converts them into a full-basket cat customer.

A naïve build would mean:

- A new pipeline against Snowflake to materialise this segment.
- A handshake on freshness ("when did this audience snapshot last run?").
- A new join key, new caching strategy, new ownership boundary.

Instead, the segments API just queries the in-memory fingerprints sideways:

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

600k iterations of a few bitwise ops and a short loop over ≤5 fingerprint entries. Returns in <100ms. Results are cached per criteria-key per generation; new fingerprint generation = automatic cache invalidation.

### What the filter language actually expresses

| Filter | Meaning |
|---|---|
| `CONTAINS code` (default tiers) | Customer's top‑5 includes this category |
| `CONTAINS code, minTier: 5` | …and they spend ≥50% on it |
| `CONTAINS code, includeAbsent: true` | …or it's not in their top‑5 (cold-start counts) |
| `NOT_CONTAINS code` | Category is absent from their top‑5 |
| `NOT_CONTAINS code, maxTier: 4` | Either absent, or present but <50% — i.e. they don't over-index on it |

The same fingerprint that says *"this customer's recommendations should lean cat-food‑heavy"* also says *"this customer is a cat-food shopper who has never bought cat litter"* — to a marketing system that has no idea what the recommendations service is.

## Why not X?

**Why not real-time CF on a vector DB?** Because the queries we need don't have <p99 latency budget — they have <p99 latency budget *and* <$10/month infra budget, and they need to run with no DBA on call. The fingerprint accepts a 1-week staleness ceiling in exchange for a memory-only request path that needs no scaling, no warm-up, and no on-call rotation.

**Why not store a full embedding per customer?** A 32-dim float32 embedding is 128 bytes per customer; 600k customers is 73 MB before any of the masks, lifestage flags, or category structure that downstream code actually needs. The fingerprint is ~20 bytes and carries the semantic structure on its face. We can read one and know what we're looking at; embeddings are opaque without their model.

**Why not just query Snowflake directly each request?** The website does hundreds of recs calls per minute at peak. The marketing tool sometimes scans the whole base in one call. Routing that through Snowflake means an always-on warehouse and per-query cost. Doing it in-memory on a Fargate task is two orders of magnitude cheaper and faster.

**Why not Skills / a model-side encoding?** The fingerprint isn't for a model — it's for a deterministic, auditable service. Marketing needs to be able to read a fingerprint and know what it says. Auditors need to know what data we hold per customer. A 20-char string with a documented field layout passes both bars; an embedding doesn't.

## Closing

The pattern, in one sentence:

> Do the expensive analytical work once a week, encode the result so tightly that the whole population fits in RAM, and make the runtime a thin layer over `Map` lookups.

Each surface of the system — recommendations, segmentation, voucher targeting — is just a different way of reading the same encoded artefact. New use cases land by querying sideways, not by building another pipeline.

The fingerprint isn't clever because it's small. It's clever because smallness is a forcing function on what counts as a signal. Every bit had to argue for its place. Everything that survived genuinely earns its keep.

## Open questions we're talking about

- We have 6 species bits and one spare. Do we use it for a 7th (Equine? Insect?) or for a "ever bought" vs "currently buying" distinction?
- Tier is 1 digit (0–9, ~10% buckets). Stretching to 2 digits gives finer resolution at +20% wire cost. Do voucher campaigns care about the difference between 50%-of-spend and 55%-of-spend?
- Should voucher redemptions feed back into the next week's fingerprint, or stay separate from organic spend? (A redeemed voucher shifts the tier; we currently don't down-weight it.)
- Lifestage masks today are junior/senior. Do we want a "puppy who is becoming a junior dog" transitional state, or is the existing junior bit enough?
- Cold-start customers (5-char header-only fingerprints) get no boost from the category multiplier. Is there a useful default for them, or is the right answer "leave the anonymous baseline alone"?
- The category codes live in Snowflake and are matched into the runtime by code. Should they live as a third, tiny CSV alongside the recs and loyalty files? (Would let the service render category names without consulting the SKU attrs map.)
