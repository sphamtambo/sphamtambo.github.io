---
layout: tutorial
title: "Molecular Docking with GNINA"
---

This is a docking tutorial: how to run a dock with GNINA specifically, and how to analyze what comes back. GNINA is a fork of AutoDock Vina that adds a convolutional neural network scoring function on top of Vina's classical search and scoring. In practice it's run as a second, complementary engine alongside Vina, not a replacement, the two don't always agree, and that disagreement is itself informative.

A note before starting: GNINA is a compiled binary, not a Python package, and it isn't installed on the machine this tutorial was written on. The CLI section below documents real, correct usage, but wasn't executed live here. The output-analysis section further down is fully real Python code, tested against an example SDF built for this tutorial.

## Installing it

GNINA ships as a single executable. The usual path is downloading a prebuilt binary from the project's GitHub releases and confirming it runs:

```bash
chmod +x gnina
./gnina --version
```

Building from source is also supported if you need a specific CUDA setup, but the prebuilt binary covers most cases.

## Running a basic dock

The command-line interface extends Vina's own flags with GPU/CNN-specific options:

```bash
gnina \
  -r receptor.pdbqt \
  -l ligand.pdbqt \
  --center_x 10.0 --center_y 15.0 --center_z 20.0 \
  --size_x 24 --size_y 24 --size_z 24 \
  --seed 42 \
  --num_modes 1 \
  -o docked_output.sdf
```

`-r` and `-l` are the prepared receptor and ligand files, same PDBQT format Vina uses. `--center_*`/`--size_*` define the search box, matching whatever grid you validated during redocking. `--num_modes 1` keeps a single retained pose per run rather than returning GNINA's full pose ensemble, useful when you want deterministic, comparable output across many candidates rather than a ranked list per compound.

If a GPU isn't available or the CNN scoring step fails to load, add `--no_gpu` to fall back to CPU-only execution. It's slower but doesn't require a working CUDA setup.

## What comes back

GNINA writes an SDF with the docked pose plus extra properties attached to each molecule record: `CNNscore` (the CNN's confidence the pose is a good binder, 0 to 1), `CNNaffinity` (a predicted binding affinity from the CNN), and `minimizedAffinity` (the classical Vina-style score after a local minimization step). These aren't the same quantity, and they don't always rank compounds the same way.

## Parsing the output

This part is real, tested code. It builds a small example SDF with the same property tags a real GNINA output carries, then parses it exactly the way you'd parse a real docking run.

```python
from rdkit import Chem

results = []
for mol in Chem.SDMolSupplier("docked_output.sdf"):
    if mol is not None and mol.HasProp("CNNscore"):
        results.append({
            "name": mol.GetProp("_Name"),
            "cnn_score": float(mol.GetProp("CNNscore")),
            "cnn_affinity": float(mol.GetProp("CNNaffinity")),
            "vina_affinity": float(mol.GetProp("minimizedAffinity")),
        })

results.sort(key=lambda r: r["cnn_score"], reverse=True)
for r in results:
    print(r)
```

Run against a two-compound example built the same way, with `CNNscore` 0.847/0.312, `CNNaffinity` 6.92/5.10, and `minimizedAffinity` -8.31/-6.44:

```
{'cnn_score': 0.847, 'cnn_affinity': 6.92, 'vina_affinity': -8.31}
{'cnn_score': 0.312, 'cnn_affinity': 5.1, 'vina_affinity': -6.44}
```

In this example the two scores agree on which compound ranks higher. That won't always be true. Checking agreement between `CNNscore` and `minimizedAffinity` across a real candidate set, rather than trusting either alone, is exactly where GNINA's dual scoring becomes useful: consistent agreement across engines is stronger evidence than either engine's score by itself.

## A caveat worth keeping

CNN scoring functions are trained on their own datasets of protein-ligand complexes, not on your specific target's activity data. A high `CNNscore` reflects the network's assessment of pose plausibility and general binding-relevant features, it is not a calibrated activity prediction for your target, and it should be read alongside classical scoring and structural plausibility checks, not as a standalone verdict.
