---
layout: post
title: "Docking Scores Aren't a Classifier"
date: 2026-09-18
---

A docking score looks like a prediction. It's a number, it ranks compounds, and a better score feels like it should mean a more likely binder. None of that makes it a classifier, and treating it like one is one of the more common ways a virtual screening pipeline quietly overclaims what it actually found.

## What a docking score is actually estimating

A scoring function is a fast, approximate estimate of binding pose quality under a fixed protocol: a chosen receptor structure, a chosen pocket definition, a search algorithm that samples poses and picks a winner. It is not a measurement of binding affinity, and it is not trained the way a classifier is trained on labelled actives and inactives for your specific target.

Three separate things get bundled into "docking works" that are worth pulling apart:

- **Redocking validation** asks whether the protocol can reproduce a known crystallographic pose for a receptor's own reference ligand. This tells you the box, the receptor prep, and the search settings are reasonable.
- **Enrichment validation** asks whether the score ranks experimentally labelled actives above inactives, across the same protocol. This tells you whether the score has any discriminative power at all for this target.
- **Rank recovery** asks whether a known good compound scores well relative to everything else in a real candidate set, not just relative to a small labelled benchmark.

Passing the first does not imply passing the second or third. A protocol can reproduce a crystal pose almost perfectly and still show enrichment barely better than random, because pose accuracy and ranking power are genuinely different properties of the same scoring function.

## Enrichment is usually weaker than people expect

When people actually check enrichment power on a real target with labelled actives and inactives, it's common to see ROC-AUC in the modest range, clearly better than chance for some targets, close to chance for others, and rarely as strong as a trained ligand-based classifier on the same task. This isn't a sign the docking was done badly. It's a structural property of scoring functions: they weren't fit to your specific target's activity data the way a QSAR model was.

This also means early-recognition metrics like enrichment factor at very early cutoffs can be actively misleading at realistic benchmark sizes. If your validation set has around a hundred compounds, the top 1% is one molecule, and a metric built on one molecule isn't a metric, it's a coin flip with extra steps.

## What docking is actually good for

None of this means docking is useless. It means the honest framing is narrower than "docking finds binders." Docking is a structural plausibility filter: it asks whether a pose makes physically and chemically sensible contacts in a real binding site, using structural information a ligand-based classifier never sees at all. That's a genuinely different, complementary source of evidence, not a replacement for one.

The practical version of this: use a validated ligand-based model to do the ranking your data actually supports, and use docking afterward to ask a narrower, structural question about the survivors, not the other way around. A docking score is evidence a pose is plausible. It is not evidence a compound is active, and it was never trained to be.
