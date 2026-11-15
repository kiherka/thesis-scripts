# Thesis scripts – compiled collection

Scripts gathered from every run folder used in the course of massively parallel insert and single insert (ATRX promoter) analyses with the original folder structure preserved. This summary, together with Snakelike analysis pipelines for the massively parallel insert project, were designed using Anthropic Claude as a tool (Opus and Sonnet models).

**Contents:** 142 scripts (`.sh`, `.py`, `Snakefile`), 32 workflow YAMLs (pipeline `config/`, SLURM/Snakemake `profile/`, conda `envs/`), 18 demultiplexing barcode lists (`r1`/`r2`, see [Demultiplexing lists](#demultiplexing-lists)) and the 2 pre-existing markdown notes. Data, logs, bigwigs, BAMs and other outputs are excluded. The symlinked `envs/` directories in SLX-26997 / SLX-27843 / SLX-27925 point at the SLX-27649 envs and are not duplicated.

**Sources used for this summary:** the only README-style documents inside the SLX folders are `SLX-27649/260216_VH01410_433_AAHHHJ2M5/PIPELINE_ANALYSIS.md` and `analysis_pros_cons.md`. No dedicated "memory" files exist in the SLX folders, so the rest of this summary is drawn from script docstrings and header comments.

## Project context

CUT&Tag / CUT&RUN-style sequencing of a library of cell lines/wells carrying a ~500 bp splinkerette insert (`insert_sequence_G3T1.fa`, 525 bp in the integrated genome) at many genomic sites, assayed with BG4 (G-quadruplex), CUTAC, CM, H3K27ac (27ac), ATRX and related antibodies. Common stack:

`demuxFQ` demultiplexing → cutadapt trimming → BWA-mem alignment (hg38 or per-well integrated genomes) → Picard MarkDuplicates → deepTools CPM bigwigs → MACS2 / SEACR peaks → FastQC/MultiQC, orchestrated with Snakemake on SLURM.

## Overview by SLX folder

| Folder | Purpose |
|---|---|
| [SLX-26996](SLX-26996) | Initial run - one replicate of BG4 for all created G3T1 insert cell lines  |
| [SLX-26997](SLX-26997) | An extension of SLX-26996 for achieving the necessary depth |
| [SLX-26997_nano](SLX-26997_nano) | The SLX-26997 library run on a MiSeq instrument to test pooling accuracy|
| [SLX-27392](SLX-27392) | Single-locus ATRX promoter edits - checks G4 presence (BG4 CUT&Tag, with H3K27ac and IgG controls) in the G3T1 / G3T1F edit clones vs WT, three biological replicates (B1–B3) |
| [SLX-27649](SLX-27649) | SPLINK insert discovery pipeline + the insert-peak analysis suite + sequencing of additional probes for the ATRX single-insert experiment |
| [SLX-27843](SLX-27843) | BG4/CUTAC alignment to per-well integrated genomes, peaks - PILOT, only first two rows of Plate 1, attempted manual size selection (BG4SIZE) as well as BG4 and CUTAC probing|
| [SLX-27925](SLX-27925) | Edited cell lines (RT-qPCR correlation, PCA, CPM analyses) - work on additional globin edited cell lines |
| [SLX-28240](SLX-28240) | Multi-replicate, multi-assay validation of insert peaks and specificity - for gold-standard insert cell lines, ones where presence of a G3T1 sequence induces G4 formation where a G4 has not been called before based on the single replicate pilot BG4 CUT&Tag of all cell lines|

---

## SLX-26996

Initial single-replicate BG4 run covering all G3T1 insert cell lines. Its reads are demultiplexed here and re-used as the "Soma" input of the SLX-26997 pipeline, so the two runs are analysed together as one deeper dataset.

| Script | Description |
|---|---|
| [slurm_demuxFQ2.sh](SLX-26996/slurm_demuxFQ2.sh) | Submits two SLURM `demuxFQ` jobs (R1, R2) for the lost-reads FASTQs into `demultiplexed/` (12 h, 32 G). |

## SLX-26997

Extension of SLX-26996 to reach the necessary depth for the single-replicate BG4 pilot. The pipeline merges five already-demultiplexed folders (lane1-4 + Soma, the latter being the SLX-26996 reads), then trims, aligns (hg38_eco), dedups, makes bigwigs and calls peaks. Its BG4 data later serve as biological replicate "BR1" in SLX-28240.

| Script | Description |
|---|---|
| [slurm_demuxFQ2.sh](SLX-26997/slurm_demuxFQ2.sh), [_lane2](SLX-26997/slurm_demuxFQ2_lane2.sh), [_lane3](SLX-26997/slurm_demuxFQ2_lane3.sh), [_lane4](SLX-26997/slurm_demuxFQ2_lane4.sh), [_Soma](SLX-26997/slurm_demuxFQ2_Soma.sh) | Per-lane `demuxFQ` submissions (R1+R2, 72 h, 32 G) into `demultiplexed_lane*/` / `demultiplexed_Soma/`. The un-suffixed script is lane 1; Soma uses the SLX-26996 lost reads. |
| [redemux_excluded.sh](SLX-26997/redemux_excluded.sh) | Re-demultiplexes the 14 excluded samples from the NAS. Run on the login node inside `screen` (NAS not visible to SLURM). |
| [redemux_resume.sh](SLX-26997/redemux_resume.sh) | Resumes the re-demux from Lane 3 R2 after the original script died. |
| [Snakefile](SLX-26997/Snakefile) | Rules: `merge_sample_r1/r2`, `repair_pairs`, `trim_paired`, `align_bwa_mem`, `flagstat_stats`, `mark_duplicates`, `index_dedup_bam`, `bigwig_cpm`, `bam_to_bedgraph`, `macs2_callpeak`, `seacr_callpeak`, `multiqc`, `fastqc_merged`, `bwa_index`. Reuses SLX-27649 conda envs. |
| [run_master.sh](SLX-26997/run_master.sh) | Launches the Snakemake master on the login node (use inside `screen -S smk26997`). |
| [config/config.yaml](SLX-26997/config/config.yaml), [profile/config.yaml](SLX-26997/profile/config.yaml) | Pipeline parameters (trim q20, `-F 780`, MAPQ 10, bigwig bin 3 / smooth 12) and Snakemake SLURM profile. |

## SLX-26997_nano

The SLX-26997 library re-run on a MiSeq to test pooling accuracy; only demultiplexing is scripted.

| Script | Description |
|---|---|
| [slurm_demuxFQ2.sh](SLX-26997_nano/slurm_demuxFQ2.sh) | `demuxFQ` R1/R2 submission for the nano library (12 h, 32 G). |

## SLX-27392

Located at `/mnt/nas-data/sblab/group_folders/herka01/SLX-27392/251022_VH01410_394_222CCJCNX/`. Single-locus ATRX promoter edits: tests for G4 presence (BG4 CUT&Tag, with H3K27ac and IgG controls) in the G3T1-1/2/3 and G3T1F-1/2/3 edit clones against WT, with three biological replicates (B1–B3, two technical replicates T1/T2 each). B3 was demultiplexed separately (`BR3_r1/r2.txt`). The Snakefile aligns to hg38 or edited reference genomes (G3T1, G3T1F, G3T4 variants, selected per sample) and produces CPM bigwigs (bs3 / s600 / s250); it has no peak-calling rules.

| Script | Description |
|---|---|
| [slurm_demuxFQ2.sh](SLX-27392/251022_VH01410_394_222CCJCNX/slurm_demuxFQ2.sh), [slurm_demuxFQ2_r2.sh](SLX-27392/251022_VH01410_394_222CCJCNX/slurm_demuxFQ2_r2.sh), [slurm_demuxFQ2_br3.sh](SLX-27392/251022_VH01410_394_222CCJCNX/slurm_demuxFQ2_br3.sh) | `demuxFQ` submissions: R1+R2 for the whole run, R2 only (re-run), and the BR3 replicate into `demultiplexed-br3/`. |
| [Snakefile](SLX-27392/251022_VH01410_394_222CCJCNX/Snakefile) | Rules: `demultiplex_r1/r2`, `trim_paired`, `align_bwa_mem`, `flagstat_stats`, `mark_duplicates`, `index_dedup_bam`, `bigwig_bs3`, `bigwig_s600`, `bigwig_s250`, `multiqc`, `fastqc_demux`, `bwa_index`. This is the earliest form of the pipeline reused in later SLX folders. |
| [analyse_cpm_v2.py](SLX-27392/251022_VH01410_394_222CCJCNX/analyse_cpm_v2.py) | Byte-identical copy of [SLX-27925/analyse_cpm_v2.py](SLX-27925/analyse_cpm_v2.py) (globin-line CPM analysis with BG4 replicate exclusions); it is not specific to the ATRX promoter edits. |
| [config/config.yaml](SLX-27392/251022_VH01410_394_222CCJCNX/config/config.yaml), `envs/` (cutadapt, bwa_samtools, picard, deeptools, fastqc, multiqc) | Pipeline configuration and conda environments. (A duplicate `config copy/` and the `Miniforge3.sh` installer are not included.) |

## SLX-27649

Main SPLINK discovery folder, which also holds sequencing of additional probes for the ATRX single-insert experiment. It contains [260216_VH01410_433_AAHHHJ2M5/](SLX-27649/260216_VH01410_433_AAHHHJ2M5) (pipeline; the ATRX CPM scripts `analyse_cpm.py`, `analyse_atrx_cpm.py` and `mean_traces.py` belong to the ATRX experiment), [insert_peak_analysis/](SLX-27649/insert_peak_analysis) (downstream analysis of the 679 integrated genomes, where "empty" = cell lines with no insert) and a top-level CPM script.

**Gold-standard definition** (`analyse_insert_peaks.py`): an insert junction is gold standard if **both** MACS2 and SEACR call a peak over the junction region **and** that peak is absent from ≥80 % of the empty (no-insert) cell lines, i.e. present in ≤20 % (`MAX_EMPTY_FRAC = 0.20`). Samples with <1 M mapped reads are skipped. This yields the 120 BG4 gold-standard junctions (`gold_standard_junctions_bg4.tsv`). The separate ≥75 % threshold in SLX-28240 is a *replicate-consistency* criterion, not part of this definition.

### Pipeline – `260216_VH01410_433_AAHHHJ2M5/`

Per [PIPELINE_ANALYSIS.md](SLX-27649/260216_VH01410_433_AAHHHJ2M5/PIPELINE_ANALYSIS.md): demultiplex → cutadapt trim → BWA-mem → Picard dedup → deepTools bigwigs (bs3 / s600 / s250) → FastQC/MultiQC. The Snakefile extends this with SPLINK junction detection: `splink_extract_split_reads`, `splink_index_junctions`, `splink_noise_filter(_combined)`, `splink_junction_consensus_combined`, `splink_integrate_inserts_combined` (builds a per-well integrated genome), `bwa_index_integrated_combined`, `splink_integration_summary_combined`, `splink_bridge_summary`.
[analysis_pros_cons.md](SLX-27649/260216_VH01410_433_AAHHHJ2M5/analysis_pros_cons.md) compares three ways to detect the 500 bp fixed insert: (1) extra-contig + split reads (explicit junctions; heavier reference), (2) trim-and-filter (simple, best for routine use, loses junction info), (3) unusual-insert/SV detection (unnecessary for a known insert, noisy for 500 bp).

| Script | Description |
|---|---|
| [Snakefile](SLX-27649/260216_VH01410_433_AAHHHJ2M5/Snakefile) | Main workflow (above). |
| [run_master.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/run_master.sh), [run_in_screen.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/run_in_screen.sh), [run_pipeline.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/run_pipeline.sh), [submit_master.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/submit_master.sh) | Alternative launchers: login-node tmux, screen, or SLURM master jobs (`cuda` partition, 7 d). |
| [slurm_demuxFQ2.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/slurm_demuxFQ2.sh) | `demuxFQ` R1/R2 submission. |
| [watch_and_launch_27843.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/watch_and_launch_27843.sh) | Waits for the `smk27649` screen session to end, then starts the SLX-27843 pipeline. |
| [restore_and_launch_27843.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/restore_and_launch_27843.sh) | Restores all 679 integrated genomes from NAS, rebuilds missing BWA indexes, launches SLX-27843. |
| [copy_combined_to_nas.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/copy_combined_to_nas.sh) | Backs up the 679 combined integrated genomes + BWA indexes to NAS. |
| [analyse_cpm.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/analyse_cpm.py) | ATRX CPM bigwig analysis; combines two naming conventions as replicates per cell line (WT, 14, 17, 25, 26). |
| [analyse_atrx_cpm.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/analyse_atrx_cpm.py), [analyse_bg4_smoothened_cpm.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/analyse_bg4_smoothened_cpm.py) | Local re-runs of the CPM analysis restricted to ATRX samples / BG4 smoothened (bs3) tracks (expEN7 dataset). |
| [mean_traces.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/mean_traces.py) | Mean ATRX CPM trace per cell line vs WT mean in a 2 kb window at the peak start. |
| **scripts/** | |
| [splink_consensus.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/splink_consensus.py) | `junctions` mode: finds insert–genome junctions in noise-filtered BAMs and builds consensus sequences; `integrate` mode: builds a new reference with the insert at each junction. Called by Snakemake. |
| [make_insert_location_beds.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/make_insert_location_beds.py) | Per-well ±25 bp BED around each hg38 insert site, plus a merged BED of all unique locations. |
| [validate_consensus_vs_bam.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/validate_consensus_vs_bam.py) | Checks consensus junctions against BWA BAMs (total / soft-clipped / chimeric `SA:Z` reads in ±100 bp). |
| [validate_consensus_and_integration.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/validate_consensus_and_integration.py) | Validates clip positions against raw reads and confirms the insert sits at the expected place in integrated genomes. |
| [histogram_signal_regions.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/histogram_signal_regions.py) | Histograms of `signal_regions` counts from peak-noise stats (stringent / combined / loose). |
| [normalised_reads_histogram.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/normalised_reads_histogram.py) | Normalises junction read counts to per-million aligned reads and plots the pooled histogram over 679 samples. |
| [junction_gencode_annotation.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/junction_gencode_annotation.py), [junction_gencode_annotation_optimized.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/junction_gencode_annotation_optimized.py) | Annotates junctions against GENCODE v44 (optimised version adds FANTOM5 enhancers); outputs table + pie chart. |
| [submit_junction_annotation.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/submit_junction_annotation.sh) | SLURM wrapper for the annotation script. |
| [patch_readgroups.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/patch_readgroups.sh) | Adds `@RG` headers to existing BAMs (`samtools addreplacerg`) so Picard MarkDuplicates works. |
| [junction_anno_env.yaml](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/junction_anno_env.yaml), [junction_venv_env.yaml](SLX-27649/260216_VH01410_433_AAHHHJ2M5/scripts/junction_venv_env.yaml) | Conda envs for the annotation scripts. |
| **lumpy/** | LUMPY structural-variant sub-pipeline (rules `lumpy_align`, `lumpy_index`, `lumpy_call`, `filter_insert_svs`, `lumpy_summary`). |
| [lumpy/Snakefile](SLX-27649/260216_VH01410_433_AAHHHJ2M5/lumpy/Snakefile), [run_lumpy.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/lumpy/run_lumpy.sh), [submit_lumpy.sh](SLX-27649/260216_VH01410_433_AAHHHJ2M5/lumpy/submit_lumpy.sh) | Workflow and launchers (screen / SLURM `epyc`, 72 h). |
| [lumpy/scripts/filter_insert_svs.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/lumpy/scripts/filter_insert_svs.py) | Keeps LUMPY VCF records on, or with BND ALT referencing, the insert contig. |
| [lumpy/scripts/lumpy_summary.py](SLX-27649/260216_VH01410_433_AAHHHJ2M5/lumpy/scripts/lumpy_summary.py) | Per-sample counts of insert-related SVs/BND events and top breakpoints. |
| `config/`, `profile/`, `envs/` (cutadapt, bwa_samtools, picard, deeptools, macs2, seacr, fastqc, multiqc, consensus), `lumpy/envs`, `lumpy/profile` | Pipeline config, Snakemake profiles and conda environments. |

### Top-level

| Script | Description |
|---|---|
| [analyse_cpm_v2.py](SLX-27649/analyse_cpm_v2.py) | Copy of the SLX-27925 (globin-edited lines) CPM analysis, v2: excludes specific QC-failed BG4 replicates before averaging/stats; CUTAC unchanged. |

### Insert-peak analysis – `insert_peak_analysis/`

| Script | Description |
|---|---|
| [generate_insert_regions.py](SLX-27649/insert_peak_analysis/generate_insert_regions.py) | BED regions (±1 kb flanks around the insert) in integrated-genome coordinates for each junction. |
| [analyse_insert_peaks.py](SLX-27649/insert_peak_analysis/analyse_insert_peaks.py) | Tests whether 2 kb junction regions overlap SEACR/MACS2 peaks (SLX-26997 BG4 + SLX-27843 BG4/BG4SIZE/CUTAC) and whether EMPTY lines peak at the same region. |
| [insert_number_analysis/insert_number_analysis.py](SLX-27649/insert_peak_analysis/insert_number_analysis/insert_number_analysis.py) | Histograms of insert counts per cell line. |
| [location_analysis/location_analysis.py](SLX-27649/insert_peak_analysis/location_analysis/location_analysis.py) | GENCODE location classes (enhancer, promoter, exon/CDS/UTR, intron, intergenic) for all (3,871) and gold-standard (120) inserts; pie/bar charts via bedtools. |
| [location_analysis_new/fold_enrichment_only.py](SLX-27649/insert_peak_analysis/location_analysis_new/fold_enrichment_only.py) | Re-computes fold-enrichment plots/TSVs from existing annotations, separately for all vs gold inserts. |
| [encode_signals/query_encode.py](SLX-27649/insert_peak_analysis/encode_signals/query_encode.py), [query_encode_batch.py](SLX-27649/insert_peak_analysis/encode_signals/query_encode_batch.py) | Query ENCODE for K562 TF/histone ChIP, ATAC and DNase fold-change bigwigs (per-experiment, and a faster batch API approach); output URL/metadata TSV. |
| [encode_signals/extract_encode_signals.py](SLX-27649/insert_peak_analysis/encode_signals/extract_encode_signals.py) | Downloads each ENCODE bigwig, extracts ±1 kb signal at gold-standard clip sites (strand-flipped), caches NPZ, deletes bigwig; resume-safe. |
| [encode_signals/plot_profiles.py](SLX-27649/insert_peak_analysis/encode_signals/plot_profiles.py) | Per-target profile plots from the NPZ cache. |
| `encode_signals_10kb/`, `encode_signals_non_gold/`, `encode_signals_non_gold_10kb/` | Variants of the extract/plot pair for a 10 kb window and/or non-gold inserts (`extract_encode_signals_10kb.py`, `extract_encode_signals_non_gold.py`, `extract_encode_signals_non_gold_10kb.py`, `plot_profiles.py`). |
| [signal_traces/extract_signal_traces.py](SLX-27649/insert_peak_analysis/signal_traces/extract_signal_traces.py) | Mean BG4 CPM across gold-standard insert regions (10 kb flank + 525 bp insert + 10 kb flank). |
| `signal_traces/cutac_gold/extract_cutac_gold_traces.py` | CUTAC signal at the 120 gold sites mapped into each well's genome; only >1M noDup-read bigwigs. |
| `signal_traces/cutac_gold_matched/…`, `cutac_gold_unmatched/…` | CUTAC signal at gold sites in the matching well (22 SLX-27843 junctions) vs all other wells lacking the insert. |
| `signal_traces/empty/…`, `empty_non_gold/…` | BG4 signal at gold / non-gold insert loci (2525 bp window) in EMPTY cell lines. |
| `signal_traces/non_gold/…` | Median trace for all non-gold BG4 junctions. |
| `signal_traces/kras_control/`, `myc_control/`, `rpa3_control/` | BG4 signal at the KRAS (chr12), MYC (chr8) and RPA3 (chr7) G4 peaks in empty lines as controls. |
| `signal_traces/nodup_filtered_analyses/{gold,cutac_gold,cutac_gold_matched,cutac_gold_unmatched,empty,empty_non_gold,non_gold}/…` | Re-runs of the above restricted to samples with >1M noDup aligned reads (Picard metrics), 2525 bp window. |

## SLX-27843

**Pilot** covering only the first two rows of Plate 1 (wells 1A1–1B12), probed with BG4 and CUTAC plus BG4SIZE (an attempted manual size selection). Samples from two merged lanes are aligned to each well's integrated genome (via `per_sample_genome_map.tsv`) with MACS2 + SEACR peak calling. Its CUTAC data supply the matched / unmatched-well controls in SLX-27649's `insert_peak_analysis`, and its BG4 data are an alternative BR1 in SLX-28240.

| Script | Description |
|---|---|
| [Snakefile](SLX-27843/Snakefile) | Rules: `merge_lanes_r1/r2`, `demultiplex_r1/r2`, `trim_paired`, `align_bwa_mem`, `flagstat_stats`, `mark_duplicates`, `index_dedup_bam`, `bigwig_cpm`, `bigwig_pilot`, `bam_to_bedgraph`, `macs2_callpeak`, `seacr_callpeak`, `multiqc`, `fastqc_demux`, `bwa_index`, `fetch_genome_from_nas`. |
| [run_master.sh](SLX-27843/run_master.sh), [run_pipeline.sh](SLX-27843/run_pipeline.sh), [submit_master.sh](SLX-27843/submit_master.sh), [run_pilot.sh](SLX-27843/run_pilot.sh) | Launchers: login-node master (needed for NAS-access localrules), SLURM masters (`cuda` / `epyc`, 7 d), and a 2-day pilot. |
| [build_envs.sh](SLX-27843/build_envs.sh), [build_envs_v2.sh](SLX-27843/build_envs_v2.sh) | Pre-build conda envs on the login node; v2 builds the SEACR env manually and then launches the pipeline. |
| [scripts/splink_consensus.py](SLX-27843/scripts/splink_consensus.py) | Copy of the junction-consensus / genome-integration tool (see SLX-27649). |
| [config/config.yaml](SLX-27843/config/config.yaml), [profile/config.yaml](SLX-27843/profile/config.yaml) | Config (demuxFQ options, per-sample genome map, MACS2/SEACR) and SLURM profile. |

## SLX-27925

Additional globin-edited cell lines (independent of the insert library) assayed with BG4 and CUTAC. Analyses focus on the edited chr16 globin target locus (chr16:113170-114196) normalised to the KRAS reference region (chr12:25,250,600-25,251,600), and on correlation with RT-qPCR alpha-globin expression. Edit categories (e.g. WT-C, 14-C, 17-C) are compared against WT.

| Script | Description |
|---|---|
| [slurm_demuxFQ2_lane1.sh](SLX-27925/slurm_demuxFQ2_lane1.sh), [_lane2](SLX-27925/slurm_demuxFQ2_lane2.sh) | Per-lane `demuxFQ` submissions (~47 h, 32 G). |
| [watch_demux_and_launch.sh](SLX-27925/watch_demux_and_launch.sh) | SLURM watcher that polls demux jobs and launches the pipeline when done. |
| [fix_gzip.sh](SLX-27925/fix_gzip.sh) | SLURM job to repair gzip files. |
| [pipeline/Snakefile](SLX-27925/pipeline/Snakefile) | merge lanes → trim → align → dedup → bigwig → FastQC/MultiQC; plus `config/` and `profile/`. |
| [analyse_cpm.py](SLX-27925/analyse_cpm.py) | Per-sample CPM tracks at the target, target/reference normalised means, per-cell-line averages (BG4 vs CUTAC) with stats. |
| [analyse_cpm_v2.py](SLX-27925/analyse_cpm_v2.py) | As above, excluding listed QC-failed BG4 replicates. |
| [correlate_rtqpcr.py](SLX-27925/correlate_rtqpcr.py), [correlate_rtqpcr_v2.py](SLX-27925/correlate_rtqpcr_v2.py) | Correlates RT-qPCR fold changes (ddCt, Livak) with BG4/CUTAC normalised CPM per clone; v2 applies replicate-level BG4 exclusions. |
| [correlate_rtqpcr_separate_edits.py](SLX-27925/correlate_rtqpcr_separate_edits.py), [correlate_rtqpcr_separate_edits_WT.py](SLX-27925/correlate_rtqpcr_separate_edits_WT.py), [correlate_rtqpcr_per_edit.py](SLX-27925/correlate_rtqpcr_per_edit.py) | One panel per focal edit category with WT (and KH-17-C) shown as reference and a regression line (WT-only reference in the `_WT` variant). |
| [mean_traces.py](SLX-27925/mean_traces.py) | Mean CPM trace per cell line vs the WT mean (±SEM), 2 kb window, per assay. |
| [pca_cutac.py](SLX-27925/pca_cutac.py) | `multiBigwigSummary` via SLURM, then sklearn PCA of CUTAC samples (coloured by edit category / biorep). |
| [cpm_analysis/](SLX-27925/cpm_analysis) | Re-organised copies: `mean_traces/`, `separate_edits/`, `excluded_samples_analysis/` (analyse_cpm, v2, correlate, `separate_edits[_WT]`), [plot_wt_cpm.py](SLX-27925/cpm_analysis/plot_wt_cpm.py) (normalised CPM for WT-C, WT-N15, 14-C, 17-C as strip/bar plots) and [pca/run_mBWsummary.sh](SLX-27925/cpm_analysis/pca/run_mBWsummary.sh) (SLURM `multiBigwigSummary`, 16 CPU / 64 G). |

## SLX-28240

Multi-replicate, multi-assay (BG4, CM, CUTAC, 27ac) dataset over 5 lanes for the gold-standard insert cell lines, i.e. those where the G3T1 insert induces G4 formation at a site with no previously called G4 in the single-replicate pilot BG4 CUT&Tag (SLX-26996/26997). Gold standard is defined in SLX-27649 (MACS2 + SEACR both call the peak, absent from ≥80 % of empty lines). The scripts here test whether these peaks replicate (a junction is flagged `gold_standard` when a SEACR or MACS2 peak is found in ≥75 % of the BG4+CM replicates for its well) and whether the signal is specific to the insert-containing line.

| Script | Description |
|---|---|
| [slurm_demuxFQ2.sh](SLX-28240/slurm_demuxFQ2.sh), [slurm_demuxFQ2_lane1-5.sh](SLX-28240/slurm_demuxFQ2_lane1.sh) | `demuxFQ` submissions (whole run, and per lane). |
| [wait_and_launch.sh](SLX-28240/wait_and_launch.sh) | Waits for all lane1-5 R1/R2 demux summaries and SLURM jobs to clear, then launches the pipeline in tmux. |
| [rerun_all_after_dedup_fix.sh](SLX-28240/rerun_all_after_dedup_fix.sh) | Sequentially reruns 8 analysis steps (debug_missing_replicates → analyse_replicate_peaks → extract_gold_standard_inserts → check_structure_confirmation → extract_assay_signal_traces → insert specificity v1/v2 → violin scripts) after the dedup-reads filter bug fix. |
| [pipeline/Snakefile](SLX-28240/pipeline/Snakefile) | merge lanes → trim → align (per-well integrated genomes, fallback hg38_eco) → dedup → bigwig → FastQC/MultiQC → bedgraph → MACS2 + SEACR. |
| [pipeline/run_pipeline.sh](SLX-28240/pipeline/run_pipeline.sh), [run_peaks.sh](SLX-28240/pipeline/run_peaks.sh), [run_bg4rl_only.sh](SLX-28240/pipeline/run_bg4rl_only.sh) | Launchers: full run (tmux), peaks + bigwigs only, and BG4_RL contig-genome realignment targets. `config/`, `profile/`, `envs/` included. |
| **peak_analysis/** | |
| [analyse_replicate_peaks.py](SLX-28240/peak_analysis/analyse_replicate_peaks.py) | Checks whether insert junction 2 kb regions are consistently called across BG4/CM replicates (SLX-26997 as BR1 + SLX-28240); flags a junction as replicate-confirmed when ≥75 % of BG4+CM replicates have a peak. |
| [debug_missing_replicates.py](SLX-28240/peak_analysis/debug_missing_replicates.py) | Verifies each of the 84 deep-profiled wells has its expected 5 BG4+CM replicates; flags low-read exclusions. |
| [extract_gold_standard_inserts.py](SLX-28240/peak_analysis/extract_gold_standard_inserts.py) | Extracts the replicate-confirmed (≥75 %) junctions from `replicate_peak_summary.tsv`. |
| [plot_BG4RL_cpm.py](SLX-28240/peak_analysis/plot_BG4RL_cpm.py) | CPM of BG4_RL samples across insert regions (±5 kb). |
| [peak_analysis_incl_low_reads/compare_gold_standards_incl_excluded_reps.py](SLX-28240/peak_analysis_incl_low_reads/compare_gold_standards_incl_excluded_reps.py) | Tests whether re-including the 12 low-read-excluded replicates changes gold-standard status or adds new gold junctions. |
| **structure_confirmation/** | |
| [check_structure_confirmation.py](SLX-28240/structure_confirmation/check_structure_confirmation.py) | For the 120 gold junctions, counts replicates per assay with a SEACR **or** MACS2 peak at the same junction/well. |
| [plot_replicates_with_peak.py](SLX-28240/structure_confirmation/plot_replicates_with_peak.py) | Histogram of junction count vs number of replicates with a peak. |
| [signal_traces/extract_assay_signal_traces.py](SLX-28240/structure_confirmation/signal_traces/extract_assay_signal_traces.py) | Median BG4/CUTAC/CM/27ac CPM at the 120 junctions, per-insert and averaged (20,525 bp window). |
| [signal_traces/plot_2525bp_window.py](SLX-28240/structure_confirmation/signal_traces/plot_2525bp_window.py) | Re-plots the cached traces zoomed to the 2525 bp insert window. |
| [signal_traces/plot_insert_vs_kras_per_replicate.py](SLX-28240/structure_confirmation/signal_traces/plot_insert_vs_kras_per_replicate.py) | Per-junction, per-assay, per-replicate insert-region vs KRAS-region traces side by side. |
| [signal_traces/plot_insert_vs_kras_per_replicate_bg4_all.py](SLX-28240/structure_confirmation/signal_traces/plot_insert_vs_kras_per_replicate_bg4_all.py) | Same, BG4 only, for every junction with ≥1 supporting replicate peak (incl. SLX-26997 single-replicate wells). |
| [signal_traces/plot_insert_vs_kras_per_replicate_gold_promoter.py](SLX-28240/structure_confirmation/signal_traces/plot_insert_vs_kras_per_replicate_gold_promoter.py) | Same, restricted to the union of the 120 gold + promoter-annotated junctions (240). |
| [signal_traces/plot_bg4_br1_insert_vs_kras.py](SLX-28240/structure_confirmation/signal_traces/plot_bg4_br1_insert_vs_kras.py) | BG4 BR1 (SLX-26997, fallback SLX-27843) insert vs KRAS traces. |
| [signal_traces/promoter_cutac_signal_change.py](SLX-28240/structure_confirmation/signal_traces/promoter_cutac_signal_change.py) | CUTAC log2FC at every promoter within 100 kb of an insert vs other wells. |
| [signal_traces/plot_promoter_and_editsite_changes.py](SLX-28240/structure_confirmation/signal_traces/plot_promoter_and_editsite_changes.py) | One PDF page per junction combining promoter log2FC and the BG4/CUTAC change at the edited site. |
| **insert_specificity_analysis/** and **insert_specificity_reanalysis/** | |
| [analyse_insert_specificity.py](SLX-28240/insert_specificity_analysis/analyse_insert_specificity.py) | Mean and peak CPM at each of the 120 junctions for BG4/CM/CUTAC/27ac, self vs other cell lines, across SLX-26997/27843/28240. |
| [analyse_insert_specificity_v2.py](SLX-28240/insert_specificity_reanalysis/analyse_insert_specificity_v2.py) | Violin plots, per-insert replicate box plots and KRAS-normalised CPM. |
| [quick_violin_raw.py](SLX-28240/insert_specificity_reanalysis/quick_violin_raw.py), [quick_violin_mean_cpm_values.py](SLX-28240/insert_specificity_reanalysis/quick_violin_mean_cpm_values.py) | Fast violin redraws from the cached `insert_specificity.tsv` (raw differences / mean CPM, self vs 83 other lines). |
| [threshold_analysis_raw.py](SLX-28240/insert_specificity_reanalysis/threshold_analysis_raw.py), [threshold_analysis_normalised.py](SLX-28240/insert_specificity_reanalysis/threshold_analysis_normalised.py) | Counts inserts per assay above a threshold (raw mean CPM > 1.0; KRAS-normalised > 0.20). |

## Demultiplexing lists

The `r1` / `r2` files are the `demuxFQ` barcode maps (tab-separated: i7 barcode, i5 barcode, output FASTQ name, sample name) read by the `slurm_demuxFQ2*.sh` scripts and the Snakemake `demux` / `sample_maps` config sections. They are included in the ZIP at their original locations.

| Run | Files |
|---|---|
| SLX-26996 | `r1.txt`, `r2.txt` |
| SLX-26997 | `r1.txt`, `r2.txt`, plus `r1_excluded.txt`, `r2_excluded.txt` (the excluded samples used by `redemux_excluded.sh`) |
| SLX-26997_nano | `r1.txt`, `r2.txt` |
| SLX-27392 | `SLX-27392_demultiplex_r1.txt`, `SLX-27392_demultiplex_r2.txt`, plus `BR3_r1.txt`, `BR3_r2.txt` (replicate 3) |
| SLX-27649 | `260216_VH01410_433_AAHHHJ2M5/SLX-27649_demultiplex_r1.txt`, `…_r2.txt` |
| SLX-27843 | none on disk, although `config/config.yaml` expects `r1.txt` / `r2.txt` |
| SLX-27925 | `r1.txt`, `r2.txt` |
| SLX-28240 | `r1.txt`, `r2.txt` |

## Notes

- Scripts contain hard-coded absolute paths (`/mnt/scratchc/...`, `/scratche/slow/...`, `/mnt/nas-data/...`) and the `demuxFQ` binary location; adjust before reuse elsewhere.
- Several scripts have versioned or relocated copies (e.g. `analyse_cpm*.py`, `correlate_rtqpcr*.py`, `mean_traces.py` in both SLX-27925 and `cpm_analysis/`); they are kept as-is to mirror the source tree.
- `PIPELINE_ANALYSIS.md` is a validation note from an early pipeline build (references SLX-27392 mapping files and a "0 samples" dry run); treat it as historical.
