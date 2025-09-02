---
layout: page
title: Output
icon: fas fa-tags
order: 2
toc: true
---

## Pipeline Workflow Overview

The pipeline is built using [Nextflow](https://www.nextflow.io) and processes data using the following steps:

## Output Directory Layout

The pipeline output is organised to mirror the logical progression of the analysis, allowing each result to be traced back to the step that generated it. The directories listed below will be created in the `outdir` directory after the pipeline has finished.

```plaintext
outdir/
├── 00-Data_download                  ## Raw FASTQ files downloaded from SRA or provided locally
├── 01-Trimming                       ## Adapter- and quality-trimmed reads after preprocessing
├── 02-QC                             ## Quality control results for raw and trimmed libraries
│    ├── 01-Raw                       ## QC results for unprocessed (raw) reads
│    │    ├── 01-FastQC               ## Per-library FastQC reports (raw reads)
│    │    └── 02-MultiQC              ## Aggregated MultiQC summary for raw reads
│    └── 02-Trimmed                   ## QC results for trimmed reads
│         ├── 01-FastQC               ## Per-library FastQC reports (trimmed reads)
│         └── 02-MultiQC              ## Aggregated MultiQC summary for trimmed reads
├── 03-Validation                     ## Analysis group validation results (depth, replicates, matrix structure)
├── 04-Filtering                      ## Removal of unwanted sequences and non-target species reads
│    ├── 01-Database_filtering        ## Reads removed by alignment to unwanted sequence databases
│    └── 02-Genome_filtering          ## Reads retained/removed based on reference genome alignment
├── 05-Quantification                 ## Sequence abundance estimation per library and per group
│    ├── 01-Counts_per_sample         ## Raw/RPM counts for each unique sequence in each library
│    ├── 02-Counts_matrix             ## Raw/RPM count matrices for each analysis group
│    └── 03-Samplesheet               ## Auto-generated input sheets to resume workflow from quantification
├── 06-DEA                            ## Differential expression analysis outputs
│    ├── 01-Exploratory_analysis      ## PCA, clustering, variance profiling, and outlier detection
│    └── 02-DESeq2                    ## DESeq2 statistical results and diagnostic plots
├── 07-Annotation                     ## miRNA/isomiR identification and annotation outputs
│    ├── 01-BLAST                     ## BLASTn alignments to reference miRNA databases
│    ├── 02-miRNA_isomiR_annotation   ## Preliminary per-library miRNA/isomiR annotations
│    └── 03-DEA_annotation            ## Fully annotated DEA results (significant sequences)
├── 08-Global_matrices                ## Global miRNA family expression matrices and mapping files
└── 09-Workflow_report                ## Workflow execution reports, logs, and consolidated summaries
```

## Data Download
The data download stage retrieves raw sequencing data from the NCBI Sequence Read Archive (SRA) using two utilities from the [SRA Toolkit](https://github.com/ncbi/sra-tools): **prefetch** and **fasterq-dump**.
Prefetch downloads the SRA run files (.sra) corresponding to accession numbers provided in the project metadata. It validates file integrity by checking checksums after transfer. Fasterq-dump then converts the downloaded .sra files into FASTQ format, supporting multi-threaded execution for improved performance. The output FASTQ files are compressed for storage efficiency and serve as the primary input for downstream quality control and preprocessing.

##### Output files

This step produces one compressed FASTQ file per sequencing library for single-end data.

Directory structure:

```plaintext
outdir/
├── 00-Data_download/<SPECIES>/<PROJECT>/
│   └── *.fastq.gz
├── 01-Trimming
├── 02-QC
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `*.fastq.gz` — Raw sequencing reads in standard FASTQ format, compressed with gzip. Each file contains all reads for a given library. These files are the direct input for the Quality Control (QC) step.


## Trimming

**Trimming** is the process of removing unwanted sequence elements and low-quality regions from raw sequencing reads. This includes the removal of adapter sequences introduced during library preparation, trimming of low-quality bases from read ends, and the exclusion of reads that fall below a defined minimum length. By eliminating these artefacts, trimming enhances both read quality and mapping performance in later stages of the pipeline.

### fastp

The trimming step is implemented using [fastp](https://github.com/OpenGene/fastp), a high-performance, all-in-one FASTQ preprocessor written in C++ with full multithreading support. Fastp automatically detects and removes adapter sequences, trims low-quality bases from the ends of reads, and discards any reads shorter than the user-defined minimum length (`--min_read_length`). This ensures that only high-quality reads are passed to downstream steps.

##### Output files

This step produces one compressed FASTQ file per library containing the adapter- and quality-trimmed reads that will be used in subsequent analyses.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming/<SPECIES>/<PROJECT>/
│   └── *.fastp.fastq.gz
├── 02-QC
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `*.fastp.fastq.gz` — Compressed FASTQ file containing reads after adapter and quality trimming. Each entry retains the original sequence identifier but with bases at the ends removed according to quality thresholds and adapter detection. Reads shorter than the minimum length are excluded from this file.

```plaintext
@SEQ_ID_1
TGACAGAAGAGAGTGAGCAC
+
IIIIIIIIIIIIIIIIIIII
@SEQ_ID_2
TGAAGCTGCCAGCATGATCTA
+
IIIIIIIIIIIIIIIIIIII
...
```