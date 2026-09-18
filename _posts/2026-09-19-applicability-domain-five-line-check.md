---
layout: post
title: "Applicability Domain: The Check Most QSAR Posts Skip"
date: 2026-09-18
---

A model trained on one chemical space doesn't know it's being asked about a different one. It will still return a number. Nothing in a standard classifier or regressor stops it from confidently scoring a molecule that looks nothing like anything it was trained on.

Applicability domain (AD) is the answer to a narrow but important question: is this query compound actually similar enough to the training data for the model's output to mean anything? It's one of the cheapest reliability checks available, and it's still missing from a lot of QSAR work that otherwise looks careful.

## The simplest version that works

You don't need anything exotic to get a usable AD estimate. A k-nearest-neighbor distance check does most of the job:

1. Featurize the training set the same way you featurize everything else.
2. For each training compound, compute its distance to its k nearest neighbors, excluding itself.
3. Set a threshold from that distribution, mean plus some number of standard deviations is a common, defensible choice.
4. For a new query compound, compute its mean distance to its k nearest training neighbors.
5. Flag it as outside the domain if that distance exceeds the threshold.

That's the whole method. No new model, no extra training step, nothing that requires touching the classifier itself.

## What it actually buys you

Applicability domain isn't a performance metric, and it won't make a bad model good. What it does is give you a place to put a specific kind of doubt: not "how confident is the model," but "has the model ever seen anything like this before."

In my own work, error rates outside the domain are consistently higher than inside it, across every target I've checked. Nobody needs to be told chemistry way outside a model's training distribution is riskier to predict. What AD gives you is a way to say so on a per-compound basis, before you act on a prediction, rather than after you've already been burned by it.

## Where people get it wrong

Two mistakes show up often enough to call out specifically:

- **Fitting the AD reference on the wrong partition.** If the training distribution used to build the AD reference has already seen your held-out test compounds, or your screening library, the domain check is measuring nothing. It needs to be built strictly from the same data the model itself was fit on, and applied unchanged to everything downstream.
- **Treating AD as binary and stopping there.** Inside/outside is a reasonable first cut, but the distance itself is informative too. A compound sitting just past the threshold is a very different situation from one sitting far outside the training cloud entirely.

## The practical takeaway

If a paper reports a great ROC-AUC and never mentions applicability domain, that's worth noticing. It's a five-step calculation that costs almost nothing computationally and directly answers a question ROC-AUC can't: not "does this model discriminate well on the data it was tested on," but "should I trust it on the compound in front of me right now." Ship domain status as a first-class output alongside every prediction, not an afterthought bolted on for the SI.
