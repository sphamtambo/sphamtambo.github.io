---
layout: tutorial
title: "Receptor Preparation with PDBFixer and Open Babel"
---

The traditional way to prepare a receptor for docking is clicking through a GUI, Chimera, Maestro, or similar, fixing missing atoms and setting protonation states by hand. A scriptable pipeline, PDBFixer plus Open Babel here, replaces the manual steps with code: every decision is logged, and running it again on the same input gives you the same result. It doesn't replace visual inspection entirely, a final look at the prepared structure in a viewer is still part of the real workflow, covered at the end.

Every number below is from an actual run against a real, public PDB structure.

## What you'll need

```bash
pip install pdbfixer openmm
brew install open-babel   # or your platform's equivalent
```

## Step 1: Get a receptor structure

Using 6CM4, a real inactive-state dopamine D2 receptor structure, as the example. Any PDB file works the same way.

```bash
curl -o receptor_raw.pdb https://files.rcsb.org/download/6CM4.pdb
```

## Step 2: Repair missing atoms with PDBFixer

Crystal structures routinely have incomplete side chains where electron density was too weak to resolve every atom. PDBFixer finds and fills these in.

```python
from pdbfixer import PDBFixer
from openmm.app import PDBFile

fixer = PDBFixer(filename="receptor_raw.pdb")
n_before = sum(1 for _ in fixer.topology.atoms())

fixer.findMissingResidues()
fixer.findMissingAtoms()
n_incomplete_residues = len(fixer.missingAtoms)

fixer.addMissingAtoms()
n_after = sum(1 for _ in fixer.topology.atoms())

with open("receptor_repaired.pdb", "w") as f:
    PDBFile.writeFile(fixer.topology, fixer.positions, f)

print(f"atoms before: {n_before}")
print(f"residues with missing atoms: {n_incomplete_residues}")
print(f"atoms after repair: {n_after}")
```

```
atoms before: 3236
residues with missing atoms: 70
atoms after repair: 3489
```

Seventy residues had incomplete side chains in the raw file, 253 atoms added to fix them. Note what this step deliberately does not do: PDBFixer can also add hydrogens and run an energy minimization, but neither is used here. Hydrogen placement and protonation state are handled explicitly in a later step instead, and skipping minimization keeps the structure's heavy-atom geometry exactly as resolved experimentally, rather than letting a force field nudge it before docking even starts.

## Step 3: Strip crystallization additives

Real crystal structures come with more than protein: bound lipids, cryoprotectants, and waters that have nothing to do with the biology you're docking against. 6CM4 specifically carries oleic acid (a membrane-mimetic lipid used in the crystallization) and PEG (a cryoprotectant), on top of ordered waters.

```python
strip_resnames = {"OLA", "PEG"}
kept, removed = [], []

with open("receptor_repaired.pdb") as f:
    for line in f:
        if line.startswith("HETATM"):
            resname = line[17:20].strip()
            if resname in strip_resnames or resname == "HOH":
                removed.append(resname)
                continue
        kept.append(line)

with open("receptor_stripped.pdb", "w") as f:
    f.writelines(kept)

from collections import Counter
print(Counter(removed))
```

```
Counter({'OLA': 45, 'PEG': 21, 'HOH': 16})
```

This version strips every water. A more careful pipeline checks each water's distance to the ligand and to any functionally relevant ion before removing it, some waters bridge a binding-site contact and are worth keeping, most don't and aren't. That distance check is a real, separate step, not shown here for space, but don't treat "strip all HOH" as the general rule, treat it as this tutorial's simplification.

## Step 4: The fusion protein problem, and why you can't automate it away

6CM4, like a lot of GPCR structures solved by X-ray crystallography, isn't just the receptor. It's a receptor-T4-lysozyme fusion construct, T4 lysozyme inserted into a flexible loop to help the whole thing crystallize. That fusion has to come out before docking, since it isn't part of the biological receptor and would otherwise occupy space and confuse any pocket-based analysis.

Here's the honest part: you cannot reliably find the exact fusion boundary from residue numbering alone. Numbering schemes vary by structure, and a fusion insertion doesn't always announce itself as a clean gap. The residue range has to be confirmed, either from the structure's own publication, or by opening the file in a viewer and checking where the receptor fold stops looking like a GPCR and starts looking like lysozyme.

Once you know the real boundary, removing it is simple:

```python
# fusion_start, fusion_end confirmed from the primary paper or visual
# inspection, NOT guessed from numbering
fusion_start, fusion_end = 220, 400  # placeholder, confirm per structure

kept = []
with open("receptor_stripped.pdb") as f:
    for line in f:
        if line.startswith(("ATOM", "HETATM")):
            resi = int(line[22:26])
            if fusion_start <= resi <= fusion_end:
                continue
        kept.append(line)

with open("receptor_no_fusion.pdb", "w") as f:
    f.writelines(kept)
```

The mechanic is trivial once you have the range. Getting the range right is the actual work, and skipping that verification is exactly how a fusion segment ends up left in, or real receptor residues end up cut out.

## Step 5: Protonate and convert with Open Babel

```bash
obabel receptor_no_fusion.pdb -O receptor_prepared.pdbqt -p 7.4 -xr
```

```
1 molecule converted
```

`-p 7.4` protonates the structure at physiological pH. `-xr` writes a rigid-receptor PDBQT, the format Vina and GNINA both expect. This is a receptor-specific step, worth being precise about: ligand protonation in a real pipeline typically uses a different tool, Dimorphite-DL, which enumerates ionization states for small molecules specifically. It isn't a substitute for Open Babel here, and Open Babel isn't a substitute for it on the ligand side. They handle different molecule types and aren't interchangeable.

## Step 6: The step that still needs a viewer

Everything above is scriptable, but the pipeline doesn't end without a human check. Load the prepared structure in a molecular viewer, PyMOL, ChimeraX, or a notebook-embedded viewer like py3Dmol if you're working in Colab, and confirm: the fusion is actually gone, the retained ligand or reference pocket looks intact, and nothing that should have been stripped is still sitting in the file. Scripting the repair, additive removal, and protonation steps makes the process reproducible. It doesn't make visual confirmation optional.
