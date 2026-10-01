# SNP Demultiplexing and Doublet Detection Pipeline

A unified SLURM-based pipeline for **single-cell RNA-seq donor demultiplexing and doublet detection**.

The pipeline combines:

- **Demuxalot** for SNP-based donor demultiplexing
- **scDblFinder** for expression-based doublet detection
- **DoubletDetection** for expression-based doublet detection
- **scds** for expression-based doublet detection
- **Scrublet** for expression-based doublet detection
- **cellSNP-lite** for SNP pileup generation
- **Vireo** for genotype-free donor demultiplexing

The pipeline automatically discovers Cell Ranger outputs and supports both **`cellranger count`** and **`cellranger multi`** directory structures.

---

## Overview

The workflow consists of three independent stages:

```text
Cell Ranger output
       │
       ├──────────────────────────────┐
       │                              │
       ▼                              ▼
Step 1: Demuxalot             Step 2: Doublet Detection
SNP demultiplexing             ├── scDblFinder
                               ├── DoubletDetection
                               ├── scds
                               └── Scrublet
       │                              │
       └──────────────┬───────────────┘
                      │
                      ▼
              Step 3: SNP Demultiplexing
                      │
                 cellSNP-lite
                      │
                      ▼
                    Vireo
```

The methods provide complementary information.

**Demuxalot and Vireo** use genetic variation to assign cells to donors and provide SNP-based evidence for cross-donor doublets.

The expression-based doublet callers use the transcriptomic profile and can therefore identify potential doublets that cannot necessarily be distinguished using donor SNPs alone.

---

## Pipeline Structure

```text
snp_demux_pipeline/
├── config.sh
├── README.md
├── bin/
│   ├── build_manifest.sh
│   ├── common.sh
│   ├── step1.sh
│   ├── step2.sh
│   ├── step3.sh
│   ├── submit.sh
│   └── summarize.sh
├── slurm/
│   ├── step1.slurm
│   ├── step2.slurm
│   └── step3.slurm
└── logs/
```

### Main files

| File | Purpose |
|---|---|
| `config.sh` | Central configuration for all pipeline stages |
| `bin/build_manifest.sh` | Automatically discovers samples, BAMs, BAM indexes, and filtered matrices |
| `bin/common.sh` | Shared helper functions |
| `bin/step1.sh` | Runs Demuxalot |
| `bin/step2.sh` | Runs expression-based doublet detection |
| `bin/step3.sh` | Runs cellSNP-lite followed by Vireo |
| `bin/submit.sh` | Builds the manifest and submits the selected SLURM stage |
| `bin/summarize.sh` | Summarizes pipeline status across samples |
| `slurm/*.slurm` | SLURM array job definitions |

---

# Automatic Cell Ranger Input Discovery

One of the main features of the pipeline is automatic detection of Cell Ranger output structure.

The user does **not** need to manually specify the filtered matrix path for every sample.

The pipeline recursively searches `BASE_DIR` for supported Cell Ranger BAM files.

## `cellranger count`

The standard structure is recognized:

```text
sample/
└── outs/
    ├── possorted_genome_bam.bam
    ├── possorted_genome_bam.bam.bai
    │
    └── filtered_feature_bc_matrix/
        ├── barcodes.tsv.gz
        ├── features.tsv.gz
        └── matrix.mtx.gz
```

The pipeline searches for:

```text
possorted_genome_bam.bam
```

and associates it with:

```text
outs/filtered_feature_bc_matrix/
```

---

## `cellranger multi`

Per-sample outputs from `cellranger multi` are also supported.

For example:

```text
multi_config_1/
└── outs/
    └── per_sample_outs/
        └── multi_config_1/
            ├── count/
            │   └── sample_filtered_feature_bc_matrix/
            │       ├── barcodes.tsv.gz
            │       ├── features.tsv.gz
            │       └── matrix.mtx.gz
            │
            └── sample_alignments.bam
```

The pipeline searches for:

```text
sample_alignments.bam
```

and resolves the corresponding:

```text
sample_filtered_feature_bc_matrix/
```

directory.

A fallback search is also performed underneath the resolved sample directory if the expected matrix location is not found.

---

