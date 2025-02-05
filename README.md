# Illumina Workflow for Metagenomic Data Processing  


This repository contains a Snakemake workflow for processing Illumina sequencing data, optimized and validated for metagenomics. The end-to-end workflow, `up_illumina_wf_snakefile.smk`, includes steps to process raw reads into taxonomic annotation: quality control, human read filtering, de novo assembly, annotation, and result summarization. The Snakefile is designed to be resource-aware, modular, and easy to configure, with outputs dynamically organized based on the current date.  


## Workflow breakdown: from raw reads to annotation

1. **Dynamic output folder & raw data linking**  
   - Feature: Automatically creates an output folder named based on the current date (e.g., `processed_ddmmyy`).  
   - Step (`hardlink_raw`): Hard-links or copies raw FASTQ files from `raw_data/`. The dynamically generated folder ensures easy data management.

2. **Quality control & deduplication**  
   - Feature: Uses [fastp](https://github.com/OpenGene/fastp) to perform both quality trimming and deduplication in a single run.  
   - Step (`QC_after_dedup`): Generates cleaned, deduplicated reads, and provides an HTML/JSON report detailing quality metrics.

3. **Human read filtering**  
   - Feature: Removes (background contaminating) human reads by aligning against the human reference genome with [bwa-mem2](https://github.com/bwa-mem2/bwa-mem2), then uses [samtools](http://www.htslib.org/) to retain only unmapped reads.  
   - Step (`filter_human`): Generates filtered reads free of host contamination to improve downstream assembly and annotation accuracy.

4. **De novo assembly**  
   - Feature: Assembles filtered reads using [metaSPAdes](https://cab.spbu.ru/software/spades/), optimized for metagenomic data.  
   - Step (`assemble_filtered`): Generates assembled contigs; includes post-processing such as renaming contigs to include run and barcode information for clarity.

5. **Annotation**  
   - Feature: Runs high-speed homology searches with [DIAMOND BLASTX](https://github.com/bbuchfink/diamond) (e-value of `10-5`) to annotate contigs against a protein database.  
   - Step (`blastx_assembled`): Produces a tab-delimited output with taxonomic and functional information for each contig.

6. **Mapping & statistics**  
   - Feature: Maps reads back to the assembled contigs (with `bwa-mem2` + `samtools`) and uses [seqkit](https://bioinf.shenwei.me/seqkit/) for read-count statistics.  
   - Steps (`map_reads_to_contigs`, Statistics & Merging):  Generates coverage information, creates BAM files, and merges coverage and annotation results into summary tables.

7. **Result organization**  
   - Feature: Creates renamed, centrally linked annotation and summary files for easier access and downstream analysis.  
   - Step (`store_completed_annotation_files`): Renames `completed_{sample}_annotation.tsv` files, links them in a central `annotations/` folder, and cleans up temporary files.

8. **Rule prioritization**  
   - Certain rules, such as blastx_assembled and assemble_filtered, have assigned priorities to optimize scheduling and execution.  


## Installation and quick start

1. **Clone the repository:**

    ```
    git clone https://github.com/divprasad/updated_illumina_workflow.git
    cd updated_illumina_workflow
    ```

2. **Set up a `conda` environment:**

    ```
    conda env create -f environment.yaml
    conda activate illumina_wf_env
    ```

3. **Set working directory:**
    Place raw Illumina paired-end FASTQ files in the following directory structure:
    ```
    raw_data/{run}/{sample}_R1_001.fastq.gz
    raw_data/{run}/{sample}_R2_001.fastq.gz
    ```
    This structure is critical for the workflow to recognize `run` and `sample` wildcards correctly. Ensure the file names follow this format.

4. **Running the workflow with minimal settings:**  
    ```
    snakemake -s up_illumina_wf_snakefile.smk --cores 8
    ```
    The pipeline will start using 8 cores, and the results will be saved in a directory named `processed_ddmmyy` (default naming format based on the current date).  


### Project structure & key outputs

**Example project layout** after cloning the repo and executing the workflow:

```
updated_illumina_workflow/
├── raw_data/                         # specified by user
│   └── runXYZ/
│       ├── sampleA_R1_001.fastq.gz
│       └── sampleA_R2_001.fastq.gz
├── up_illumina_wf_snakefile.smk      # snakefile
├── environment.yaml                  # dependencies for conda installation
├── execute-and-log.sh                # wrapper
├── README.md                         # this file
└── processed_ddmmyy/                 # generated by executing snakefile
    ├── runXYZ/
    │   └── sampleA/
    │       ├── raw/
    │       ├── dedup_qc/
    │       ├── filtered/
    │       ├── assembly/
    │       ├── mappings/
    │       ├── summary/
    │       └── ...
    └── ...
```

- **`raw_data/`** holds the raw FASTQ files to be processed.
- **`processed_ddmmyy/`** is generated by the workflow and contains processed outputs, organized into subfolders for each `{run}` and `{sample}`, following the structure `{OUTPUT_FOLDER}/{run}/{sample}`  


### Output directories & files

1. **`dedup_qc/`** Contains the **QC**ed and **deduplicated** reads and fastp reports.
2. **`filtered/`** Holds the **human-filtered** reads after removing host contamination.
3. **`assembly/contigs.fasta`** Final **assembled** contigs from metaSPAdes.
4. **`{sample}_annotation.tsv`** DIAMOND BLASTX annotation results for each assembled contig.
5. **`mappings/{sample}_mappings.bam`** BAM files with reads mapped back to contigs.
6. **`summary/`** Summary tables of coverage, read statistics, and merged annotation data for quick reference.  


## Advanced usage and configuration

**To run the workflow with logging, use** `execute-and-log.sh`, which automates the process.  
Alternatively, launch the workflow by specifying the number of cores, memory, and other parameters.  

```bash
snakemake -s up_illumina_wf_snakefile.smk \
    --resources mem_gb=192 \
    --cores 24 \
    --rerun-triggers mtime \
    --rerun-incomplete \
    --latency-wait 30 \
    --max-jobs-per-second 2 \
    --max-status-checks-per-second 4
```  

- **Default output**: If no custom folder is specified by passing the `config` flag to snakemake, default output directory is `processed_ddmmyy`.
- **`cores`**: Use up to 24 CPU cores
- **`resources mem_gb`**: Allocate 192 GB memory (roughly 8× the number of cores).  
- **`rerun-triggers`**: Force a rerun if file modification times indicate that inputs have changed.  
- **`rerun-incomplete`**: Rerun incomplete jobs from previous executions.
- **`latency-wait`**: Waits up to 30 seconds for input files (useful for network filesystems).
- **Job submission and status check rate**: Limit new job submission rate (`max-jobs-per-second`) to 2 new jobs per second; limit job status check rate (`max-status-checks-per-second`) to 4 checks per second  
- **Configuration**: Pass configuration variable flag to snakemake - to set a custom output folder:

  ```
  snakemake -s up_illumina_wf_snakefile.smk \
      --config OUTPUT_FOLDER="processed_mydate" \
      --cores 16
  ```  


### Resource Allocation and Priorities
  - Threads and Memory: Each rule in the Snakefile can dynamically allocate threads and memory.  
  - Adjusting Resources: Modify the `threads:` or `resources:` directives inside each rule for finer control.
  - Adjusting Priorities: Some rules include **priority settings** to optimize execution order. This ensures that computationally intensive steps start earlier, preventing bottlenecks and reducing total runtime. **Example Priority Assignments**:  
    - **`blastx_assembled`**  → **Priority 2** - As the most time-consuming step, it is executed first.  
    - **`assemble_filtered`** → **Priority 1** - This step runs with higher priority than all other jobs.

> **Note:** By default, rules have priority 0. Raise or lower priority levels as needed.  


## Acknowledgements  

**Adapted by:** [Div Prasad](https://github.com/divprasad/) (Jul'24–Feb'25)  
**Original Workflow by:** Nathalie Worp & David Nieuwenhuijse  

This Snakemake pipeline is an adaptation of the original work by **Nathalie Worp** and **David Nieuwenhuijse**, incorporating updates, enhancements, and additional features.  

A special thanks to **Nathalie Worp** and **David Nieuwenhuijse** for their contributions; their work laid the foundation for this repository.  
