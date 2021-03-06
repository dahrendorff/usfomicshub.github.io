---
title: 'Make GO bubble-plot: essential genes version'
author: "J. Oberstaller"
date: "6/4/2019"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## JG: assuming you have already run the topGO pipeline, but including some info on that part for context

### run.topGO ###

My run.topGO-function makes the GOdata object for topGO**^c^**, performs GO analysis by ontology (molecular function, biological process, cellular compartment) on all essentiality-groups at once, and then outputs results to several tables.
  * run.topGO defines which genes are interesting and which should be defined as background for each essentiality category. Genes in the essentiality category of interest are tested for enrichment against all the other genes included in the comparison (the "gene universe").
  * The "gene universe" here consists of all genes in the comparison (i.e., for RNAseq, all genes for which you have expression data). topGO automatically accounts for genes that cannot be mapped to GO terms with the "feasible genes" indicated in the topGO GOdata object (which you can see in the "genes_by_GOterm*.txt" output file from this function)--no need to make separate GO databases for each run. Just use the same GO db of ALL genes, and topGO will run the appropriate comparisons.
#### run.topGO outputs ####
For each of the 8 binary essentiality categories:
  * output a table with all SIGNIFICANT (p <=0.05) GO-terms by ontology (MF, BP, CC)
    + Routput/GO/significant.results_by_bin*.tab.txt
  * output a table with all GO-term results (top 30 terms, no significance cut-off)
    + Routput/GO/results_by_bin*.tab.txt
      
And all essentiality-categories combined, with an added column for essentiality category:
  * significant results
    + "Routput/GO/all.bin.combined.significant.GO.results.tab.txt"
  * all results (no significance cut-off, top 30 terms)
    + "Routput/GO/all.bin.combined.GO.results.tab.txt"
    
**Primary output from run.topGO is "Routput/GO/all.bin.combined.significant.GO.results.tab.txt", which we'll use for input to the next part of the analyses.**

**^b^** *See the "/Ranalysis/scripts_040219.R" file and the formatGOdb_commas function for more info on creating and formatting the PfGOdb file.*

**^c^** *See the topGO manual from Alexa et al.:* https://rdrr.io/bioc/topGO/f/inst/doc/topGO.pdf


## Data visualization ##

```{r testing GO-enrichment bubble-plots}
# read in the output of my topGO pipeline
input <- "Routput/GO/all.bin.combined.GO.results.tab.txt"
GO.results.txt <- read.table(input, header = TRUE, sep = "\t")
GO.results.txt <- as.data.frame(GO.results.txt)
bubblePlotDf <- cbind.data.frame(GO.results.txt$GO.ID, GO.results.txt$Significant, GO.results.txt$Annotated, GO.results.txt$topGO, GO.results.txt$go.category, GO.results.txt$essentiality.category)
colnames(bubblePlotDf) <- c("GO.ID", "Significant", "Annotated", "topGO", "GO.category", "essentiality.category")
```

```{r calculations for bubble-plot}
# make another column for zScore, which will be the values for the x-axis of the plot . . . comes in to play again below in the plot function (and y-axis is -log2 of the p-value)
# there are a number of possible ways to calculate zScore/normalize the data; here I'm scaling each row by average GO-term significance by essentiality category.
# dat$zScore <- ave(dat$y, dat$group, FUN=scale)
bubblePlotDf$zScore <- ave(bubblePlotDf$topGO, bubblePlotDf$essentiality.category, FUN = scale)
# the "P.value" here will be either the topGO-determined p-value or the scaled zscore, whichever is smaller.
bubblePlotDf$P.value<- apply(bubblePlotDf, 1, function(x) {return(min(c(x[4], x[7])))})
```

```{r set bubbleplot aesthetics}
# add a column to assign shapes by GO category
bubblePlotDf$shape <- rep(0, nrow(bubblePlotDf))
bubblePlotDf$shape[which(bubblePlotDf$GO.category == 'MF')]    <- 25 # filled inverted-triangle
bubblePlotDf$shape[which(bubblePlotDf$GO.category == 'BP')]    <- 21 # filled circle
bubblePlotDf$shape[which(bubblePlotDf$GO.category == 'CC')]    <- 22 # filled square
# add a column to assign colors by essentiality category
bubblePlotDf$color <- rep(0, nrow(bubblePlotDf))
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '111')]    <- rgb(255,0,127 ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '112')]    <- rgb(96, 155, 49 ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '121')]    <- rgb(107,155, 244 ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '122')]    <- rgb(255,204, 255 ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '211')]    <- rgb(204,204, 255 ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '212')]    <- rgb(204,204, 255 ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '221')]    <- rgb(246, 156, 156  ,max = 255, alpha = 180)
bubblePlotDf$color[which(bubblePlotDf$essentiality.category == '222')]    <- rgb(153,255, 153 ,max = 255, alpha = 180)
# order df in decreasing order
bubblePlotDf <- bubblePlotDf[order(bubblePlotDf$P.value, decreasing = T),]
```

