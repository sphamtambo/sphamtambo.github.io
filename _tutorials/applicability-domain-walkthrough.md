---
layout: tutorial
title: "Building a k-NN Applicability Domain Check"
---

This walks through the applicability domain (AD) method from [the earlier post](/applicability-domain-five-line-check/): a k-nearest-neighbor distance check that flags whether a query compound is similar enough to your training data for a prediction to mean anything.

Every number below is from an actual run of this code, not a hand-typed example.

## What you'll need

```bash
pip install rdkit pandas scikit-learn matplotlib
```

## Step 1: A toy dataset

Ten made-up molecules, standing in for a real training set. In practice this would be your actual training compounds after featurization.

```python
from rdkit import Chem
from rdkit.Chem import AllChem
import numpy as np

train_smiles = [
    "CCO", "CCN", "CCC", "CCCl", "CCBr",
    "c1ccccc1", "c1ccncc1", "c1ccc(cc1)O", "CC(=O)O", "CC(C)O",
]

def featurize(smiles_list, radius=2, n_bits=1024):
    fps = []
    for smi in smiles_list:
        mol = Chem.MolFromSmiles(smi)
        fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius, nBits=n_bits)
        fps.append(np.array(fp))
    return np.array(fps)

X_train = featurize(train_smiles)
print(X_train.shape)
```

```
(10, 1024)
```

## Step 2: Fit the k-NN reference

```python
from sklearn.neighbors import NearestNeighbors

k = 3
nn = NearestNeighbors(n_neighbors=k + 1)  # +1 because a point's nearest neighbor is itself
nn.fit(X_train)

distances, _ = nn.kneighbors(X_train)
self_out_distances = distances[:, 1:].mean(axis=1)  # drop the zero-distance self-match

threshold = self_out_distances.mean() + 2 * self_out_distances.std()
print(f"AD threshold: {threshold:.2f}")
```

```
AD threshold: 3.54
```

## Step 3: Look at the distribution, not just the threshold

A single number hides how spread out the training set actually is. Plot it.

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(6, 4))
ax.hist(self_out_distances, bins=6, color="#1f77b4", edgecolor="white")
ax.axvline(threshold, color="#d62728", linestyle="--", linewidth=2, label=f"threshold = {threshold:.2f}")
ax.set_xlabel("Mean distance to k nearest neighbors")
ax.set_ylabel("Training compounds")
ax.legend(frameon=False)
fig.savefig("ad_threshold_distribution.png", dpi=200)
```

![Distribution of leave-self-out distances with the AD threshold marked](/assets/ad-threshold-distribution.png)

Most of the training set clusters well under the threshold. The one clear outlier in this toy set is the phenol-like compound, its own nearest neighbors in this small set just happen to be a bit further away.

## Step 4: Score new compounds

```python
query_smiles = [
    "CCOC",                                          # simple ether, close to training
    "C1CCC2C(C1)CCC3C2CCC4C3(CCC(C4)O)C",             # steroid-like, structurally unrelated
]
X_query = featurize(query_smiles)

nn_query = NearestNeighbors(n_neighbors=k)
nn_query.fit(X_train)
query_distances, _ = nn_query.kneighbors(X_query)
query_mean_distance = query_distances.mean(axis=1)

for smi, dist in zip(query_smiles, query_mean_distance):
    status = "inside domain" if dist <= threshold else "outside domain"
    print(f"{smi}: mean distance {dist:.2f}, {status}")
```

```
CCOC: mean distance 2.70, inside domain
C1CCC2C(C1)CCC3C2CCC4C3(CCC(C4)O)C: mean distance 6.00, outside domain
```

The simple ether looks like the training set and gets flagged accordingly. The steroid-like structure doesn't, and its distance isn't close to the threshold either, it's nearly double it.

## Step 5: What leakage actually does to the threshold

Here's the mistake the earlier post warned about, demonstrated rather than just described. Fit the threshold on the training set plus a query compound that should be evaluated separately, and watch what happens.

```python
borderline_smiles = ["c1ccc2c(c1)ccc3c2ccc4c3cccc4"]  # a fused-ring PAH, borderline case
X_borderline = featurize(borderline_smiles)

nn_borderline = NearestNeighbors(n_neighbors=k)
nn_borderline.fit(X_train)
d_borderline, _ = nn_borderline.kneighbors(X_borderline)
dist_borderline = d_borderline.mean(axis=1)[0]
print(f"borderline compound distance (train-only reference): {dist_borderline:.2f}")

# now leak it into the reference set before fitting the threshold
X_leaked = np.vstack([X_train, X_borderline])
nn_leak = NearestNeighbors(n_neighbors=k + 1)
nn_leak.fit(X_leaked)
d_leak, _ = nn_leak.kneighbors(X_leaked)
self_out_leaked = d_leak[:, 1:].mean(axis=1)
threshold_leaked = self_out_leaked.mean() + 2 * self_out_leaked.std()
print(f"threshold, train-only: {threshold:.3f}")
print(f"threshold, leaked:     {threshold_leaked:.3f}")
```

```
borderline compound distance (train-only reference): 3.83
threshold, train-only: 3.543
threshold, leaked:     3.797
```

With the correct, train-only threshold, this compound sits outside the domain, 3.83 against 3.54. Fit the threshold on a set that already includes it, and the threshold jumps to 3.80, right up against the compound's own distance. It doesn't flip the verdict for this exact compound, but it comes within hundredths of doing so, and a marginally less unusual compound would have flipped. That's the actual mechanism behind "fitting the AD reference on the wrong partition gives you nothing": every point you leak into the reference set pulls the threshold toward itself.

## What to do with this

Attach `within_applicability_domain` as a column next to every prediction your model makes, not just as a diagnostic you check once. The threshold itself is a judgment call, mean plus two standard deviations is a reasonable default, but the important part demonstrated above: fit it once, on the training partition only, and apply it unchanged to everything downstream, held-out test compounds, external validation sets, and any new screening library.
