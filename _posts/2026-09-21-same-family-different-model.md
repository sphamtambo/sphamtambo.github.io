---
layout: post
title: "Same Receptor Family, Different Model"
date: 2026-09-18
image: /assets/zeroshot-transfer-vs-baseline.png
---

It's tempting to assume that if two receptors belong to the same family, a model trained on one should say something useful about the other. They share a fold, they're often screened against overlapping chemical libraries, and a lot of medicinal chemistry intuition carries across the family in practice. None of that guarantees a machine learning model trained on one target transfers to another, even a close relative.

## What the model actually learned

A classifier trained on one receptor's activity data isn't learning "what makes a good ligand for this receptor family." It's learning the specific structure-activity relationships present in that target's own training set, its own pocket geometry, its own chemotype coverage, its own assay conventions. Two receptors can share a fold and still have meaningfully different pockets, different key anchor residues, and different chemical series represented in the public data that trained each model.

Family membership is a hint about shared biology. It isn't a guarantee about shared feature-space geometry, and a model's decision boundary lives in feature space, not in evolutionary relatedness.

## What zero-shot transfer actually tests

The clean way to check this, rather than assume it, is a direct zero-shot experiment: take a model trained on one target, apply it without any retraining to a different target's compounds, and compare that against a model trained specifically on the destination target's own data. If the transferred model matches or beats the destination-specific one, family-level transfer is real for that pair. If it doesn't, the destination target needed its own model all along.

I ran exactly this across a panel of five pharmacologically distinct GPCR targets, 20 directed source-to-destination pairs in total. Mean transfer ROC-AUC across all 20 pairs was 0.585. Mean ROC-AUC for the destination-specific models, trained on their own target's data, was 0.903. Not one of the 20 directed transfers outperformed the corresponding destination-specific baseline.

![Cross-target transfer versus destination-specific model performance](/assets/zeroshot-transfer-vs-baseline.png)

Zero of twenty. Every single directed transfer lost to a model that had simply been trained on the target it was actually being asked to predict.

## The practical takeaway

Treat family membership as a reason to expect some shared chemistry, not as a reason to skip target-specific validation. If you're building models across a receptor family, budget for training and validating each target on its own data, and if you want to claim cross-target transfer works, test it directly with a proper zero-shot comparison against a destination-specific baseline. Don't infer it from the fact that the receptors happen to be related. In this panel, relatedness didn't buy a single one of the 20 pairs anything.
