# PGRP_Snakemake
PGRP_Snakemake

## Introduction

## Overview of pipeline





## Installation

Install the PGRP_Snakemake repo to a suitable directory using git clone.

```bash
git clone git@github.com:plantgenomicslab/PGRP_Snakemake.git
```

### Dependencies
- Trim Galore 0.6.7
- SRA Toolkit 2.11.0
- STAR
- HTseq 1.99.2
- Subread 2.0.1
- multiqc 1.11
- Snakemake 6.15.0
- parallel-fastq-dump 0.6.7
- Samtools 1.14
- ggplot2
- Trinity 2.13.2
- Graphviz
- Gffread
- TPMcalculator
- Bioconductor Qvalue

### Setting up a Conda environment 

We recommend setting up a conda environment for PGRP_Snakemake to easily install the above dependencies. The latest version of Miniconda can be installed [here](https://docs.conda.io/en/latest/miniconda.html#latest-miniconda-installer-links). Use the commands below to set up the PGRP_Snakemake environment.

```bash
conda create -n PGRP_Snakemake -c bioconda -c conda-forge python=3.7 mamba

conda activate PGRP_Snakemake

mamba install -c bioconda -c conda-forge -c anaconda trim-galore=0.6.7 sra-tools=2.11.0 STAR htseq=1.99.2 subread=2.0.1 multiqc=1.11 snakemake=7.5.0 parallel-fastq-dump=0.6.7 bioconductor-tximport samtools=1.14 r-ggplot2 trinity=2.13.2 hisat2 bioconductor-qvalue sambamba graphviz gffread tpmcalculator lxml

```
### Configuration

Configure SRA Toolkit. The following commands allow you to specify where large SRA files get stored and ensure that your connection doesn't time out when downloading data from NCBI's SRA database.

```bash
# Set up the root dierctory for SRA files
# Enter '4' in interactive editor. Then enter your path: [absolute_path]/PGRP_Snakemake/output
vdb-config  -i --interactive-mode textual

# Set timeout to 100 seconds
vdb-config -s /http/timeout/read=100000
```
The pipeline requires an indexed reference genome and GTF file as input. By default, the pipeline expects the reference genome and gtf file to be located in the ref/ directory. You can either create a directory called 'ref' to store the reference genome and GTF file or update config.json to use a different location for the reference materials.

```bash
# Add the ref directory to your working directory
mkdir ref && cd ref

# Download a reference genome fasta file using curl, wget, ...etc.
# Download the reference GTF or GFF file. If starting from a GFF, convert it to GTF using the following (make sure gff is unzipped)
gffread [GFF_file] -T -F --keep-exon-attrs -o [output_name].gtf

# Update config.json with the path to the GTF file
vim config.json
# update "ref/[GTF_name]" under GTFName

# Index the reference genome with STAR (make sure genome fasta is unzipped)
# Helps to run on a compute cluster (computationally expensive)
STAR  --runThreadN 48g --runMode genomeGenerate --genomeDir . --genomeFastaFiles [genome.fa] --sjdbGTFfile [reference.gtf] --sjdbOverhang 99   --genomeSAindexNbases 12
```
### Input Files
The pipeline requires two input files to run: sraRunsbyExperiment.tsv and contrasts.txt. Examples of each input file are provided in the examples/ directory.

**sraRunsbyExperiment.tsv** provides the pipleline with information about the data that you want to process. These can be either SRA runs or locally stored data. When downloading SRA data, there may be multiple SRA runs (SRR...) for each SRA experiment (SRX...) where experiments represents the sequencing performed on a particular sample. Experiments should be given meainingful titles to aid in the interpretation of the pipeline output. Finally, the relationships between replicates and treatments should be flushed out. See the example below for an example sraRunsbyExperiment.tsv input file.

```bash
# Example format of sraRunsbyExperiment.tsv

head sraRunsbyExperiment.tsv
Run	Experiment	Replicate	Treatment
SRR5210841	SRX2524297	ZT0_rep1	ZT0
SRR5210842	SRX2524297	ZT0_rep1	ZT0
SRR5210843	SRX2524297	ZT0_rep1	ZT0
SRR5210844	SRX2524298	ZT4_rep1	ZT4
SRR5210845	SRX2524298	ZT4_rep1	ZT4
SRR5210846	SRX2524298	ZT4_rep1	ZT4
SRR5210847	SRX2524299	ZT8_rep1	ZT8
SRR5210848	SRX2524299	ZT8_rep1	ZT8
SRR5210849	SRX2524299	ZT8_rep1	ZT8
```
Because these files can tedious to generate for projects with many samples, a script called joinSraRelations.py is provided. This script takes an SRA project id (SRP...) as input and fetches the associated run information over the SRA API. The final two inputs are regex expressions to parse out the human readable treatment/replicate text from the full SRA run titles. See the example below for usage:

```bash
# Example usage for SRA project SRP098160
# Full run titles for this project take the form: 'GSM2471308: ZT0_rep1; Glycine max; RNA-Seq'
# Experiment titles will be of the form 'ZT0_rep1'

./joinSraRelations.py SRP098160 "ZT\d{1,2}_rep\d" "ZT\d{1,2}"
```
**contrasts.txt** provides tab separated contrasts to be used in differential gene expression analysis using DESeq2. The first column defines treatments and the second column defines replicates associated with each treatment. Pairwise differnetially expressed genes will be computed for each possible combination of treatments. This file must be generated by the user, see the example below:

```bash
# Example format of contrasts.txt

cat contrasts.txt
ZT0	ZT0_rep1
ZT0	ZT0_rep2
ZT0	ZT0_rep3
ZT4	ZT4_rep1
ZT4	ZT4_rep2
ZT4	ZT4_rep3
ZT8	ZT8_rep1
ZT8	ZT8_rep2
ZT8	ZT8_rep3
```

### If Runnning locally stored data...

Set up the output folder structure by runnning:
```bash
snakemake --snakefile Snakefile_local -np
```
Familiarize yourself with the output folder structure in the output/ directory. You will need to copy your raw, paired-end fastq data into 

```bash
# Move raw data into output folder structure (modify to suit needs)
cut -f 1 RunsbyExperiment.tsv  | sed 1d | while read rep; do 
  cp "/path/to/raw/data/${rep}-R1.fastq.gz" "output/${rep}/${rep}/raw/"
  cp "/path/to/raw/data/${rep}-R2.fastq.gz" "output/${rep}/${rep}/raw/"
done
```

## Running the pipeline 

### Inputs
The pipeline requires two input files to run: sraRunsbyExperiment.tsv and contrasts.txt.

sraRunsbyExperiment.tsv provides the pipleline with information about the SRA runs that you want to process. There may be multiple SRA runs (SRR...) for each SRA experiment (SRX...) and each experiment represents the sequencing performed on a particular sample. Experiments should be given meainingful titles to aid in the interpretation of the pipeline output. Finally, the relationships between replicates and treatments should be flushed out. See the example below for an example of a valid sraRunsbyExperiment.tsv input file.

```bash
# Example format of sraRunsbyExperiment.tsv

head sraRunsbyExperiment.tsv
Run	Experiment	Replicate	Treatment
SRR5210841	SRX2524297	ZT0_rep1	ZT0
SRR5210842	SRX2524297	ZT0_rep1	ZT0
SRR5210843	SRX2524297	ZT0_rep1	ZT0
SRR5210844	SRX2524298	ZT4_rep1	ZT4
SRR5210845	SRX2524298	ZT4_rep1	ZT4
SRR5210846	SRX2524298	ZT4_rep1	ZT4
SRR5210847	SRX2524299	ZT8_rep1	ZT8
SRR5210848	SRX2524299	ZT8_rep1	ZT8
SRR5210849	SRX2524299	ZT8_rep1	ZT8
```
Because these files can tedious to generate for projects with many samples, a script called joinSraRelations.py is provided. This script takes an SRA project id (SRP...) as input and fetches the associated run information over the SRA API. The final two inputs are regex expressions to parse out the human readable treatment/replicate text from the full SRA run titles. See the example below for usage:

```bash
# Example usage for SRA project SRP098160
# Full run titles for this project take the form: 'GSM2471308: ZT0_rep1; Glycine max; RNA-Seq'

./joinSraRelations.py SRP098160 "ZT\d{1,2}_rep\d" "ZT\d{1,2}"
```

contrasts.txt provides tab separated contrasts to be used in differential gene expression analysis using DESeq2. This file must be generated by the user. See the example below:
```bash
# Example format of contrasts.txt

cat contrasts.txt
ZT0	ZT0_rep1
ZT0	ZT0_rep2
ZT0	ZT0_rep3
ZT4	ZT4_rep1
ZT4	ZT4_rep2
ZT4	ZT4_rep3
ZT8	ZT8_rep1
ZT8	ZT8_rep2
ZT8	ZT8_rep3
```

### Running without scheduler
```bash
# Check the pipeline prior to run
snakemake -np

# Visualize the pipeline as a DAG
snakemake --dag [output] | dot -Tpdf -Gnodesep=0.75 -Granksep=0.75 > dag.pdf

# Run the pipeline 
snakemake --cores [available cores]
```

### Running with SLURM scheduler

```bash
sbatch --mem=16g -c 4 --time=13-11:00:00 -o snakemake.out -e snakemake.err --wrap="./run.sh"
```

## Citations

