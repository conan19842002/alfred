# Usage

Alfred uses subcommands for [quality control](#alignment-quality-control) ([qc](#alignment-quality-control)), [feature counting](#bam-feature-counting) ([count_dna](#bam-read-counting-for-dna-seq), [count_rna](#bam-read-counting-for-rna-seq), [count_jct](#bam-read-counting-for-rna-seq)), [feature annotation](#bam-feature-annotation) ([annotate](#chip-seq-or-atac-seq-peak-annotation), [tracks](#browser-tracks)), [alignment](#pairwise-sequence-alignment) ([pwalign](#pairwise-sequence-alignment), [consensus](#bam-consensus-computation)) and [haplotype-resolved analysis](#haplotype-specific-bam-files) ([split](#haplotype-specific-bam-files), [ase](#allele-specific-expression)). The subcommands are explained below.

## Alignment Quality Control

Alfred supports a command-line interface to run alignment quality control and a [web application](https://www.gear-genomics.com) can be used to render all QC metrics.

## Command-line Interfact for BAM Quality Control

Alfred computes various alignment metrics and summary statistics by read group

```bash
alfred qc -r <ref.fa> -o qc.tsv.gz <align.bam>
```

Plotting alignment statistics

```bash
Rscript scripts/stats.R qc.tsv.gz
```

To convert all the alignment metrics from column format to row format for readability

```bash
zgrep ^ME qc.tsv.gz | cut -f 2- | datamash transpose | column -t
```

## Interactive Quality Control Browser

Quality control metrics can be browsed interactively using the [web front end of Alfred](https://www.gear-genomics.com/alfred).

```bash
alfred qc -r <ref.fa> -j qc.json.gz -o qc.tsv.gz <align.bam>
```

Then just upload the qc.json.gz file to the Alfred GUI [https://www.gear-genomics.com/alfred](https://www.gear-genomics.com/alfred). A convenient feature of the web-front end is that multiple samples can be uploaded and compared.


## BAM Alignment Quality Control for Targeted Sequencing

If target regions are provided, Alfred computes the average coverage for each target and the on-target rate.

```bash
alfred qc -r <ref.fa> -b <targets.bed.gz> -o <qc.tsv.gz> <align.bam>
```

For instance, for a human whole-exome data set.

```bash
cd maps/ && Rscript exon.R
alfred qc -r <hg19.fa> -b maps/exonic.hg19.bed.gz -j qc.json.gz -o qc.tsv.gz <exome.bam>
Rscript scripts/stats.R qc.tsv.gz
```

Alternatively, one can use the [interactive GUI](https://www.gear-genomics.com/alfred) and upload the json file.

```bash
alfred qc -r <hg19.fa> -b maps/exonic.hg19.bed.gz -j qc.json.gz -o qc.tsv.gz <exome.bam>
```


## BAM Alignment Quality Control for ATAC-Seq

For ATAC-Seq data, the insert size distribution should reveal the DNA pitch and a clear nucleosome pattern with a peak for single nucleosomes and dimers. The transcription start site (TSS) enrichment should be >5 for a good ATAC-Seq library and ideally the duplicate rate is <20%, the alignment rate >70% and the standardized SD in coverage >0.3.

```bash
cd maps/ && Rscript promoter.R
alfred qc -r <hg19.fa> -b maps/hg19.promoter.bed.gz -o qc.tsv.gz <atac.bam>
Rscript scripts/stats.R qc.tsv.gz
zgrep ^ME qc.tsv.gz | datamash transpose | egrep "^Dup|^MappedFraction|^SD|^Enrich"
```

ATAC-Seq libraries often have a large number of mitochondrial reads depending on the library preparation.

```bash
zgrep ^CM qc.tsv.gz | egrep "Mapped|chrM"
```

Alternatively, one can use the [interactive GUI](https://www.gear-genomics.com/alfred) and upload the json file.

```bash
alfred qc -r <hg19.fa> -b maps/hg19.promoter.bed.gz -j qc.json.gz -o qc.json.gz <atac.bam>
```

## BAM Feature Counting

Alfred supports counting reads in overlapping or non-overlapping windows, at predefined intervals in BED format,
or as gene and transcript counting for RNA-Seq in stranded or unstranded mode using a gtf or gff3 gene annotation
file. Expression values can be normalized as raw counts, FPKM, or FPKM-UQ values.

## BAM Read Counting for DNA-Seq

For DNA sequencing, Alfred can be used to calculate the coverage in overlapping or non-overlapping windows or in given set of intervals.

```bash
alfred count_dna -o <cov.gz> <align.GRCh37.bam>
```

To plot the whole-chromosome coverage profile for chr1-22 and chrX.

```bash
Rscript scripts/rd.R <cov.gz>
```

## BAM Read Counting for RNA-Seq

Alfred can also assign reads to gene annotation features from a GTF file such as counting reads by gene or transcript identifier.

```bash
cd gtf/ && ./downloadGTF.sh
alfred count_rna -g gtf/Homo_sapiens.GRCh37.75.gtf.gz <align.GRCh37.bam>
```

An experimental feature of Alfred is to count splice junction supporting reads. This method generates exon-exon junction counts for intra-gene exon-exon junctions, inter-gene exon-exon junctions and completely novel (not annotated) intra-chromosomal junctions.

```bash
alfred count_jct -g gtf/Homo_sapiens.GRCh37.75.gtf.gz <align.GRCh37.bam>
```

## BAM Feature Annotation

Alfred supports annotation of ChIP-Seq and ATAC-Seq peaks for neighboring genes or transcription factor binding sites (based on motif alignments). Additionally, browser tracks in UCSC bedgraph format can be computed with configurable resolution.

## ChIP-Seq or ATAC-Seq peak annotation

To annotate overlapping/neighboring genes up to a distance of 10,000bp:

```bash
alfred annotate -d 10000 -g gtf/Homo_sapiens.GRCh37.75.gtf.gz <peaks.bed>
```

The two output files summarize nearby genes by peak and vice versa (peaks by gene).


## Motif annotation

Motif annotation of peaks based on alignment scores of motif-specific position weight matrices can also be obtained. 

```bash
cd motif/ && ./downloadMotifs.sh
alfred annotate -r <hg19.fa> -m motif/jaspar.gz <peaks.bed>
```

Alfred further implements functionality to output the exact motif hits in each peak to perform for instance transcription factor footprinting.

```bash
alfred annotate -p hits.gz -r <hg19.fa> -m motif/jaspar.gz <peaks.bed>
```


## Joined motif hits

You can also obtain all motif hits across a reference genome. For instance, to identify all hits for recombination signal sequences (RSS) you can use:

```bash
cat <hg19.fa.fai>  | cut -f 1,2 | awk '{print $1"\t0\t"$2;}' > all.chrom.bed
alfred annotate -m motif/rss.gz -r <hg19.fa> -q 0.9 -p motif.hits.gz all.chrom.bed
```

To then join all motif hits of the conserved heptamer sequence (7bp), a spacer sequence (12bp or 23bp), and a conserved nonamer sequence (9bp) you can use:

```bash
alfred spaced_motif -l 11 -h 13 motif.hits.gz
```



## Browser Tracks

Normalized and file size optimized browser tracks are essential to compare peak profiles across samples and upload them quickly in online genome browsers such as the [UCSC Genome Browser](https://genome.ucsc.edu/). Alfred generates browser tracks in [UCSC bedgraph format](https://genome.ucsc.edu/goldenpath/help/bedgraph.html) with configurable resolution. Lower resolution values leed to coarse-grained windows at reduced file size. Contrarily, high resolution values leed to fine-grained windows at greater file size.

```bash
alfred tracks -r 0.2 -o track.bedGraph.gz <aligned.bam>
```

[IGV tools](https://software.broadinstitute.org/software/igv/igvtools) can be used to convert bedgraph files to IGV's proprietory tdf format.

```bash
igvtools totdf track.bedGraph.gz track.tdf hg19
```

This conversion enables comparing dozens of ATAC-Seq or ChIP-Seq samples at greatly improved speed in IGV compared to using the raw BAM files. By default the Alfred tracks command normalizes all input files to 30 million reads so peak heights are directly comparable across samples.

## Pairwise Sequence Alignment

Alfred supports global and local pairwise sequence alignments with configurable match scores and gap/mismatch penalties. Overlap and nested alignments can be achieved using penalty-free leading/trailing gaps. Affine gap penalties with separate gap open and gap extension penalties are supported.

```bash
alfred pwalign -a align.fa.gz <seq1.fasta> <seq2.fasta>
```

All computed pairwise alignments are "linear" alignments that is the order of sequence nucleotides is preserved. For more complex pairwise alignments involving inversions or duplications you can use [Maze](https://www.gear-genomics.com/maze/).

## BAM Consensus Computation

Alfred supports consensus computation of error-prone long reads using multiple sequence alignment principles. To compute a consensus sequence for a set of long reads at a given alignment position:

```bash
alfred consensus -f bam -t ont -p chr1:218992200 <ont_pacbio.bam>
```

The consensus method generates two output files. A simple fasta file with the consensus sequence and a FASTA align file that shows the multiple sequence alignment used for consensus generation in either horizontal or vertical format. The horizontal format is the classical FASTA align format. The vertical format transposes the horizontal alignment and in addition shows the consensus nucleotide for each alignment column after the vertical bar.

The consensus command is probably most useful by first separating haplotypes (Alfred's [split](#haplotype-specific-bam-files) subcommand) and then computing consensus sequences independently for each haplotype.

```bash
alfred split -r <ref.fa> -s NA12878 -p <haplotype1.bam> -q <haplotype2.bam> -v <phased.snps.bcf> <input.bam>
alfred consensus -c <hap1.fa.gz> -f bam -t ont -p chr1:chr4:500500 <haplotype1.bam>
alfred consensus -c <hap2.fa.gz> -f bam -t ont -p chr1:chr4:500500 <haplotype2.bam>
```

To identify variants, you can then compare the two locally assembled haplotypes using our online dotplot method [Maze](https://www.gear-genomics.com/maze) or align them pairwise against each other using the subcommand [pwalign](#pairwise-sequence-alignment).


## Haplotype-specific BAM files

Long reads, Hi-C or novel single-cell sequencing technologies such as Strand-Seq enable (local) haplotyping or phasing of variants. Once such a phased SNP scaffold has been obtained one can split a BAM file into the corresponding haplotypes.

```bash
alfred split -r <ref.fa> -s NA12878 -p <haplotype1.bam> -q <haplotype2.bam> -v <phased.snps.bcf> <input.bam>
```

## Allele-specific expression

Alfred implements methods to generate allele-specific expression tables. If the input SNPs are phased Alfred annotates the allele-specific expression haplotype-aware.

```bash
alfred ase -r <ref.fa> -s NA12878 -v <snps.bcf> -a <ase.tsv> <input.bam>
```

## Analysis of replication timing by NGS

Alfred implements a method to analyze replication timing using next-generation sequencing (Repli-Seq). The order of BAM files on the command-line must follow the cell-cycle.

```bash
alfred replication -r <ref.fa> -o outprefix <g1b.bam> <s1.bam> <s2.bam> <s3.bam> <s4.bam> <g2.bam>
```

There is a supporting script that plots the tag density for each cell-cycle fraction.

```bash
Rscript scripts/reppattern.R -f outprefix.profile.tsv -r chr12:24000000-26000000
```

There is also a script for plotting the replication time along a given chromosome.

```bash
Rscript scripts/reptime.R -f outprefix.reptime.tsv -r chr12
```
