---
layout: page
title: Getting started
icon: fas fa-tags
order: 1
toc: true
---

**mirDeX-nf** is a Nextflow-based pipeline for differential expression analysis of microRNAs (miRNAs) from small RNA sequencing (sRNA-seq) data. One of the key features of mirDeX-nf is its ability to process and analyze data from multiple comparisons, projects, or species simultaneously. Users only need to include the data for the projects they wish to analyze, and the pipeline will return results for all of them in a single run.

1. SRA download (`prefetch` and `fasterq-dump`)
2. Support for direct input: FASTQ or precomputed count matrices
3. Read preprocessing (FASTQ):
   - Adapter and quality trimming (`fastp`)
   - Quality control (`FastQC`, `MultiQC`)
   - Filtering against user-provided reference FASTA (`Bowtie`)
   - Genome alignment with user-supplied genome (`Bowtie`)
4. Validation of sequencing depth for each library and minimum number of biological replicates per condition.
5. Small RNA quantification:
   - Raw counts of small RNAs
   - Generation of raw count matrices
6. Exploratory and differential expression analysis:
   - Low-count filtering
   - Principal Component Analysis (PCA)
   - `DESeq2`
7. miRNA annotation and isomiR classification:
   - Initial alignment of sRNA sequences to reference mature and precursor miRNAs from `miRBase`, `sRNAanno` or `PmiREN` databases (`BLASTn`)
   - Classification of aligned sequences into:
     - Canonical miRNAs (perfect match to mature sequence)
     - Templated isomiRs (sequence variants that align perfectly with the precursor)
     - Non-templated isomiRs (variants with mismatches or additions not present in the precursor)
   - Re-alignment of non-templated sequences to the user-provided genome to discard perfect matches elsewhere (potential false positives) (`Bowtie`)
   - Abundance filtering based on minimum Reads Per Million (RPM) or relative abundance compared to the canonical miRNA:
     - Sequences passing the threshold are labeled as `PASS`
     - Sequences below threshold are labeled as `REJECT`
   - Final classification of detected isomiRs
   - Annotation of differentially expressed sRNAs to identify those corresponding to miRNAs or isomiRs
8. Global patterns of DE miRNAs
   - Generation of binary matrices indicating which miRNAs are differentially expressed across the analyzed conditions
   - Creation of log₂ fold-change matrices showing the direction and magnitude of expression changes
   - Summary provided at both individual miRNA and miRNA family levels
   - Designed to support global analysis and identification of shared or condition-specific expression patterns

This section guides you through **installation** and a minimal **first run** of miRdeX-nf.

## Installation

Before running **mirDeX-nf**, you need to install **Nextflow**, which is required to execute the workflow. Detailed installation instructions for Nextflow can be found here:

