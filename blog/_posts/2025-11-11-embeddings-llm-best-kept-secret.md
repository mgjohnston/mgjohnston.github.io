---
layout: post
title: "Embeddings: LLM's best kept secret?"
categories: [ai]
tags: [embeddings, llm, retail, jollyes]
---
*Three retail problems solved with embeddings - total spend ~£30, total time an afternoon.*

<div class="tldr" markdown="1">
**TL;DR**

- Embeddings are the cheap power tool of applied AI - underrated next to the GPT / Claude headline acts. 30,000 baskets vector-embedded for five pence in five minutes.
- We used them at Jollyes for three things in three weeks: labelling 30k baskets by motivation, tracking what those motivations turn into a year later, and replacing a human surveyor for new-store footfall scoring.
- The recipe each time: get an LLM to produce high-quality labels on a small sample, embed the full population cheaply, then let classical ML map between them.
</div>

A lot of the AI conversation lands on the headline model - what GPT-5 / Claude / etc can or can't do. Embeddings get less airtime, but they are the workhorse that quietly does a lot of the work in production. They're cheap, they compose well with the classical ML toolbox of the last decade, and they unlock problems that used to need a human in the loop.

Three places we put them to work at Jollyes recently.

## Part 1 - Why do people shop?

Question: of the 30,000 baskets that went through our tills last week, can we tag each one with a *reason for visit* - "puppy starter pack", "everyday food top-up", "frozen raw food", "treats and chews"?

The traditional answer is to either rules-engine it (brittle, and falls apart the moment promotions change) or pay a labelling agency (slow and expensive). What we did instead:

1. **Cast each basket as a string an LLM can read.**

   ```
   { quantity one, name gourmet perle g chef col, department cat,
     category food, subcategory prem.wet, subcategory adult,
     brand gourmet, promo no, size small }
   { quantity two, name cat litter 30ltr, department cat,
     category litter, subcategory wood, brand jollyes,
     promo yes, size big }
   ```

2. **Ask an LLM, in free text, why this customer shopped.** Output is messy but informative: *"new pet, dog treats and food, new puppy, dog care essentials"*. Cost: **£0.50 for 100 baskets, ~2 minutes.**

3. **Re-prompt with a closed taxonomy** - 13 leaf reasons laddering up to 4 top-level categories. The LLM must answer only inside the taxonomy. Output: `DIET_SPECIFIC_FOOD, EVERYDAY_FOOD`. Cost: **£25 for 2,000 baskets, ~4 hours.**

4. **Vector-embed every basket** (all 30,000 of them) into a high-dimensional semantic space. Cost: **£0.05, ~5 minutes.**

5. **Train a small classifier** that maps the embeddings to the labels from step 3. ~5 minutes.

6. **Predict labels for the remaining 28,000 baskets** in seconds, at near-zero marginal cost.

The trick is steps 4–6: we pay the LLM for high-quality labels on a small sample, then teach a cheap model to imitate it across the whole dataset using the embeddings as features. The LLM never sees the other 28,000 baskets. It does what it's expensive at (deep reading) on a small sample; the embeddings do what they're cheap at (similarity) across the population.

**Net result: 30k baskets labelled with a meaningful shopping-reason taxonomy, end to end, for ~£25.55.** A labelling agency wouldn't even quote you for that.

## Part 2 - What do people do next?

Once every basket has a reason-for-visit, the much more interesting question is whether the *first* basket predicts what happens over the next year.

Tracking 20,000 customers across 240,000 baskets in their first 365 days:

| Reason for first visit | Sub-reason | Share of first baskets | Mean first-basket value | Stickiness (returned within 90d) | LTV (first 365d) |
|---|---|---|---|---|---|
| Food | Everyday food | 26% | £23 | 58% | £151 |
| Food | Frozen raw food | 5% | £24 | **67%** | **£235** |
| Non-food | Toys | 7% | £23 | **45%** | £106 |
| Non-food | Treats and chews | 27% | £22 | 55% | £138 |

