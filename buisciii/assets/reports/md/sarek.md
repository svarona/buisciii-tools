# nf-core/sarek: Output <!-- omit in toc -->

## Introduction <!-- omit in toc -->

This document describes the output produced by the pipeline. Most of the plots are taken from the MultiQC report, which summarises results at the end of the pipeline.

The directories listed below will be created in the results directory after the pipeline has finished. All paths are relative to the top-level results directory.

## Pipeline overview <!-- omit in toc -->

The pipeline is built using [Nextflow](https://www.nextflow.io/) and processes data using the following steps:

- [Directory Structure](#directory-structure)
- [Preprocessing](#preprocessing)
  - [Preparation of input files (FastQ or (u)BAM)](#preparation-of-input-files-fastq-or-ubam)
    - [Trim adapters](#trim-adapters)
    - [Split FastQ files](#split-fastq-files)
    - [UMI consensus](#umi-consensus)
  - [Map to Reference](#map-to-reference)
    - [BWA](#bwa)
    - [BWA-mem2](#bwa-mem2)
    - [DragMap](#dragmap)
    - [Sentieon BWA mem](#sentieon-bwa-mem)
  - [Mark Duplicates](#mark-duplicates)
    - [GATK MarkDuplicates (Spark)](#gatk-markduplicates-spark)
  - [Sentieon LocusCollector and Dedup](#sentieon-locuscollector-and-dedup)
  - [Base Quality Score Recalibration](#base-quality-score-recalibration)
    - [GATK BaseRecalibrator (Spark)](#gatk-baserecalibrator-spark)
    - [GATK ApplyBQSR (Spark)](#gatk-applybqsr-spark)
  - [CSV files](#csv-files)
- [Variant Calling](#variant-calling)
  - [SNVs and small indels](#snvs-and-small-indels)
    - [bcftools](#bcftools)
    - [DeepVariant](#deepvariant)
    - [FreeBayes](#freebayes)
    - [GATK HaplotypeCaller](#gatk-haplotypecaller)
      - [GATK Germline Single Sample Variant Calling](#gatk-germline-single-sample-variant-calling)
      - [GATK Joint Germline Variant Calling](#gatk-joint-germline-variant-calling)
    - [GATK Mutect2](#gatk-mutect2)
    - [Sentieon DNAscope](#sentieon-dnascope)
      - [Sentieon DNAscope joint germline variant calling](#sentieon-dnascope-joint-germline-variant-calling)
    - [Sentieon Haplotyper](#sentieon-haplotyper)
      - [Sentieon Haplotyper joint germline variant calling](#sentieon-haplotyper-joint-germline-variant-calling)
    - [Strelka2](#strelka2)
  - [Structural Variants](#structural-variants)
    - [Manta](#manta)
    - [TIDDIT](#tiddit)
  - [Sample heterogeneity, ploidy and CNVs](#sample-heterogeneity-ploidy-and-cnvs)
    - [ASCAT](#ascat)
    - [CNVKit](#cnvkit)
    - [Control-FREEC](#control-freec)
  - [Microsatellite instability (MSI)](#microsatellite-instability-msi)
    - [MSIsensorPro](#msisensorpro)
  - [Concatenation](#concatenation)
- [Variant annotation](#variant-annotation)
  - [snpEff](#snpeff)
  - [VEP](#vep)
  - [BCFtools annotate](#bcftools-annotate)
- [Quality control and reporting](#quality-control-and-reporting)
  - [Quality control](#quality-control)
    - [FastQC](#fastqc)
    - [FastP](#fastp)
    - [Mosdepth](#mosdepth)
    - [NGSCheckMate](#ngscheckmate)
    - [GATK MarkDuplicates reports](#gatk-markduplicates-reports)
    - [Sentieon Dedup reports](#sentieon-dedup-reports)
    - [samtools stats](#samtools-stats)
    - [bcftools stats](#bcftools-stats)
    - [VCFtools](#vcftools)
    - [snpEff reports](#snpeff-reports)
    - [VEP reports](#vep-reports)
  - [Reporting](#reporting)
    - [MultiQC](#multiqc)
  - [Pipeline information](#pipeline-information)
- [Reference files](#reference-files)

## Directory Structure

The default directory structure is as follows

```
{outdir}
├── csv
├── multiqc
├── pipeline_info
├── preprocessing
│   ├── markduplicates
│       └── <sample>
│   ├── recal_table
│       └── <sample>
│   └── recalibrated
│       └── <sample>
├── reference
└── reports
    ├── <tool1>
    └── <tool2>
work/
.nextflow.log
```

## Preprocessing

Sarek pre-processes raw FastQ files or unmapped BAM files, based on [GATK best practices](https://gatk.broadinstitute.org/hc/en-us/sections/360007226651-Best-Practices-Workflows).

### Preparation of input files (FastQ or (u)BAM)

[FastP](https://github.com/OpenGene/fastp) is a tool designed to provide all-in-one preprocessing for FastQ files and as such is used for trimming and splitting. By default, these files are not published. However, if publishing is enabled, please be aware that these files are only published once, meaning if trimming and splitting is enabled, then the resulting files will be sharded FastQ files with trimmed reads. If only one of them is enabled then the files contain either trimmed or split reads, respectively.

#### Trim adapters

[FastP](https://github.com/OpenGene/fastp) supports global trimming, which means it trims all reads in the front or the tail. This function is useful since sometimes you want to drop some cycles of a sequencing run. In the current implementation in Sarek
`--detect_adapter_for_pe` is set by default which enables auto-detection of adapter sequences. For more information on how to fine-tune adapter trimming, take a look into the parameter docs.

The resulting files are intermediate and by default not kept in the final files delivered to users. Set `--save_trimmed` to enable publishing of the files in:

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/fastp/<sample>`**

- `<sample>_<lane>_{1,2}.fastp.fastq.gz>`
  - Bgzipped FastQ file

</details>

#### Split FastQ files

[FastP](https://github.com/OpenGene/fastp) supports splitting of one FastQ file into multiple files allowing parallel alignment of sharded FastQ file. To enable splitting, the number of reads per output can be specified. For more information, take a look into the parameter `--split_fastq`in the parameter docs.

These files are intermediate and by default not placed in the output-folder kept in the final files delivered to users. Set `--save_split` to enable publishing of these files to:

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/fastp/<sample>/`**

- `<sample_lane_{1,2}.fastp.fastq.gz>`
  - Bgzipped FastQ file

</details>

#### UMI consensus

Sarek can process UMI-reads, using [fgbio](http://fulcrumgenomics.github.io/fgbio/tools/latest/) tools.

These files are intermediate and by default not placed in the output-folder kept in the final files delivered to users. Set `--save_split` to enable publishing of these files to:

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/umi/<sample>/`**

- `<sample_lane_{1,2}.umi-consensus.bam>`

**Output directory: `{outdir}/reports/umi/`**

- `<sample_lane_{1,2}_umi_histogram.txt>`

</details>

### Map to Reference

#### BWA

[BWA](https://github.com/lh3/bwa) is a software package for mapping low-divergent sequences against a large reference genome. The aligned reads are then coordinate-sorted (or name-sorted if [`GATK MarkDuplicatesSpark`](https://gatk.broadinstitute.org/hc/en-us/articles/5358833264411-MarkDuplicatesSpark) is used for duplicate marking) with [samtools](https://www.htslib.org/doc/samtools.html).

#### BWA-mem2

[BWA-mem2](https://github.com/bwa-mem2/bwa-mem2) is a software package for mapping low-divergent sequences against a large reference genome.The aligned reads are then coordinate-sorted (or name-sorted if [`GATK MarkDuplicatesSpark`](https://gatk.broadinstitute.org/hc/en-us/articles/5358833264411-MarkDuplicatesSpark) is used for duplicate marking) with [samtools](https://www.htslib.org/doc/samtools.html).

#### DragMap

[DragMap](https://github.com/Illumina/dragmap) is an open-source software implementation of the DRAGEN mapper, which the Illumina team created so that we would have an open-source way to produce the same results as their proprietary DRAGEN hardware. The aligned reads are then coordinate-sorted (or name-sorted if [`GATK MarkDuplicatesSpark`](https://gatk.broadinstitute.org/hc/en-us/articles/5358833264411-MarkDuplicatesSpark) is used for duplicate marking) with [samtools](https://www.htslib.org/doc/samtools.html).

These files are intermediate and by default not placed in the output-folder kept in the final files delivered to users. Set `--save_mapped` to enable publishing, furthermore add the flag `save_output_as_bam` for publishing in BAM format.

#### Sentieon BWA mem

Sentieon [bwa mem](https://support.sentieon.com/manual/usages/general/#bwa-mem-syntax) is a subroutine for mapping low-divergent sequences against a large reference genome. It is part of the proprietary software package [DNAseq](https://www.sentieon.com/detailed-description-of-pipelines/#dnaseq) from [Sentieon](https://www.sentieon.com/).

The aligned reads are coordinate-sorted with Sentieon.

<details markdown="1">
<summary>Output files for all mappers and samples</summary>

The alignment files (BAM or CRAM) produced by the chosen aligner are not published by default. CRAM output files will not be saved in the output-folder (`outdir`), unless the flag `--save_mapped` is used. BAM output can be selected by setting the flag `--save_output_as_bam`.

**Output directory: `{outdir}/preprocessing/mapped/<sample>/`**

- if `--save_mapped`: `<sample>.sorted.cram` and `<sample>.sorted.cram.crai`

  - CRAM file and index

- if `--save_mapped --save_output_as_bam`: `<sample>.sorted.bam` and `<sample>.sorted.bam.bai`
  - BAM file and index
  </details>

### Mark Duplicates

During duplicate marking, read pairs that are likely to have originated from duplicates of the same original DNA fragments through some artificial processes are identified. These are considered to be non-independent observations, so all but a single read pair within each set of duplicates are marked, causing the marked pairs to be ignored by default during the variant discovery process.

For further reading and documentation see the [data pre-processing for variant discovery from the GATK best practices](https://gatk.broadinstitute.org/hc/en-us/articles/360035535912-Data-pre-processing-for-variant-discovery).

#### GATK MarkDuplicates (Spark)

By default, Sarek will use [GATK MarkDuplicates](https://gatk.broadinstitute.org/hc/en-us/articles/5358880192027-MarkDuplicates-Picard-).

To use the corresponding spark implementation [`GATK MarkDuplicatesSpark`](https://gatk.broadinstitute.org/hc/en-us/articles/5358833264411-MarkDuplicatesSpark), please specify `--use_gatk_spark markduplicates`. The resulting files are converted to CRAM with either [samtools](https://www.htslib.org/doc/samtools.html), when GATK MarkDuplicates is used, or, implicitly, by GATK MarkDuplicatesSpark.

The resulting CRAM files are delivered to the users.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/markduplicates/<sample>/`**

- `<sample>.md.cram` and `<sample>.md.cram.crai`
  - CRAM file and index
- if `--save_output_as_bam`:
  - `<sample>.md.bam` and `<sample>.md.bam.bai`

</details>

### Sentieon LocusCollector and Dedup

The subroutines LocusCollector and Dedup are part of Sentieon DNAseq packages with speedup versions of the standard GATK tools, and together those two subroutines correspond to GATK's MarkDuplicates.

The subroutine [LocusCollector](https://support.sentieon.com/manual/usages/general/#driver-algorithm-syntax) collects read information that will be used for removing or tagging duplicate reads; its output is the score file indicating which reads are likely duplicates.

The subroutine [Dedup](https://support.sentieon.com/manual/usages/general/#dedup-algorithm) marks or removes duplicate reads based on the score file supplied by LocusCollector, and produces a BAM or CRAM file.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/sentieon_dedup/<sample>/`**

- `<sample>.dedup.cram` and `<sample>.dedup.cram.crai`
  - CRAM file and index
- if `--save_output_as_bam`:
  - `<sample>.dedup.bam` and `<sample>.dedup.bam.bai`

</details>

### Base Quality Score Recalibration

During Base Quality Score Recalibration, systematic errors in the base quality scores are corrected by applying machine learning to detect and correct for them. This is important for evaluating the correct call of a variant during the variant discovery process. However, this is not needed for all combinations of tools in Sarek. Notably, this should be turned off when having UMI tagged reads or using DragMap (see [here](https://gatk.broadinstitute.org/hc/en-us/articles/4407897446939--How-to-Run-germline-single-sample-short-variant-discovery-in-DRAGEN-mode)) as mapper.

For further reading and documentation see the [technical documentation by GATK](https://gatk.broadinstitute.org/hc/en-us/articles/360035890531-Base-Quality-Score-Recalibration-BQSR-).

#### GATK BaseRecalibrator (Spark)

[GATK BaseRecalibrator](https://gatk.broadinstitute.org/hc/en-us/articles/360042477672-BaseRecalibrator) generates a recalibration table based on various co-variates.

To use the corresponding spark implementation [GATK BaseRecalibratorSpark](https://gatk.broadinstitute.org/hc/en-us/articles/5358896138011-BaseRecalibrator), please specify `--use_gatk_spark baserecalibrator`.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/recal_table/<sample>/`**

- `<sample>.recal.table`
  - Recalibration table associated to the duplicates-marked CRAM file.

</details>

#### GATK ApplyBQSR (Spark)

[GATK ApplyBQSR](https://gatk.broadinstitute.org/hc/en-us/articles/5358826654875-ApplyBQSR) recalibrates the base qualities of the input reads based on the recalibration table produced by the [GATK BaseRecalibrator](#gatk-baserecalibrator) tool.

Specify `--use_gatk_spark baserecalibrator` to use [GATK ApplyBQSRSpark](https://gatk.broadinstitute.org/hc/en-us/articles/5358898266011-ApplyBQSRSpark-BETA-) instead, the respective spark implementation.

The resulting recalibrated CRAM files are delivered to the user. Recalibrated CRAM files are usually 2-3 times larger than the duplicate-marked CRAM files.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/preprocessing/recalibrated/<sample>/`**

- `<sample>.recal.cram` and `<sample>.recal.cram.crai`
  - CRAM file and index
- if `--save_output_as_bam`:
  - `<sample>.recal.bam` and `<sample>.recal.bam.bai` - BAM file and index
  </details>

### CSV files

The CSV files are auto-generated and can be used by Sarek for further processing and/or variant calling.

See the [`input`](usage#input-sample-sheet-configurations) section in the usage documentation for further reading and documentation on how to make the most of them.

<details markdown="1">
<summary>Output files:</summary>

**Output directory: `{outdir}/preprocessing/csv`**

- `mapped.csv`
  - if `--save_mapped`
  - CSV containing an entry for each sample with the columns `patient,sample,sex,status,bam,bai`
- `markduplicates_no_table.csv`
  - CSV containing an entry for each sample with the columns `patient,sample,sex,status,cram,crai`
- `markduplicates.csv`
  - CSV containing an entry for each sample with the columns `patient,sample,sex,status,cram,crai,table`
- `recalibrated.csv`
  - CSV containing an entry for each sample with the columns`patient,sample,sex,status,cram,crai`
- `variantcalled.csv`
  - CSV containing an entry for each sample with the columns `patient,sample,vcf`
  </details>

## Variant Calling

The results regarding variant calling are collected in `{outdir}/variantcalling/`.
If some results from a variant caller do not appear here, please check out the `--tools` section in the parameter [documentation](https://nf-co.re/sarek/latest/parameters).

(Recalibrated) CRAM files can used as an input to start the variant calling.

### SNVs and small indels

For single nucleotide variants (SNVs) and small indels, multiple tools are available for normal (germline), tumor-only, and tumor-normal (somatic) paired data. For a list of the appropriate tool(s) for the data and sequencing type at hand, please check [here](usage#which-tool).

#### bcftools

[bcftools mpileup](https://samtools.github.io/bcftools/bcftools.html#mpileup) generates pileup of a CRAM file, followed by [bcftools call](https://samtools.github.io/bcftools/bcftools.html#call) and filtered with `-i 'count(GT==\"RR\")==0`.
For further reading and documentation see the [bcftools manual](https://samtools.github.io/bcftools/howtos/variant-calling.html).

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/variantcalling/bcftools/<sample>/`**

- `<sample>.bcftools.vcf.gz` and `<sample>.bcftools.vcf.gz.tbi`
  - VCF with tabix index

</details>

#### DeepVariant

[DeepVariant](https://github.com/google/deepvariant) is a deep learning-based variant caller that takes aligned reads, produces pileup image tensors from them, classifies each tensor using a convolutional neural network and finally reports the results in a standard VCF or gVCF file. For further documentation take a look [here](https://github.com/google/deepvariant/tree/r1.4/docs).

<details markdown="1">
<summary>Output files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/deepvariant/<sample>/`**

- `<sample>.deepvariant.vcf.gz` and `<sample>.deepvariant.vcf.gz.tbi`
  - VCF with tabix index
- `<sample>.deepvariant.g.vcf.gz` and `<sample>.deepvariant.g.vcf.gz.tbi`
  - gVCF with tabix index
  </details>

#### FreeBayes

[FreeBayes](https://github.com/ekg/freebayes) is a Bayesian genetic variant detector designed to find small polymorphisms, specifically SNPs, indels, MNPs, and complex events smaller than the length of a short-read sequencing alignment. For further reading and documentation see the [FreeBayes manual](https://github.com/ekg/freebayes/blob/master/README.md#user-manual-and-guide).

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/variantcalling/freebayes/{sample,normalsample_vs_tumorsample}/`**

- `<sample>.freebayes.vcf.gz` and `<sample>.freebayes.vcf.gz.tbi`
  - VCF with tabix index

</details>

#### GATK HaplotypeCaller

[GATK HaplotypeCaller](https://gatk.broadinstitute.org/hc/en-us/articles/5358864757787-HaplotypeCaller) calls germline SNPs and indels via local re-assembly of haplotypes.

<details markdown="1">
<summary>Output files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/haplotypecaller/<sample>/`**

- `<sample>.haplotypecaller.vcf.gz` and `<sample>.haplotypecaller.vcf.gz.tbi`
  - VCF with tabix index

</details>

##### GATK Germline Single Sample Variant Calling

[GATK Single Sample Variant Calling](https://gatk.broadinstitute.org/hc/en-us/articles/360035535932-Germline-short-variant-discovery-SNPs-Indels-)
uses HaplotypeCaller in its default single-sample mode to call variants. The VCF that HaplotypeCaller emits errors on the side of sensitivity, therefore they are filtered by first running the [CNNScoreVariants](https://gatk.broadinstitute.org/hc/en-us/articles/5358904862107-CNNScoreVariants) tool. This tool annotates each variant with a score indicating the model's prediction of the quality of each variant. To apply filters based on those scores run the [FilterVariantTranches](https://gatk.broadinstitute.org/hc/en-us/articles/5358928898971-FilterVariantTranches) tool with SNP and INDEL sensitivity tranches appropriate for your task.

If the haplotype-called VCF files are not filtered, then Sarek should be run with at least one of the options `--dbsnp` or `--known_indels`.

<details markdown="1">
<summary>Output files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/haplotypecaller/<sample>/`**

- `<sample>.haplotypecaller.filtered.vcf.gz` and `<sample>.haplotypecaller.filtered.vcf.gz.tbi`
  - VCF with tabix index

</details>

##### GATK Joint Germline Variant Calling

[GATK Joint germline Variant Calling](https://gatk.broadinstitute.org/hc/en-us/articles/360035535932-Germline-short-variant-discovery-SNPs-Indels-) uses Haplotypecaller per sample in `gvcf` mode. Next, the gVCFs are consolidated from multiple samples into a [GenomicsDB](https://gatk.broadinstitute.org/hc/en-us/articles/5358869876891-GenomicsDBImport) datastore. After joint [genotyping](https://gatk.broadinstitute.org/hc/en-us/articles/5358906861083-GenotypeGVCFs), [VQSR](https://gatk.broadinstitute.org/hc/en-us/articles/5358906115227-VariantRecalibrator) is applied for filtering to produce the final multisample callset with the desired balance of precision and sensitivity.

<details markdown="1">
<summary>Output files from joint germline variant calling</summary>

**Output directory: `{outdir}/variantcalling/haplotypecaller/<sample>/`**

- `<sample>.haplotypecaller.g.vcf.gz` and `<sample>.haplotypecaller.g.vcf.gz.tbi`
  - gVCF with tabix index

**Output directory: `{outdir}/variantcalling/haplotypecaller/joint_variant_calling/`**

- `joint_germline.vcf.gz` and `joint_germline.vcf.gz.tbi`
  - VCF with tabix index
- `joint_germline_recalibrated.vcf.gz` and `joint_germline_recalibrated.vcf.gz.tbi`
  - variant recalibrated VCF with tabix index (if VQSR is applied)

</details>

#### GATK Mutect2

[GATK Mutect2](https://gatk.broadinstitute.org/hc/en-us/articles/5358911630107-Mutect2) calls somatic SNVs and indels via local assembly of haplotypes.
When `--joint_mutect2` is used, Mutect2 subworkflow outputs will be saved in a subfolder named with the patient ID and `{patient}.mutect2.vcf.gz` file will contain variant calls from all of the normal and tumor samples of the patient.
For further reading and documentation see the [Mutect2 manual](https://gatk.broadinstitute.org/hc/en-us/articles/360035531132).
It is not required, but recommended to have a [panel of normals (PON)](https://gatk.broadinstitute.org/hc/en-us/articles/360035890631-Panel-of-Normals-PON) using at least 40 normal samples to get filtered somatic calls. When using `--genome GATK.GRCh38`, a panel-of-normals file is available. However, it is _highly_ recommended to create one matching your tumor samples. Creating your own panel-of-normals is currently not natively supported by the pipeline. See [here](https://gatk.broadinstitute.org/hc/en-us/articles/360035531132) for how to create one manually.

<details markdown="1">
<summary>Output files for tumor-only and tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/mutect2/{sample,tumorsample_vs_normalsample,patient}/`**

Files created:

- `{sample,tumorsample_vs_normalsample,patient}.mutect2.vcf.gz` and `{sample,tumorsample_vs_normalsample,patient}.mutect2.vcf.gz.tbi`
  - unfiltered (raw) Mutect2 calls VCF with tabix index
- `{sample,tumorsample_vs_normalsample,patient}.mutect2.vcf.gz.stats`
  - a stats file generated during calling of raw variants (needed for filtering)
- `{sample,tumorsample_vs_normalsample}.mutect2.contamination.table`
  - table calculating the fraction of reads coming from cross-sample contamination
- `{sample,tumorsample_vs_normalsample}.mutect2.segmentation.table`
  - table containing segmentation of the tumor by minor allele fraction
- `{sample,tumorsample_vs_normalsample,patient}.mutect2.artifactprior.tar.gz`
  - prior probabilities for read orientation artifacts
- `{sample,tumorsample,normalsample}.mutect2.pileups.table`
  - tabulates pileup metrics for inferring contamination
- `{sample,tumorsample_vs_normalsample,patient}.mutect2.filtered.vcf.gz` and `{sample,tumorsample_vs_normalsample,patient}.mutect2.filtered.vcf.gz.tbi`
  - filtered Mutect2 calls VCF with tabix index based on the probability that a variant is somatic
- `{sample,tumorsample_vs_normalsample,patient}.mutect2.filtered.vcf.gz.filteringStats.tsv`
  - a stats file generated during the filtering of Mutect2 called variants

</details>

#### Sentieon DNAscope

[Sentieon DNAscope](https://support.sentieon.com/appnotes/dnascope_ml/#dnascope-germline-variant-calling-with-a-machine-learning-model) is a variant-caller which aims at outperforming GATK's Haplotypecaller in terms of both speed and accuracy. DNAscope allows you to use a machine learning model to perform variant calling with higher accuracy by improving the candidate detection and filtering.

<details markdown="1">
<summary>Unfiltered VCF-files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/sentieon_dnascope/<sample>/`**

- `<sample>.dnascope.unfiltered.vcf.gz` and `<sample>.dnascope.unfiltered.vcf.gz.tbi`
  - VCF with tabix index

</details>

The output from Sentieon's DNAscope can be controlled through the option `--sentieon_dnascope_emit_mode` for Sarek, see [Basic usage of Sentieon functions](#basic-usage-of-sentieon-functions).

Unless `dnascope_filter` is listed under `--skip_tools` in the nextflow command, Sentieon's [DNAModelApply](https://support.sentieon.com/manual/usages/general/#dnamodelapply-algorithm) is applied to the unfiltered VCF-files in order to obtain filtered VCF-files.

<details markdown="1">
<summary>Filtered VCF-files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/sentieon_dnascope/<sample>/`**

- `<sample>.dnascope.filtered.vcf.gz` and `<sample>.dnascope.filtered.vcf.gz.tbi`
  - VCF with tabix index

</details>

##### Sentieon DNAscope joint germline variant calling

In Sentieon's package DNAscope, joint germline variant calling is done by first running Sentieon's Dnacope in emit-mode `gvcf` for each sample and then running Sentieon's [GVCFtyper](https://support.sentieon.com/manual/usages/general/#gvcftyper-algorithm) on the set of gVCF-files. See [Basic usage of Sentieon functions](#basic-usage-of-sentieon-functions) for information on how joint germline variant calling can be done in Sarek using Sentieon's DNAscope.

<details markdown="1">
<summary>Output files from joint germline variant calling</summary>

**Output directory: `{outdir}/variantcalling/sentieon_dnascope/<sample>/`**

- `<sample>.dnascope.g.vcf.gz` and `<sample>.dnascope.g.vcf.gz.tbi`
  - VCF with tabix index

**Output directory: `{outdir}/variantcalling/sentieon_dnascope/joint_variant_calling/`**

- `joint_germline.vcf.gz` and `joint_germline.vcf.gz.tbi`
  - VCF with tabix index

</details>

#### Sentieon Haplotyper

[Sentieon Haplotyper](https://support.sentieon.com/manual/usages/general/#haplotyper-algorithm) is Sention's speedup version of GATK's Haplotypecaller (see above).

<details markdown="1">
<summary>Unfiltered VCF-files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/sentieon_haplotyper/<sample>/`**

- `<sample>.haplotyper.unfiltered.vcf.gz` and `<sample>.haplotyper.unfiltered.vcf.gz.tbi`
  - VCF with tabix index

</details>

The output from Sentieon's Haplotyper can be controlled through the option `--sentieon_haplotyper_emit_mode` for Sarek, see [Basic usage of Sentieon functions](#basic-usage-of-sentieon-functions).

Unless `haplotyper_filter` is listed under `--skip_tools` in the nextflow command, GATK's CNNScoreVariants and FilterVariantTranches (see above) is applied to the unfiltered VCF-files in order to obtain filtered VCF-files.

<details markdown="1">
<summary>Filtered VCF-files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/sentieon_haplotyper/<sample>/`**

- `<sample>.haplotyper.filtered.vcf.gz` and `<sample>.haplotyper.filtered.vcf.gz.tbi`
  - VCF with tabix index

</details>

##### Sentieon Haplotyper joint germline variant calling

In Sentieon's package DNAseq, joint germline variant calling is done by first running Sentieon's Haplotyper in emit-mode `gvcf` for each sample and then running Sentieon's [GVCFtyper](https://support.sentieon.com/manual/usages/general/#gvcftyper-algorithm) on the set of gVCF-files. See [Basic usage of Sentieon functions](#basic-usage-of-sentieon-functions) for information on how joint germline variant calling can be done in Sarek using Sentieon's DNAseq. After joint genotyping, Sentieon's version of VQSR ([VarCal](https://support.sentieon.com/manual/usages/general/#varcal-algorithm) and [ApplyVarCal](https://support.sentieon.com/manual/usages/general/#applyvarcal-algorithm)) is applied for filtering to produce the final multisample callset with the desired balance of precision and sensitivity.

<details markdown="1">
<summary>Output files from joint germline variant calling</summary>

**Output directory: `{outdir}/variantcalling/sentieon_haplotyper/<sample>/`**

- `<sample>.haplotyper.g.vcf.gz` and `<sample>.haplotyper.g.vcf.gz.tbi`
  - VCF with tabix index

**Output directory: `{outdir}/variantcalling/sentieon_haplotyper/joint_variant_calling/`**

- `joint_germline.vcf.gz` and `joint_germline.vcf.gz.tbi`
  - VCF with tabix index
- `joint_germline_recalibrated.vcf.gz` and `joint_germline_recalibrated.vcf.gz.tbi`
  - variant recalibrated VCF with tabix index (if VarCal is applied)

</details>

#### Strelka2

[Strelka2](https://github.com/Illumina/strelka) is a fast and accurate small variant caller optimized for analysis of germline variation in small cohorts and somatic variation in tumor/normal sample pairs. For further reading and documentation see the [Strelka2 user guide](https://github.com/Illumina/strelka/blob/master/docs/userGuide/README.md). If [Strelka2](https://github.com/Illumina/strelka) is used for somatic variant calling and [Manta](https://github.com/Illumina/manta) is also specified in tools, the output candidate indels from [Manta](https://github.com/Illumina/manta) are used according to [Strelka Best Practices](https://github.com/Illumina/strelka/blob/master/docs/userGuide/README.md#somatic-configuration-example).
For further downstream analysis, take a look [here](https://github.com/Illumina/strelka/blob/v2.9.x/docs/userGuide/README.md#interpreting-the-germline-multi-sample-variants-vcf).

<details markdown="1">
<summary>Output files for all single samples (normal or tumor-only)</summary>

**Output directory: `{outdir}/variantcalling/strelka/<sample>/`**

- `<sample>.strelka.genome.vcf.gz` and `<sample>.strelka.genome.vcf.gz.tbi`
  - genome VCF with tabix index
- `<sample>.strelka.variants.vcf.gz` and `<sample>.strelka.variants.vcf.gz.tbi`
  - VCF with tabix index with all potential variant loci across the sample. Note this file includes non-variant loci if they have a non-trivial level of variant evidence or contain one or more alleles for which genotyping has been forced.
  </details>

<details markdown="1">
<summary>Output files for tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/strelka/<tumorsample_vs_normalsample>/`**

- `<tumorsample_vs_normalsample>.strelka.somatic_indels.vcf.gz` and `<tumorsample_vs_normalsample>.strelka.somatic_indels.vcf.gz.tbi`
  - VCF with tabix index with all somatic indels inferred in the tumor sample.
- `<tumorsample_vs_normalsample>.strelka.somatic_snvs.vcf.gz` and `<tumorsample_vs_normalsample>.strelka.somatic_snvs.vcf.gz.tbi`
  - VCF with tabix index with all somatic SNVs inferred in the tumor sample.

</details>

### Structural Variants

#### Manta

[Manta](https://github.com/Illumina/manta) calls structural variants (SVs) and indels from mapped paired-end sequencing reads.
It is optimized for analysis of germline variation in small sets of individuals and somatic variation in tumor/normal sample pairs.
[Manta](https://github.com/Illumina/manta) provides a candidate list for small indels that can be fed to [Strelka2](https://github.com/Illumina/strelka) following [Strelka Best Practices](https://github.com/Illumina/strelka/blob/master/docs/userGuide/README.md#somatic-configuration-example). For further reading and documentation see the [Manta user guide](https://github.com/Illumina/manta/blob/master/docs/userGuide/README.md).

<details markdown="1">
<summary>Output files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/manta/<sample>/`**

- `<sample>.manta.diploid_sv.vcf.gz` and `<sample>.manta.diploid_sv.vcf.gz.tbi`
  - VCF with tabix index containing SVs and indels scored and genotyped under a diploid model for the sample.
  </details>

<details markdown="1">
<summary>Output files for tumor-only samples</summary>

**Output directory: `{outdir}/variantcalling/manta/<sample>/`**

- `<sample>.manta.tumor_sv.vcf.gz` and `<sample>.manta.tumor_sv.vcf.gz.tbi`
  - VCF with tabix index containing a subset of the candidateSV.vcf.gz file after removing redundant candidates and small indels less than the minimum scored variant size (50 by default). The SVs are not scored, but include additional details: (1) paired and split read supporting evidence counts for each allele (2) a subset of the filters from the scored tumor-normal model are applied to the single tumor case to improve precision.
  </details>

<details markdown="1">
<summary>Output files for tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/manta/<tumorsample_vs_normalsample>/`**

- `<tumorsample_vs_normalsample>.manta.diploid_sv.vcf.gz` and `<tumorsample_vs_normalsample>.manta.diploid_sv.vcf.gz.tbi`
  - VCF with tabix index containing SVs and indels scored and genotyped under a diploid model for the sample. In the case of a tumor/normal subtraction, the scores in this file do not reflect any information from the tumor sample.
- `<tumorsample_vs_normalsample>.manta.somatic_sv.vcf.gz` and `<tumorsample_vs_normalsample>.manta.somatic_sv.vcf.gz.tbi`
  - VCF with tabix index containing SVs and indels scored under a somatic variant model.
  </details>

#### TIDDIT

[TIDDIT](https://github.com/SciLifeLab/TIDDIT) identifies intra and inter-chromosomal translocations, deletions, tandem-duplications and inversions. For further reading and documentation see the [TIDDIT manual](https://github.com/SciLifeLab/TIDDIT/blob/master/README.md).

<details markdown="1">
<summary>Output files for normal and tumor-only samples</summary>

**Output directory: `{outdir}/variantcalling/tiddit/<sample>/`**

- `<sample>.tiddit.vcf.gz` and `<sample>.tiddit.vcf.gz.tbi`
  - VCF with tabix index containing SV calls
- `<sample>.tiddit.ploidies.tab`
  - tab file describing the estimated ploidy and coverage across each contig

</details>

<details markdown="1">
<summary>Output files for tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/tiddit/<tumorsample_vs_normalsample>/`**

- `<tumorsample_vs_normalsample>.tiddit.normal.vcf.gz` and `<tumorsample_vs_normalsample>.tiddit.normal.vcf.gz.tbi`
  - VCF with tabix index containing SV calls
- `<tumorsample_vs_normalsample>.tiddit.tumor.vcf.gz` and `<tumorsample_vs_normalsample>.tiddit.tumor.vcf.gz.tbi`
  - VCF with tabix index containing SV calls
- `<tumorsample_vs_normalsample>_sv_merge.tiddit.vcf.gz` and `<tumorsample_vs_normalsample>_sv_merge.tiddit.vcf.gz.tbi`
  - merged tumor/normal VCF with tabix index
- `<tumorsample_vs_normalsample>.tiddit.ploidies.tab`
  - tab file describing the estimated ploidy and coverage across each contig

</details>

### Sample heterogeneity, ploidy and CNVs

#### ASCAT

[ASCAT](https://github.com/VanLoo-lab/ascat) is a software for performing allele-specific copy number analysis of tumor samples and for estimating tumor ploidy and purity (normal contamination).
It infers tumor purity and ploidy and calculates whole-genome allele-specific copy number profiles.
The [ASCAT](https://github.com/VanLoo-lab/ascat) process gives several images as output, described in detail in this [book chapter](http://www.ncbi.nlm.nih.gov/pubmed/22130873).
Running ASCAT on NGS data requires that the BAM files are converted into BAF and LogR values.
This is done internally using the software [AlleleCount](https://github.com/cancerit/alleleCount). For further reading and documentation see the [ASCAT manual](https://www.crick.ac.uk/research/labs/peter-van-loo/software).

<details markdown="1">
<summary>Output files for tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/ascat/<tumorsample_vs_normalsample>/`**

- `<tumorsample_vs_normalsample>.tumour.ASCATprofile.png`
  - image with information about allele-specific copy number profile
- `<tumorsample_vs_normalsample>.tumour.ASPCF.png`
  - image with information about allele-specific copy number segmentation
- `<tumorsample_vs_normalsample>.before_correction_Tumour.<tumorsample_vs_normalsample>.tumour.png`
  - image with information about raw profile of tumor sample of logR and BAF values before GC correction
- `<tumorsample_vs_normalsample>.before_correction_Tumour.<tumorsample_vs_normalsample>.germline.png`
  - image with information about raw profile of normal sample of logR and BAF values before GC correction
- `<tumorsample_vs_normalsample>.after_correction_GC_Tumour.<tumorsample_vs_normalsample>.tumour.png`
  - image with information about GC and RT corrected logR and BAF values of tumor sample after GC correction
- `<tumorsample_vs_normalsample>.after_correction_GC_Tumour.<tumorsample_vs_normalsample>.germline.png`
  - image with information about GC and RT corrected logR and BAF values of normal sample after GC correction
- `<tumorsample_vs_normalsample>.tumour.sunrise.png`
  - image visualising the range of ploidy and tumor percentage values
- `<tumorsample_vs_normalsample>.metrics.txt`
  - file with information about different metrics from ASCAT profiles
- `<tumorsample_vs_normalsample>.cnvs.txt`
  - file with information about CNVS
- `<tumorsample_vs_normalsample>.purityploidy.txt`
  - file with information about purity and ploidy
- `<tumorsample_vs_normalsample>.segments.txt`
  - file with information about copy number segments
- `<tumorsample_vs_normalsample>.tumour_tumourBAF.txt` and `<tumorsample_vs_normalsample>.tumour_normalBAF.txt`
  - file with beta allele frequencies
- `<tumorsample_vs_normalsample>.tumour_tumourLogR.txt` and `<tumorsample_vs_normalsample>.tumour_normalLogR.txt`
  - file with total copy number on a logarithmic scale

The text file `<tumorsample_vs_normalsample>.cnvs.txt` contains predictions about copy number state for all the segments.
The output is a tab delimited text file with the following columns:

- _chr_: chromosome number
- _startpos_: start position of the segment
- _endpos_: end position of the segment
- _nMajor_: number of copies of one of the allels (for example the chromosome inherited of one parent)
- _nMinor_: number of copies of the other allele (for example the chromosome inherited of the other parent)

The file `<tumorsample_vs_normalsample>.cnvs.txt` contains all segments predicted by ASCAT, both those with normal copy number (nMinor = 1 and nMajor =1) and those corresponding to copy number aberrations.

</details>

#### CNVKit

[CNVKit](https://cnvkit.readthedocs.io/en/stable/) is a toolkit to infer and visualize copy number from high-throughput DNA sequencing data. It is designed for use with hybrid capture, including both whole-exome and custom target panels, and short-read sequencing platforms such as Illumina. For further reading and documentation, see the [CNVKit Documentation](https://cnvkit.readthedocs.io/en/stable/plots.html)

<details markdown="1">
<summary>Output files for normal and tumor-only samples</summary>

**Output directory: `{outdir}/variantcalling/cnvkit/<sample>/`**

- `<sample>.antitargetcoverage.cnn`
  - file containing coverage information
- `<sample>.targetcoverage.cnn`
  - file containing coverage information
- `<sample>-diagram.pdf`
  - file with plot of copy numbers or segments on chromosomes
- `<sample>-scatter.png`
  - file with plot of bin-level log2 coverages and segmentation calls
- `<sample>.bintest.cns`
  - file containing copy number segment information
- `<sample>.cnr`
  - file containing copy number ratio information
- `<sample>.cns`
  - file containing copy number segment information
- `<sample>.call.cns`
  - file containing copy number segment information
- `<sample>.genemetrics.tsv`
  - file containing per gene copy number information (if input files are annotated)
  </details>

<details markdown="1">
<summary>Output files for tumor/normal samples</summary>

**Output directory: `{outdir}/variantcalling/cnvkit/<tumorsample_vs_normalsample>/`**

- `<normalsample>.antitargetcoverage.cnn`
  - file containing coverage information
- `<normalsample>.targetcoverage.cnn`
  - file containing coverage information
- `<tumorsample>.antitargetcoverage.cnn`
  - file containing coverage information
- `<tumorsample>.targetcoverage.cnn`
  - file containing coverage information
- `<tumorsample>.bintest.cns`
  - file containing copy number segment information
- `<tumorsample>-scatter.png`
  - file with plot of bin-level log2 coverages and segmentation calls
- `<tumorsample>-diagram.pdf`
  - file with plot of copy numbers or segments on chromosomes
- `<tumorsample>.cnr`
  - file containing copy number ratio information
- `<tumorsample>.cns`
  - file containing copy number segment information
- `<tumorsample>.call.cns`
  - file containing copy number segment information
- `<tumorsample>.genemetrics.tsv`
  - file containing per gene copy number information (if input files are annotated)
  </details>

#### Control-FREEC

[Control-FREEC](https://github.com/BoevaLab/FREEC) is a tool for detection of copy-number changes and allelic imbalances (including loss of heterozygoity (LOH)) using deep-sequencing data.
[Control-FREEC](https://github.com/BoevaLab/FREEC) automatically computes, normalizes, segments copy number and beta allele frequency profiles, then calls copy number alterations and LOH.
It also detects subclonal gains and losses and evaluates the most likely average ploidy of the sample. For further reading and documentation see the [Control-FREEC Documentation](http://boevalab.inf.ethz.ch/FREEC/tutorial.html).

<details markdown="1">
<summary>Output files for tumor-only and tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/controlfreec/{tumorsample,tumorsample_vs_normalsample}/`**

- `config.txt`
  - Configuration file used to run Control-FREEC
- `<tumorsample>_BAF.png` and `<tumorsample_vs_normalsample>_BAF.png`
  - image of BAF plot
- `<tumorsample>_ratio.log2.png` and `<tumorsample_vs_normalsample>_ratio.log2.png`
  - image of ratio log2 plot
- `<tumorsample>_ratio.png` and `<tumorsample_vs_normalsample>_ratio.png`
  - image of ratio plot
- `<tumorsample>.bed` and `<tumorsample_vs_normalsample>.bed`
  - translated output to a .BED file (so to view it in the UCSC Genome Browser)
- `<tumorsample>.circos.txt` and `<tumorsample_vs_normalsample>.circos.txt`
  - translated output to the Circos format
- `<tumorsample>.p.value.txt` and `<tumorsample_vs_normalsample>.p.value.txt`
  - CNV file containing p_values for each call
- `<tumorsample>_BAF.txt` and `<tumorsample_vs_normalsample>.mpileup.gz_BAF.txt`
  - file with beta allele frequencies for each possibly heterozygous SNP position
- `<tumorsample_vs_normalsample>.tumor.mpileup.gz_CNVs`
  - file with coordinates of predicted copy number alterations
- `<tumorsample>_info.txt` and `<tumorsample_vs_normalsample>.tumor.mpileup.gz_info.txt`
  - parsable file with information about FREEC run
- ` <tumorsample>_ratio.BedGraph` and `<tumorsample_vs_normalsample>.tumor.mpileup.gz_ratio.BedGraph `
  - file with ratios in BedGraph format for visualization in the UCSC genome browser. The file contains tracks for normal copy number, gains and losses, and copy neutral LOH (\*).
- `<tumorsample>_ratio.txt` and `<tumorsample_vs_normalsample>.tumor.mpileup.gz_ratio.txt`
  - file with ratios and predicted copy number alterations for each window
- `<tumorsample>_sample.cpn` and `<tumorsample_vs_normalsample>.tumor.mpileup.gz_sample.cpn`
  - files with raw copy number profiles for the tumor sample
- `<tumorsample_vs_normalsample>.normal.mpileup.gz_control.cpn`
  - files with raw copy number profiles for the control sample
- `<GC_profile.<tumorsample>.cpn>`
  - file with GC-content profile

</details>

### Microsatellite instability (MSI)

[Microsatellite instability](https://en.wikipedia.org/wiki/Microsatellite_instability) is a genetic condition associated with deficiencies in the mismatch repair (MMR) system which causes a tendency to accumulate a high number of mutations (SNVs and indels).
An altered distribution of microsatellite length is associated with a missed replication slippage which would be corrected under normal MMR conditions.

#### MSIsensorPro

[MSIsensorPro](https://github.com/xjtu-omics/msisensor-pro) is a tool to detect the MSI status of a tumor scanning the length of the microsatellite regions.
It requires a normal sample for each tumour to differentiate the somatic and germline cases. For further reading see the [MSIsensor paper](https://www.ncbi.nlm.nih.gov/pubmed/24371154).

<details markdown="1">
<summary>Output files for tumor/normal paired samples</summary>

**Output directory: `{outdir}/variantcalling/msisensor/<tumorsample_vs_normalsample>/`**

- `<tumorsample_vs_normalsample>`
  - MSI score output, contains information about the number of somatic sites.
- `<tumorsample_vs_normalsample>_dis`
  - The normal and tumor length distribution for each microsatellite position.
- `<tumorsample_vs_normalsample>_germline`
  - Somatic sites detected.
- `<tumorsample_vs_normalsample>_somatic`
  - Germline sites detected.
  </details>

### Concatenation

Germline VCFs from `DeepVariant`, `FreeBayes`, `HaplotypeCaller`, `Haplotyper`, `Manta`, `bcftools mpileup`, `Strelka2`, or `Tiddit` are concatenated with `bcftools concat`. The field `SOURCE` is added to the VCF header to report the variant caller.

<details markdown="1">
<summary>Concatenated VCF-files for normal samples</summary>

**Output directory: `{outdir}/variantcalling/concat/<sample>/`**

- `<sample>.germline.vcf.gz` and `<sample>.germline.vcf.gz.tbi`
  - VCF with tabix index

</details>

## Variant annotation

This directory contains results from the final annotation steps: two tools are used for annotation, [snpEff](http://snpeff.sourceforge.net/) and [VEP](https://www.ensembl.org/info/docs/tools/vep/index.html). Both results can also be combined by setting `--tools merge`.
All variants present in the called VCF files are annotated. For some variant callers this can mean that the variants are already filtered by `PASS`, for some this needs to be done during post-processing.

### snpEff

[snpeff](http://snpeff.sourceforge.net/) is a genetic variant annotation and effect prediction toolbox.
It annotates and predicts the effects of variants on genes (such as amino acid changes) using multiple databases for annotations.
The generated VCF header contains the software version and the used command line. For further reading and documentation see the [snpEff manual](http://snpeff.sourceforge.net/SnpEff_manual.html#outputSummary).

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/annotation/{sample,tumorsample_vs_normalsample}`**

- `{sample,tumorsample_vs_normalsample}.<variantcaller>_snpEff.ann.vcf.gz` and `{sample,tumorsample_vs_normalsample}.<variantcaller>_snpEff.ann.vcf.gz.tbi`
  - VCF with tabix index
  </details>

### VEP

[VEP (Variant Effect Predictor)](https://www.ensembl.org/info/docs/tools/vep/index.html), based on `Ensembl`, is a tool to determine the effects of all sorts of variants, including SNPs, indels, structural variants, CNVs.
The generated VCF header contains the software version, also the version numbers for additional databases like [Clinvar](https://www.ncbi.nlm.nih.gov/clinvar/) or [dbSNP](https://www.ncbi.nlm.nih.gov/snp/) used in the [VEP](https://www.ensembl.org/info/docs/tools/vep/index.html) line.
The format of the [consequence annotations](https://www.ensembl.org/info/genome/variation/prediction/predicted_data.html) is also in the VCF header describing the `INFO` field.
For further reading and documentation see the [VEP manual](https://www.ensembl.org/info/docs/tools/vep/index.html).

Currently, it contains:

- _Consequence_: impact of the variation, if there is any
- _Codons_: the codon change, i.e. cGt/cAt
- _Amino_acids_: change in amino acids, i.e. R/H if there is any
- _Gene_: ENSEMBL gene name
- _SYMBOL_: gene symbol
- _Feature_: actual transcript name
- _EXON_: affected exon
- _PolyPhen_: prediction based on [PolyPhen](http://genetics.bwh.harvard.edu/pph2/)
- _SIFT_: prediction by [SIFT](http://sift.bii.a-star.edu.sg/)
- _Protein_position_: Relative position of amino acid in protein
- _BIOTYPE_: Biotype of transcript or regulatory feature

plus any additional filed selected via the plugins: [dbNSFP](https://sites.google.com/site/jpopgen/dbNSFP), [LOFTEE](https://github.com/konradjk/loftee), [SpliceAI](https://spliceailookup.broadinstitute.org/), [SpliceRegion](https://www.ensembl.info/2018/10/26/cool-stuff-the-vep-can-do-splice-site-variant-annotation/).

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/annotation/{sample,tumorsample_vs_normalsample}`**

- `{sample,tumorsample_vs_normalsample}.<variantcaller>_VEP.ann.vcf.gz` and `{sample,tumorsample_vs_normalsample}.<variantcaller>_VEP.ann.vcf.gz.tbi`
  - VCF with tabix index

</details>

### BCFtools annotate

[BCFtools annotate](https://samtools.github.io/bcftools/bcftools.html#annotate) is used to add annotations to VCF files. The annotations are added to the INFO column of the VCF file. The annotations are added to the VCF header and the VCF header is updated with the new annotations. For further reading and documentation see the [BCFtools annotate manual](https://samtools.github.io/bcftools/bcftools.html#annotate).

<details markdown="1">
<summary>Output files for all samples</summary>

- `{sample,tumorsample_vs_normalsample}.<variantcaller>_bcf.ann.vcf.gz` and `{sample,tumorsample_vs_normalsample}.<variantcaller>_bcf.ann.vcf.gz.tbi`
  - VCF with tabix index

</details>

## Quality control and reporting

### Quality control

#### FastQC

[FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) gives general quality metrics about your sequenced reads. It provides information about the quality score distribution across your reads, per base sequence content (%A/T/G/C), adapter contamination and overrepresented sequences. For further reading and documentation see the [FastQC help pages](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/Help/).

The plots display:

- Sequence counts for each sample.
- Sequence Quality Histograms: The mean quality value across each base position in the read.
- Per Sequence Quality Scores: The number of reads with average quality scores. Shows if a subset of reads has poor quality.
- Per Base Sequence Content: The proportion of each base position for which each of the four normal DNA bases has been called.
- Per Sequence GC Content: The average GC content of reads. Normal random library typically have a roughly normal distribution of GC content.
- Per Base N Content: The percentage of base calls at each position for which an N was called.
- Sequence Length Distribution.
- Sequence Duplication Levels: The relative level of duplication found for each sequence.
- Overrepresented sequences: The total amount of overrepresented sequences found in each library.
- Adapter Content: The cumulative percentage count of the proportion of your library which has seen each of the adapter sequences at each position.

<details markdown="1">
<summary>Output files for all samples</summary>

:::note
The FastQC plots displayed in the MultiQC report shows _untrimmed_ reads. They may contain adapter sequence and potentially regions with low quality.
:::
**Output directory: `{outdir}/reports/fastqc/<sample-lane>`**

- `<sample-lane_1>_fastqc.html` and `<sample-lane_2>_fastqc.html`
  - [FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) report containing quality metrics for your untrimmed raw FastQ files
- `<sample-lane_1>_fastqc.zip` and `<sample-lane_2>_fastqc.zip`
  - Zip archive containing the [FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) report, tab-delimited data file and plot images

> **NB:** The FastQC plots displayed in the [MultiQC](https://multiqc.info/) report shows _untrimmed_ reads.
> They may contain adapter sequence and potentially regions with low quality.

</details>

#### FastP

[FastP](https://github.com/OpenGene/fastp) is a tool designed to provide all-in-one preprocessing for FastQ files and is used for trimming and splitting. The tool then determines QC metrics for the processed reads.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/fastp/<sample>`**

- `<sample-lane>_fastp.html`
  - report in HTML format
- `<sample-lane>_fastp.json`
  - report in JSON format
- `<sample-lane>_fastp.log`
  - FastQ log file

</details>

#### Mosdepth

[Mosdepth](https://github.com/brentp/mosdepth) reports information for the evaluation of the quality of the provided alignment data.
In short, the basic statistics of the alignment (number of reads, coverage, GC-content, etc.) are summarized and a number of useful graphs are produced.
For further reading and documentation see the [Mosdepth documentation](https://github.com/brentp/mosdepth).

Plots will show:

- cumulative coverage distribution
- absolute coverage distribution
- average coverage per contig/chromosome

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/mosdepth/<sample>`**

- `<sample>.{sorted,md,recal}.mosdepth.global.dist.txt`
  - file used by [MultiQC](https://multiqc.info/), if `.region` file does not exist
- `<sample>.{sorted,md,recal}.mosdepth.region.dist.txt`
  - file used by [MultiQC](https://multiqc.info/)
- `<sample>.{sorted,md,recal}.mosdepth.summary.txt`
  -A summary of mean depths per chromosome and within specified regions per chromosome.
- `<sample>.{sorted,md,recal}.{per-base,regions}.bed.gz`
  - per-base depth for targeted data, per-window (500bp) depth of WGS
- `<sample>.{sorted,md,recal}.regions.bed.gz.csi`
  - CSI index for per-base depth for targeted data, per-window (500bp) depth of WGS
  </details>

#### NGSCheckMate

[NGSCheckMate](https://github.com/parklab/NGSCheckMate) is a tool for determining whether samples come from the same genetic individual, using a set of commonly heterozygous SNPs. This enables for the detecting of sample mislabelling events. The output includes a text file indicating whether samples have matched or not according to the algorithm, as well as a dendrogram visualising these results.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/ngscheckmate/`**

- `ngscheckmate_all.txt`
  - Tab delimited text file listing all the comparisons made, whether they were considered as a match, with the correlation and a normalised depth.
- `ngscheckmate_matched.txt`
  - Tab delimited text file listing only the comparison that were considered to match, with the correlation and a normalised depth.
- `ngscheckmate_output_corr_matrix.txt`
  - Tab delimited text file containing a matrix of all correlations for all comparisons made.
- `vcfs/<sample>.vcf.gz`
  - Set of vcf files for each sample. Contains calls for the set of SNP positions used to calculate sample relatedness.
  </details>

#### GATK MarkDuplicates reports

More information in the [GATK MarkDuplicates section](#gatk-markduplicates)

Duplicates can arise during sample preparation _e.g._ library construction using PCR.
Duplicate reads can also result from a single amplification cluster, incorrectly detected as multiple clusters by the optical sensor of the sequencing instrument.
These duplication artifacts are referred to as optical duplicates. If [GATK MarkDuplicates](https://gatk.broadinstitute.org/hc/en-us/articles/5358880192027-MarkDuplicates-Picard-) is used, the metrics file generated by the tool is used, if [`GATK MarkDuplicatesSpark`](https://gatk.broadinstitute.org/hc/en-us/articles/5358833264411-MarkDuplicatesSpark) is used the report is generated by [GATK4 EstimateLibraryComplexity](https://gatk.broadinstitute.org/hc/en-us/articles/5358838684187-EstimateLibraryComplexity-Picard-) on the mapped BAM files.
For further reading and documentation see the [MarkDuplicates manual](https://software.broadinstitute.org/gatk/documentation/tooldocs/4.1.2.0/picard_sam_markduplicates_MarkDuplicates.php).

The plot will show:

- duplication statistics

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/markduplicates/<sample>`**

- `<sample>.md.cram.metrics`
  - file used by [MultiQC](https://multiqc.info/)
  </details>

#### Sentieon Dedup reports

Sentieon's DNAseq subroutine Dedup produces a metrics report much like the one produced by GATK's MarkDuplicates. The Dedup metrics are imported into MultiQC as custom content and displayed in a table.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/sentieon_dedup/<sample>`**

- `<sample>.dedup.cram.metrics`
  - file used by [MultiQC](https://multiqc.info/).
  </details>

#### samtools stats

[samtools stats](https://www.htslib.org/doc/samtools.html) collects statistics from CRAM files and outputs in a text format.
For further reading and documentation see the [`samtools` manual](https://www.htslib.org/doc/samtools.html#COMMANDS_AND_OPTIONS).

The plots will show:

- Alignment metrics.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/samtools/<sample>`**

- `<sample>.{sorted,md,recal}.samtools.stats.out`
  - Raw statistics used by `MultiQC`

</details>

#### bcftools stats

[bcftools stats](https://samtools.github.io/bcftools/bcftools.html#stats) produces a statistics text file which is suitable for machine processing and can be plotted using plot-vcfstats.
For further reading and documentation see the [bcftools stats manual](https://samtools.github.io/bcftools/bcftools.html#stats).

Plots will show:

- Stats by non-reference allele frequency, depth distribution, stats by quality and per-sample counts, singleton stats, etc.
- Note: When using [Strelka2](https://github.com/Illumina/strelka), there will be no depth distribution plot, as Strelka2 does not report the INFO/DP field

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/bcftools/`**

- `<sample>.<variantcaller>.bcftools_stats.txt`
  - Raw statistics used by `MultiQC`
  </details>

#### VCFtools

[VCFtools](https://vcftools.github.io/) is a program package designed for working with VCF files. For further reading and documentation see the [VCFtools manual](https://vcftools.github.io/man_latest.html#OUTPUT%20OPTIONS).

Plots will show:

- the summary counts of each type of transition to transversion ratio for each `FILTER` category.
- the transition to transversion ratio as a function of alternative allele count (using only bi-allelic SNPs).
- the transition to transversion ratio as a function of SNP quality threshold (using only bi-allelic SNPs).

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/vcftools/`**

- `<sample>.<variantcaller>.FILTER.summary`
  - Raw statistics used by `MultiQC` with a summary of the number of SNPs and Ts/Tv ratio for each FILTER category
- `<sample>.<variantcaller>.TsTv.count`
  - Raw statistics used by `MultiQC` with the Transition / Transversion ratio as a function of alternative allele count. Only uses bi-allelic SNPs.
- `<sample>.<variantcaller>.TsTv.qual`
  - Raw statistics used by `MultiQC` with Transition / Transversion ratio as a function of SNP quality threshold. Only uses bi-allelic SNPs.
  </details>

#### snpEff reports

[snpeff](http://snpeff.sourceforge.net/) is a genetic variant annotation and effect prediction toolbox.
It annotates and predicts the effects of variants on genes (such as amino acid changes) using multiple databases for annotations. For further reading and documentation see the [snpEff manual](http://snpeff.sourceforge.net/SnpEff_manual.html#outputSummary).

The plots will show:

- locations of detected variants in the genome and the number of variants for each location.
- the putative impact of detected variants and the number of variants for each impact.
- the effect of variants at protein level and the number of variants for each effect type.
- the quantity as function of the variant quality score.

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/SnpEff/{sample,tumorsample_vs_normalsample}/<variantcaller>/`**

- `<sample>.<variantcaller>_snpEff.csv`
  - Raw statistics used by [MultiQC](http://multiqc.info)
- `<sample>.<variantcaller>_snpEff.html`
  - Statistics to be visualised with a web browser
- `<sample>.<variantcaller>_snpEff.genes.txt`
  - TXT (tab separated) summary counts for variants affecting each transcript and gene
  </details>

#### VEP reports

[VEP (Variant Effect Predictor)](https://www.ensembl.org/info/docs/tools/vep/index.html), based on `Ensembl`, is a tool to determine the effects of all sorts of variants, including SNPs, indels, structural variants, CNVs. For further reading and documentation see the [VEP manual](https://www.ensembl.org/info/docs/tools/vep/index.html)

<details markdown="1">
<summary>Output files for all samples</summary>

**Output directory: `{outdir}/reports/EnsemblVEP/{sample,tumorsamplt_vs_normalsample}/<variantcaller>/`**

- `<sample>.<variantcaller>_VEP.summary.html`
  - Summary of the VEP run to be visualised with a web browser
  </details>

### Reporting

#### MultiQC

[MultiQC](http://multiqc.info) is a visualization tool that generates a single HTML report summarizing all samples in your project.
Most of the pipeline QC results are visualised in the report and further statistics are available in the report data directory.
Results generated by MultiQC collect pipeline QC from supported tools e.g. FastQC. The pipeline has special steps which also allow the software versions to be reported in the MultiQC output for future traceability. For more information about how to use MultiQC reports, see <http://multiqc.info>.

<details markdown="1">
<summary>Output files</summary>

- `multiqc/`
  - `multiqc_report.html`: a standalone HTML file that can be viewed in your web browser.
  - `multiqc_data/`: directory containing parsed statistics from the different tools used in the pipeline.
  - `multiqc_plots/`: directory containing static images from the report in various formats.
  </details>

### Pipeline information

<details markdown="1">
<summary>Output files</summary>

- `pipeline_info/`
  - Reports generated by Nextflow: `execution_report_<timestamp>.html`, `execution_timeline_<timestamp>.html`, `execution_trace_<timestamp>.txt`, `pipeline_dag_<timestamp>.dot`/`pipeline_dag_<timestamp>.svg` and `manifest_<timestamp>.bco.json`.
  - Reports generated by the pipeline: `pipeline_report.html`, `pipeline_report.txt` and `software_versions.yml`. The `pipeline_report*` files will only be present if the `--email` / `--email_on_fail` parameter's are used when running the pipeline.
  - Parameters used by the pipeline run: `params_<timestamp>.json`.

</details>

[Nextflow](https://www.nextflow.io/docs/latest/tracing.html) provides excellent functionality for generating various reports relevant to the running and execution of the pipeline. This will allow you to troubleshoot errors with the running of the pipeline, and also provide you with other information such as launch commands, run times and resource usage.

## Reference files

Contains reference folders generated by the pipeline. These files are only published, if `--save_reference` is set.

<details markdown="1">
<summary>Output files</summary>

- `bwa/`
  - Index corresponding to the [BWA](https://github.com/lh3/bwa) aligner
- `bwamem2/`
  - Index corresponding to the [BWA-mem2](https://github.com/bwa-mem2/bwa-mem2) aligner
- `cnvkit/`
  - Reference files generated by [CNVKit](https://cnvkit.readthedocs.io/en/stable/)
- `dragmap/`
  - Index corresponding to the [DragMap](https://github.com/Illumina/dragmap) aligner
- `dbsnp/`
  - Tabix index generated by [Tabix](http://www.htslib.org/doc/tabix.html) from the given dbsnp file
- `dict/`
  - Sequence dictionary generated by [GATK4 CreateSequenceDictionary](https://gatk.broadinstitute.org/hc/en-us/articles/5358872471963-CreateSequenceDictionary-Picard-) from the given fasta
- `fai/`
  - Fasta index generated with [samtools faidx](http://www.htslib.org/doc/samtools-faidx.html) from the given fasta
- `germline_resource/`
  - Tabix index generated by [Tabix](http://www.htslib.org/doc/tabix.html) from the given gernline resource file
- `intervals/`
  - Bed files in various stages: .bed, .bed.gz, .bed per chromosome, .bed.gz per chromsome
- `known_indels/`
  - Tabix index generated by [Tabix](http://www.htslib.org/doc/tabix.html) from the given known indels file
- `msi/`
  - [MSIsensorPro](https://github.com/xjtu-omics/msisensor-pro) scan of the reference genome to get microsatellites information
- `pon/`
  - Tabix index generated by [Tabix](http://www.htslib.org/doc/tabix.html) from the given panel-of-normals file
  </details>

# Mapping stats with Picard.

[Picard](https://broadinstitute.github.io/picard/) is used to gather mapping quality metrics from sarek's output into a single table that can be easily inspected. The module used for this task is CollectHsMetrics. You may find more information regarding this module and its output [here](http://broadinstitute.github.io/picard/picard-metric-definitions.html#HsMetrics).

# Post-processing and Annotation

[GATK toolkit](https://gatk.broadinstitute.org/hc/en-us) is used to separate SNPs, indels and apply several filters to process the vcf files generated by sarek.

[AWK](https://www.gnu.org/software/gawk/manual/gawk.html) and [BCFtools query](https://samtools.github.io/bcftools/bcftools.html#query) are used to modify the VCF previously processed with GATK toolkit, creating a new variants table which is easier to merge with the annotation data from VEP and Exomiser.

The annotation step is performed apart from sarek in order to provide a more thorough annotation of the variants, which will include prediction of effect and correlation with the disease thanks to [Ensembl's Variant Effect Predictor (VEP)](https://www.ensembl.org/info/docs/tools/vep/index.html) and [exomiser](https://exomiser.readthedocs.io/en/latest/advanced_analysis.html). This latter will also include inheritance typing.

# VEP documentation

You can find information on how to interpret results from VEP toolkit [here](https://www.ensembl.org/info/docs/tools/vep/vep_formats.html#output).

# The Exomiser - A Tool to Annotate and Prioritize Exome Variants

[![GitHub release](https://img.shields.io/github/release/exomiser/Exomiser.svg)](https://github.com/exomiser/Exomiser/releases)
[![CircleCI](https://circleci.com/gh/exomiser/Exomiser/tree/development.svg?style=shield)](https://circleci.com/gh/exomiser/Exomiser/tree/development)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/b518a9448b5b4889a40b3dc660ef585a)](https://www.codacy.com/app/monarch-initiative/Exomiser?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=exomiser/Exomiser&amp;utm_campaign=Badge_Grade)
[![Documentation](https://readthedocs.org/projects/exomiser/badge/?version=latest)](http://exomiser.readthedocs.io/en/latest)
#### Overview:

The Exomiser is a Java program that finds potential disease-causing variants from whole-exome or whole-genome sequencing data.

Starting from a [VCF](https://samtools.github.io/hts-specs/VCFv4.3.pdf) file and a set of phenotypes encoded using the [Human Phenotype Ontology](http://www.human-phenotype-ontology.org) (HPO) it will annotate, filter and prioritise likely causative variants. The program does this based on user-defined criteria such as a variant's predicted pathogenicity, frequency of occurrence in a population and also how closely the given phenotype matches the known phenotype of diseased genes from human and model organism data.

The functional annotation of variants is handled by [Jannovar](https://github.com/charite/jannovar) and uses any of [UCSC](http://genome.ucsc.edu), [RefSeq](https://www.ncbi.nlm.nih.gov/refseq/) or [Ensembl](https://www.ensembl.org/Homo_sapiens/Info/Index) KnownGene transcript definitions and hg19 or hg38 genomic coordinates.

Variants are prioritised according to user-defined criteria on variant frequency, pathogenicity, quality, inheritance pattern, and model organism phenotype data. Predicted pathogenicity data is extracted from the [dbNSFP](http://www.ncbi.nlm.nih.gov/pubmed/21520341) resource. Variant frequency data is taken from the [1000 Genomes](http://www.1000genomes.org/), [ESP](http://evs.gs.washington.edu/EVS), [TOPMed](http://www.uk10k.org/studies/cohorts.html), [UK10K](http://www.uk10k.org/studies/cohorts.html), [ExAC](http://exac.broadinstitute.org) and [gnomAD](http://gnomad.broadinstitute.org/) datasets. Subsets of these frequency and pathogenicity data can be defined to further tune the analysis. Cross-species phenotype comparisons come from our PhenoDigm tool powered by the OWLTools [OWLSim](https://github.com/owlcollab/owltools) algorithm.

The Exomiser was developed by the Computational Biology and Bioinformatics group at the Institute for Medical Genetics and Human Genetics of the Charité - Universitätsmedizin Berlin, the Mouse Informatics Group at the Sanger Institute and other members of the [Monarch initiative](https://monarchinitiative.org).

# Interpreting the Results

Depending on the output options provided, Exomiser will write out at least an HTML results file in the `exomiser` sub-directory of the Exomiser installation.

As a general rule all output files contain a ranked list of genes and/or variants with the top-ranked gene/variant displayed first. The exception being the VCF output which, since version 13.1.0, is sorted according to VCF convention and tabix indexed.

Exomiser attempts to predict the variant or variants likely to be causative of a patient's phenotype and does so by associating them with the gene (or genes in the case of large structural variations) they intersect with on the genomic sequence. Variants occurring in intergenic regions are associated to the closest gene and those overlapping two genes are associated with the gene in which they are predicted to have the largest consequence.

Once associated with a gene, Exomiser uses the compatible modes of inheritance for a variant to assess it in the context of any diseases associated with the gene or any mouse knockout models of that gene. These are all bundled together into a `GeneScore` which features filtered variants located in that gene compatible with a given mode of inheritance. After the filtering steps Exomiser ranks these GeneScores according to descending combined score. The results are then written out to the files and formats specified in the output settings.

As of release 13.2.0 the output files only feature a single, combined output file of ranked genes/variants e.g. when supplying the output options (via an output-options.yaml file) `outputFileName: Pfeiffer-hiphive-exome-PASS_ONLY` and `outputFormats: [TSV_VARIANT, TSV_GENE, VCF]`, the following files will be written out: `Pfeiffer-hiphive-exome-PASS_ONLY.variants.tsv` `Pfeiffer-hiphive-exome-PASS_ONLY.genes.tsv`, `Pfeiffer-hiphive-exome-PASS_ONLY.vcf`

These formats are detailed below.

## HTML

![HTML Description 1](images/exomiser-html-description-1.png)

![HTML Description 2](images/exomiser-html-description-2.png)

## JSON

The JSON file represents the most accurate representation of the data, as it is referenced internally by Exomiser. As such, we don't provide a schema for this, but it has been pretty stable and breaking changes will only occur with major version changes to the software. Minor additions are to be expected for minor releases, as per the [SemVer](https://semver.org) specification.

We recommend using [Python](https://docs.python.org/3/library/json.html?highlight=json#module-json) or [JQ](https://stedolan.github.io/jq/) to extract data from this file.

## TSV GENES

In the genes.tsv file it is possible for a gene to appear multiple times, depending on the MOI it is compatible with, given the filtered variants. For example in the example below MUC6 is ranked 7th under the AD model and 8th under an AR model.

```tsv
#RANK    ID          GENE_SYMBOL    ENTREZ_GENE_ID    MOI    P-VALUE    EXOMISER_GENE_COMBINED_SCORE    EXOMISER_GENE_PHENO_SCORE    EXOMISER_GENE_VARIANT_SCORE    HUMAN_PHENO_SCORE    MOUSE_PHENO_SCORE    FISH_PHENO_SCORE    WALKER_SCORE    PHIVE_ALL_SPECIES_SCORE    OMIM_SCORE    MATCHES_CANDIDATE_GENE    HUMAN_PHENO_EVIDENCE    MOUSE_PHENO_EVIDENCE    FISH_PHENO_EVIDENCE    HUMAN_PPI_EVIDENCE    MOUSE_PPI_EVIDENCE    FISH_PPI_EVIDENCE
1        FGFR2_AD    FGFR2          2263              AD     0.0000     0.9981                          1.0000                         1.0000                         0.8808               1.0000               0.0000               0.5095          1.0000                   1.0000          0                      Jackson-Weiss syndrome (OMIM:123150): Brachydactyly (HP:0001156)-Broad hallux (HP:0010055), Craniosynostosis (HP:0001363)-Craniosynostosis (HP:0001363), Broad thumb (HP:0011304)-Broad metatarsal (HP:0001783), Broad hallux (HP:0010055)-Broad hallux (HP:0010055),       Brachydactyly (HP:0001156)-abnormal sternum morphology (MP:0000157), Craniosynostosis (HP:0001363)-premature cranial suture closure (MP:0000081), Broad thumb (HP:0011304)-abnormal sternum morphology (MP:0000157), Broad hallux (HP:0010055)-abnormal sternum morphology (MP:0000157),           Proximity to FGF14 associated with Spinocerebellar ataxia 27 (OMIM:609307): Broad hallux (HP:0010055)-Pes cavus (HP:0001761),           Proximity to FGF14 Brachydactyly (HP:0001156)-abnormal digit morphology (MP:0002110), Broad thumb (HP:0011304)-abnormal digit morphology (MP:0002110), Broad hallux (HP:0010055)-abnormal digit morphology (MP:0002110),
2        ENPP1_AD    ENPP1          5167              AD     0.0049     0.8690                          0.5773                         0.9996                         0.6972               0.5773               0.5237               0.5066          0.6972                   1.0000          0                      Autosomal recessive hypophosphatemic rickets (ORPHA:289176): Brachydactyly (HP:0001156)-Genu varum (HP:0002970), Craniosynostosis (HP:0001363)-Craniosynostosis (HP:0001363), Broad thumb (HP:0011304)-Tibial bowing (HP:0002982), Broad hallux (HP:0010055)-Genu varum (HP:0002970),       Brachydactyly (HP:0001156)-fused carpal bones (MP:0008915), Craniosynostosis (HP:0001363)-abnormal nucleus pulposus morphology (MP:0006392), Broad thumb (HP:0011304)-fused carpal bones (MP:0008915), Broad hallux (HP:0010055)-fused carpal bones (MP:0008915),       Craniosynostosis (HP:0001363)-ceratohyal cartilage premature perichondral ossification, abnormal (ZP:0012007), Broad thumb (HP:0011304)-cleithrum nodular, abnormal (ZP:0006782),       Proximity to PAPSS2 associated with Brachyolmia 4 with mild epiphyseal and metaphyseal changes (OMIM:612847): Brachydactyly (HP:0001156)-Brachydactyly (HP:0001156), Broad thumb (HP:0011304)-Brachydactyly (HP:0001156), Broad hallux (HP:0010055)-Brachydactyly (HP:0001156),       Proximity to PAPSS2 Brachydactyly (HP:0001156)-abnormal long bone epiphyseal plate morphology (MP:0003055), Craniosynostosis (HP:0001363)-domed cranium (MP:0000440), Broad thumb (HP:0011304)-abnormal long bone epiphyseal plate morphology (MP:0003055), Broad hallux (HP:0010055)-abnormal long bone epiphyseal plate morphology (MP:0003055),
//
7        MUC6_AD      MUC6           4588              AD     0.0096     0.7532                          0.5030                         0.9990                         0.0000               0.0000               0.0000               0.5030          0.5030                   1.0000          0                      Proximity to GKN2 Brachydactyly (HP:0001156)-brachydactyly (MP:0002544), Broad thumb (HP:0011304)-brachydactyly (MP:0002544), Broad hallux (HP:0010055)-brachydactyly (MP:0002544),
8        MUC6_AR      MUC6           4588              AR     0.0096     0.7531                          0.5030                         0.9990                         0.0000               0.0000               0.0000               0.5030          0.5030                   1.0000          0                      Proximity to GKN2 Brachydactyly (HP:0001156)-brachydactyly (MP:0002544), Broad thumb (HP:0011304)-brachydactyly (MP:0002544), Broad hallux (HP:0010055)-brachydactyly (MP:0002544),
```


## TSV VARIANTS

In the variants.tsv file it is possible for a variant, like a gene, to appear multiple times, depending on the MOI it is
compatible with. For example in the example below MUC6 has two variants ranked 7th under the AD model and two ranked 8th
under an AR (compound heterozygous) model. In the AD case the CONTRIBUTING_VARIANT column indicates whether the variant
was (1) or wasn't (0) used for calculating the EXOMISER_GENE_COMBINED_SCORE and EXOMISER_GENE_VARIANT_SCORE.

``` tsv

    #RANK	ID	GENE_SYMBOL	ENTREZ_GENE_ID	MOI	P-VALUE	EXOMISER_GENE_COMBINED_SCORE	EXOMISER_GENE_PHENO_SCORE	EXOMISER_GENE_VARIANT_SCORE	EXOMISER_VARIANT_SCORE	CONTRIBUTING_VARIANT	WHITELIST_VARIANT	VCF_ID	RS_ID	CONTIG	START	END	REF	ALT	CHANGE_LENGTH	QUAL	FILTER	GENOTYPE	FUNCTIONAL_CLASS	HGVS	EXOMISER_ACMG_CLASSIFICATION	EXOMISER_ACMG_EVIDENCE	EXOMISER_ACMG_DISEASE_ID	EXOMISER_ACMG_DISEASE_NAME	CLINVAR_VARIANT_ID	CLINVAR_PRIMARY_INTERPRETATION	CLINVAR_STAR_RATING	GENE_CONSTRAINT_LOEUF	GENE_CONSTRAINT_LOEUF_LOWER	GENE_CONSTRAINT_LOEUF_UPPER	MAX_FREQ_SOURCE	MAX_FREQ	ALL_FREQ	MAX_PATH_SOURCE	MAX_PATH	ALL_PATH
    1	10-123256215-T-G_AD	FGFR2	2263	AD	0.0000	0.9981	1.0000	1.0000	1.0000	1	1		rs121918506	10	123256215	123256215	T	G	0	100.0000	PASS	1|0	missense_variant	FGFR2:ENST00000346997.2:c.1688A>C:p.(Glu563Ala)	LIKELY_PATHOGENIC	PM2,PP3_Strong,PP4,PP5	OMIM:123150	Jackson-Weiss syndrome	28333	LIKELY_PATHOGENIC	1	0.13692	0.074	0.27				REVEL	0.965	REVEL=0.965,MVP=0.9517972
    2	6-132203615-G-A_AD	ENPP1	5167	AD	0.0049	0.8690	0.5773	0.9996	0.9996	1	0		rs770775549	6	132203615	132203615	G	A	0	922.9800	PASS	0/1	splice_donor_variant	ENPP1:ENST00000360971.2:c.2230+1G>A:p.?	UNCERTAIN_SIGNIFICANCE	PVS1_Strong	OMIM:615522	Cole disease		NOT_PROVIDED	0	0.41042	0.292	0.586	GNOMAD_E_SAS	0.0032486517	TOPMED=7.556E-4,EXAC_NON_FINNISH_EUROPEAN=0.0014985314,GNOMAD_E_NFE=0.0017907989,GNOMAD_E_SAS=0.0032486517
    //
    7	11-1018088-TG-T_AD	MUC6	4588	AD	0.0096	0.7532	0.5030	0.9990	0.9990	1	0		rs765231061	11	1018088	1018089	TG	T	-1	441.8100	PASS	0/1	frameshift_variant	MUC6:ENST00000421673.2:c.4712del:p.(Pro1571Hisfs*21)	UNCERTAIN_SIGNIFICANCE					NOT_PROVIDED	0	0.79622	0.656	0.971	GNOMAD_G_NFE	0.0070363074	GNOMAD_E_AMR=0.0030803352,GNOMAD_G_NFE=0.0070363074
    7	11-1018093-G-GT_AD	MUC6	4588	AD	0.0096	0.7532	0.5030	0.9990	0.9989	0	0		rs376177791	11	1018093	1018093	G	GT	1	592.4500	PASS	0/1	frameshift_elongation	MUC6:ENST00000421673.2:c.4707dup:p.(Pro1570Thrfs*136)	NOT_AVAILABLE					NOT_PROVIDED	0	0.79622	0.656	0.971	GNOMAD_G_NFE	0.007835763	GNOMAD_G_NFE=0.007835763
    8	11-1018088-TG-T_AR	MUC6	4588	AR	0.0096	0.7531	0.5030	0.9990	0.9990	1	0		rs765231061	11	1018088	1018089	TG	T	-1	441.8100	PASS	0/1	frameshift_variant	MUC6:ENST00000421673.2:c.4712del:p.(Pro1571Hisfs*21)	UNCERTAIN_SIGNIFICANCE					NOT_PROVIDED	0	0.79622	0.656	0.971	GNOMAD_G_NFE	0.0070363074	GNOMAD_E_AMR=0.0030803352,GNOMAD_G_NFE=0.0070363074
    8	11-1018093-G-GT_AR	MUC6	4588	AR	0.0096	0.7531	0.5030	0.9990	0.9989	1	0		rs376177791	11	1018093	1018093	G	GT	1	592.4500	PASS	0/1	frameshift_elongation	MUC6:ENST00000421673.2:c.4707dup:p.(Pro1570Thrfs*136)	UNCERTAIN_SIGNIFICANCE					NOT_PROVIDED	0	0.79622	0.656	0.971	GNOMAD_G_NFE	0.007835763	GNOMAD_G_NFE=0.007835763
```

## VCF

In the VCF file it is possible for a variant, like a gene, to appear multiple times, depending on the MOI it is compatible with. For example in the example below MUC6 has two variants ranked 7th under the AD model and two ranked 8th under an AR (compound heterozygous) model. In the AD case the CONTRIBUTING_VARIANT column indicates whether the variant was (1) or wasn't (0) used for calculating the EXOMISER_GENE_COMBINED_SCORE and EXOMISER_GENE_VARIANT_SCORE. 
The ``INFO`` field with the ``ID=Exomiser`` describes the internal format of this sub-field. Be aware that for multi-allelic sites, Exomiser will decompose and trim them for the proband sample and this is what will be displayed in the Exomiser ``ID`` sub-field e.g. ``11-1018088-TG-T_AD``.

``` vcf

    ##INFO=<ID=Exomiser,Number=.,Type=String,Description="A pipe-separated set of values for the proband allele(s) from the record with one per compatible MOI following the format: {RANK|ID|GENE_SYMBOL|ENTREZ_GENE_ID|MOI|P-VALUE|EXOMISER_GENE_COMBINED_SCORE|EXOMISER_GENE_PHENO_SCORE|EXOMISER_GENE_VARIANT_SCORE|EXOMISER_VARIANT_SCORE|CONTRIBUTING_VARIANT|WHITELIST_VARIANT|FUNCTIONAL_CLASS|HGVS|EXOMISER_ACMG_CLASSIFICATION|EXOMISER_ACMG_EVIDENCE|EXOMISER_ACMG_DISEASE_ID|EXOMISER_ACMG_DISEASE_NAME}">
    #CHROM	POS	ID	REF	ALT	QUAL	FILTER	INFO	sample
    10	123256215	.	T	G	100	PASS	Exomiser={1|10-123256215-T-G_AD|FGFR2|2263|AD|0.0000|0.9981|1.0000|1.0000|1.0000|1|1|missense_variant|FGFR2:ENST00000346997.2:c.1688A>C:p.(Glu563Ala)|LIKELY_PATHOGENIC|PM2,PP3_Strong,PP4,PP5|OMIM:123150|"Jackson-Weiss syndrome"};GENE=FGFR2;INHERITANCE=AD;MIM=101600	GT:DS:PL	1|0:2.000:50,11,0
    11	1018088	.	TG	T	441.81	PASS	AC=1;AF=0.50;AN=2;BaseQRankSum=7.677;DP=162;DS;Exomiser={7|11-1018088-TG-T_AD|MUC6|4588|AD|0.0096|0.7532|0.5030|0.9990|0.9990|1|0|frameshift_variant|MUC6:ENST00000421673.2:c.4712del:p.(Pro1571Hisfs*21)|UNCERTAIN_SIGNIFICANCE|||""},{8|11-1018088-TG-T_AR|MUC6|4588|AR|0.0096|0.7531|0.5030|0.9990|0.9990|1|0|frameshift_variant|MUC6:ENST00000421673.2:c.4712del:p.(Pro1571Hisfs*21)|UNCERTAIN_SIGNIFICANCE|||""};FS=25.935;HRun=3;HaplotypeScore=1327.2952;MQ=43.58;MQ0=6;MQRankSum=-5.112;QD=2.31;ReadPosRankSum=2.472;set=variant	GT:AD:DP:GQ:PL	0/1:146,45:162:99:481,0,5488
    11	1018093	.	G	GT	592.45	PASS	AC=1;AF=0.50;AN=2;BaseQRankSum=8.019;DP=157;Exomiser={7|11-1018093-G-GT_AD|MUC6|4588|AD|0.0096|0.7532|0.5030|0.9990|0.9989|0|0|frameshift_elongation|MUC6:ENST00000421673.2:c.4707dup:p.(Pro1570Thrfs*136)|NOT_AVAILABLE|||""},{8|11-1018093-G-GT_AR|MUC6|4588|AR|0.0096|0.7531|0.5030|0.9990|0.9989|1|0|frameshift_elongation|MUC6:ENST00000421673.2:c.4707dup:p.(Pro1570Thrfs*136)|UNCERTAIN_SIGNIFICANCE|||""};FS=28.574;HRun=1;HaplotypeScore=1267.6968;MQ=44.06;MQ0=4;MQRankSum=-5.166;QD=3.26;ReadPosRankSum=1.328;set=variant	GT:AD:DP:GQ:PL	0/1:140,42:157:99:631,0,4411
    6	132203615	.	G	A	922.98	PASS	AC=1;AF=0.50;AN=2;BaseQRankSum=-0.671;DP=94;Dels=0.00;Exomiser={2|6-132203615-G-A_AD|ENPP1|5167|AD|0.0049|0.8690|0.5773|0.9996|0.9996|1|0|splice_donor_variant|ENPP1:ENST00000360971.2:c.2230+1G>A:p.?|UNCERTAIN_SIGNIFICANCE|PVS1_Strong|OMIM:615522|"Cole disease"};FS=0.805;HRun=0;HaplotypeScore=3.5646;MQ=56.63;MQ0=0;MQRankSum=1.807;QD=9.82;ReadPosRankSum=-0.900;set=variant2	GT:AD:DP:GQ:PL	0/1:53,41:94:99:953,0,1075
```

The VCF file is tabix-indexed and exomiser ranked alleles can be extracted using ``grep``. For example, to display the top 5 ranked variants, you would issue the command:

``` shell
    zgrep -E '\{[1-5]{1}\|' Pfeiffer-hiphive-exome-PASS_ONLY.vcf.gz
```
