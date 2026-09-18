---
layout: post
title: "GPCRs: Drug Discovery's Favorite, Most Frustrating Target"
date: 2026-09-18
---

G protein-coupled receptors sit at the center of an enormous share of human physiology. They mediate neurotransmission, inflammation, immune signaling, pain, cardiovascular control, metabolism, essentially every major extracellular communication system the body has. It's no surprise they've been a persistent focus of drug discovery for decades, and no surprise they remain one of the most heavily pursued therapeutic target families in pharmacology.

It's also no surprise they're difficult to model computationally, and the reasons are worth being specific about rather than waving at "GPCRs are hard."

## The data problem

Public bioactivity data for GPCRs is uneven across receptors, and the unevenness isn't cosmetic. Some targets have tens of thousands of curated measurements, others have a few thousand. Assay endpoints are heterogeneous within a single target, Ki, IC50, EC50, binding versus functional readouts, often pooled together because that's what the available data supports, not because they're pharmacologically interchangeable. Mechanism annotations, agonist versus antagonist versus partial agonist, are frequently missing or too sparse to use as a modelling axis at all.

None of this is a reason to avoid the data. It's a reason to be explicit about what a model trained on it can and can't claim.

## The polypharmacology problem

Many GPCR ligands aren't selective for one receptor. Chemotypes get shared across related receptor subfamilies, sometimes deliberately, sometimes as an artifact of how medicinal chemistry programs explore a pocket. A compound flagged "active" against one receptor in a training set may be genuinely active against a related one too, which complicates both training set construction and the interpretation of a high-confidence prediction against a single target.

## The structural problem

GPCRs aren't static. Active and inactive conformations differ meaningfully at the binding pocket, and a docking protocol built against one receptor state answers a narrower question than "does this compound bind the receptor." It answers "does this compound plausibly fit this specific structure, in this specific state." For some targets, only one state has a usable crystal or cryo-EM structure at all, which limits what structure-based methods can even ask.

## Why this matters for how you read GPCR ML papers

None of these problems are reasons to distrust GPCR modelling work wholesale. They're reasons to expect specific disclosures: which endpoints were pooled and why, whether evaluation used a split that actually stresses analogue-series overlap, whether applicability domain or some other reliability check was applied before a prediction was acted on, and which receptor state a docking result is actually conditional on. A paper that's explicit about these things is telling you something more useful than a single headline accuracy number ever could.
