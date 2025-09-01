---
layout: page
title: Usage
icon: fas fa-tags
order: 2
toc: true
---

# Usage
# Options
## Pipeline parameters

Pipeline parameters must be specified explicitly either on the command line(CLI) or via a parameter file provided with the `-params-file` option.
Execution-related settings (e.g., resource allocation, execution queue, container engine) may be configured in a custom Nextflow configuration file using the `-c` option, or by selecting an appropriate execution profile with `-profile`.

## Samplesheet input

The `samplesheet.tsv ` file is a mandatory input that specifies the samples to be processed by miRdeX-nf. Its structure and required fields depend on both the type of input data and the intended scope of the analysis. The location of the file must be provided using the `--input` parameter when launching the workflow:

```
--input '[path to samplesheet file]'
```

The samplesheet must be a tab-delimited text file with a header row, containing one entry per sample (or input set). The workflow supports the following input file types:
- **FASTQ files** (`.fastq` / `.fastq.gz`) – For analyses starting from raw sequencing libraries.
- **SRA accession lists** (`.txt`) – For automated retrieval of public sequencing libraries from the SRA database.
- **Raw count matrices** (`.tsv`) – For analyses starting from pre-computed sRNA expression matrices. This mode skips all read processing stages and executes only the downstream analysis phases (Differential Expression Analysis, miRNA/IsomiR Identification and Annotation, Global miRNA Family Summary).

The `samplesheet.tsv` file must contain the following columns:

| Column     | Description |
|------------|-------------|
| `Id`     | Identifier for the sample or project. For FASTQ input, this must match the `Run` column in the metadata file. For SRA accessions or count matrices, this must match the `Project` column. |
| `File`   | Path to the input file. May be a FASTQ file (`.fastq`, `.fastq.gz`), an SRA accession list (`.txt`), or a raw count matrix (`.tsv`). |
| `Metadata` | Path to a metadata TSV file describing the experimental design, including sample conditions, biological replicates, the statistical test to apply, and the design formula for DEA. |
| `Genome` (*Optional*) | Path to the reference genome FASTA file. Required when genome-based filtering or isomiR annotation is enabled. |
| `Group` (*Optional*)  | Defines experimental groups or contrasts. This field is only required when the input is a count matrix. |


## Options

This section details the configurable **options** available in the pipeline, grouped by functional category for ease of reference.

### Execution mode options

These parameters **control how the pipeline is executed**, allowing partial runs or resuming from intermediate steps. The `--only_preprocessing` parameter limits the workflow to trimming, validation, and quality control with FastQC and MultiQC, leaving quantification, DEA, and annotation out of the run. A more advanced variant, `--only_preprocessing_and_counts`, performs the same preprocessing steps but also carries out quantification, producing raw count matrices and an accompanying samplesheet. This generated samplesheet is designed to serve as input for a future run using the `--from_counts` option, which instructs the pipeline to skip preprocessing and quantification entirely and begin directly at the DEA stage using precomputed count tables. In this way, `--only_preprocessing_and_counts` and `--from_counts` can be used together in separate runs to break the analysis into distinct stages.

> [!NOTE]  
> You can use your own count matrices directly with the `--from_counts` parameter, but you must ensure they follow the required structure.


### Trimming options

Trimming controls how sequencing **reads** are **cleaned** and prepared before downstream analysis. The `--trimming_adapters` parameter points to a FASTA file containing adapter sequences that must be removed, with a default path of `${projectDir}/assets/illumina_adapters.fa`. The trimming process uses **fastp** with a set of predefined options: quality-based clipping is applied at both ends (`--cut_right` and `--cut_front`), using a right window size of 4 with a mean quality threshold of 20, and a front window size of 1 with a mean quality threshold of 3. Poly-X tails of length 10 or more are removed with `--trim_poly_x` and `--poly_x_min_len` 10. Read length constraints are enforced through fastp’s `--length_required` and `--length_limit` options, while `--n_base_limit` 0 ensures that reads containing ambiguous bases are discarded. The values used for these length constraints are determined by the pipeline parameters `--trimming_min_len` and `--trimming_max_len`, which can be adjusted by the user. Custom fastp options may be supplied via the pipeline parameter `--trimming_custom_args`, allowing the default behavior to be extended or overridden when specific trimming requirements are needed.

> [!NOTE]
> **Adapter sequences** should match the technology and library preparation protocol used in your experiment. Using the wrong adapter file can lead to loss of valid reads or incomplete trimming. **Custom arguments** must follow the syntax and options supported by fastp. Refer to the official fastp documentation for valid parameters.


