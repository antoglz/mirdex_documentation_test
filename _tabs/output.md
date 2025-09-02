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


## Quality Control (QC)

The **quality control (QC)** stage evaluates sequencing read quality both before and after trimming. QC is performed with FastQC for per-library assessments and MultiQC for aggregated project-level summaries.

### FastQC
The quality assessment of **individual libraries** is performed using [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). FastQC runs a series of diagnostic checks on raw and trimmed reads, including per-base sequence quality, GC content, sequence length distribution, overrepresented sequences, and adapter contamination. Running FastQC on both pre- and post-trimmed datasets allows direct comparison to verify sequencing quality and to assess the effectiveness of the trimming process.

##### Output files
This step produces one HTML report and one compressed archive per library, both containing the results of all FastQC quality metrics.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
│    ├── 01-FastQC/<SPECIES>/<PROJECT>/
│    │    ├── *.fastqc.html
│    │    └── *.fastqc.zip
│    └── 02-MultiQC
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `*.fastqc.html` — Interactive HTML report containing plots and tables for all FastQC quality checks, viewable in any web browser.
- `*.fastqc.zip` — Compressed directory containing the raw data and images from the FastQC analysis, including the fastqc_data.txt file with all numerical metrics.

### MultiQC

[MultiQC](https://multiqc.info/) compiles all individual FastQC reports into a single aggregated HTML report. This global summary provides an at-a-glance overview of sequencing quality across all libraries, enabling rapid detection of systematic issues, batch effects, or quality trends that might affect downstream analyses.

##### Output files

MultiQC generates one interactive HTML report at the **project level**.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
│    ├── 01-FastQC
│    └── 02-MultiQC/<SPECIES>/<PROJECT>/
│         └── *_report.html
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `*_report.html` — Aggregated, interactive HTML report combining all individual FastQC outputs. Includes side-by-side comparisons of quality metrics, summary tables, and plots across all libraries in the project.

## Validation

The **validation** stage verifies that datasets meet minimum quality and replication standards before proceeding to Differential Expression Analysis (DEA). This step ensures that downstream statistical analyses are based on sufficiently deep and adequately replicated datasets. Two distinct validation modes are implemented depending on the input type: **Libraries validation** and **Count matrix validation**. In both modes, validation operates at the analysis group level, ensuring that all sample groups to be compared in DEA satisfy the thresholds specified by the user.

### Libraries validation (*Option 1*)

The **libraries validation** step evaluates the suitability of both individual libraries and their corresponding analysis groups for inclusion in downstream analyses. Two independent criteria are applied: the **sequencing depth** of each library and the **minimum number of biological replicates** per sample group within an analysis group. These thresholds are defined by the user via the `--validation_depth` and `--validation_rep` parameters, respectively.

Validation ensures that only analysis groups meeting both criteria — sufficient sequencing depth in all relevant libraries and adequate replication in all compared sample groups — are carried forward into DEA.

##### Output files
This step produces two tab-delimited reports — one detailing library-level validation (depth and replicates) and another summarising analysis group validity based on these criteria. Libraries or groups failing validation are excluded from subsequent steps.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation/<SPECIES>/<PROJECT>/
│    ├── *sum_libraries.tsv
│    └── *sum_project.tsv
├── 04-Filtering
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `*sum_libraries.tsv` — Lists each processed library with its validation status. Columns:
  - `File` — Library identifier (matches the Run column in the metadata).
  - `Depth` — Total number of reads in the library.
  - `Depth_validity` — `valid` if the library meets the minimum depth threshold, `not-valid` otherwise.
  - `Replicates_validity` — `valid` if the library belongs to a sample group with at least the required number of biological replicates, and the other group in the planned DEA comparison also meets this requirement; `not-valid` otherwise.

```plaintext
File	Depth	Depth_validity	Replicates_validity
SRR14182731.fastp.fastq.gz	19042524	valid	valid
SRR14182732.fastp.fastq.gz	12561071	valid	valid
SRR14182733.fastp.fastq.gz	18047262	valid	valid
SRR14182734.fastp.fastq.gz	14794553	valid	valid
SRR14182735.fastp.fastq.gz	16977623	valid	valid
SRR14182736.fastp.fastq.gz	18485787	valid	valid
SRR14182738.fastp.fastq.gz	17822264	valid	valid
SRR14182739.fastp.fastq.gz	12127819	valid	valid
SRR14182740.fastp.fastq.gz	11545483	valid	valid
SRR14182741.fastp.fastq.gz	6659745	valid	valid
SRR14182742.fastp.fastq.gz	8379530	valid	valid
SRR14182743.fastp.fastq.gz	16865976	valid	valid
SRR14182747.fastp.fastq.gz	16022716	valid	valid
SRR14182749.fastp.fastq.gz	17600922	valid	valid
SRR14182750.fastp.fastq.gz	15849560	valid	valid
```

- `*sum_project.tsv` — Aggregates validation results across libraries within each analysis group. Columns:
  - `Project` — Project identifier (Project column in the metadata).
  - `Group_id` — Analysis group identifier (Group column in the metadata).
  - `Number_valid_files` — Number of libraries passing both depth and replicate requirements.
  - `Number_not_valid_files` — Number of libraries failing one or both requirements.
  - `Group_validity` — `valid` if the analysis group contains at least two sample groups meeting the minimum replicate threshold and sequencing depth requirement; `not-valid` otherwise.

```plaintext
Project	Group_id	Number_valid_files	Number_not_valid_files	Group_validity
PROJECT1	1	2	4	not-valid
PROJECT1	2	0	6	not-valid
PROJECT1	3	1	5	not-valid
PROJECT1	4	0	6	not-valid
PROJECT1	5	3	3	not-valid
PROJECT1	6	6	0	valid
PROJECT1	7	6	0	valid
PROJECT1	8	6	0	valid
PROJECT1	9	6	0	valid
```

### Count matrix validation (*Option 2*)

The **count matrix validation** step is applied when the input provided in the samplesheet is a pre-computed count matrix for a specific analysis group (`--from_counts`) rather than raw sequencing libraries. In this scenario, sequencing depth is not applicable, and validation instead considers two main aspects. First, the **structural integrity** of the matrix is checked to ensure that the file follows the expected format, including correct column names and appropriate data types. Second, the **number of biological replicates** in each sample group intended for DEA is verified against the user-defined threshold (`--validation_rep`). An analysis group passes validation only if the matrix structure is correct and all sample groups satisfy the replicate requirement, and any group failing either criterion is excluded from DEA.


##### Output files
This step produces two tab-delimited reports — one detailing library-level validation (depth and replicates) and another summarising analysis group validity based on these criteria. Libraries or groups failing validation are excluded from subsequent steps.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation/<SPECIES>/<PROJECT>/
│    └── *sum.tsv
├── 04-Filtering
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `*sum.tsv` — Aggregates validation results across libraries within each analysis group. Columns:
  - `Group` — Full analysis group identifier formed by concatenating the project identifier and the group_id.
  - `Project` — Project identifier (`Project` column in the metadata).
  - `Group_id` — Analysis group identifier (`Group` column in the metadata).
  - `Group_validity` — Indicates whether the group meets the validation criteria (`valid`) or fails (`not-valid`)
  - `Exclusion_reason` — Specifies the reason why a group is marked as `not-valid` (`NA` if the group is `valid`)
  - `Number_valid_samples` — Number of samples passing both depth and replicate requirements.
  - `Number_not_valid_samples` — Number of samples failing one or both requirements.

```
Group	Project	Group_id	Group_validity	Exclusion_reason	Number_valid_samples	Number_not_valid_samples
PROJECT1_1	PROJECT1	1	not-valid	replicates	2	4
```

## Filtering

The **filtering** stage aims to remove undesired sequences and retain only those relevant for downstream analysis, using [Bowtie](https://bowtie-bio.sourceforge.net/manual.shtml) — an ultrafast, memory-efficient short-read aligner that leverages the Burrows–Wheeler index to minimise memory usage while maintaining high mapping speed. In mirDeX-nf, Bowtie runs in single-end mode and supports full multi-threading, enabling the rapid alignment of millions of small RNA reads. This step can be applied in two modes: **Database** and **Genome** filtering.

Both modes produce, for each library, an alignment summary file along with compressed FASTQ files containing reads classified as either aligned (retained or discarded, depending on the mode) or unaligned. Additional Bowtie parameters can be passed using `--bowtie_filt_db_ext_args` or `--bowtie_filt_genome_ext_args`.


### Database Filtering

In this mode, reads are aligned against a user-defined reference sequence **database** specified with `--filt_db /path/to/database.fasta`. The database typically contains unwanted sequences such as rRNA, tRNA, snRNA, snoRNA, etc. Reads mapping to this database are discarded, and only the unaligned reads are passed to downstream analysis. Additional alignment parameters for Bowtie can be provided via `--bowtie_filt_db_ext_args`.

##### Output files

This *optional* step generates, for each library, an alignment report and two FASTQ files containing reads classified as **aligned** (discarded) or **unaligned** (retained for further analysis).

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation
├── 04-Filtering
│    ├── 01-Database_filtering/<SPECIES>/<PROJECT>/
│    │    ├── 01-Alignment_out/
│    │    │    └── *.out
│    │    └── 02-Libraries/
│    │         ├── *.database.aligned.fastq.gz
│    │         └── *.database.unaligned.fastq.gz
│    └── 02-Genome_filtering/
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `01-Alignment_out/*.out` — Alignment summary generated by Bowtie for each library, reporting total reads processed, number and percentage of reads aligned, and other alignment statistics.

```plaintext
## reads processed: 8754320
## reads with at least one alignment: 201614 (2.30%)
## reads that failed to align: 8552706 (97.70%)
Reported 201614 alignments
```

- `02-Libraries/*.database.aligned.fastq.gz` — Compressed FASTQ file containing reads that aligned to the filtering database (discarded in downstream analysis).

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

- `02-Libraries/*.database.unaligned.fastq.gz` — Compressed FASTQ file containing reads that did not align to the database (retained for downstream analysis).

```plaintext
@SEQ_ID_3
TTGACAGAAGATAGAGAGCAC
+
IIIIIIIIIIIIIIIIIIII
@SEQ_ID_4
TGACAGAAGAGAGTGAGCAC
+
IIIIIIIIIIIIIIIIIIII
...
```

### Genome Filtering

In this mode, reads are aligned to the reference **genome** specified in the Genome column of the samplesheet when `--filt_genome` is enabled. Reads mapping to the genome are retained, while unaligned reads are discarded, ensuring that only sequences originating from the target species proceed to downstream steps. Additional Bowtie parameters can be provided via `--bowtie_filt_genome_ext_args`.

##### Output files
This *optional* step produces an alignment summary and two FASTQ files per library, containing reads classified as **aligned** (retained) or **unaligned** (discarded).

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation
├── 04-Filtering
│    ├── 01-Database_filtering
│    └── 02-Genome_filtering/<SPECIES>/<PROJECT>/
│         ├── 01-Alignment_out/
│         │    └── *.out
│         └── 02-Libraries/
│              ├── *.genome.aligned.fastq.gz
│              └── *.genome.unaligned.fastq.gz
├── 05-Quantification
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `01-Alignment_out/*.out` — Alignment summary from Bowtie for each library, detailing processed reads, number and percentage aligned, and additional statistics.

```plaintext
## reads processed: 8754320
## reads with at least one alignment: 201614 (2.30%)
## reads that failed to align: 8552706 (97.70%)
Reported 201614 alignments
```

- `02-Libraries/*.genome.aligned.fastq.gz` — Compressed FASTQ file containing reads that aligned to the genome (retained in downstream analysis).


```plaintext
@SEQ_ID_4
TGACAGAAGAGAGTGAGCAC
+
IIIIIIIIIIIIIIIIIIII
@SEQ_ID_5
TTGACAGATTATAGAGAGCAC
+
IIIIIIIIIIIIIIIIIIII
...
```

- `02-Libraries/*.genome.unaligned.fastq.gz` — Compressed FASTQ file containing reads that did not align to the genome (discarded in downstream analysis).

```plaintext
@SEQ_ID_3
TTGACAGAAGATAGAGAGCAC
+
IIIIIIIIIIIIIIIIIIII
@SEQ_ID_7
TGACAGTTGAGAGTGAGCAC
+
IIIIIIIIIIIIIIIIIIII
...
```

## Quantification

The quantification step measures the abundance of each unique small RNA sequence detected in the dataset and structures this information for downstream Differential Expression Analysis (DEA). It produces two main outputs: **per-library read counts**, which provide absolute or normalised counts for each sequence in individual libraries, and **analysis group-specific count matrices**, which combine counts from multiple samples belonging to the same analysis group.

### Per-Library Sequence Quantification

For each library, the pipeline quantifies the **abundance of every unique sequence** present in the preprocessed reads. This is achieved by counting the number of times each sequence occurs, producing raw counts that represent the absolute sequencing depth for that sequence within the library. Optionally, counts can be expressed as **Reads Per Million (RPM)**, where counts are scaled by the total number of reads in the library. Raw counts are always generated for downstream DEA, while RPM values are only produced if explicitly requested.

##### Output files

This step produces **one file per library** containing the abundance values for every detected sequence.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
│    ├── 01-Counts_per_sample
│    │    ├── 01-Raw/<SPECIES>/<PROJECT>/
│    │    │    └── *.raw.tsv
│    │    └── 02-RPM/<SPECIES>/<PROJECT>/
│    │         └── *.rpm.tsv
│    ├── 02-Counts_matrix
│    └── 03-Samplesheet
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `01-Raw/<SPECIES>/<PROJECT>/*.raw.tsv` — Table reporting the **raw read counts** for all sequences detected in a single library. The first column (`seq`) contains the nucleotide sequence of the small RNA, while the second column (`raw`) provides the corresponding raw counts.

```plaintext
seq	raw
TTGACAGAAGATAGAGAGCAC	1555213
TGACAGAAGAGAGTGAGCAC	1034539
TGAAGCTGCCAGCATGATCTA	708522
...
```

- `02-RPM/<SPECIES>/<PROJECT>/*.rpm.tsv` — Table reporting the **Reads Per Million (RPM)** for all sequences in a single library (*optional*). The first column (`seq`) contains the nucleotide sequence, and the second column (`RPM`) gives the abundance value after normalisation to the total number of mapped reads in that library.

```plaintext
seq	RPM
TTGACAGAAGATAGAGAGCAC	177503.45
TGACAGAAGAGAGTGAGCAC	118076.98
TGAAGCTGCCAGCATGATCTA	80845.67
...
```

### Count Matrix Construction

For each **analysis group** defined in the metadata file (`Group` column), the pipeline merges the counts from all libraries belonging to that group into a single matrix. An analysis group represents a set of samples analysed together in a single DEA, typically containing at least two distinct experimental conditions (e.g. Control vs Treatment, Wild-type vs Mutant).

This grouping strategy enables the pipeline to handle multi-project or multi-condition datasets efficiently, generating separate matrices for each group and ensuring that comparisons are performed only between biologically relevant samples. Raw count matrices are required for DEA, while RPM matrices are optional and intended for visualisation or exploratory analyses outside DEA.

##### Output files

This step produces **one matrix per analysis group**, with either absolute counts or RPM-normalised counts for all samples in that group. Each matrix is a tab-delimited file where rows correspond to unique sequences and columns correspond to individual samples within the analysis group.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
│    ├── 01-Counts_per_sample
│    ├── 02-Counts_matrix
│    │    ├── 01-Raw/<SPECIES>/<PROJECT>/
│    │    │    └── *.raw.tsv
│    │    └── 02-RPM/<SPECIES>/<PROJECT>/
│    │         └── *.rpm.tsv
│    └── 03-Samplesheet
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `01-Raw/<SPECIES>/<PROJECT>/*.raw.tsv` — **Raw count matrix** for all samples in the analysis group. The first column (`seq`) contains the nucleotide sequence, and each subsequent column corresponds to a sample in the group, reporting the raw count for that sequence. Sample column names correspond to the `Run` identifiers in the metadata.
  
```plaintext
seq	SRR1848791	SRR1848792	SRR1848793
TGAAGCTGCCAGCATGATCTA	881696	793949	1008586
TTGACAGAAGATAGAGAGCAC	834634	780906	1046367
TGACAGAAGAGAGTGAGCAC	326492	373550	513835
...
```

- `02-RPM/<SPECIES>/<PROJECT>/*.rpm.tsv` — **RPM-normalised count matrix** for all samples in the analysis group (*optional*). The first column (`seq`) contains the nucleotide sequence, and each subsequent column corresponds to a sample in the group, reporting the normalised abundance to the total mapped reads of that sample. Sample column names correspond to the `Run` identifiers in the metadata.

```plaintext
seq	SRR1848791	SRR1848792	SRR1848793
TGAAGCTGCCAGCATGATCTA	100512.67	94587.21	112345.88
TTGACAGAAGATAGAGAGCAC	95123.44	92987.53	116223.56
TGACAGAAGAGAGTGAGCAC	37219.88	44410.56	57012.31
...
```

### Samplesheet Generation (*optional*)

In addition to the count matrices, the pipeline can optionally generate a **samplesheet at the quantification stage**. This file follows the same format as the standard pipeline input samplesheet and contains metadata entries for each analysis group, along with the file paths to the corresponding **raw count matrices**.
This functionality is designed to allow the workflow to be resumed directly from the quantification stage in a future run, without repeating earlier steps such as data download, trimming, filtering, or quality control. It is particularly useful when the first part of the pipeline is executed separately—e.g., to preprocess and quantify sequencing data—while deferring DEA and annotation to a later execution.

##### Output files

One samplesheet file is generated per project, located under 05-Quantification/03-Samplesheet/.

Directory structure:

```plaintext
outdir/
├── 00-Data_download
├── 01-Trimming
├── 02-QC
├── 03-Validation
├── 04-Filtering
├── 05-Quantification
│    ├── 01-Counts_per_sample
│    ├── 02-Counts_matrix
│    └── 03-Samplesheet
│         └── samplesheet_counts.tsv
├── 06-DEA
├── 07-Annotation
├── 08-Global_matrices
└── 09-Workflow_report
```

File description:

- `samplesheet_counts.tsv` — Tab-delimited file containing one row per count matrix generated during quantification. It is composed of five columns:
  - `Id` — Project identifier associated with the generated count matrix.
  - `File` — Path to the count matrix file stored within the output directory.
  - `Metadata` — Path to the metadata file specified for this project in the samplesheet used during the previous pipeline execution.
  - `Genome` — Path to the genome file of the species associated with this project, if it was provided in the previous execution’s samplesheet; otherwise, this field remains empty.
  - `Group` — Group identifier (`Group_id`) corresponding to the analysis group to which the generated count matrix belongs.

```plaintext
Id	File	Metadata	Genome	Group
PROJECT1	/path/to/PROJECT1_6.raw.tsv	/path/to/metadata.tsv	/path/to/genome.fa	6
...
```