```{r define bubble-plot related functions}
ccn <- function(x) {return(as.numeric(as.character(x)))}
# pointPlot function to define parameters for drawing points
pointPlot <- function(x, CutOff = 1, biggestSize = 10, col = 'gray'){
    
          #id <- which(ccn(x$P.value) < CutOff)
          #print(length(id))
          
          # scale the size of the point for each GO-term to where 10 is the biggest point; then cut the size-vector into sizes relative to the biggest point
          sizeVector = (x$Annotated)
          sizePoints = c(1:biggestSize)[cut(sizeVector, biggestSize)]
          
          # JO changed the 'pch' parameter to vary based on the GO.category column
          points( (ccn(x$zScore)),
                  -log2(ccn(x$topGO)), pch = as.numeric(x$shape),
                  bg = as.character(x$color), cex = sizePoints)
                  
          }
```

```{r make bubble-plot pdf}
#  make a .pdf of your bubble-plot
pdf('Rfigs/GO.bubble.pdf', height = 6, width = 7.5)
par(mar=c(5,5,0,0))
plot(x=NA,y=NA,xlim = c(-2, 2),
ylim = c(1,13),
yaxt= "n",
xaxt = "n",
xlab = 'Z-score',
ylab = '-log2(Pvalue)',bty = 'n',
cex.lab=1.2, cex.axis=1.2, cex.main=1, cex.sub=1)
axis(1, at = seq(-2, 6, 2), cex.axis = 1.2, lwd = 1)
axis(2, at = seq(0,12,2), las = 2, cex.axis = 1.2, lwd = 1)
abline(h = -log2(0.05), lty= 2)
abline(v = 0          , lty = 2)
pointPlot(bubblePlotDf,
0.1, 8, col = bubblePlotDf$col)
dev.off()
```

```{r echo=FALSE}
# print bubble-plot to screen
par(mar=c(5,5,0,0))
plot(x=NA,y=NA,xlim = c(-2, 2),
ylim = c(1,13),
yaxt= "n",
xaxt = "n",
xlab = 'Z-score',
ylab = '-log2(Pvalue)',bty = 'n',
cex.lab=1.2, cex.axis=1.2, cex.main=1, cex.sub=1)
axis(1, at = seq(-2, 6, 2), cex.axis = 1.2, lwd = 1)
axis(2, at = seq(0,12,2), las = 2, cex.axis = 1.2, lwd = 1)
abline(h = -log2(0.05), lty= 2)
abline(v = 0          , lty = 2)
pointPlot(bubblePlotDf,
1, 8, col = bubblePlotDf$col)
```

```{r function to draw bubbleplot to .pdf}
##### another option: make a function to draw your bubble-plot to .pdf ####
draw <- function(x, id = nrow(x))
{
  pdf('Rfigs/GO.bubble.pdf', height = 6, width = 7.5)
  par(mar=c(5,5,0,0))
  plot(x=NA,y=NA,xlim = c(-2, 2),
       ylim = c(1,13),
       yaxt= "n",
       xaxt = "n",
       xlab = 'Z-score',
       ylab = '-log2(Pvalue)',bty = 'n',
       cex.lab=1.2, cex.axis=1.2, cex.main=1, cex.sub=1)
  axis(1, at = seq(-2, 6, 2), cex.axis = 1.2, lwd = 1)
  axis(2, at = seq(0,12,2), las = 2, cex.axis = 1.2, lwd = 1)
    abline (v = (ccn(x$zScore)[id]))
    abline (h = -log2(ccn(x$topGO))[id])
    pointPlot(x,
              0.1, 8, col = x$color)
    print(x$Significant[id])
    dev.off()
}
draw(bubblePlotDf)
```