# Shared Manifest

All pipeline stages use a **single automatically generated manifest**.

Running any stage regenerates:

```text
pipeline_manifest.tsv
```

Example:

```text
sample          mode    sample_dir    bam    bai    matrix_dir    barcodes_gz    json
multi_config_1  multi   ...           ...    ...    ...           ...            NA
sample_A        count   ...           ...    ...    ...           ...            NA
```

The manifest records:

| Column | Description |
|---|---|
| `sample` | Sample name |
| `mode` | `count` or `multi` |
| `sample_dir` | Resolved Cell Ranger sample directory |
| `bam` | BAM file |
| `bai` | BAM index |
| `matrix_dir` | Filtered feature-barcode matrix |
| `barcodes_gz` | Filtered cell barcode file |
| `json` | Optional JSON file used by Solo |

The manifest should **not be edited manually**.

---

# Requirements

The pipeline assumes an HPC environment with:

- SLURM
- Singularity/Apptainer
- Bash
- Demuxafy-compatible container
- Cell Ranger output
- Donor/SNP VCF files

The configured container must contain the required tools:

```text
Demuxalot
scDblFinder
DoubletDetection
scds
Scrublet
cellSNP-lite
Vireo
```

---

# Configuration

All settings are controlled through:

```text
config.sh
```

The most important setting is:

```bash
BASE_DIR="/path/to/cellranger/results"
```

The pipeline searches underneath this directory for Cell Ranger outputs.

---

## Container

Set the Demuxafy Singularity image:

```bash
SIF="${BASE_DIR}/SNPS/Demuxafy.sif"
```

---

## Singularity Bind Mount

On systems where the data filesystem is not automatically visible inside Singularity, configure the filesystem root explicitly.

For example:

```bash
BIND_ROOT="/data/chi"
SINGULARITY_BIND="${BIND_ROOT}:${BIND_ROOT}"
```

Pipeline commands are then executed using:

```bash
singularity exec \
    --bind "$SINGULARITY_BIND" \
    "$SIF" \
    ...
```

This ensures that BAM files, matrices, VCF files, and output directories are visible inside the container.

---

# SNP Configuration

## Demuxalot VCF

```bash
DEMUX_VCF="${BASE_DIR}/SNPS/liftover2.vcf"
```

## Vireo VCF

```bash
VIREO_VCF="${BASE_DIR}/SNPS/liftover2.vcf.gz"
```

The Vireo VCF should be bgzip-compressed and indexed.

Expected index:

```text
liftover2.vcf.gz.tbi
```

or:

```text
liftover2.vcf.gz.csi
```

If an index is missing, Step 3 attempts to generate one using `tabix`.

---

# Donor Configuration

For Demuxalot, donor IDs can be supplied using a file:

```bash
DONORS="${BASE_DIR}/SNPS/ind.txt"
DONORS_IS_FILE=true
```

Alternatively, a donor count can be configured:

```bash
DONORS_IS_FILE=false
DONOR_COUNT=8
```

---

# Vireo Configuration

Set the expected number of donors:

```bash
VIREO_N=12
```

Other settings include:

```bash
THREADS=8
RAND_SEED=12345
```

Vireo is run in **genotype-free mode** after cellSNP-lite.

---

# Step 1 — Demuxalot

Step 1 performs SNP-based donor demultiplexing.

For each sample, the pipeline:

1. Retrieves the BAM and filtered barcodes from the shared manifest.
2. Converts compressed barcodes to an uncompressed TSV when necessary.
3. Verifies that the BAM is visible inside Singularity.
4. Runs Demuxalot.
5. Stores logs and pipeline status.

Conceptually:

```text
BAM
 +
filtered cell barcodes
 +
donor SNP VCF
 +
donor IDs
 │
 ▼
Demuxalot
 │
 ▼
donor assignments
```

Outputs are written under:

```text
SNPS9/DEMUX/
```

with a sample-specific directory such as:

```text
SNPS9/DEMUX/multi_config_1_demux/
```

---

# Step 2 — Expression-Based Doublet Detection

Step 2 runs multiple independent doublet-detection algorithms against the filtered 10x matrix.

Enabled by default:

