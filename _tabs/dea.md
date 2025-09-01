---
layout: page
title: Differential Expression Analysis
icon: fas fa-tags
order: 4
toc: true
---

## Configuring Differential Expression Analysis

This section explains **how to configure a differential expression analysis (DEA) using the project’s metadata table**. Users specify the design formula, statistical test, reference levels, and contrasts directly in the metadata, allowing the pipeline to adapt automatically to a wide range of experimental designs—from simple two-group comparisons to complex multi-factor or nested analyses.

<!-- no toc -->
- [**1. DEA using a design formula (Wald Test)**](#1-dea-using-a-design-formula-wald-test)
  - [1.1. One Factor, Two Levels](#11-one-factor-two-levels)
  - [1.2. One Factor, Three/more Levels](#12-one-factor-threemore-levels)
  - [1.3. Two Factors with Interaction](#13-two-factors-with-interaction)
- [**2. DEA using a design formula and custom contrast (Wald Test)**](#2-dea-using-a-design-formula-and-custom-contrast-wald-test)
  - [2.1. One Factor with Custom Contrast](#21-one-factor-with-custom-contrast)
  - [2.2. Three Factors with Nesting](#22-three-factors-with-nesting)
- [**3. DEA using Likelihood Ratio Test (LRT)**](#3-dea-using-likelihood-ratio-test-lrt)
- [**4. Handling Batch Effects**](#4-handling-batch-effects)


Each of the examples below represents a specific type of design, ranging from the simplest (one factor with two levels) to more advanced models (interaction terms, nested designs, and LRT tests). For each case, we provide:

- A short explanation of the design and its interpretation.
- A minimal example of how the metadata table should be structured.
  
By following these examples, users can better understand how to correctly structure their metadata and take full advantage of the pipeline’s capabilities to perform reproducible and robust differential expression analyses.

### 1. **DEA using a design formula (Wald Test)**

To perform differential expression analysis with DESeq2, a common approach is to define an experimental design using an R formula. This formula (e.g., `~ condition`), specifies the variables that will be used to model the data and estimate expression changes. The formula must begin with a tilde (`~`) followed by the relevant factors separated by plus signs. These factors should correspond to columns in the metadata table, such as treatment groups (`Treatment`), treatment levels (`Level`), or time points (`Time`). This formula instructs DESeq2 on how to model the data based on the experimental variables to identify significant changes between groups.

Differential expression analysis using a design formula requires the utilization of the `Test`, `Design`, and `DesignRef` columns from the metadata table:


| Column     | Description                                                                                      | Example           |
|------------|------------------------------------------------------------------------------------------------|-------------------|
| `Test`     | Specifies the statistical test to be applied, such as the Wald test or the likelihood ratio test (LRT). | Wald or LRT       |
| `Design`   | Defines the design formula used for modeling the data.                                           | `~ Condition`     |
| `DesignRef`| Indicates the baseline level for each factor included in the formula, serving as the reference point for comparisons. | `Condition(None)` |

The following code provides a simplified example of how the metadata table defined above is used to construct the DESeq2 object and run the differential expression analysis:

#### R code example

```r
# Get the design formula
design_formula <- unique(metadata$Design)[1]
test_type <- unique(metadata$Test)[1]      

# Create a default colData dataframe
coldata <- data.frame(
  Group     = metadata$Group,
  species   = metadata$Species,
  Project   = metadata$Project,
  Run       = metadata$Run,
  Replicate = metadata$Replicate, 
  Test      = metadata$Test,
  Design    = metadata$Design,
  row.names = metadata$Run
)

# Create DESeqDataSet
dds <- DESeqDataSetFromMatrix(
  countData = counts_matrix,
  colData   = coldata,
  design    = as.formula(design_formula)
)

# Run differential expression analysis
dds <- DESeq(dds, test = test)
```

The following sections illustrate practical examples of experimental designs involving different combinations of factors and levels. These examples are intended to demonstrate how the design formula and associated metadata columns (`Test`, `Design`, and `DesignRef`) can be adapted to match specific biological questions and the structure of the dataset.

#### 1.1. One Factor, Two Levels

In many experiments, samples are grouped by a key characteristic called factor, which has different categories called levels. For example, a factor could be `Treatment`, with levels like Cold and None (non-treated), or `Time`, with levels such as 0h and 12h. Comparing two levels of a single factor is the simplest form of differential expression analysis.

In this case, the goal is to compare two experimental conditions using a single categorical variable, such as `Treatment` with levels Cold and None (baselines).

- **Design formula**: `~ Treatment`
- **Test type**: Wald (default)
- **Baseline**: None

#### Metadata Table Example

| Group | Species             | Project     | Run         | Treatment | Replicate | Level | Time | Cultivar | Tissue | Stage             | Genotype | Batch | Test | Design  | DesignRef | DesignRed | Contrast |
|-------|---------------------|-------------|-------------|-----------|-----------|-------|------|----------|--------|--------------------|----------|-------|------|---------|-----------|-----------|----------|
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1 | cold   | 1         | L.0   | 12h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2 | cold   | 2         | L.0   | 12h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3 | cold   | 3         | L.0   | 12h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4 | None      | 1         | L.0   | 0h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample5 | None      | 2         | L.0   | 0h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample6 | None      | 3         | L.0   | 0h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment  | Treatment(None)  | DR.0      | CT.0     |


#### 1.2. One Factor, Three/more Levels

This case is quite similar to *One Factor, Two Levels*, but here the single factor includes three or more distinct levels. For example, the factor might be `Time`, with levels like `0h`, `6h`, and `12h`. In this setup, each level is compared individually in a pairwise manner against a baseline level within the same analysis.

- **Design formula**: `~ Time`
- **Test type**: Wald (default)
- **Baseline**: 0h

#### Metadata Table Example

| Group | Species             | Project     | Run         | Treatment | Replicate | Level | Time | Cultivar | Tissue | Stage             | Genotype | Batch | Test | Design  | DesignRef | DesignRed | Contrast |
|-------|---------------------|-------------|-------------|-----------|-----------|-------|------|----------|--------|--------------------|----------|-------|------|---------|-----------|-----------|----------|
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1 | None   | 1         | L.0   | 0h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2 | None   | 2         | L.0   | 0h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3 | None   | 3         | L.0   | 0h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4 | cold      | 1         | L.0   | 6h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample5 | cold      | 2         | L.0   | 6h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample6 | cold      | 3         | L.0   | 6h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample7 | cold      | 1         | L.0   | 12h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample8 | cold      | 2         | L.0   | 12h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample9 | cold      | 3         | L.0   | 12h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Time  | Time(0h)  | DR.0      | CT.0     |


#### 1.3. Two Factors with Interaction

Sometimes experiments involve studying the effect of two different factors simultaneously, for example `Treatment` and `Time`. Each factor can have multiple levels, and the combined effect of these factors may not be simply additive but interactive. An interaction means that the effect of one factor depends on the level of the other factor. Modeling this interaction allows detecting sRNAs whose expression changes differently depending on the combination of factor levels, rather than just the individual effects of each factor.

For example, you might want to know if the effect of Treatment (e.g., Cold vs None) differs at different time points (0h, 6h, 12h). In this case, the design formula includes both factors and their interaction term: `~ Treatment + Time + Treatment:Time`. Since there are two factors, the `DesignRef` column specifies the baseline level of each factor, separating them with a colon (`:`). For instance: `Treatment(None):Time(0h)` indicates that the baseline is 'None' for `Treatment` and '0h' for `Time`.

- **Design formula**: `~ Treatment + Time + Treatment:Time`
- **Test type**: Wald
- **Baseline**: Treatment = None, Time = 0h

#### Metadata Table Example

| Group | Species             | Project     | Run         | Treatment | Replicate | Level | Time | Cultivar | Tissue | Stage             | Genotype | Batch | Test | Design  | DesignRef | DesignRed | Contrast |
|-------|---------------------|-------------|-------------|-----------|-----------|-------|------|----------|--------|--------------------|----------|-------|------|---------|-----------|-----------|----------|
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1 | None   | 1         | L.0   | 0h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time  | Treatment(None):Time(0h)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2 | None   | 2         | L.0   | 0h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3 | None   | 3         | L.0   | 0h  | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time  | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4 | cold      | 1         | L.0   | 6h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time  | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample5 | cold      | 2         | L.0   | 6h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time  | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample6 | cold      | 3         | L.0   | 6h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time  | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample7 | cold      | 1         | L.0   | 12h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time  | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample8 | cold      | 2         | L.0   | 12h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time | Treatment(None):Time(0h)   | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample9 | cold      | 3         | L.0   | 12h   | CV.0     | leaves | D.0  | G.0      | B.0   | Wald | ~ Treatment + Time + Treatment:Time | Treatment(None):Time(0h) T  | DR.0      | CT.0     |


### 2. **DEA using a design formula and custom contrast (Wald Test)**

In some experiments, you may want to test specific hypotheses that are not captured by the default coefficients of the model. This is particularly useful when you want to compare specific levels of a factor that are not directly adjacent in the design formula, or when you want to test combined effects of multiple conditions or levels that require a custom comparison. In these cases, the pipeline allows the use of **custom contrasts** to extract the relevant comparisons from the fitted DESeq2 model. This approach provides flexibility to test hypotheses beyond simple pairwise comparisons defined by the model coefficients.

To perform differential expression analysis using a custom contrast, the pipeline uses the metadata columns `Test`, `Design`, `DesignRef`, and `Contrast`:

| Column     | Description                                                                                      | Example                                                                 |
|------------|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| `Test`     | Specifies the statistical test to apply (Wald test in this case).                               | Wald                                                                    |
| `Design`   | Defines the full design formula modeling all relevant factors.                                   | `~ Time`                                                                 |
| `DesignRef`| Indicates the baseline level for each factor, serving as the reference point for comparisons.  | `Time(0h)`                                                              |
| `Contrast` | Specifies the specific comparison of interest, which can include combinations or differences of factor levels. | `(50h&12hreinf − 0h) − ((50h − 0h) + (12h − 0h))` |

#### R code example


FALTA VER COMO SE HACE ESTO EN EL PIPELINE PARA REPRESENTARLO BIEN AQUI
```r
# Get design formula and contrast
design_formula <- as.formula(unique(metadata$Design)[1])
contrast_vector <- strsplit(unique(metadata$Contrast)[1], " - ")[[1]]  # simplified parsing

# Create colData
coldata <- data.frame(
  Group     = metadata$Group,
  Species   = metadata$Species,
  Project   = metadata$Project,
  Run       = metadata$Run,
  Replicate = metadata$Replicate,
  Time      = metadata$Time,
  Test      = metadata$Test,
  row.names = metadata$Run
)

# Create DESeqDataSet
dds <- DESeqDataSetFromMatrix(
  countData = counts_matrix,
  colData   = coldata,
  design    = design_formula
)

# Run DESeq2 (Wald test)
dds <- DESeq(dds)

# Extract results using custom contrast
res <- results(dds, contrast = c("Time", "50h&12hreinf", "0h"))
```

#### 2.1. One Factor with Custom Contrast 

#### 2.1. One Factor with Custom Contrast

In this example, we illustrate the use of a custom contrast for a single factor with multiple levels. The goal is to test whether the combined effect of two specific conditions differs from the sum of their individual effects, which can reveal potential interactions or synergy that are not captured by standard pairwise comparisons. Here, the factor of interest is `Time`, with levels `0h`, `12h`, `50h`, and Superinfection (`50h&12hreinf`). The custom contrast compares the combined level `50h&12hreinf` against what would be expected from adding the individual effects of `50h` and `12h` relative to `0h`.

The following metadata and design setup show how this contrast is specified in the pipeline, using `Wald` as the statistical test and `Time(0h)` as the baseline. The `Contrast` column defines the specific comparison of interest:

- **Design formula**: `~ Time`
- **Test type**: Wald (default)
- **Baseline**: 0h
- **Contrast**: `(50h&12hreinf − 0h) − ((50h − 0h) + (12h − 0h))`

#### Metadata Table Example

| Group | Species              | Project  | Run       | Treatment | Replicate | Level | Time          | Cultivar | Tissue | Stage | Genotype | Batch | Test | Design | DesignRef | DesignRed | Contrast                                                   |
|-------|----------------------|----------|-----------|-----------|-----------|-------|---------------|----------|--------|-------|----------|-------|------|--------|-----------|-----------|------------------------------------------------------------|
| 1     | Arabidopsis thaliana | PROJECT1 | sample1   | None      | 1         | L.0   | 0h            | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample2   | None      | 2         | L.0   | 0h            | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample3   | None      | 3         | L.0   | 0h            | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample4   | vir       | 1         | L.0   | 50h           | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample5   | vir       | 2         | L.0   | 50h           | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample6   | vir       | 3         | L.0   | 50h           | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample7   | vir       | 1         | L.0   | 12h           | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample8   | vir       | 2         | L.0   | 12h           | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample9   | vir       | 3         | L.0   | 12h           | CV.0     | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample10  | vir       | 1         | L.0   | Superinfection | CV.0    | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample11  | vir       | 2         | L.0   | Superinfection | CV.0    | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |
| 1     | Arabidopsis thaliana | PROJECT1 | sample12  | vir       | 3         | L.0   | Superinfection | CV.0    | TS.0   | 50h   | G.0      | B.0   | Wald | ~ Time | Time(0h)  | DR.0      | (50h&12hreinf - 0h) - ((50h - 0h) + (12h - 0h))             |

#### 2.2. Three Factors with Nesting

When one factor is nested within another, it means that the levels of the nested
factor are specific to (and only ocurr within) certain levels of the main factor.
In other words, each level of the nested factor belongs exclusively to one level
of the higher order factor and does not appear elsewhere.

For example, imagine a study with three factors: developmental stage (with levels
"V1" and "D1"), tissue type (with levels "leaves" and "roots"), and time (with
levels "0h" and "6h"). Suppose that leaves are only sampled at stage V1, and roots
are only sampled at stage D1. In this case, tissue type is nested within developmental
stage, because each tissue type is unique to a specific stage.

This nesting structure has important implications for the experimental design
and statistical analysis. Since tissue type depends on developmental stage,
both factors must be included in the model along with their interaction with
time. This allows the model to capture differences in sRNA expression across
stages, tissues, and times, as well as how the effect of time varies depending
on the tissue.

In this example, you cannot directly compare leaves and roots across stages using
tissue alone, because each tissue only exists in one stage. Instead, to study changes
over time within each tissue-stage combination, the model shoud include the
following design formula and contrasts to compare time points within each tissue:

- **Design formula**: `~ Stage + Tissue + Time + Tissue:Time`
- **Contrasts**: `(leaves at 6h) – (leaves at 0h), (roots at 6h) – (roots at 0h)`
- **Test type**: Wald (default)
- **Baseline**: Tissue(leaves):Time(0h):Stage(V1)

This approach correctly models the nested structure and interaction effects,
allowing valid interpretation of time-dependent changes within each tissue
nested in its developmental stage.

#### Metadata Table Example

| Group | Species             | Project  | Run     | Treatment | Replicate | Time | Cultivar | Tissue  | Stage | Genotype | Batch | Test | Design                                         | DesignRef                          | Contrast                                                 |
|-------|---------------------|----------|---------|-----------|-----------|------|----------|---------|-------|----------|-------|------|------------------------------------------------|-----------------------------------|----------------------------------------------------------|
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1 | cold      | 1         | 0h   | CV.0      | leaves  | V1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)            | (leaves & 6h) - (leaves & 0h)                            |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1 | cold      | 2         | 0h   | CV.0      | leaves  | V1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)          | (leaves & 6h) - (leaves & 0h)                            |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2 | cold      | 1         | 6h   | CV.0      | leaves  | V1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)           | (leaves & 6h) - (leaves & 0h)                            |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2 | cold      | 2         | 6h   | CV.0      | leaves  | V1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)            | (leaves & 6h) - (leaves & 0h)                            |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3 | cold      | 1         | 0h   | CV.0      | roots   | D1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)             | (roots & 6h) - (roots & 0h)                              |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3 | cold      | 2         | 0h   | CV.0      | roots   | D1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)             | (roots & 6h) - (roots & 0h)                              |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4 | cold      | 1         | 6h   | CV.0      | roots   | D1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)             | (roots & 6h) - (roots & 0h)                              |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4 | cold      | 2         | 6h   | CV.0      | roots   | D1    | G.0       | B.0    | Wald | ~ Stage + Tissue + Time + Tissue:Time       | Tissue(leaves):Time(0h):Stage(V1)             | (roots & 6h) - (roots & 0h)     


### 3. **DEA using Likelihood Ratio Test (LRT)**

The **Likelihood Ratio Test (LRT)** is a statistical method used to determine whether adding a factor or interaction significantly improves the fit of a model. It works by comparing **two nested models**:

- **Full model** – includes all factors of interest.  
- **Reduced model** – excludes the factor(s) you want to test.  

LRT evaluates whether the difference in fit between the full and reduced models is statistically significant. This makes it particularly useful for:

- **Nested models** – to test if extra complexity is justified.  
- **Multi-level factors** – to assess overall differences across several levels simultaneously.  
- **Time-series or longitudinal experiments** – to detect miRNAs whose expression changes over time, avoiding multiple pairwise comparisons at each time point.  

In the pipeline, LRT can be applied by setting the `Test` column in the metadata table to `LRT`. The `Design` column specifies the full model, while the `DesignRed` column defines the reduced (simpler) model for comparison. This allows users to focus on factors or interactions that truly drive expression changes in their experiments.

In this example, we demonstrate how to configure an LRT to evaluate whether the interaction between `Treatment` and `Time` has a significant impact on miRNA expression. The full model includes the interaction term, while the reduced model excludes it. By specifying these models in the `Design` and `DesignRed` columns of the metadata table and setting `Test = LRT`, the pipeline can assess the contribution of the interaction to the overall expression changes.

- **Design formula**: `~ Treatment + Time + Treatment:Time`  
- **Reduced formula**: `~ Treatment + Time`  
- **Test type**: LRT  
- **Baseline**: Treatment = None, Time = 0h  

#### Metadata Table Example

| Group | Species             | Project     | Run       | Treatment | Replicate | Time | Test | Design                       | DesignRed                  |
|-------|-------------------|------------|-----------|-----------|-----------|------|------|-------------------------------|---------------------------|
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1   | None      | 1         | 0h   | LRT  | ~ Treatment + Time + Treatment:Time | ~ Treatment + Time      |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2   | None      | 2         | 0h   | LRT  | ~ Treatment + Time + Treatment:Time | ~ Treatment + Time      |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3   | cold      | 1         | 6h   | LRT  | ~ Treatment + Time + Treatment:Time | ~ Treatment + Time      |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4   | cold      | 2         | 6h   | LRT  | ~ Treatment + Time + Treatment:Time | ~ Treatment + Time      |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample5   | cold      | 3         | 12h  | LRT  | ~ Treatment + Time + Treatment:Time | ~ Treatment + Time      |

#### R code example

```r
dds <- DESeq(dds, test = "LRT", reduced = as.formula("~ Treatment + Time"))
```

### 4. **Handling Batch Effects**

In many experiments, technical or biological factors unrelated to the experimental conditions—such as sequencing run, library preparation batch, lab technician, or sample source—can introduce systematic differences in the measured expression values. These unwanted variations are collectively referred to as **batch effects**. If not accounted for, batch effects can obscure true biological signals, create false positives or negatives, and lead to incorrect conclusions. Correcting for batch effects ensures that observed differential expression truly reflects the factor(s) of interest rather than technical or confounding variability.

In the pipeline, batch effects can be modeled by including covariates in the design formula. Users simply add columns representing the batch or other confounding variables to the metadata table, and include them in the `Design` formula. DESeq2 then accounts for these variables when estimating expression changes, effectively removing unwanted variability.

The following example demonstrates how to incorporate a `Batch` variable into the design formula to correct for batch effects while testing for differential expression between treatments:

- **Design formula**: `~ Batch + Treatment`  
- **Test type**: Wald  
- **Baseline**: Treatment = None  

#### Metadata Table Example

| Group | Species             | Project     | Run         | Treatment | Replicate | Level | Time | Cultivar | Tissue | Stage             | Genotype | Batch | Test | Design  | DesignRef | DesignRed | Contrast |
|-------|---------------------|-------------|-------------|-----------|-----------|-------|------|----------|--------|--------------------|----------|-------|------|---------|-----------|-----------|----------|
| 1     | Arabidopsis thaliana | PROJECT1 | Sample1 | cold   | 1         | L.0   | 12h  | CV.0     | leaves | D.0  | G.0      | 1   | Wald | ~ Batch + Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample2 | cold   | 2         | L.0   | 12h  | CV.0     | leaves | D.0  | G.0      | 2   | Wald | ~ Batch + Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample3 | None      | 1         | L.0   | 0h   | CV.0     | leaves | D.0  | G.0      | 1   | Wald | ~ Batch + Treatment  | Treatment(None)  | DR.0      | CT.0     |
| 1     | Arabidopsis thaliana | PROJECT1 | Sample4 | None      | 2         | L.0   | 0h   | CV.0     | leaves | D.0  | G.0      | 2   | Wald | ~ Batch + Treatment  | Treatment(None)  | DR.0      | CT.0     |

#### R code example

```r
dds <- DESeqDataSetFromMatrix(countData = counts_matrix, colData = coldata, design = '~ Batch + Treatment')
dds <- DESeq(dds)
```