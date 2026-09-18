---
layout: tutorial
title: "Calculating Heavy-Atom RMSD Without Getting Fooled by Atom Order"
---

Redocking validation asks one question: can your docking protocol reproduce a receptor's own co-crystallized ligand pose? The answer is a heavy-atom RMSD between two files, the crystallographic reference pose and your docking engine's redocked output, judged against a threshold, commonly 2.0 A.

The part that trips people up isn't the RMSD formula. It's that the reference ligand file and the redocked output almost never share the same atom order, they came from different tools, and even within one molecule, symmetry means more than one atom-to-atom mapping can be chemically valid. Get the correspondence wrong and a genuinely good redocking result can report a large, wrong RMSD.

Every number below is from an actual run, not typed by hand.

## What you'll need

```bash
pip install rdkit numpy
```

## Step 1: The reference pose and a redocked pose

In a real validation run, `reference_pose` would be loaded from the crystal structure's ligand file, and `redocked_pose` from your docking engine's output SDF for that same ligand. Here, to make the atom-ordering problem concrete and reproducible, `redocked_pose` starts as an exact copy of the reference geometry with its atoms relabeled in a different order, exactly what you'd get from two tools exporting the same pose with their own internal numbering.

```python
from rdkit import Chem
from rdkit.Chem import AllChem, rdMolAlign
import numpy as np
import random

reference_pose = Chem.MolFromSmiles("c1ccc(cc1)C(=O)Nc1ccccc1")
reference_pose = Chem.AddHs(reference_pose)
AllChem.EmbedMolecule(reference_pose, randomSeed=42)
AllChem.MMFFOptimizeMolecule(reference_pose)

perm = list(range(reference_pose.GetNumAtoms()))
random.seed(1)
random.shuffle(perm)
redocked_pose = Chem.RenumberAtoms(reference_pose, perm)
```

`reference_pose` and `redocked_pose` describe the exact same 3D structure here, standing in for a docking engine that reproduced the crystal pose essentially perfectly. Only the atom order differs, which is realistic: nothing about a good redocking result guarantees matching atom numbering.

## Step 2: The naive RMSD, and why it fails

```python
def naive_rmsd(pose_a, pose_b):
    conf_a, conf_b = pose_a.GetConformer(), pose_b.GetConformer()
    diffs = []
    for i in range(pose_a.GetNumAtoms()):
        p_a, p_b = conf_a.GetAtomPosition(i), conf_b.GetAtomPosition(i)
        diffs.append((p_a.x - p_b.x)**2 + (p_a.y - p_b.y)**2 + (p_a.z - p_b.z)**2)
    return float(np.sqrt(np.mean(diffs)))

print(f"naive RMSD: {naive_rmsd(reference_pose, redocked_pose):.3f}")
```

```
naive RMSD: 6.346
```

By this number, redocking looks like it failed badly, well outside any reasonable threshold. It hasn't. Atom `i` in `reference_pose` and atom `i` in `redocked_pose` are simply different atoms, so this is comparing the wrong pairs, not measuring a real structural difference.

## Step 3: The atom-order-invariant RMSD

RDKit's `GetBestRMS` searches over chemically valid atom mappings between the two structures, accounting for molecular symmetry, and reports the minimum RMSD across them.

```python
best_rmsd = rdMolAlign.GetBestRMS(redocked_pose, reference_pose)
print(f"heavy-atom RMSD: {best_rmsd:.3f}")
```

```
heavy-atom RMSD: 0.000
```

Correct answer: the redocked pose reproduces the reference exactly, RMSD of zero, a clear pass against any redocking threshold.

## Step 4: Confirming it still catches a real failure

The method should only report a low RMSD when the poses genuinely agree. Check that against a redocked pose that's actually different, standing in for a docking attempt that missed the correct binding mode.

```python
failed_redock = Chem.MolFromSmiles("c1ccc(cc1)C(=O)Nc1ccccc1")
failed_redock = Chem.AddHs(failed_redock)
AllChem.EmbedMolecule(failed_redock, randomSeed=7)
AllChem.MMFFOptimizeMolecule(failed_redock)

print(f"heavy-atom RMSD, different conformer: {rdMolAlign.GetBestRMS(failed_redock, reference_pose):.3f}")
```

```
heavy-atom RMSD, different conformer: 0.536
```

A real pose difference produces a real, non-zero RMSD. In this toy example it's still well under a typical 2.0 A pass threshold, but the method is doing its job: it isn't just returning zero for everything, it's correctly distinguishing "same pose, different atom order" from "actually different pose," which is exactly what a redocking pass/fail decision depends on getting right.

## Where this matters

Before trusting any redocking pass/fail call, confirm your RMSD calculation is doing the symmetry-aware comparison shown in Step 3, not the naive per-index comparison in Step 2. The naive version can fail a genuinely good redocking result, or in principle pass a bad one, purely because of atom ordering that has nothing to do with the actual pose.

A note on PyMOL specifically: its `align` and `super` commands are built for macromolecular structural alignment via sequence correspondence, which is a different problem than atom-order-invariant RMSD between two small-molecule poses. For ligand pose comparison specifically, a tool built for that job, RDKit's `GetBestRMS` here, or a dedicated package like `spyrmsd`, is the more direct fit.