### FastQC/MultiQC options

These parameters allow you to customize the **quality control** steps carried out by FastQC and MultiQC. The `--extra_fastqc_args` parameter can be used to pass additional command-line options to FastQC, enabling you to adjust quality analysis settings beyond the defaults. Similarly, `--extra_multiqc_args` lets you provide extra arguments to MultiQC, allowing greater control over how reports are generated, including the ability to incorporate extra data sources or modify the report’s appearance.

> [!NOTE]
> Extra arguments must follow the syntax and options supported by each tool. Refer to the official FastQC documentation and MultiQC documentation for valid parameters.


### Validation options

The validation stage ensures that datasets meet the **minimum quality and replication standards** before being included in Differential Expression Analysis (DEA). The process operates at the analysis group level and uses thresholds defined by the pipeline parameters `--validation_depth` and `--validation_rep`.

When the input consists of **raw sequencing libraries**, the procedure checks each library’s sequencing depth against the `--validation_depth` threshold and verifies that all sample groups within each planned comparison have at least the number of biological replicates specified by `--validation_rep.`. Both conditions must be met for an analysis group to be considered valid, and groups failing either are excluded from DEA.

When the input is a **precomputed count matrix**, as in runs started with `--from_counts`, sequencing depth is no longer relevant. Instead, validation first checks the structural integrity of the matrix to ensure it matches the expected format and metadata mapping, and then applies the replicate requirement using the `--validation_rep `parameter. Groups with incorrectly formatted matrices or insufficient replication are marked as invalid and excluded from further analysis.

### Filtering options

Filtering **removes unwanted sequences** from the dataset before quantification by aligning reads either to a reference database or to the species genome.

When a FASTA file path is provided in the pipeline parameter `--filt_db`, database-based filtering is performed. In this mode, Bowtie aligns reads against the specified reference and, by default, runs with the `-v 0` setting, meaning only perfect matches are considered. **Reads that match the database are removed** from the dataset, effectively eliminating sequences that originate from known contaminants or unwanted sources.

When `--filt_genome` is enabled, genome-based filtering is applied using the reference genome specified in the `Genome` column of the input samplesheet. In this case, Bowtie runs with a default setting of `-v 2`, allowing up to two mismatches. **Reads that align to the genome are retained**, while unaligned reads are discarded, ensuring that only sequences originating from the target species proceed to downstream steps.

Both filtering modes can be fine-tuned by supplying additional Bowtie parameters via `--bowtie_filt_db_ext_args` and `--bowtie_filt_genome_ext_args`, respectively. Adjusting these options allows users to control alignment stringency, which can be important when dealing with highly similar sequences or when a more permissive filter is desirable.

> [!NOTE]  
> Modifying Bowtie parameters can greatly change filtering behavior. Relaxing the mismatch threshold or adding permissive options may increase the number of sequences retained but also risks passing contaminants, while overly strict settings could remove biologically relevant reads.


### Quantification options

The **quantification** stage measures the abundance of each unique small RNA sequence detected after preprocessing and filtering, structuring the results for downstream Differential Expression Analysis (DEA). For each sequencing library, the pipeline counts every occurrence of each unique sequence, producing per-library raw counts that represent the absolute abundance of each sequence within a sample. These raw counts are then aggregated by analysis group, as defined in the `Group` column of the input metadata, to create raw count matrices combining data from all samples in the same group. Raw count matrices are required for DEA and are always generated.

When the parameter `--calculate_rpm` is enabled, the pipeline also generates RPM-normalised matrices in which counts are scaled to Reads Per Million based on the total number of mapped reads in each sample. This produces both per-library RPM tables and RPM matrices for each analysis group.

**Filtering of low-abundance sequences** before DEA is controlled by `--min_counts` and `--min_samples`. The first sets the minimum number of reads a sequence must have in a single sample to be retained (default: `5`), while the second specifies the minimum number of samples in which that threshold must be met (default: `3`). Together, these criteria help reduce noise by excluding sequences that are rare or inconsistently detected across sampl
es.

The parameter `--counts_not_memory` modifies how the SQLite database used for count matrix construction is handled. By default, this database is loaded into memory to speed up processing, but enabling --counts_not_memory keeps it on disk, reducing memory usage at the cost of longer runtimes.


### Differential Expression Analysis options
Differential Expression Analysis (DEA) in this pipeline is performed with **DESeq2**, which models count data using negative binomial distributions, estimates size factors and dispersion, and applies shrinkage to improve log₂ fold-change estimation. Each analysis group, as defined in the `Group` column of the input metadata, undergoes two phases: **Exploratory Analysis** and **Statistical Testing**.

