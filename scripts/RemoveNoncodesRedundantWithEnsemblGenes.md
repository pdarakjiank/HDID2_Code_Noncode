---
title: "Remove genes from NONCODE gtf reduntant with Ensemble gtf"
author: "Priscila Darakjian"
output: html_document
---
# Script: RemoveNoncodesRedundantWithEnsemblGenes.Rmd
## Description: This script removes noncode rows present in the NONCODE2016_mouse_mm10_lncRNA.gtf file that have Ensembl counterparts listed in the Mus_musculus.GRCm38.85.gtf file under an actual gene name
#### Author: Priscila Darakjian - OHSU - Research - Behavioral Neuroscience
#### Date: 09/13/2015

### Load gtf files

```r
library(GenomicRanges)
library(rtracklayer)
setwd("/lawrencedata/ongoing_analyses/RNASeq016/RNASeq016_noncode/NonCode_Analysis_2016")
Ensembl.gtf <- readGFF("data/Mus_musculus.GRCm38.85.gtf")
Noncode.gtf <- readGFF("data/NONCODE2016_mouse_mm10_lncRNA.gtf")
```

### We will use all biotypes from Ensembl's annotation gtf file but just for the usual chromosomes and for exons (noncodes as well). 

```r
Ensembl.gtf <- Ensembl.gtf[Ensembl.gtf[, 1] %in% c("1", "2", 
    "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", 
    "14", "15", "16", "17", "18", "19", "X", "Y"), ]
Ensembl.gtf <- Ensembl.gtf[Ensembl.gtf[, 3] %in% c("exon"), ]
Noncode.gtf$seqid <- sub("chr", "", Noncode.gtf$seqid)
Noncode.gtf <- Noncode.gtf[Noncode.gtf[, 1] %in% c("1", "2", 
    "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", 
    "14", "15", "16", "17", "18", "19", "X", "Y"), ]
Noncode.gtf <- Noncode.gtf[Noncode.gtf[, 3] %in% c("exon"), ]
```

### Create Genomic Ranges from the gtf files

```r
Ensembl.ranges <- GRanges(seqnames = Rle(Ensembl.gtf$seqid), 
    ranges = IRanges(start = Ensembl.gtf$start, end = Ensembl.gtf$end), 
    strand = Ensembl.gtf$strand, gene_name = Ensembl.gtf$gene_name, 
    biotype = Ensembl.gtf$gene_biotype)

Noncode.ranges <- GRanges(seqnames = Rle(Noncode.gtf$seqid), 
    ranges = IRanges(start = Noncode.gtf$start, end = Noncode.gtf$end), 
    strand = Noncode.gtf$strand, gene_name = Noncode.gtf$gene_id, 
    biotype = "noncode")

Ensembl.ranges <- unique(Ensembl.ranges)
Noncode.ranges <- unique(Noncode.ranges)
```
### Find Exact Overlaps - (just a precaution) (there should be none)

```r
Ensembl.ranges.ovls <- findOverlaps(Ensembl.ranges, drop.self = T, 
    drop.redundant = T, type = "equal")

Noncode.ranges.ovls <- findOverlaps(Noncode.ranges, drop.self = T, 
    drop.redundant = T, type = "equal")
```
### If the code and noncode overlaps above come up empty, continue below i.e. find exact overlaps between the code and noncode ranges (we want to eliminate the noncodes that actually have gene symbols in Ensembl)

```r
Combined_annot <- rbind(as.data.frame(Ensembl.ranges), as.data.frame(Noncode.ranges))
Combined_annot_ordered <- Combined_annot[order(Combined_annot$strand, 
    Combined_annot$seqnames, Combined_annot$start), ]

Combined_annot_ranges <- GRanges(seqnames = Rle(Combined_annot_ordered$seqnames), 
    ranges = IRanges(start = Combined_annot_ordered$start, end = Combined_annot_ordered$end), 
    strand = Combined_annot_ordered$strand, gene.names = Combined_annot_ordered$gene_name, 
    biotype = Combined_annot_ordered$biotype)

Combined_annot_equal_ovls <- findOverlaps(Combined_annot_ranges, 
    drop.self = T, drop.redundant = T, type = "equal")
Combined_annot_equal_ovls_mat <- as.matrix(Combined_annot_equal_ovls)
Combined_annot_equal_ovls_dta <- data.frame(as.data.frame(Combined_annot_ranges)[Combined_annot_equal_ovls_mat[, 
    1], ], as.data.frame(Combined_annot_ranges)[Combined_annot_equal_ovls_mat[, 
    2], ])
```
### Verify that there are no NONCODEs in gene.names.1

```r
Combined_annot_equal_ovls_dta[-grep("NONMMUG", Combined_annot_equal_ovls_dta$gene.names.1), 
    ]  # should return no rows
```
### Fetch the indexes of subject hits (noncodes) so we can remove those redundant noncodes from the data frame

```r
Combined_annot_equal_ovls_rm <- as.integer(Combined_annot_equal_ovls_mat[, 
    2])
```
### Remove redundant noncodes from the combined data frame and write the resulting df to a file

```r
Combined_annot_clean <- as.data.frame(Combined_annot_ranges)[-Combined_annot_equal_ovls_rm, 
    ]

write.table(Combined_annot_clean, "data/Combined_code_noncode_mm10_GRCm38.85_NONCODE2016.txt", 
    sep = "\t", row.names = F, col.names = T, quote = F)

save(Combined_annot_clean, file = "data/Combined_annot_clean.RData")
```
