---
layout: post
title: "Embeddings: LLM's best kept secret?"
categories: [ai]
tags: [embeddings, llm, retail, jollyes]
---
*Two retail problems solved with embeddings - total spend under £30, total time an afternoon.*

<div class="tldr" markdown="1">
**TL;DR**

- Embeddings are the cheap power tool of applied AI - underrated next to the GPT / Claude headline acts. 30,000 baskets vector-embedded for five pence in five minutes.
- Two Jollyes use cases: classifying baskets into human-readable shopping *modes* (the kind classical PCA can't surface), and replacing a human surveyor for new-store footfall scoring from a postcode alone.
- The recipe each time: get an LLM to produce expensive labels on a small sample, embed the full population cheaply, then let classical ML map between them. Embed once, query many.
</div>

A lot of the AI conversation lands on the headline model - what GPT-5 / Claude / etc can or can't do. Embeddings get less airtime, but they are the workhorse that quietly does a lot of the work in production. They're cheap, they compose well with the classical ML toolbox of the last decade, and they unlock problems that used to need a human in the loop.

Two places we put them to work at Jollyes recently.

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

### Meaning, not just dimensionality

Here's the part that matters more than the price tag.

We could have come at this classically - run PCA on the basket vectors, find clusters, write up the principal components. We did. The first principal component fell out as *animal type* (dog / cat / small animal); the second as *food / non-food*. Both are already in the basket header. PCA recovered the column names and called it insight.

The embedding-and-label route produces something different in kind. The labels it surfaces are shopping *modes* a human would actually recognise:

- *"weekly main shop with a chew thrown in"*
- *"pure treat run"*
- *"medical shop with an apology treat to make it up to the pet"*

These aren't orthogonal axes - they're stories about what the basket was *for*. A merchandiser can act on them. A category manager can plan a promotion against them. A marketing team can write a voucher around them. PCA1 = "dog" is just a tautology of how the basket was filled out.

The wider point: classical decomposition finds the geometry of the data; an LLM finds the *meaning* humans put into it. Embeddings are how you bridge the two.

## Part 2 - Take the human out of the loop

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

### Your top dimensions are not the embedding's top dimensions

A subtlety worth flagging.

Embedding dimensions come pre-ordered by the model that produced them, roughly by how much semantic information they carry across the corpus the model was trained on. The intuition most people start with is that the early dimensions are the most useful.

For *the embedding model*, they are. For *your task*, they often aren't. The top SHAP features on our footfall task are `embedding_160`, `_472`, `_329` - not `embedding_0`, `_1`, `_2`. If we re-use the same population embedding on a different downstream feature - dwell time, basket size, demographic profile of the catchment - a different `embedding_XXX` short-list bubbles up each time.

The implication: an embedding is a compressed snapshot of *meaning in general*; any given task is a specific cross-section through it. One embedding pass gives you many feature spaces. **Embed once, query many.**

## The pattern

Two different problems. The same shape each time:

1. **Get an LLM to produce small-sample, expensive-but-high-quality outputs** (closed-taxonomy labels, expert-tone surveys).
2. **Embed the full population cheaply** - five pence for 30,000 records.
3. **Use the small-sample outputs as labels, the embeddings as features, and a classical model as the glue.**

Steps 1 and 3 are well-trodden. Step 2 is what still surprises people: a 30,000-row embedding job costs five pence. The classical-ML toolbox of the last decade gets a new feature space for free, and a lot of the analytical questions you used to delegate to a labelling agency become a £20 internal job.

Embeddings aren't the headline of the LLM story. They might be the most useful part of it.
