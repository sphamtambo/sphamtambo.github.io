---
layout: post
title: "Your Model's AUC Doesn't Tell You When to Trust It"
date: 2026-09-18
---

A model with 0.90 ROC-AUC sounds trustworthy. It isn't automatically. AUC measures one thing: whether the model ranks actives above inactives. It says nothing about whether a predicted probability of 0.9 actually means "90% likely active."

Those are two different properties, and it's easy to have one without the other.

## Ranking vs. believing the number

Take a classifier that outputs 0.99 for almost everything it calls active and 0.01 for almost everything it calls inactive. If the ranking is right, AUC will look great. But if the true hit rate among its "0.99" predictions is actually 60%, not 99%, the model is badly miscalibrated. AUC never sees this, because AUC only cares about order, not magnitude.

For virtual screening, this distinction matters more than it looks. Ranking is what you need to build a shortlist. But the moment you start reading the probability itself as a confidence statement, or using a threshold like "only advance predictions above 0.9," calibration is doing real work, and a high-AUC model can still get that number wrong.

## Calibrating isn't a free fix either

The obvious response is to calibrate the output, Platt scaling, isotonic regression, whatever's on hand. This helps, but it isn't a guarantee, and it comes with a subtlety worth knowing: calibrating a model can make sharpness worse even while making it more honest. A method that widens or narrows probability estimates to better match reality doesn't always improve summary scores like Brier score or expected calibration error, because those scores blend calibration and sharpness together. Improving one can look flat, or even worse, on the other.

This is why I've stopped trusting a single scalar calibration metric to tell the whole story. It's worth checking calibration behavior directly, plotted, stratified, not just one number, before deciding a model is well-behaved.

## What to actually check

Two habits catch most of this early:

- **Plot predicted probability against observed frequency**, don't just report a single calibration score. A reliability diagram makes overconfidence or underconfidence visible in a way Brier score alone won't.
- **Ask what "well calibrated" needs to guarantee.** Distribution-free tools, conformal prediction, Venn-Abers, exist specifically because scalar post-hoc calibration doesn't come with a validity guarantee. They cost you sharpness in exchange for a statement you can actually stand behind.

None of this replaces good discrimination. A model still needs to rank correctly before calibration is worth discussing. But AUC and calibration are answering different questions, and a paper, or a screening pipeline, that only reports AUC hasn't actually told you when to trust an individual prediction.
