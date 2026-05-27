---
layout: post
title: "Embeddings: LLM's best kept secret?"
categories: [ai]
tags: [embeddings, llm, retail, jollyes]
---
*Combining LLMs with traditional AI to solve new problems in minutes for pennies*

<div class="tldr" markdown="1">
**TL;DR**

- Embeddings are a cheap power tool of LLMs - underrated next to the GPT / Claude GenAI headlines.
- You can integrate an LLM into your XGBoost workflow: get an LLM to produce expensive labels on a small sample, embed the full population cheaply and quickly, then let classical ML map between them.
- The most important embeddings to the model are seldom the most important vectors for your use case.
- You can reuse the embeddings across different use cases, picking out different vectors which are important to you.
</div>

Most AI conversations are around the headline models - GPT-5 / Claude - in generative use cases. Embeddings get almost no look in, but they are at the heart of the maths which drive LLMs. Most LLM providers provide incredibly cheap and fast embedding endpoints. I will show here, with two examples, how you can use embeddings with the classical ML toolbox, and unlock problems that used to need a human in the loop or lengthy research.

## Part 1 - Why do people shop?

Motivation: for the 30,000 baskets that went through our tills in the last couple of days, can we tag each one with a *reason for visit* - "puppy starter pack", "everyday food top-up", "frozen raw food", "treats and chews"?

The traditional answer is to either rules-engine it (brittle) or pay a labelling agency (slow and expensive). What we did instead:

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

3. **Re-prompt with a closed taxonomy derived from the free-text answer** - 13 leaf reasons laddering up to 4 top-level categories. The LLM must answer only inside the taxonomy. Output: `DIET_SPECIFIC_FOOD, EVERYDAY_FOOD`. Cost: **£25 for 2,000 baskets, ~4 hours.**

4. **Vector-embed every basket** (all 30,000 of them) into a high-dimensional semantic space. Cost: **£0.05, ~5 minutes.**

5. **Train a classifier** that maps the embeddings to the labels from step 3. ~5 minutes.

6. **Predict labels for the remaining 28,000 baskets** in seconds, at near-zero marginal cost.

The trick is steps 4-6: we pay the LLM for high-quality labels on a small sample, then teach a cheap classical model to imitate it across the whole dataset using the embeddings as features. The embedding model still reads all 30,000 baskets - we just don't pay it to *answer*, only to project each basket into vector space.

**Net result: 30k baskets labelled with a meaningful shopping-reason taxonomy, end to end, for ~£25.55.** The marginal cost for future baskets: near zero.

### Meaning, not just dimensionality

This could have been done solely classically - run PCA on the basket shopping by category and run a k-means clustering analysis. That was my first approach too, but the first principal component fell out as *animal type* (dog / cat / small animal); the second as *food / non-food*.

The embedding-and-label route produces missions instead of a simple descriptor of animal and product type:

- *"weekly main shop with a chew thrown in"*
- *"pure treat run"*
- *"medical shop with an apology treat to make it up to the pet"*

Classical decomposition finds the geometry of the raw data - % food, % cat, basket size. Embeddings give you a *different* geometry, one where the coordinates encode semantic meaning rather than category mix.

## Part 2 - Take the human out of the loop

Our new-store revenue model leans on a footfall score for the proposed retail park - a subjective 1-10 number, currently produced by our property surveyor walking the location and grading it. SHAP on the revenue model shows footfall features are the top two predictors by a clear margin.

The footfall score is the most predictive input *and* the most subjective one. It's the bottleneck on both new-store decisions and on auditing the existing estate. So, the question we asked was: **can an LLM, with no field visit, reproduce it from a postcode alone?**

Pipeline:

```
Ground "truth"   →   Data (LLM-described)   →   Predictions
from a human         from a postcode             (XGBoost on embeddings)
```

1. **Ask an LLM to survey the postcode.**

   ```
   System: "You are an expert in retail surveying..."
   Prompt:  ask, example; ask, example; ask {DE11 9AA}
   Answer (bringing together many sources from a Google search):
            "The retail park comprises approximately 76,000 sq ft
             and is anchored by Morrisons (supermarket food..."
   ```

2. **Clean the text.**
3. **Embed it.**
4. **Train XGBoost** on the embeddings against the human surveyor's score across our existing estate.

This gave us a good correlation between the model's prediction and the human score. Top SHAP-importance features are specific embedding dimensions (`embedding_160`, `embedding_472`, `embedding_329`, ...) - opaque to humans, but encoding the kind of things a surveyor implicitly weights when grading a site.

It's a neat dual use of models. The expensive and human-time-intensive step - scouring the web and generating expert-tone prose about a site - is exactly what LLMs are good at. The cheap and unbiased step - turning that prose into a number a downstream model can use - is the same as classifying baskets. Satisfyingly, the same retail-park embedding can be reused as input to other models entirely: pet preference, catchment demographics, etc.

### Your top dimensions are not the embedding's top dimensions

Embedding dimensions (usually[^matryoshka]) come pre-ordered by the model that produced them, by how much semantic information they carry across the corpus the model was trained on. The intuition most people start with is that the early dimensions are the most useful - reinforced by OpenAI [explicitly telling you](https://openai.com/index/new-embedding-models-and-api-updates/) that you can truncate their embeddings and keep most of the semantic meaning (a `text-embedding-3-large` vector chopped from 3,072 down to 256 dimensions still beats the older `ada-002` at full length).

For *the embedding model*, they are. For *your task*, they often aren't. The top SHAP features on our footfall task are `embedding_160`, `_472`, `_329` - not `embedding_0`, `_1`, `_2`. If we re-use the same retail park embedding on a different downstream feature, a different `embedding_XXX` short-list bubbles up.

In short, an embedding is a compressed snapshot of *meaning in general*; any given task is a specific cross-section through it. One embedding pass gives you many feature spaces - you need to pick the right set for your use case.

## Closing

Two different problems. The same shape for both:

1. **Get an LLM to produce small-sample, expensive-but-high-quality outputs** (closed-taxonomy labels, expert-tone surveys). Expensive in time or agency fees!
2. **Embed the full population cheaply and quickly** - five pence for 30,000 records.
3. **Use the small-sample outputs as labels, the embeddings as features, and a classical model as the glue.**

I'm still surprised you can get high-quality embeddings at such speed and low prices, and I don't hear people talking about it. The classical-ML toolbox has just gotten a new feature space for (almost) free.

Embeddings aren't the headline of the LLM story, but they definitely should be part of the conversation.

[^matryoshka]: If the model was trained with Matryoshka Representation Learning, which is now common but worth checking.
