# Docking Pipeline User Guide

## Overview

The **Docking Pipeline** is a fully automated molecular docking workflow designed for protein-ligand interactions using AutoDock Vina. It handles the entire process from receptor preparation to ligand docking and result analysis.

### Key Features

- **Automated Receptor Preparation**: Downloads from RCSB PDB or uses pre-prepared structures
- **Ligand Processing**: Converts ligand files (SDF, MOL2) to PDBQT format
- **Batch Docking**: Performs molecular docking with AutoDock Vina
- **Score Summarization**: Extracts and summarizes binding affinity scores
- **Organized Output**: Generates timestamped run directories with structured results

---

## System Requirements

### Dependencies

- **Python 3.8+**
- **Conda** (recommended) or pip
- **AutoDock Vina**
- **Open Babel** (for file format conversions)
- **Biopython**
- **ADFR** (for PDBQT preparation)

### Installation

1. **Create the conda environment**:
   ```bash
   conda env create -f environment.yml
   conda activate docking_env
   ```

2. **Verify installations**:
   ```bash
   which vina
   obabel -V
   python3 -c "import Bio; print(Bio.__version__)"
   ```

---

## Directory Structure

```
docking/
├── run_pipeline.py              # Main pipeline orchestrator
├── run_vina_batch.py            # Batch docking runner
├── convert_to_pdbqt.py          # Ligand PDBQT conversion
├── summarize_vina_scores.py     # Score extraction and analysis
├── add_ligands.py               # Add new ligands to the library
├── sdf_to_pdbqt.py              # SDF-to-PDBQT conversion utility
├── prepare_top_ligands_for_chimerax.py  # ChimeraX visualization prep
├── docking_config.yml           # Configuration file
├── environment.yml              # Conda dependencies
├── ligands/                     # Input ligand structures
│   ├── *.sdf                    # SDF format (multi-conformer)
│   └── *.mol2                   # MOL2 format
└── runs/                        # Output directory
    └── docking_run_YYYY-MM-DD_HH-MM-SS/
        ├── receptor_prepped.pdbqt
        ├── combined_ligands.sdf
        ├── config_used.yml
        ├── vina_config.txt
        ├── pdbqt_files/
        └── results/
            ├── scores.csv
            └── *.pdbqt         # Docked ligand poses
```

---

## Configuration

The pipeline is configured via **`docking_config.yml`**:

### Receptor Setup

```yaml
# Option 1: Download from RCSB
pdb_id: 3EST

# Option 2: Use pre-prepared file (overrides pdb_id)
receptor: ""

# Protein chain(s) to keep
chains: "A"

# Keep cofactors/ligands from PDB
keep_cofactors: false
```

### Ligands

```yaml
# Folder containing input ligand files
ligand_dir: ligands/

# Output combined SDF (auto-generated)
combined_sdf: combined_ligands.sdf
```

### Docking Box

Define the search space box center and dimensions:

```yaml
center_x: -7.141
center_y: 26.551
center_z: 38.452

# Box size in Ångströms
size_x: 18
size_y: 18
size_z: 18
```

**Tip**: Use PyMOL, UCSF Chimera, or online tools to determine the binding site center.

### Vina Settings

```yaml
# Number of CPU cores to use
cpu: 8

# Output filename for Vina configuration
config_out: vina_config.txt
```

---

## Workflow

### Step 1: Prepare Input Ligands

Place your ligand structure files in the `ligands/` directory. Supported formats:
- **SDF** (Structure Data Format) - Recommended for multi-conformer libraries
- **MOL2** (Sybyl format)
- **PDB** (via conversion)

#### Adding Ligands

```bash
python3 add_ligands.py /path/to/ligand.sdf
```

#### Converting Formats

Convert SDF to PDBQT:
```bash
python3 sdf_to_pdbqt.py input.sdf output.pdbqt
```

### Step 2: Update Configuration

Edit `docking_config.yml`:

1. Set the **receptor** (PDB ID or file path)
2. Set the **docking box** coordinates and size
3. Adjust **CPU cores** if needed

