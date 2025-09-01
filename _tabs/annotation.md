---
layout: page
title: Annotation
icon: fas fa-tags
order: 5
toc: true
---

# miRNA Annotation Workflow

This document provides a detailed overview of **miRNA and isomiR identification and classification** within the pipeline. Starting from pre-processed sequencing libraries, the workflow identifies canonical miRNAs and their variants (isomiRs) through a series of structured steps. The purpose of this document is to give users a clear understanding of **each stage of the annotation process and the criteria used to classify different variant types**.

The annotation workflow can be summarized in the following main stages:
1. [**miRNA database download and preparation**](#miRNA-database-download-and-preparation) – Download and prepare mature and precursor miRNA reference databases (miRBase, sRNAanno, PmiREN).
2. [**sRNA Sequence Alignment to miRNA databases**](#srna-sequence-alignment-to-mirna-databases) – Align sRNA sequences to mature and precursor miRNA databases.  
3. [**Precursor-based classification**](#precursor-based-classification) – Assign sequences as canonical miRNAs, templated isomiRs, or non-templated isomiRs.  
4. [**Non-templated isomiR filtering**](#non-templated-isomir-filtering) – Remove non-templated isomiRs that map to unrelated genomic regions.  
5. [**Variant classification**](#variant-classification) – Categorize isomiRs by type (5'/3' shifts, nucleotide additions, substitutions, etc.).  
6. [**Abundance filtering**](#abundance-filtering) – Retain only high-confidence isomiRs based on RPM or Relative Abundance.  
7. [**DEA integration**](#dea-integration) – Annotate sequences found to be differentially expressed according to their classification.


## miRNA Database Download and Preparation

The first step of the annotation workflow focuses on **acquiring and preparing reference data for miRNAs**. The pipeline draws from well-established databases, including **miRBase, PmiREN, and sRNAanno**. **miRBase** provides curated miRNA sequences from a wide range of species, covering both animals and plants, while **PmiREN** and **sRNAanno** focus specifically on plant miRNAs. Each database offers collections of both mature and precursor (hairpin) sequences along with species-specific identifiers, which are essential for accurate and consistent annotation. Users can control **which databases to include** in the analysis and **the order of preference** using the `--databases` option (e.g., `--databases 'mirbase,pmiren'`). This ensures that the pipeline uses reference data from the most appropriate sources according to the user’s specifications.

The preparation process includes:

1. **Database download** – The pipeline first downloads the databases specified by the user via the `--databases` option, ensuring that only the selected resources are retrieved.  

2. **Species identifier standarization** – To avoid inconsistencies across databases, species identifiers are standardized. For example, if one database lists a species as "idk" and another as "kdi", one identifier is chosen and all sequences in the databases are updated to match it. This ensures consistent species naming across all reference data.  

3. **Output generation** – The processed reference sequences are saved as FASTA files for each database and for each miRNA type (mature and precursor).

4. **Database assignment for species-specific data** – During the workflow, the data corresponding to each species are assigned to a database based on whether the species is present in the selected databases and according to the user-specified order of preference.

>[!Warning]
> If you run the pipeline multiple times using different sets of databases, species identifiers may not be consistent across executions. This is because the generation of identifiers depends on the specific databases selected for each run.


## sRNA Sequence Alignment to miRNA databases

The second step of the annotation workflow involves **aligning sRNA sequences from the sequencing libraries to reference miRNA databases**. This step is essential for identifying both **canonical miRNAs** and potential **isomiRs**. Alignments are performed separately against **mature** and **precursor (hairpin)** sequences from the selected reference databases. The workflow uses **BLASTn** with parameters optimized for short sequences, chosen to balance sensitivity and specificity:

- `-task blastn-short` – Optimized for aligning short sequences, such as sRNAs (~18–25 nt), improving detection of short miRNAs and isomiRs.
- `-word_size 13` – Sets the initial exact match word length. A smaller word size increases sensitivity for short sequences while avoiding excessive false positives.
- `-gapopen 5` – Sets the penalty for opening a gap, balancing between detecting variants with small indels and avoiding spurious alignments.
- `-gapextend 2` – Sets the penalty for extending an existing gap, fine-tuning detection of small insertions/deletions.
- `-strand plus` – Only considers alignments in the sense orientation, consistent with the expected orientation of sRNAs relative to miRNA sequences.
- `-evalue 0.01` – Defines the threshold for reporting matches, controlling the trade-off between sensitivity and noise.
  

VER COMO CITAR ESTO:

These settings are applied consistently for both **mature** and **precursor** miRNA alignments. This strategy is based on the approach described in [**A Comprehensive Approach to Sequence-oriented IsomiR annotation (CASMIR): demonstration with IsomiR profiling in colorectal neoplasia**](https://bmcgenomics.biomedcentral.com/articles/10.1186/s12864-018-4794-7), which demonstrated robust identification of miRNA variants using BLAST-based alignments.

These settings are applied consistently for both **mature** and These settings are applied consistently for both **mature** and **precursor** miRNA alignments. This strategy is based on the approach described by [Wu et al. (2018)](https://pubmed.ncbi.nlm.nih.gov/29801434/), which demonstrated robust identification of miRNA variants using BLAST-based alignments.



## Precursor-based Classification

Once the sequencing reads have been aligned to **mature** and **precursor** miRNA reference sequences, the next step is to classify them according to their relationship with the precursor structure. This step enables the distinction between **canonical miRNAs** and their sequence variants (**isomiRs**), which may arise due to alternative Dicer/Drosha cleavage or  post-transcriptional modifications.

IMAGEN

The classification relies on the alignment position of each read relative to the annotated mature and precursor sequences:

1. **Canonical miRNAs** – Reads that perfectly match the annotated mature miRNA sequence (or its star/3p/5p form) within the precursor hairpin. These represent the reference form of the miRNA.  

2. **Templated isomiRs** – Reads that align to the precursor sequence but differ slightly from the annotated mature miRNA. These differences are explained by **alternative cleavage** of Drosha/Dicer, leading to 5′ or 3′ shifts in the start/end positions. Since they remain fully templated by the precursor, they reflect biologically relevant processing variants.  

3. **Non-templated isomiRs** – Reads that extend beyond the precursor template or contain mismatches not explained by shifts in processing. These often correspond to **post-transcriptional modifications** (e.g., 3′ adenylation or uridylation) or technical noise. Their classification requires additional filtering steps (see the [next section](#non-templated-isomir-filtering)).

## Non-templated isomiR filtering

**Non-templated isomiRs** are sequences that differ from the canonical miRNA by additions, trimming, or nucleotide substitutions that cannot be explained by the precursor structure alone. While many of these represent true biological variants, others may in fact originate from unrelated genomic regions, such as repetitive elements or other classes of small RNAs. Distinguishing between these cases is essential to avoid misannotation.

To achieve this, **all candidate non-templated isomiRs undergo a genomic alignment step** using **Bowtie** under stringent conditions that only allow perfect matches. Sequences that align exclusively within the annotated precursor coordinates—or that lack alternative perfect genomic alignments—are retained as valid non-templated isomiRs. In contrast, any sequence that can also align perfectly to genomic loci outside the precursor is considered ambiguous and is excluded from the final annotation.

This filtering strategy **minimizes false positives** by ensuring that sequences originating from other genomic regions are not misannotated as isomiRs.

## Variant classification

Once reads have been aligned and filtered, **all sequences are organized into variant classes based on their relationship to the canonical miRNA**. These include canonical miRNAs, templated isomiRs generated by alternative precursor processing, and non-templated isomiRs arising from post-transcriptional modifications.

IMAGEN

The system **follows the mirGFF3 standard for isomiR classification** ([Loher et al., 2020](https://academic.oup.com/bioinformatics/article/36/3/698/5556118)), which is itself an adaptation of the format originally introduced in [Urgese et al., 2016](enlace_al_articulo). Variants are categorized according to the type of sequence change and its position within the miRNA, providing a structured framework for consistent annotation and downstream analyses:

- **5′ end shifts (**`iso_5p`**)** – Arise from alternative cleavage by Drosha or Dicer at the 5′ end of the precursor. The position of the start relative to the canonical miRNA is indicated with a `–` if upstream or a `+` if downstream. These shifts can affect the seed sequence (positions 2–7), potentially altering target specificity. Only templated nucleotides (encoded in the genome) are considered for this category.

- **3′ end shifts (**`iso_3p`**)** – Result from alternative 3′ cleavage of the precursor. The 3′ end displacement is also annotated with `–` (upstream) or `+` (downstream). These shifts are generally well-tolerated and do not usually affect the seed region, but can influence miRNA stability or Argonaute loading. 

- **Non-templated nucleotide additions (**`iso_add5p`**, **`iso_add3p`**)** – Nucleotides added post-transcriptionally that are **not encoded by the precursor sequence**.  
  - **5′ non-templated additions (**`iso_add5p`**)** are extremely rare and may reflect sequencing artifacts; therefore, they are disabled by default (`five_add = 0`).  
  - **3′ non-templated additions (`iso_add3p`)**, such as uridylation or adenylation, are more frequent and can modulate miRNA stability or activity. Only sequences passing abundance and genomic filtering are retained. By default, up to 3 non-templated nucleotides are allowed at the 3′ end (`three_add = 3`) to accommodate common biological modifications like adenylation or uridylation.

- **Single-nucleotide variants (SNVs)** – Variants involving internal nucleotide substitutions, without changes at the miRNA ends. They may originate from RNA editing, transcription errors, or other post-transcriptional modifications. SNVs are further stratified by their position:  
  - **Seed region (**`iso_snv_seed`**)** – The isomiR carries a nucleotide change within the seed region, spanning positions 2 to 7 of the miRNA.  
  - **Central offset (**`iso_snv_central_offset`**)** – The nucleotide variation occurs at position 8, immediately adjacent to the seed.
  - **Central domain (**`iso_snv_central`**)** – The substitution is located in the central portion of the miRNA, covering nucleotides 9 to 12.  
  - **Central supplementary region (**`iso_snv_central_supp`**)** – The change falls within nucleotides 13 to 17, corresponding to the supplementary region of the miRNA.  
  - **Other positions (**`iso_snv`**)** – Variations occurring outside the above regions, including nucleotide 1 and positions 18 to the miRNA end.

In practice, sequences are considered valid isomiRs and then classified only if they fall within user-defined thresholds for deviations from the reference sequence. These thresholds are configurable **Nextflow pipeline parameters**. For example, **internal substitutions** are restricted to at most one by default (`substitutions = 1`), balancing sensitivity with specificity. **5' non-templated additions are disabled (`five_add = 0`)**, since these variants lack a known biogenesis mechanism and are generally considered sequencing or alignment artifacts. In contrast, up to **3 non-templated nucleotides are permitted at the 3′ end (`three_add = 3`)**, reflecting the biological prevalence of 3′ tailing events such as adenylation or uridylation. Finally, the **total number of modifications** across both ends is capped (`ends_modification = 4`), ensuring that sequences with excessive alterations are excluded to minimize false positives.

## Abundance filtering

After variant classification, the pipeline applies **abundance filtering** to retain only **high-confidence isomiRs**. This step is essential to exclude sequences likely arising from sequencing noise or technical artifacts. The filtering procedure evaluates isomiRs based on **one of these metrics**:

- **Reads Per Million (RPM)** – Absolute expression normalized to sequencing depth. This threshold filters out sequences with very low read counts.

- **Relative abundance** – Proportion of the isomiR’s RPM relative to the RPM of its canonical miRNA within the same sample. This metric filters sequences that contribute minimally to the canonical miRNA population, helping remove potential sequencing errors even when the absolute read count is high.
 
Both thresholds can be configured to apply **across a minimum number of samples** (`min_samples_filt`, default: 2), ensuring that only consistently expressed isomiRs are considered for downstream analyses. Once an isomiR passes the filter in at least this minimum number of libraries, it is considered valid across all libraries for that species, under the assumption that a sequence observed as real in one library is unlikely to be a sequencing artifact in other libraries.

> [!important]
> By default, the pipeline applies **relative abundance filtering** with a threshold of `0.01`.

> [!Warning]
> For relative abundance filtering, if an isomiR is detected in a sample but its canonical miRNA is absent (**orphan isomiR**), the sequence is excluded.

> [!Note]
> Filtering by relative abundance is supported by simulation studies demonstrating that this strategy effectively removes false-positive isomiRs originating from sequencing errors, even for highly expressed canonical miRNAs ([Sand et al., 2025](https://doi.org/10.1007/s00125-025-06397-4)). In these simulations, synthetic reads of canonical miRNAs were generated with low-probability sequencing errors. The relative abundance of each resulting erroneous variant was calculated as the proportion of reads belonging to that variant relative to the total reads of the canonical miRNA. Most sequencing-error-derived variants had extremely low relative abundance (<1%), allowing the application of a threshold to reliably distinguish true biological isomiRs from technical artifacts, even for miRNAs with very high expression levels.


## DEA Integration

The final stage of the pipeline involves **integrating differential expression analysis (DEA) results** with the miRNA and isomiR annotation. This step enables users to identify which canonical miRNAs and isomiRs are differentially expressed across conditions or treatments.

The procedure works as follows:

- **Filter DEA results by annotation** – The output from DESeq2 is cross-referenced with the annotation tables, retaining only those sequences that have been classified as canonical miRNAs or isomiRs. Non-annotated sequences are excluded.
  
- **Merge statistical and annotation metadata** – For each retained sequence, DEA statistics (e.g., adjusted p-values, log2 fold changes, shrinkage estimates) are combined with annotation details such as miRNA name, precursor, variant type, and family assignment. This produces enriched result tables that couple expression dynamics with biological context.

By combining DEA outcomes with detailed annotation, the workflow provides a biologically meaningful view of differential expression, enabling the interpretation of changes not only at the level of individual miRNAs but also across isomiR variants and families.

> [!IMPORTANT]  
> Only sequences that are **significantly differentially expressed** are included in the output tables.

# References

Desvignes, T. et al. (2020). *Unification of miRNA and isomiR research: the mirGFF3 format and the mirtop API*. Bioinformatics, 36(3), 698–703. [https://doi.org/10.1093/bioinformatics/btz675](https://doi.org/10.1093/bioinformatics/btz675)  

Urgese, G. et al. (2016). *isomiR-SEA: an RNA-Seq analysis tool for miRNAs/isomiRs expression level profiling and miRNA-mRNA interaction sites evaluation*. BMC Bioinformatics, 17, 148. [https://doi.org/10.1186/s12859-016-0958-0](https://doi.org/10.1186/s12859-016-0958-0)  

Sand, S. et al. (2025). *Comprehensive sequencing profile and functional analysis of IsomiRs in human pancreatic islets and beta cells*. Diabetologia, 68(6), 1261–1278. [https://doi.org/10.1007/s00125-025-06397-4](https://doi.org/10.1007/s00125-025-06397-4)  

Wu, C. W. et al. (2018). *CASMIR: sequence-oriented isomiR annotation and profiling in colorectal neoplasia*. BMC Genomics, 19, 401. [https://doi.org/10.1186/s12864-018-4794-7](https://doi.org/10.1186/s12864-018-4794-7)  






