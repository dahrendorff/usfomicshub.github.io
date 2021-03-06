
# Microbiome Data-Analysis Workshop materials: Day 2 
This document is written in R-Markdown format (file-extension .Rmd). It can be run interactively in R-studio (after setting up a Project in the "day2" directory we had you download for the hands-on portion of the workshop, as we did in class). The day2 directory contains all necessary practice-data and code (including automated installation of required R-packages) to complete the tutorial.

---
Title: Running Anujit Sarkar's dada2 microbiome-analysis pipeline

Author: J. Oberstaller

Guinea-pig and guest-editor: T. E. Keller

Date: 11/06/2019

---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo=TRUE)
source(here::here("Rsource/microbiome.source.scripts.TEK-JO_110619.R"))
```

# Pipeline-overview #

*Goal*: The purpose of this analysis is to obtain an Amplicon Sequence Variant (ASV) table for all of our microbiome-sample example-data.

*Input data*: We will start with demultiplexed fastq files for all samples. *This analysis is for paired-end data.* Thus, for each sample, there will be two files, named according to Illumina platform conventions:

  1. Forward-reads, named *_R1_001.fastq
  2. Reverse-reads, named *_R2_001.fastq

## Before we begin ##

Before we begin, let's take a moment to get organized. The importance of documentation and good record-keeping are *essential* to producing high-quality and reproducible computational analyses, just as they are at the bench! 

We recommend you keep your analyses organized by project (just as we organized this example). Looking around: 
    
  - **Rdata**: this folder contains our input .fastq.gz files and our input database of 16S-sequences that we'll use to identify taxa present in our samples.

  - **Ranalysis**: this folder contains any scripts we create to analyze our data, like this R-Markdown (.Rmd) document.
  - **Routput**: we will direct any output data-files from our analyses to this folder.
  - **Rfigs**: we will direct any figures we generate from our analyses to this folder.
  - **Rsource**: this folder contains any R source-scripts we create to set up our environment for our analyses--custom functions, which packages to load, etc. etc. You don't need to worry about this one since we made it for you.
  
      You can think of these as set-up scripts--just load it at the beginning of your session and forget about it.
  

## Begin analyses ##

Now back to RStudio.

```{r}

# Let's give the path to the directory where all the fastq files are stored a name ("demo_microbiome_fasqfiles").
demo_microbiome_fasqfiles <- here::here("Rdata/fastq")


```

```{r load}
# Let's load the appropriate R package (dada2) for the analysis. It should have automatically been installed when you ran this markdown.
library(dada2)
```

*Important:* make sure to note the package-version of dada2 you're using!
```{r version-check}

packageVersion("dada2")
```

```{r separate_fastq_files}
# Check that the fastq directory contains all our fastq files
list.files(demo_microbiome_fasqfiles)

# Now make two variables to separate and store the forward and the reverse reads
demo_F <- sort(list.files(demo_microbiome_fasqfiles, pattern="_R1_001.fastq", full.names = TRUE))
demo_R <- sort(list.files(demo_microbiome_fasqfiles, pattern="_R2_001.fastq", full.names = TRUE))

# Extract filenames of all samples for future steps in the analysis
demo_samplenames <- sapply(strsplit(basename(demo_F), "_"), '[', 1)
```


## Evaluating data-quality ##

Let's check our data-quality by making plots and viewing them directly in RStudio. Your plots will appear in the RStudio "Plots" pane to the lower-right.

```{r check_fastq_quality}
# First we'll check the forward-reads:
plotQualityProfile(demo_F[1:4], n=1e+06)

# Now let's plot to check the data-quality of our reverse-reads.
plotQualityProfile(demo_R[1:4], n=1e+06)

```

Let's also output the plots as .pdf files so we can view them later. They'll be saved in the Rfigs directory.
  *Helpful tip: It is important to save any data or figures you generate in R that you want to keep to file; they are not saved when you quit RStudio, and you'll have to regenerate them!*
  
```{r make_quality_pdfs, include=TRUE, echo=FALSE, warning=FALSE}
## save data-quality plot for forward-reads:
pdf(here::here("Rfigs/demo_F_quality.pdf"), width = 12, height = 8, pointsize = 8)
plotQualityProfile(demo_F[1:4], n=1e+06)
dev.off()

# save the data-quality plot for our reverse-reads:
pdf(here::here("Rfigs/demo_R_quality.pdf"), width = 12, height = 8, pointsize = 8)
plotQualityProfile(demo_R[1:4], n=1e+06)
dev.off()
```

## Filter reads based on data-quality ##

The next step is to filter the sequences appropriately, the parameters for which will depend on the data. 

Conceptually, we will discard the bad reads, trim the ends of the good reads, and then save the trimmed good reads to a new directory.

First we will specify the path and name the output-files to which the good sequences will be written. 
  **the directory and output-files we specify here will be created in the next step (filterAndTrim).**
```{r filter_bad_reads1}
# The first step here is to specify the path and name the output-files to which the good sequences will be written. 
  # the directory and output-files we specify here will be created in the next step (filterAndTrim).