Two stories jump out. **First-basket value is roughly the same across reasons** (£22–£24) - the till receipt does not tell you who is about to be a great customer. But **stickiness and LTV diverge sharply**. A customer whose first visit is frozen raw food returns within 90 days 67% of the time and is worth £235 in their first year. A customer whose first visit is toys returns 45% of the time and is worth £106. Same basket size at the till, ~2.2× difference in first-year value.

The cohort dynamics get more interesting when you compare *first visit* against *all subsequent visits*:

| Reason for visit | First visit share | Future visits share | Enrichment |
|---|---|---|---|
| FOOD / EVERYDAY_FOOD | 26% | 35% | 0.73× |
| NEW PET / PET_SETUP_ESSENTIALS | 3% | 1% | 5.54× |
| NEW PET / PUPPY_OR_KITTEN_FOOD | 3% | 1% | 1.88× |
| NON-FOOD / ACCESSORIES_AND_KIT | 11% | 5% | 2.12× |
| SERVICE / VET_CARE | 4% | 0% | 8.76× |

Read the **Enrichment** column as "how over-represented this reason is in *first* visits versus *all* visits". The big enrichments name what brings people through the door for the first time but stops appearing afterwards: new-pet kit (5.54×), vet care (8.76×), accessories (2.12×). The under-1 enrichment on Everyday Food (0.73×) tells you the opposite - it's under-represented in first visits, over-represented in repeat ones.

> Customers come for their new pets, accessories, and vets. They stay for the food.

That's a one-line strategy statement derived from an embedding-powered labelling job that cost less than a takeaway.

## Part 3 - Take the human out of the loop

Our new-store revenue model leans on a footfall score for the proposed retail park - a subjective 1–10 number, currently produced by our property surveyor walking the location and grading it. SHAP on the revenue model shows footfall features are the top two predictors by a clear margin.

The footfall score is the most predictive input *and* the most subjective one. It's the bottleneck on both new-store decisions and on auditing the existing estate. So the question we asked was: **can an LLM, with no field visit, reproduce it from a postcode alone?**

Pipeline:

```
Ground "truth"   →   Data (LLM-described)   →   Predictions
from a human         from a postcode             (XGBoost on embeddings)
```

1. **Ask an LLM to survey the postcode.**

   ```
   System: "You are an expert in retail surveying..."
   Prompt: ask, example; ask, example; ask {DE11 9AA}
   Answer:  "The retail park comprises approximately 76,000 sq ft
             and is anchored by Morrisons (supermarket food..."
   ```

2. **Clean the text.**
3. **Embed it.**
4. **Train XGBoost** on the embeddings against the human surveyor's score across our existing estate.

The result: a reasonable correlation between the model's prediction and the human score. Top SHAP-importance features are specific embedding dimensions (`embedding_160`, `embedding_472`, `embedding_329`, ...) - opaque in isolation, but evidently encoding the kind of things a surveyor implicitly weights when grading a site.

This isn't a victory over the surveyor - it's a chance to scale them. They now spend time on borderline sites and on auditing the model's outliers, instead of on every postcode in the country. The bottleneck became a sampling target.

## The pattern

Three different problems. The same shape each time:

1. **Get an LLM to produce small-sample, expensive-but-high-quality outputs** (free-text reasons, closed-taxonomy labels, expert-tone surveys).
2. **Embed the full population cheaply** - five pence for 30,000 records.
3. **Use the small-sample outputs as labels, the embeddings as features, and a classical model as the glue.**

Steps 1 and 3 are well-trodden. Step 2 is what still surprises people: a 30,000-row embedding job costs five pence. The classical-ML toolbox of the last decade gets a new feature space for free, and a lot of the analytical questions you used to delegate to a labelling agency become a £20 internal job.

Embeddings aren't the headline of the LLM story. They might be the most useful part of it.
