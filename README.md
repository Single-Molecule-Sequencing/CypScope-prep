# CypScope-prep

Preparatory FASTQ extraction and ONT alignment on per-sample BAM files for [CypScope](https://github.com/Single-Molecule-Sequencing/CypScope).

This repository provides a suite of Python scripts to convert unmapped BAM files to FASTQ, align them with minimap2, and index the resulting BAMs — tailored for high-performance computing (HPC) environments using the Slurm workload manager.

## Manifest

- `bam2fq`: Generates `samtools fastq` commands for BAM to FASTQ conversion.
- `mapONT`: Generates `minimap2 | samtools sort` commands for ONT alignment with automatic thread allocation.
- `indexBAM`: Indexes all BAM files in a directory using parallel processing.

## Usage

### `bam2fq`

Generates `samtools fastq` commands for converting unmapped BAM files to FASTQ.

```bash
bam2fq -i <bam_directory> -o <output_directory>
```

- `-i, --input`: Directory containing input BAM files (required).
- `-o, --output`: Directory to store output FASTQ files (required). Created if it does not exist.
- `-t, --threads`: Total threads per `samtools fastq` process (default: `4`).
- `--cmd-file`: Path for the generated command file (default: `./bam2fq_cmd.txt`).

### `mapONT`

Generates piped `minimap2 | samtools sort` commands for ONT read alignment. Threads are split between minimap2 and samtools using a 3:1 ratio by default.

```bash
mapONT -i <fastq_directory> -o <output_directory> -r <reference_fasta>
```

- `-i, --input`: Directory containing FASTQ files (`*.fastq`, `*.fq`, `*.fastq.gz`, `*.fq.gz`) (required).
- `-o, --output`: Directory to store sorted BAM files (required). Created if it does not exist.
- `-r, --ref`: Path to the reference FASTA file (required).
- `-tm`: Threads for minimap2 (default: `6`).
- `-ts`: Threads for samtools sort (default: auto, `tm / 3`).
- `--cmd-file`: Path for the generated command file (default: `./mapONT_cmd.txt`).

Output files are named `{sample}.sorted.bam`. A SLURM CPU hint is printed to stdout after generation.

### `indexBAM`

Indexes BAM files in parallel on a single node.

```bash
indexBAM -i <bam_directory>
```

- `-i, --input`: Directory containing BAM files (required).
- `-t, --threads`: Total threads per `samtools index` process (default: `1`).
- `-j, --jobs`: Number of parallel jobs (default: auto, `ncpus / threads`).
- `--dry-run`: Print commands to stdout instead of running them.

## Workflow

The workflow converts unmapped BAM files into analysis-ready, sorted, and indexed BAM files.

```txt
Input: Directory of unmapped BAM files (e.g., 'uBAMs/')
  │
  ▼
┌──────────────────────────┐
│  1. Extract FASTQ files  │
└──────────────────────────┘
  │
  │ Input: 'uBAMs/'
  │ Output: 'FastQ/' directory with FASTQ files
  │
  ├─► bam2fq -i uBAMs -o FastQ
  │
  └─► Submit bam2fq_cmd.txt to Slurm
  │
  ▼
┌──────────────────────────────┐
│  2. Align and sort with ONT  │
└──────────────────────────────┘
  │
  │ Input: 'FastQ/', reference FASTA
  │ Output: 'Aligned/' directory with sorted BAMs
  │
  ├─► mapONT -i FastQ -o Aligned -r ref.fasta
  │
  └─► Submit mapONT_cmd.txt to Slurm
  │
  ▼
┌─────────────────────────┐
│  3. Index sorted BAMs   │
└─────────────────────────┘
  │
  │ Input: 'Aligned/'
  │ Output: '.bai' index files in 'Aligned/'
  │
  └─► nohup indexBAM -i Aligned -t 2 -j 8 > index.log 2>&1 &
  │
  ▼
Final Output: Sorted, indexed BAM files ready for CypScope.
```

### Step 1: Extract FASTQ from unmapped BAMs

Generate the `samtools fastq` commands. This creates a command file with one command per sample.

```bash
bam2fq -i uBAMs -o FastQ -t 4
```

[Slurmify](https://github.com/Vanishborn/Slurmify) is recommended for generating formatted sbatch scripts at scale.

Submit the commands to Slurm.

### Step 2: Align with minimap2 and sort

Generate the alignment commands. `mapONT` hardcodes `-ax map-ont` for Oxford Nanopore data.

```bash
mapONT -i FastQ -o Aligned -r /path/to/GRCh38.fasta -tm 6
```

The SLURM CPU hint from `mapONT` shows the number of CPUs to request per job:

```bash
# [mapONT] SLURM hint: request at least 8 CPUs per job (minimap2=6 + samtools=2)
```

Submit the commands to Slurm.

### Step 3: Index the sorted BAMs

After alignment jobs complete, index the BAM files. `indexBAM` runs directly (no Slurm submission needed) using parallel processes. It is recommended to run with `nohup`.

```bash
nohup indexBAM -i Aligned -t 2 -j 8 > index.log 2>&1 &
```

Alternatively, generate commands for Slurm submission:

```bash
indexBAM -i Aligned -t 2 --dry-run > index_cmd.txt
```