demo_goodF <- file.path(here::here("Routput/demo_good_filtered"), paste0(demo_samplenames, "F_good.fastq.gz"))
demo_goodR <- file.path(here::here("Routput/demo_good_filtered"), paste0(demo_samplenames, "R_good.fastq.gz"))
names(demo_goodF) <- demo_samplenames
names(demo_goodR) <- demo_samplenames
```


Now we perform the very important step of filtering and trimming each fastq. 

*These parameters are flexible and should depend on your data!*

```{r filter_bad_reads2}

demo_good_proper <- filterAndTrim(demo_F, demo_goodF, demo_R, demo_goodR, trimLeft = c(17, 21), truncLen = c(145, 135), maxN = 0, truncQ = 2, minQ=1, maxEE = c(2, 4), rm.phix = TRUE, n = 1e+5, compress = TRUE, verbose = TRUE)

# save the output of previous step (a summary table indicating how many reads there were for each sample before and after quality-filtering):
write.table(demo_good_proper, here::here("Routput/demo_filteredout.txt"), sep = "\t", quote = FALSE)

```




## Dereplicate sequences in each sample ##

Dereplicating the data collapses together reads that encode the same sequence this ends up saving computational time in later stages. (see section 4 https://bioconductor.org/packages/devel/bioc/vignettes/dada2/inst/doc/dada2-intro.html)
```{r dereplicate}
# The next step is to dereplicate the sequences in each sample
derep_demo_F <- derepFastq(demo_goodF, n = 1e+06, verbose = TRUE)
derep_demo_R <- derepFastq(demo_goodR, n = 1e+06, verbose = TRUE)

names(derep_demo_F) <- demo_samplenames
names(derep_demo_R) <- demo_samplenames

```


## Calculate and plot error-rates ##

Now let's calculate the error-rates (see below) for the forward and reverse sequences, plot them directly in RStudio and save to .pdf.

[see section 5 ]
(https://bioconductor.org/packages/devel/bioc/vignettes/dada2/inst/doc/dada2-intro.html)
From the dada2 vignette:

" The dada algorithm uses a parametric model of the errors introduced by PCR amplification and sequencing. Those error parameters typically vary between sequencing runs and PCR protocols, so our method provides a way to estimate those parameters from the data itself."

```{r error_calc}
## forward-reads:
demo_error_F <- learnErrors(derep_demo_F, nbases = 1e+07, randomize = TRUE, MAX_CONSIST = 12, multithread = TRUE, verbose = TRUE)
plotErrors(demo_error_F, obs = TRUE, nominalQ = TRUE)

## and reverse-reads:
demo_error_R <- learnErrors(derep_demo_R, nbases = 1e+07, randomize = TRUE, MAX_CONSIST = 12, multithread = TRUE, verbose = TRUE)
plotErrors(demo_error_R, obs = TRUE, nominalQ = TRUE)

```

We'll also save both plots to .pdf in our Rfigs directory for our records:
```{r error_plots, include=TRUE, echo=FALSE, warning=FALSE}

pdf(here::here("Rfigs/demo_error_F_plot.pdf"), width = 10, height = 10, pointsize = 8)
plotErrors(demo_error_F, obs = TRUE, nominalQ = TRUE)
dev.off()

pdf(here::here("Rfigs/demo_error_R_plot.pdf"), width = 10, height = 10, pointsize = 8)
plotErrors(demo_error_R, obs = TRUE, nominalQ = TRUE)
dev.off()
```


## Running dada2: Calculating ASVs ##

Now it is time to run the actual dada2 algorithm to determine the ASVs in the dataset.

  ** This step is run separately for forward and reverse sets of paired-end reads **

```{r running_dada2}

demo_dada_F <- dada(derep_demo_F, err=demo_error_F, pool = TRUE, multithread = TRUE)
demo_dada_R <- dada(derep_demo_R, err=demo_error_R, pool = TRUE, multithread = TRUE)

# Let's see how many sequence variants we have got in the forward set
demo_dada_F[[1]]
```


Next we will merge the forward and reverse sets (the paired-end reads for all our samples), and output a sequence-table of all ASVs.

```{r mergePairs}
demo_merged <- mergePairs(demo_dada_F, derep_demo_F, demo_dada_R, derep_demo_R, minOverlap = 20, maxMismatch = 0, verbose = TRUE)