```bash
RUN_SCDBLFINDER=true
RUN_DOUBLETDETECTION=true
RUN_SCDS=true
RUN_SCRUBLET=true
```

These correspond to:

- scDblFinder
- DoubletDetection
- scds
- Scrublet

Each program runs independently.

If one program fails, the remaining programs are still attempted, and the status of each method is recorded.

Outputs are stored under:

```text
SNPS9/DOUBLETD/<sample>/
```

For example:

```text
SNPS9/DOUBLETD/multi_config_1/
├── scDblFinder/
├── DoubletDetection/
├── scds/
├── Scrublet/
├── logs/
└── status.tsv
```

---

## Solo

Solo is included as an optional method but is disabled by default:

```bash
RUN_SOLO=false
```

The current Solo workflow requires an appropriate:

```text
samplesheet.json
```

If Solo is enabled but the JSON file is unavailable, Solo is skipped rather than causing the entire doublet-detection stage to fail.

---

# Step 3 — cellSNP-lite + Vireo

Step 3 performs an independent SNP-based donor demultiplexing workflow.

It consists of two parts.

## Step 3A — cellSNP-lite

cellSNP-lite uses:

```text
BAM
 +
filtered cell barcodes
 +
VCF
```

to generate allele-count matrices.

The pipeline runs approximately:

```bash
cellsnp-lite \
    -s <BAM> \
    -b <BARCODES> \
    -O <OUTPUT> \
    -R <VCF> \
    -p <THREADS> \
    --minMAF 0.1 \
    --minCOUNT 20 \
    --gzip
```

Outputs are written under:

```text
SNPS9/CELLSNP_ONLY/<sample>/cellsnp/
```

Before continuing, the pipeline verifies that the required Vireo inputs were generated, including:

```text
cellSNP.base.vcf
cellSNP.tag.AD.mtx
cellSNP.tag.DP.mtx
```

Compressed versions are also accepted.

---

## Step 3B — Vireo

The cellSNP-lite output is passed directly to Vireo.

Conceptually:

```text
BAM + VCF + barcodes
        │
        ▼
   cellSNP-lite
        │
        ▼
 allele-count matrices
        │
        ▼
      Vireo
        │
        ▼
 donor assignments
```

Vireo is run genotype-free using:

```bash
vireo \
    -c <CELLSNP_DIR> \
    -o <VIREO_DIR> \
    -N <NUMBER_OF_DONORS> \
    -p <THREADS>
```

If the installed Vireo version supports `--randSeed`, the configured random seed is also supplied.

The pipeline verifies the creation of:

```text
donor_ids.tsv
```

or:

```text
donor_ids.tsv.gz
```

Vireo outputs are written under:

```text
SNPS9/VIREO_GF/<sample>/vireo/
```

---

# Installation

Clone or copy the pipeline into the desired location.

Make the scripts executable:

```bash
chmod +x bin/*.sh
chmod +x slurm/*.slurm
```

Then edit:

```text
config.sh
```

for the current experiment.

---

# Running the Pipeline

The three stages are deliberately submitted separately.

This makes it possible to verify each stage before moving to the next one.

## Step 1

```bash
bash bin/submit.sh step1
```

This:

1. Searches for Cell Ranger samples.
2. Rebuilds `pipeline_manifest.tsv`.
3. Determines `count` versus `multi`.
4. Resolves BAM and matrix locations.
5. Submits one Demuxalot SLURM array task per sample.

---

## Step 2

```bash
bash bin/submit.sh step2
```

This rebuilds the shared manifest and submits the doublet-detection array.

---

## Step 3

```bash
bash bin/submit.sh step3
```

This rebuilds the shared manifest and submits the cellSNP-lite/Vireo array.

---

# SLURM Arrays

Each sample corresponds to one row of:

```text
pipeline_manifest.tsv
```

and one SLURM array task.

For example, if the manifest contains seven samples:

```text
sample 1
sample 2
sample 3
sample 4
sample 5
sample 6
sample 7
```

the submission is equivalent to:

```bash
sbatch --array=1-7 ...
```

This allows samples to run independently and in parallel.

---

# Monitoring Jobs

Standard SLURM commands can be used to monitor jobs:

```bash
squeue -u "$USER"
```

To inspect a specific job:

```bash
scontrol show job <JOB_ID>
```

Pipeline-level SLURM logs are written under:

```text
logs/
```

---

# Pipeline Status Summary

After running the pipeline:

```bash
bash bin/summarize.sh
```

Example output:

```text
sample          mode    step1      step2         step3
multi_config_1  multi   SUCCESS    SUCCESS       SUCCESS
multi_config_2  multi   SUCCESS    SUCCESS       SUCCESS
sample_A        count   SUCCESS    PARTIAL_FAIL  SUCCESS
```

This provides a quick overview of which samples completed each stage.

---

# Output Structure

A completed analysis will generally resemble:

```text
SNPS9/
├── DEMUX/
│   ├── sample_1_demux/
│   ├── sample_2_demux/
│   └── ...
│
├── DOUBLETD/
│   ├── sample_1/
│   │   ├── scDblFinder/
│   │   ├── DoubletDetection/
│   │   ├── scds/
│   │   ├── Scrublet/
│   │   ├── logs/
│   │   └── status.tsv
│   └── ...
│
├── CELLSNP_ONLY/
│   ├── sample_1/
│   │   ├── barcodes.tsv
│   │   └── cellsnp/
│   └── ...
│
└── VIREO_GF/
    ├── sample_1/
    │   ├── vireo/
    │   ├── logs/
    │   └── status.tsv
    └── ...
```

---

# Doublet Detection Strategy

The pipeline deliberately combines **SNP-based demultiplexing** with **expression-based doublet detection**.

## SNP-based methods

Demuxalot and Vireo use donor genetic variation.

These methods are particularly useful for identifying cells whose SNP evidence is inconsistent with assignment to a single donor.

## Expression-based methods

The pipeline additionally runs:

```text
scDblFinder
DoubletDetection
scds
Scrublet
```

These methods evaluate expression characteristics associated with doublets.

Using both approaches provides complementary evidence rather than relying on a single doublet-detection algorithm.

---

# Why Dropulation Is Not Included

Dropulation is not included in the final pipeline.

The workflow already contains two SNP-based demultiplexing approaches:

```text
Demuxalot
Vireo
```

as well as four expression-based doublet detection methods.

The intended downstream approach is therefore to **compare and integrate the existing calls** rather than continually adding additional demultiplexing/doublet algorithms.

---

# Recommended Workflow

```text
                    Cell Ranger
                         │
             ┌───────────┴───────────┐
             │                       │
       Cell Ranger Count       Cell Ranger Multi
             │                       │
             └───────────┬───────────┘
                         │
                         ▼
               build_manifest.sh
                         │
                         ▼
                pipeline_manifest.tsv
                         │
             ┌───────────┼────────────┐
             │           │            │
             ▼           ▼            ▼
         Demuxalot    Doublets     cellSNP-lite
                      │                │
                scDblFinder            ▼
             DoubletDetection        Vireo
                    scds               │
                  Scrublet             │
             │           │            │
             └───────────┼────────────┘
                         │
                         ▼
              Downstream integration
                         │
                         ▼
            Final donor / cell QC calls
```

---

# Notes

- Do not manually edit `pipeline_manifest.tsv`; it is regenerated by the pipeline.
- Confirm the donor number before running Vireo.
- Confirm that the appropriate VCF is being used for the experiment.
- The Vireo VCF should be bgzip-compressed and indexed.
- BAM indexes must be present.
- Both compressed and uncompressed 10x matrix components are supported by sample discovery.
- The `/data/chi:/data/chi` Singularity bind is important on systems where `/data/chi` is not automatically mounted inside the container.
- Step 2 does not require a Seurat object.
- Dropulation, DoubletFinder, and DoubletDecon are not part of the final pipeline.
- Each stage can be rerun independently.

---

## Summary

This pipeline provides a reusable workflow for:

**automatic Cell Ranger sample discovery → SNP donor demultiplexing → multi-method doublet detection → independent Vireo demultiplexing → downstream consensus QC**

with support for both:

```text
cellranger count
```

and:

```text
cellranger multi
```

from a single shared configuration and sample manifest.