During **exploratory analysis**, the pipeline evaluates sample relationships and variance patterns before formal testing. **Principal Component Analysis (PCA)** is applied to variance-stabilised data, and an optional clustering significance test compares intra-condition and inter-condition distances using a **Mann–Whitney–Wilcoxon (MWW)** test. The threshold for this test is set with `--ea_p_value ` (default: 0.05). When this parameter is specified, the result of the MWW test is used to **filter analysis groups**: only those where samples cluster tightly within conditions and show clear separation between conditions are retained for downstream steps, while groups failing this criterion are excluded.

In the **statistical testing** phase, DESeq2 identifies sequences with significant differential expression between experimental conditions. Significance is defined by `--dea_alpha` (default: `0.05`), which sets the adjusted p-value cutoff and is also used by DESeq2 for independent filtering. Effect size constraints are introduced with `--log2fc_threshold` (default: 0), modifying the Wald test null hypothesis so that only sequences with an absolute log₂ fold-change above this threshold can be significant. All significance and effect size thresholds are applied within DESeq2’s `results()` function, ensuring consistency between reported p-values, adjusted p-values, and the set significance criteria.

> [!Warning]  
> Filtering analysis groups at the exploratory stage based on the MWW test (`--ea_p_value`) is generally not recommended. This can lead to the loss of biologically meaningful results, as excluded groups will not proceed to any subsequent steps in the pipeline. A safer approach is to run the full workflow on all groups and review clustering performance afterwards, allowing filtering decisions to be made with both exploratory and downstream results in hand.

> [!NOTE]  
> Setting a `--log2fc_threshold` makes the statistical test more conservative by modifying the null hypothesis itself, rather than applying a fold-change filter after testing. This means only features with an absolute log₂ fold-change above the specified value are considered significant during hypothesis testing, which typically results in fewer detected features but with stronger effect sizes.


### miRNA/IsomiR annotation Options

The **miRNA/IsomiR Annotation** stage identifies known miRNAs and their isomiR variants by aligning sequences against curated small RNA reference databases.

The pipeline supports three curated reference databases — **miRBase**, **sRNAanno**, and **PmiREN** — which can be listed in order of preference via the `--databases `parameter. Annotation is performed using a single database per species, determined by the first database in the user-specified list that contains miRNA and precursor sequences for that species. For example, with '`--databases 'mirbase,srnaanno,pmiren`', the pipeline will first download the databases, adjust species identifiers in the downloaded files to ensure compatibility, and then check whether the species in the dataset is present in miRBase. If not, it will proceed to sRNAanno, and finally to PmiREN. This database selection applies consistently to all datasets of the same species in the run. When `--reuse_dbs` is enabled, any previously downloaded and preprocessed databases in the output directory are reused instead of being downloaded again, reducing runtime in repeated analyses. Databases without entries for the target species are skipped automatically.

Once the appropriate database is selected, sequences are first **aligned to it using BLAST to identify known miRNAs and their isomiR variants**. They are then classified after alignment according to `--mirna_classes`, which defines the types of miRNA-related sequences to be retained in the analysis (e.g. canonical miRNAs, 5′- or 3′-templated isomiRs, non-templated variants). By default, all supported classes are included.

The classification stage then applies post-alignment tolerance thresholds: `--substitutions` defines the maximum number of internal mismatches allowed; `--five_add` and `--three_add` limit the number of non-templated nucleotides added at the 5′ and 3′ ends, respectively; and `--ends_modification` caps the total number of modifications across both ends. For sequences classified as **non-templated isomiRs**, an additional genomic alignment step is performed with Bowtie to check whether they could originate from other genomic regions. This Bowtie run uses a default strict `-v 0` setting, which can be modified with `--bowtie_nti_ext_args` to adjust stringency.

The annotation stage also includes **abundance-based filtering of isomiRs** (reference/canonical miRNAs are never filtered). Filters can be applied either to RPM values (`--min_rpm_filt`) or to the abundance of isomiRs relative to their associated canonical miRNA (`--min_relative_abundance_filt`). Each of these thresholds must be met in a minimum number of samples, set via `--min_samples_filt`. In the case of relative abundance filtering, isomiRs without a detected canonical counterpart in the same sample (orphan isomiRs) are automatically discarded.

> [!Tip]
> If you plan to run the pipeline multiple times with `--reuse_dbs` but prefer to keep the downloaded and preprocessed databases outside the outdir of the first execution, you can move them to another directory and reference their new paths using `--mirbase`, `--srnaanno` and `--pmiren`. In this case, you must also provide the `species_ids.csv` file generated in the initial run via `--species_ids`. **The pipeline will not work with databases that have not been generated by this pipeline**, as differences in identifier formats or preprocessing steps would cause execution problems.


### Global patterns of DE miRNAs options

When `--global_matrix` is enabled, the pipeline summarises Differential Expression Analysis (DEA) results across all analysis groups into two matrices: a **binary presence/absence matrix** indicating whether each miRNA family has at least one significantly differentially expressed member in a group, and an **effect-size matrix** reporting the shrunken log₂ fold-change of the most representative member.

In both matrices, rows correspond to miRNA families and columns correspond to analysis groups, represented by encoded identifiers constructed from the metadata fields specified in `--global_fields`. These identifiers are composed of numeric tokens, where each token corresponds to a specific metadata field (in the order provided to `--global_fields`) and represents the code assigned to its value. This encoding keeps column names concise while preserving the ability to fully reconstruct the metadata. The `--global_fields` parameter must reference valid columns in the input metadata for the encoding to work correctly.

## Running the pipeline

The pipeline can be launched in different execution modes depending on the type of input data and the desired scope of analysis. The following examples illustrate typical usage patterns.

### Full analysis from raw FASTQ or SRA data

Runs the complete workflow, starting from raw sequencing libraries (FASTQ or SRA accessions), performing preprocessing, quantification, differential expression analysis, and annotation.

```
nextflow run main.nf \
    --input samplesheet.csv \
    --outdir Output_directory/ \
    -profile docker
