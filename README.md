# Merging & direct-joining workflow
## Overview
This computational workflow follows a combination of merging and direct-joining approaches to reduce the loss of amplicon data generated by the Illumina MiSeq platform. This computational approach could be followed when the Illumina PE dataset contains insufficient high-quality reads and/or when the conventional merging approach drops a significantly copious amount of reads due to poor quality data. In this workflow, the trimmed PE reads are merged when the reads have sufficient length of overlaps; otherwise, the reads are concatenated. For more details, please refer to this [publication](https://link.springer.com/article/10.1007/s10529-024-03509-9).

This bioinformatics workflow could also be used to infer prokaryotic and eukaryotic diversity using mixed amplicon sequencing data without splitting the 16S and 18S rRNA reads. For more information, please read this [publication](https://academic.oup.com/femsle/advance-article/doi/10.1093/femsle/fnae009/7596553?searchresult=1).
## Citation
Please consider citing the following publications, if you use the code in your research.

[Ramakodi, M.P. Don’t let valuable microbiome data go to waste: combined usage of merging and direct-joining of sequencing reads for low-quality paired-end amplicon data. Biotechnol Lett (2024). https://doi.org/10.1007/s10529-024-03509-9](https://link.springer.com/article/10.1007/s10529-024-03509-9).

[Meganathan P Ramakodi, Merging and concatenation of sequencing reads: a bioinformatics workflow for the comprehensive profiling of microbiome from amplicon data, FEMS Microbiology Letters, 2024;, fnae009, https://doi.org/10.1093/femsle/fnae009](https://academic.oup.com/femsle/advance-article/doi/10.1093/femsle/fnae009/7596553?searchresult=1).
## Prerequisites
The following R packages should be installed on your workstation before running the codes. The codes were tested on R version 4.2.2. 
- `dada2`
- `phyloseq`
- `stringr`
- `Biostrings`
### Code
**Calling the required packages**
```
library(dada2)
library(phyloseq)
library(stringr)
library(Biostrings)
```
**Assigning the path where the fastq files are stored**
```
path <- "/home/mega/PROJECTS/DATA/Input_directory"
```
**Reading the files**
```
fnFs <- sort(list.files(path, pattern="_R1.fastq.gz", full.names = TRUE))
fnRs <- sort(list.files(path, pattern="_R2.fastq.gz", full.names = TRUE))
```
**Assigning sample names**
```
sample.names <- sapply(strsplit(basename(fnFs), "_"), `[`, 1)
```
**Creating path to store filered data**
```
filtFs<- file.path(path, "filtered", paste0(sample.names,"_F_filt.fastq.gz"))
filtRs<- file.path(path, "filtered", paste0(sample.names,"_R_filt.fastq.gz"))
names(filtFs) <- sample.names
names(filtRs) <- sample.names
```
**Filtering the reads**
> _NOTE: Users should change the values of the parameters as per their requirements_
```
out <- filterAndTrim(fnFs, filtFs, fnRs, filtRs, truncLen=c(230,210), maxEE=c(2,2), trimLeft=c(35,35), compress = TRUE, multithread = TRUE)
```
**Learning errors**
```
errF <- learnErrors(filtFs, multithread=TRUE)
errR <- learnErrors(filtRs, multithread=TRUE)
```
**Processing the data**
```
# Sample inference and merger of paired-end reads
mergers <- vector("list", length(sample.names))
mergers_ <- vector("list", length(sample.names))
jConS <- vector("list", length(sample.names))
sLICEs <- vector("list", length(sample.names))
derepFs <- vector("list", length(sample.names))
derepRs <- vector("list", length(sample.names))
ddFs <- vector("list", length(sample.names))
ddRs <- vector("list", length(sample.names))

cAT <- "NNNNNNNNNN"

names(mergers) <- sample.names
names(mergers_) <- sample.names
names(jConS) <- sample.names
names(sLICEs) <- sample.names
names(derepFs) <- sample.names
names(derepRs) <- sample.names
names(ddFs) <- sample.names
names(ddRs) <- sample.names

for(sam in sample.names) {
	cat("Processing:", sam, "\n")
	derepF <- derepFastq(filtFs[[sam]])
	ddF <- dada(derepF, err=errF, multithread=TRUE)
	derepR <- derepFastq(filtRs[[sam]])
	ddR <- dada(derepR, err=errR, multithread=TRUE)

	derepFs[[sam]] <- derepF
	derepRs[[sam]] <- derepR
	ddFs[[sam]] <- ddF
	ddRs[[sam]] <- ddR
#merging the data
	merger <- mergePairs(ddF, derepF, ddR, derepR, verbose=TRUE)
	mergers[[sam]] <- merger
#merging the data and retaining the information of rejected dadaFs and dadaRs pairs
	merger_<-mergePairs(ddF, derepF, ddR, derepR, verbose=TRUE, returnRejects = TRUE)
	mergers_[[sam]] <- merger_
#Concatenation/ direct-joining of reads
	jCon <- mergePairs(ddF, derepF, ddR, derepR, verbose=TRUE, justConcatenate = TRUE)
#Correcting the directly-joined data of unmerged reads
	jCon$dummy_nmatch <- merger_$nmatch
	for (i in 1:nrow(jCon))
	{
		jCon$r1[i] <- unlist(str_split(jCon$sequence[i], cAT))[1]
		jCon$r2[i] <- unlist(str_split(jCon$sequence[i], cAT))[2]
		jCon$r2_[i] <- str_sub(jCon$r2[i], jCon$dummy_nmatch[i] + 1)
		jCon$seq[i] <- paste0(jCon$r1[i],cAT,jCon$r2_[i])
	}
	jCon$original_seq <- jCon$sequence
	jCon$sequence <- jCon$seq
	jConS[[sam]] <- jCon
#Retrieve the unmerged reads that were joined directly
	sLICE <- jCon[-(as.numeric(row.names(merger))),]
	sLICEs[[sam]] <- sLICE
}
```
**Make Sequence Table**
> _NOTE: st_jCon is optional._
```
st_mergers <- makeSequenceTable(mergers)
st_jCon <- makeSequenceTable(jConS)
st_sLICE <- makeSequenceTable(sLICEs)
```
**Remove chimeras**
> _NOTE: st.nochim.jCon is optional_
```
st.nochim.mergers <- removeBimeraDenovo(st_mergers, method="consensus", multithread=TRUE, verbose=TRUE)
st.nochim.jCon <- removeBimeraDenovo(st_jCon, method="consensus", multithread=TRUE, verbose=TRUE)
st.nochim.sLICE <- removeBimeraDenovo(st_sLICE, method="consensus", multithread=TRUE, verbose=TRUE)
```
**Assigning taxonomy & creating phyloseq object**
>***For the convenience of users, the assignTaxonomy module from dada2 is given below. However, the users can use the [IdTaxa](https://rdrr.io/bioc/DECIPHER/man/IdTaxa.html) tool for taxonomy assignment and create the phyloseq object but the codes need to be changed***
1. Loading the reference database
> _NOTE: User should change the path and name of the database in the code as required_
```
Ref <- "/home/mega/DATABASE/reference_database.fasta"
```
2. Processing st.nochim.mergers
> _This object contains the merged amplicon sequences_
```
taxa <- assignTaxonomy(st.nochim.mergers, Ref, multithread = TRUE, tryRC = TRUE)
#Create the phyloseq object
ps.m <- phyloseq(otu_table(st.nochim.mergers, taxa_are_rows=FALSE), tax_table(taxa))
#add refseq
dna<-Biostrings::DNAStringSet(taxa_names(ps.m))
names(dna)<-taxa_names(ps.m)
ps.m<- merge_phyloseq(ps.m, dna)
ps.m
```
3. Processing st.nochim.sLICE
> _This object contains the directly-joined sequences that could not be merged_
```
taxa <- assignTaxonomy(st.nochim.sLICE, Ref, multithread = TRUE, tryRC = TRUE)
#Create the phyloseq object
ps.um <- phyloseq(otu_table(st.nochim.sLICE, taxa_are_rows=FALSE), tax_table(taxa))
#add refseq
dna<-Biostrings::DNAStringSet(taxa_names(ps.um))
names(dna)<-taxa_names(ps.um)
ps.um<- merge_phyloseq(ps.um, dna)
ps.um 
```
4. Processing st.nochim.jCon
> _This object contains directly-joined sequences and processing this object is optional_
```
taxa <- assignTaxonomy(st.nochim.jCon, Ref, multithread = TRUE, tryRC = TRUE)
#Create the phyloseq object
ps.dj <- phyloseq(otu_table(st.nochim.jCon, taxa_are_rows=FALSE), tax_table(taxa))
#add refseq
dna<-Biostrings::DNAStringSet(taxa_names(ps.dj))
names(dna)<-taxa_names(ps.dj)
ps.dj<- merge_phyloseq(ps.dj, dna)
ps.dj
```
**Creating the M-DJ dataset**
```
ps.m.dj <- merge_phyloseq(ps.m,ps.um)
```
**Users can save the ps.m.dj object as rds file and can use it later for downstream analyses**