### Step 3: Run the Pipeline

```bash
python3 run_pipeline.py docking_config.yml
```

**What happens**:
1. Downloads/validates receptor
2. Prepares receptor (cleaning, adding hydrogens, PDBQT conversion)
3. Converts ligands to PDBQT
4. Runs AutoDock Vina
5. Collects and summarizes scores

### Step 4: Review Results

A timestamped directory is created: `runs/docking_run_YYYY-MM-DD_HH-MM-SS/`

**Key output files**:
- `results/scores.csv` - Binding affinity scores for all ligands
- `results/*.pdbqt` - Docked ligand poses
- `vina_config.txt` - Vina configuration used
- `config_used.yml` - Pipeline config snapshot

### Step 5: Analyze Results

#### View Score Summary

```bash
cat runs/docking_run_*/results/scores.csv
```

#### Sort by Binding Affinity

```bash
tail -n +2 runs/docking_run_*/results/scores.csv | sort -t',' -k2 -n
```

#### Prepare for Visualization

```bash
python3 prepare_top_ligands_for_chimerax.py runs/docking_run_*/results/
```

Then open in **UCSF ChimeraX** or **PyMOL** for visual inspection.

---

## Output Files

### `scores.csv`

CSV file with docking results:

```
Ligand,Affinity (kcal/mol),RMSD_LB,RMSD_UB
1EIN_PLC_A_601,-7.8,0.0,3.4
1W52_DDQ_X_501,-7.2,0.5,2.1
...
```

Lower binding affinity (more negative) = stronger predicted binding.

### Docked Poses

Each ligand is saved as `results/<ligand_name>_mode_1.pdbqt` with the best pose.

Vina can generate multiple binding modes (poses) if configured.

---

## Advanced Usage

### Running Just Vina (Batch Mode)

If you already have prepped ligands in PDBQT format:

```bash
python3 run_vina_batch.py docking_config.yml
```

### Manual Receptor Preparation

If downloading fails or you have a custom receptor:

1. Prepare PDB file (remove water, ligands, etc.)
2. Convert to PDBQT using ADFR or MGL:
   ```bash
   prepare_receptor4.py -r protein.pdb -o protein.pdbqt
   ```
3. Set `receptor: path/to/protein.pdbqt` in config

### Multiple Docking Runs

Create variant configs:
- `docking_config_site1.yml` - Different box
- `docking_config_site2.yml` - Different receptor

Then run each separately:
```bash
python3 run_pipeline.py docking_config_site1.yml
python3 run_pipeline.py docking_config_site2.yml
```

---

## Troubleshooting

### Error: "Vina not found"

Ensure conda environment is activated and Vina installed:
```bash
conda activate docking_env
vina --version
```

### Error: "Failed to download PDB"

Check PDB ID is correct (e.g., `3EST` not `3est`). Verify internet connection.

### Error: "Ligand conversion failed"

- Verify ligand file format is valid (SDF/MOL2)
- Test with Open Babel:
  ```bash
  obabel -isdf input.sdf -opdbqt -O output.pdbqt
  ```

### Poor Docking Results

- **Verify docking box**: Ensure center and size encompass the binding site
- **Check ligand preparation**: Ensure protonation states are correct
- **Adjust Vina settings**: Exhaustiveness/num_modes in `vina_config.txt`

---

## Performance Tips

1. **Use multiple CPU cores**: Set `cpu: 8` or higher in config
2. **Batch process**: Run multiple docking experiments in sequence
3. **Filter ligands**: Pre-screen by drug-likeness before docking
4. **Parallelize**: Create config variants and run on different machines

---

## Citation & References

- **AutoDock Vina**: O. Trott & A. J. Olson, J. Comp. Chem. 2010
- **RCSB PDB**: www.rcsb.org
- **Open Babel**: Guha, R., et al., J. Cheminform. 2016

---

## Support & Customization

For questions or customization needs:
- Review the source code comments in main scripts
- Check the YAML config file for available parameters
- Examine output logs: `docking_run.log`

---

**Last Updated**: April 2026  
**Pipeline Version**: 2.0