```
### Preprocessing only

Runs trimming, validation, and quality control steps, producing cleaned FASTQ files and MultiQC reports but skipping quantification and downstream analysis.

```
nextflow run main.nf \
    --input samplesheet.csv \
    --outdir Output_directory/ \
    --only_preprocessing \
    -profile docker 
```

### Preprocessing and quantification

Performs preprocessing and quantification, generating count matrices and a new samplesheet that can be reused in future runs starting from the counts.
```
nextflow run main.nf \
    --input samplesheet.csv \
    --outdir Output_directory/ \
    --only_preprocessing_and_counts \
    -profile docker 
```

### Starting from count matrices

Skips preprocessing and quantification entirely, starting directly from pre-computed raw count matrices to perform differential expression analysis and annotation.
```
nextflow run main.nf -profile docker --input counts_samplesheet.tsv --from_counts
nextflow run main.nf \
    --input samplesheet.csv \
    --outdir Output_directory/ \
    --from_counts \
    -profile docker 
```

### Running on Linux ARM architectures

The pipeline can be launched on ARM-based hardware, such as Apple Silicon Macs or ARM servers, by combining the `docker` and `arm` profiles. This enables the use of ARM-compatible containers and applies the necessary overrides to ensure compatibility with amd64 images when required.

```
nextflow run main.nf \
    --input samplesheet.tsv \
    --outdir Output_directory/ \
    -profile docker,arm
```

## Core Nextflow arguments

Before executing the pipeline, it is essential to understand several core Nextflow arguments that govern workflow execution independently of the pipeline’s own parameters. These arguments are part of Nextflow’s native interface and are applicable to any Nextflow workflow.

## -profile
The `-profile` option specifies one or more predefined configuration profiles declared in the `nextflow.config` file or in associated configuration modules.
Profiles define environment-specific settings, such as execution platform, container engine, resource allocation, and other operational parameters.

- `docker` - A profile to execute the pipeline entirely with Docker. This is the recommended option for most users.
- `singularity` - A profile to run the pipeline with Singularity containers. Useful in HPC systems where Docker is not available.
- `conda` - A profile to use Conda for dependency management. Provides an alternative to containers but is less reproducible.
- `arm` - A configuration profile for ARM-based systems such as Apple Silicon. Runs Docker in amd64 mode to ensure container compatibility. 

Example:
```
nextflow run main.nf -profile singularity
```
Executes the pipeline using the singularity profile for containerised execution

## -resume
The `-resume` option allows Nextflow to continue a previous execution by reusing cached results. Steps that have already been successfully completed with the same inputs will not be re-run, saving time and resources. Both the file names and their contents must be identical for the cache to be valid.

It is also possible to provide the name of a specific run, which makes it easier to resume a chosen execution. Previous run names can be listed with the nextflow log command.

## -c
The `-c` option specifies a custom configuration file to be used when launching the pipeline. This allows users to override or extend the default nextflow.config with their own settings for resources, execution profiles, or environment-specific parameters. This option is particularly useful in shared systems or clusters where different users may require different configurations without modifying the main pipeline files.