- [Nextflow installation](https://www.nextflow.io/docs/latest/getstarted.html)

In addition, you must have at least one supported execution environment installed: **Docker**, **Singularity**, or **Conda**. These environments provide the necessary containers or package management to run the pipeline reliably. Installation information for each is available at:

- [Docker installation](https://docs.docker.com/get-docker/)  
- [Singularity installation](https://docs.sylabs.io/guides/3.0/user-guide/installation.html)  
- [Conda installation](https://docs.conda.io/en/latest/)

After installing the prerequisites, the pipeline can be obtained directly from GitHub. You can either **download a release archive** or **clone the repository** to get all workflow scripts and configuration files required to run **miRdeX-nf**:

```bash
git clone https://github.com/<username>/mirDeX-nf.git
cd mirDeX-nf
```

## Usage

To run **miRdeX-nf**, you must provide a properly formatted samplesheet specifying the inputs and metadata for each analysis. The pipeline supports multiple input types, including FASTQ files, SRA accession lists, and raw count matrices. Only the FASTQ input format is shown below; for full details on all supported input types, see the usage documentation.

**samplesheet.tsv**:

```tsv
Id       File                    Metadata                   Genome             Group
sample1  data/sample1.fastq.gz   metadata/PROJECT1_meta.tsv genomes/ath.fa    
sample2  data/sample2.fastq.gz   metadata/PROJECT1_meta.tsv genomes/ath.fa    
sample3  data/sample3.fastq.gz   metadata/PROJECT1_meta.tsv genomes/ath.fa    
sample4  data/sample4.fastq.gz   metadata/PROJECT1_meta.tsv genomes/ath.fa    
sample5  data/sample5.fastq.gz   metadata/PROJECT1_meta.tsv genomes/ath.fa    
sample6  data/sample6.fastq.gz   metadata/PROJECT1_meta.tsv genomes/ath.fa    
```

In this example, each row represents a single-end FASTQ file associated with a sample to be analyzed. On the other hand, the columns represent:

- **Id**: FASTQ file identifier. It represents the sample ID and must match the Run column in the metadata file.
- **File**: Path to the input FASTQ file.
- **Metadata**: Path to a metadata file describing the experimental design (e.g., sample conditions, replicates).
- **Genome** *(optional)*: Path to a reference genome FASTA file. Required if genome-based filtering or isomiR annotation is enabled.
- **Group** *(optional)*: Used when input is a count matrix. Not required when using FASTQ files.

The pipeline can be executed as follows, specifying the input samplesheet, the output directory, and the execution environment (Docker, Singularity, or Conda):

```bash
nextflow run miRdeX-nf/main.nf
   --input <SAMPLESHEET>
   --outdir <OUTDIR>
   -profile <docker/singularity/.../>
```

For full details on all pipeline features, supported input types, and configurable parameters, it is strongly recommended to consult the usage documentation and the parameter documentation before running the workflow.

> [!Note]
> Make sure to select an execution profile compatible with your system (`docker`, `singularity`, or `conda`). The profile determines how dependencies and containers are managed during workflow execution.

{% include note.html content="This is my note. All the content I type here is treated as a single paragraph. <br/><br/> Now I'm typing on a  new line." %}

## Pipeline output

The pipeline produces a structured set of results, including processed libraries, quality control reports, quantification tables, differential expression results, and miRNA/isomiR annotations:

```text
outdir/
├── 00-Data_download                  # Raw FASTQ files downloaded from SRA or provided locally
├── 01-Trimming                       # Adapter- and quality-trimmed reads after preprocessing
├── 02-QC                             # Quality control results for raw and trimmed libraries
│    ├── 01-Raw                       # QC results for unprocessed (raw) reads
│    │    ├── 01-FastQC               # Per-library FastQC reports (raw reads)
│    │    └── 02-MultiQC              # Aggregated MultiQC summary for raw reads
│    └── 02-Trimmed                   # QC results for trimmed reads
│         ├── 01-FastQC               # Per-library FastQC reports (trimmed reads)
│         └── 02-MultiQC              # Aggregated MultiQC summary for trimmed reads
├── 03-Validation                     # Analysis group validation results (depth, replicates, matrix structure)
├── 04-Filtering                      # Removal of unwanted sequences and non-target species reads
│    ├── 01-Database_filtering        # Reads removed by alignment to unwanted sequence databases
│    └── 02-Genome_filtering          # Reads retained/removed based on reference genome alignment
├── 05-Quantification                 # Sequence abundance estimation per library and per group
│    ├── 01-Counts_per_sample         # Raw/RPM counts for each unique sequence in each library
│    ├── 02-Counts_matrix             # Raw/RPM count matrices for each analysis group
│    └── 03-Samplesheet               # Auto-generated input sheets to resume workflow from quantification
├── 06-DEA                            # Differential expression analysis outputs
│    ├── 01-Exploratory_analysis      # PCA, clustering, variance profiling, and outlier detection
│    └── 02-DESeq2                    # DESeq2 statistical results and diagnostic plots
├── 07-Annotation                     # miRNA/isomiR identification and annotation outputs
│    ├── 01-BLAST                     # BLASTn alignments to reference miRNA databases
│    ├── 02-miRNA_isomiR_annotation   # Preliminary per-library miRNA/isomiR annotations
│    └── 03-DEA_annotation            # Fully annotated DEA results (significant sequences)
├── 08-Global_matrices                # Global miRNA family expression matrices and mapping files
└── 09-Workflow_report                # Workflow execution reports, logs, and consolidated summaries
```

For a **comprehensive description of the generated outputs and accompanying reports**, please refer to the output documentation.
