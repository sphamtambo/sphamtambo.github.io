---
layout: tutorial
title: "Ligand Preparation for Docking: Protonation, Chirality, and Conformers"
---

Receptor preparation gets one structure ready. Ligand preparation has to get a whole screening library ready, and each compound brings its own version of the same open questions: what ionization state is it in at physiological pH, does it have a stereocenter the input SMILES never specified, and what 3D shape should represent it going into a docking box that only sees one conformer at a time.

Every number and every output below is from an actual run.

## What you'll need

```bash
pip install rdkit dimorphite-dl meeko
```

## Step 1: Check for unspecified stereochemistry before you do anything else

This has to come first, not last, because every step after it can silently commit you to a stereochemistry decision you never actually made.

```python
from rdkit import Chem

smi = "CC(N)C(=O)O"  # alanine, stereocenter left unspecified
mol = Chem.MolFromSmiles(smi)

centers = Chem.FindMolChiralCenters(mol, includeUnassigned=True, useLegacyImplementation=False)
print("chiral centers:", centers)
```

```
chiral centers: [(1, '?')]
```

One stereocenter, unassigned. This SMILES describes both enantiomers at once. Nothing downstream knows that unless you tell it.

## Step 2: Protonate at physiological pH

```python
from dimorphite_dl import protonate_smiles

states = protonate_smiles(smi, ph_min=7.4, ph_max=7.4, precision=0.5)
print("protonation states at pH 7.4:", states)
```

```
protonation states at pH 7.4: ['CC([NH3+])C(=O)[O-]', 'CC(N)C(=O)[O-]']
```

Two states came back. Alanine's dominant real-world form at pH 7.4 is the zwitterion, protonated amine, deprotonated carboxylic acid, and that's exactly what's listed first. A defensible, simple rule is to keep the first returned state as the dominant one; a more thorough pipeline would carry multiple states through when they're close enough in population to matter for a specific compound.

## Step 3: The trap, embedding silently resolves stereochemistry for you

Take the protonated SMILES from Step 2, still stereochemically unspecified, and generate a 3D conformer the ordinary way.

```python
from rdkit.Chem import AllChem

mol = Chem.MolFromSmiles("CC([NH3+])C(=O)[O-]")
mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol, randomSeed=42)
AllChem.MMFFOptimizeMolecule(mol)

print(Chem.MolToSmiles(Chem.RemoveHs(mol)))
```

```
C[C@H]([NH3+])C(=O)[O-]
```

Look closely: the output SMILES now has `@H`, a specific, assigned stereocenter, even though nothing was specified going in. The embedding step had to place atoms in actual 3D space, so it picked one enantiomer to build. Silently. If this ligand goes on to docking without anyone noticing, you've docked one specific enantiomer of a compound that was never actually specified as that enantiomer, and whoever reads the result has no way to know which one, or that a choice was even made.

## Step 4: Do it properly, enumerate instead of letting embedding decide

```python
from rdkit.Chem.EnumerateStereoisomers import EnumerateStereoisomers, StereoEnumerationOptions

mol = Chem.MolFromSmiles("CC([NH3+])C(=O)[O-]")
opts = StereoEnumerationOptions(onlyUnassigned=True, unique=True)
isomers = list(EnumerateStereoisomers(mol, options=opts))
for iso in isomers:
    print(Chem.MolToSmiles(iso))
```

```
C[C@H]([NH3+])C(=O)[O-]
C[C@@H]([NH3+])C(=O)[O-]
```

Now there are two explicit, distinct molecules, and the decision about which one, or both, to carry forward into docking is a decision someone actually made, not an accident of how a distance-geometry algorithm happened to place atoms. For a real screening library, this means every unspecified stereocenter should be enumerated before conformer generation, not discovered afterward by reading the output SMILES closely, the way it was found here.

## Step 5: Generate the conformer for the isomer you're keeping

```python
mol = isomers[0]
mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol, randomSeed=42)
AllChem.MMFFOptimizeMolecule(mol)

Chem.MolToMolFile(mol, "ligand_3d.mol")
```

The fixed seed matters here for a different reason than the stereochemistry issue above: it makes the specific 3D conformer chosen reproducible across runs, so re-running the pipeline on the same molecule doesn't silently produce a geometrically different starting point.

## Step 6: Convert to a docking-ready format with Meeko

```python
from meeko import MoleculePreparation, PDBQTWriterLegacy

mol = Chem.MolFromMolFile("ligand_3d.mol", removeHs=False)
preparator = MoleculePreparation()
setup = preparator.prepare(mol)[0]

pdbqt_string, success, error_msg = PDBQTWriterLegacy.write_string(setup)
print("conversion succeeded:", success)
```

```
conversion succeeded: True
```

## Step 7: Confirm the atom types are actually usable

Not every valid PDBQT atom type is accepted by every docking engine. GNINA in particular rejects the macrocycle pseudo-atom types Meeko can emit for large ring systems, which won't come up for a small molecule like this one, but matters as soon as your library includes anything macrocyclic.

```python
GNINA_COMPATIBLE_TYPES = {
    'H', 'HD', 'HS', 'C', 'A', 'N', 'NA', 'NS', 'OA', 'OS', 'F',
    'Mg', 'MG', 'P', 'SA', 'S', 'Cl', 'CL', 'Ca', 'CA', 'Mn', 'MN',
    'Fe', 'FE', 'Zn', 'ZN', 'Br', 'BR', 'I',
}

found_types = set()
for line in pdbqt_string.splitlines():
    if line.startswith(("ATOM", "HETATM")):
        atom_type = line[77:].strip().split()[-1]
        found_types.add(atom_type)

print("atom types found:", found_types)
print("incompatible with GNINA:", found_types - GNINA_COMPATIBLE_TYPES)
```

```
atom types found: {'HD', 'N', 'C', 'OA'}
incompatible with GNINA: set()
```

Clean here. For a macrocyclic candidate, the fix is disabling macrocycle sampling in Meeko and preparing it as a rigid ring instead, trading some conformational flexibility for a file every downstream engine can actually read.

## Putting it together

The order matters: check for unspecified stereochemistry before embedding, not after, protonate before generating 3D coordinates, enumerate stereoisomers explicitly rather than trusting whichever one the embedding algorithm happens to produce, and validate atom-type compatibility with whichever engine is actually going to consume the file. Skip the stereochemistry check specifically, and the mistake doesn't announce itself. It just quietly docks a specific enantiomer nobody chose.