# Let's make a sequence table of all the ASVs
demo_sequence_table <- makeSequenceTable(demo_merged, orderBy = "abundance")

# We can check the distribution of the ASVs by length
table(nchar(getSequences(demo_sequence_table)))

```

## Filtering chimeric sequences ##

```{r}

# Remove chimeric sequences
demo_nochim <- removeBimeraDenovo(demo_sequence_table, method = "consensus", minFoldParentOverAbundance = 1, verbose = TRUE, multithread = TRUE)

# Let's see how many ASVs remain after filtering chimeric sequences:
dim(demo_nochim)

# Let's see the proportion of sequences we retained after filtering for chimeric sequences:
sum(demo_nochim)/sum(demo_sequence_table)

```

Now we have completed all the filtering, trimming, cleanup etc. to arrive at our final data-set. Here we should check and record how many sequences we retained after each step.
  
  ** This test is important for trouble-shooting purposes. **

```{r}  
# Create a function to calculate reads retained
fetch_numbers <- function(a) sum(getUniques(a))

# Then apply this function to the output of each step in our pipeline to generate a counts-table of reads remaining after each step
demo_track_steps <- cbind(demo_good_proper, sapply(demo_dada_F, fetch_numbers), sapply(demo_dada_R, fetch_numbers), sapply(demo_merged, fetch_numbers), rowSums(demo_nochim))
colnames(demo_track_steps) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nochim")
rownames(demo_track_steps) <- demo_samplenames

# And save the output to a new file for our records:
write.table(demo_track_steps, here::here("Routput/demo_filtering_steps_track.txt"), sep = "\t", quote = FALSE)

```

## Assign taxonomy to all ASVs ##

We will next determine taxa present in our samples using the Silva database v.132.


dada2 helpfully maintains specially formatted databases for 3 of the most popular 16S microbiome-databases: Silva, Greengenes, and RDP (also UNITE for ITS).

We will be using the [dada2 Silva database](https://zenodo.org/record/1172783#.XcClW9VOnb1)

  *You downloaded this file to your Rdata directory previously (Rdata/Silva_db/silva_nr_v132_train_set.fa)*


```{r taxonomy}

demo_taxonomy <- assignTaxonomy(demo_nochim, here::here("Rdata/Silva_db/silva_nr_v132_train_set.fa"), minBoot = 80, verbose = TRUE, multithread = TRUE)
write.table(demo_taxonomy, here::here("Routput/demo_taxaout.txt"), sep = "\t", quote = FALSE)
```

Now we will generate output-files critical for further analyses and data-visualization. These include:

    a table summarizing ASVs by taxa
    a fasta-file of all ASVs
    a table of ASV-counts per sample (the OTU-table)

```{r make_final_tables}    
# Let's create a table by replacing the ASV sequences with ids (ASV_1, ASV_2 etc.) and their corresponding classifications
demo_taxa_summary <- demo_taxonomy
row.names(demo_taxa_summary) <- NULL
head(demo_taxa_summary)

# Let's make a file listing all the ASVs and their sequences in fasta format
demo_asv_seqs <- colnames(demo_nochim)
demo_asv_headers <- vector(dim(demo_nochim)[2], mode = "character")
for (i in 1:dim(demo_nochim)[2]) {demo_asv_headers[i] <- paste(">ASV", i, sep = "_")}
demo_asv.fasta <- c(rbind(demo_asv_headers, demo_asv_seqs))
write(demo_asv.fasta, here::here("Routput/demo_out_asv.fasta"))

# At this step, we need to make a table of ASV counts for each sample (which is going to be most important for all statistical analyses)
demo_asv_tab <- t(demo_nochim)
row.names(demo_asv_tab) <- sub(">", "", demo_asv_headers)
write.table(demo_asv_tab, here::here("Routput/demo_asv_counts.tsv"), sep = "\t", quote=F, col.names = NA)

# Finally, let's make a table with the taxonomy of all the ASVs
demo_asv_taxa <- demo_taxonomy
row.names(demo_asv_taxa) <- sub(">", "", demo_asv_headers)
write.table(demo_asv_taxa, here::here("Routput/demo_asvs_taxonomy.tsv"), sep = "\t", quote=F, col.names = NA)
dim(demo_asv_taxa)

# We'll also need to make fake sample-bmi data for tomorrow's visualization-exercises (phyloseq)
bmi=c('obese','obese','lean','lean')
demo_fake_sample_data <-data.frame(bmi_group=bmi)
rownames(demo_fake_sample_data)=c("demo1","demo2","demo3","demo4")
write.table(demo_fake_sample_data,here::here("Routput/made_up_sample_data.tsv"),sep="\t",quote=FALSE)

```


Tomorrow, we'll plot the data we analyzed today.


## ** THANK YOU EVERYONE ** ##


