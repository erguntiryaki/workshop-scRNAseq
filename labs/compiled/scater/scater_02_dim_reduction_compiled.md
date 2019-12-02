---
title: "Scater/Scran: Dimensionality reduction"
author: "Åsa Björklund  &  Paulo Czarnewski"
date: "Sept 13, 2019"
output:
  html_document:
    self_contained: true
    highlight: tango
    df_print: paged
    toc: yes
    toc_float:
      collapsed: false
      smooth_scroll: true
    toc_depth: 3
    keep_md: yes
    fig_caption: true
  html_notebook:
    self_contained: true
    highlight: tango
    df_print: paged
    toc: yes
    toc_float:
      collapsed: false
      smooth_scroll: true
    toc_depth: 3
---

# Dimensionality reduction

Paulo Czarnewski


 h1, .h1, h2, .h2, h3, .h3, h4, .h4 {
    margin-top: 50px;
}
<style>
p.caption {font-size: 0.9em;font-style: italic;color: grey;margin-right: 10%;margin-left: 10%;text-align: justify;}
</style>

## Data preparation
***

First, let's load all necessary libraries and the QC-filtered dataset from the previous step.


```r
suppressPackageStartupMessages({
  library(scater)
  library(scran)
  library(cowplot)
  library(ggplot2)
  library(rafalib)
  library(umap)
})

sce <- readRDS("data/3pbmc_qc.rds")
```

### Feature selection

Next, we first need to define which features/genes are important in our dataset to distinguish cell types. For this purpose, we need to find genes that are highly variable across cells, which in turn will also provide a good separation of the cell clusters.


```r
sce <- computeSumFactors(sce, sizes=c(20, 40, 60, 80))
sce <- normalize(sce)
var.fit <- trendVar(sce, use.spikes=FALSE,method="loess",loess.args=list(span=0.02))
var.out <- decomposeVar(sce, var.fit)

mypar(1,2)
#plot mean over TOTAL variance
plot(var.out$mean, var.out$total, pch=16, cex=0.4, xlab="Mean log-expression",
     ylab="Variance of log-expression")
o <- order(var.out$mean)
lines(var.out$mean[o], var.out$tech[o], col="dodgerblue", lwd=2)

cutoff_value <- 0.2
cutoff <- var.out$bio > cutoff_value
points(var.out$mean[cutoff], var.out$total[cutoff], col="red", pch=16,cex=.6)

#plot mean over BIOLOGICAL variance
plot(var.out$mean, var.out$bio, pch=16, cex=0.4, xlab="Mean log-expression",
     ylab="Variance of log-expression")
lines(c(min(var.out$mean),max(var.out$mean)), c(0,0), col="dodgerblue", lwd=2)
points(var.out$mean[cutoff], var.out$bio[cutoff], col="red", pch=16,cex=.6)
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

```r
hvg.out <- var.out[which(var.out$FDR <= 0.05 & var.out$bio >= cutoff_value),]
hvg.out <- hvg.out[order(hvg.out$bio, decreasing=TRUE),]
```

### Z-score transformation

Now that the data is prepared, we now proceed with PCA. Since each gene has a different expression level, it means that genes with higher expression values will naturally have higher variation that will be captured by PCA. This means that we need to somehow give each gene a similar weight when performing PCA (see below). The common practice is to center and scale each gene before performing PCA. This exact scaling is called Z-score normalization it is very useful for PCA, clustering and plotting heatmaps. Additionally, we can use this function to remove any unwanted sources of variation from the dataset, such as `cell cycle`, `sequencing depth`, `percent mitocondria`. This is achieved by doing a generalized linear regression using these parameters as covariates in the model. Then the residuals of the model are taken as the "regressed data". Although not in the best way, batch effect regression can also be done here.


```r
# sce@assays$data@listData$scaled.data <- apply(exprs(sce)[rownames(hvg.out),,drop=FALSE],2,function(x) scale(x,T,T))
# rownames(sce@assays$data@listData$scaled.data) <- rownames(hvg.out)
```


## PCA
***

Performing PCA has many useful applications and interpretations, which much depends on the data used. In the case of life sciences, we want to segregate samples based on gene expression patterns in the data.

As said above, we use the `logcounts` and then set `scale_features` to TRUE in order to scale each gene.


```r
#Default Scater way
sce <- runPCA(sce, exprs_values = "logcounts",  scale_features = T,
              ncomponents = 30, feature_set = rownames(hvg.out),method = "prcomp")

#For some reason Scater removes the dimnames of "logcounts" after PCA, so we put it back
dimnames(sce@assays$data@listData$logcounts) <- dimnames(sce@assays$data@listData$counts)

#2nd way:
#sce <- runPCA(sce, exprs_values = "scaled.data", scale_features = FALSE,
#              ncomponents = 30, feature_set = rownames(hvg.out) )
```

We can plot the first 6 dimensions like so.


```r
plot_grid(ncol = 3,
  plotReducedDim(sce,use_dimred = "PCA",colour_by = "sample_id",ncomponents = 1:2,add_ticks = F),
  plotReducedDim(sce,use_dimred = "PCA",colour_by = "sample_id",ncomponents = 3:4,add_ticks = F),
  plotReducedDim(sce,use_dimred = "PCA",colour_by = "sample_id",ncomponents = 5:6,add_ticks = F) )
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

To identify which genes (Seurat) or metadata paramters (Scater/Scran) contribute the most to each PC, one can retreive the loading matrix information. Unfortunatelly this is not implemented in Scater/Scran, so you will need to compute PCA using `logcounts`.


```r
plot_grid(ncol = 2, plotExplanatoryPCs(sce))
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

We can also plot the amount of variance explained by each PC.


```r
mypar()
plot(attr(sce@reducedDims$PCA,"percentVar")[1:50]*100,type="l",ylab="% variance",xlab="Principal component #")
points(attr(sce@reducedDims$PCA,"percentVar")[1:50]*100,pch=21,bg="grey",cex=.5)
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

Based on this plot, we can see that the top 7 PCs retain a lot of information, while other PCs contain pregressivelly less. However, it is still advisable to use more PCs since they might contain informaktion about rare cell types (such as platelets and DCs in this dataset)

## tSNE
***

We can now run [BH-tSNE](https://arxiv.org/abs/1301.3342).


```r
set.seed(42)
sce <- runTSNE(sce, use_dimred = "PCA", n_dimred = 30, 
               perplexity = 30)
#see ?Rtsne and ?runTSNE for more info
reducedDimNames(sce)[reducedDimNames(sce)=="TSNE"] <- "tSNE_on_PCA"
```

We can now plot the tSNE colored per dataset. We can start now clearly see the effect of batches present in the dataset.


```r
plot_grid(ncol = 3,plotReducedDim(sce,use_dimred = "tSNE_on_PCA",colour_by = "sample_id",add_ticks = F))
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-9-1.png)<!-- -->


## UMAP
***

We can now run [UMAP](https://arxiv.org/abs/1802.03426).


```r
sce <- runUMAP(sce,use_dimred = "PCA", n_dimred = 30,   ncomponents = 2)

#We need to rename it to not overide with other UMAP computations
try(sce@reducedDims$UMAP_on_RNA <- NULL)
reducedDimNames(sce)[reducedDimNames(sce)=="UMAP"] <- "UMAP_on_PCA"
#see ?umap and ?runUMAP for more info
```

Another usefullness of UMAP is that it is not limitted by the number of dimensions the data cen be reduced into (unlike tSNE). We can simply reduce the dimentions altering the `n.components` parameter.


```r
sce <- runUMAP(sce,use_dimred = "PCA", n_dimred = 30,   ncomponents = 10)
#see ?umap and ?runUMAP for more info

#We need to rename it to not overide with other UMAP computations
try(sce@reducedDims$UMAP10_on_RNA <- NULL)
reducedDimNames(sce)[reducedDimNames(sce)=="UMAP"] <- "UMAP10_on_PCA"
```

We can now plot the UMAP colored per dataset. Although less distinct as in the tSNE, we still see quite an effect of the different batches in the data.


```r
plot_grid(ncol = 3,
          plotReducedDim(sce,use_dimred = "UMAP_on_PCA",colour_by = "sample_id",add_ticks = F)+
            ggplot2::ggtitle(label ="UMAP_on_PCA"),
          plotReducedDim(sce,use_dimred = "UMAP10_on_PCA",colour_by = "sample_id",ncomponents = 1:2,add_ticks = F)+
            ggplot2::ggtitle(label ="UMAP10_on_PCA"),
          plotReducedDim(sce,use_dimred = "UMAP10_on_PCA",colour_by = "sample_id",ncomponents = 3:4,add_ticks = F)+
            ggplot2::ggtitle(label ="UMAP10_on_PCA")
)
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-12-1.png)<!-- -->


## Using ScaledData and graphs for DR
***

Althought running a sencond dimmensionality reduction (i.e tSNE or UMAP) on PCA would be a standard approach (because it allows higher computation efficiency), the options are actually limiteless. Below we will show a couple of other common options such as running directly on the scaled data (which was used for PCA) or on a graph built from scaled data. We will show from now on only UMAP, but the same applies for tSNE.

### Using ScaledData for UMAP

To run tSNE or UMAP on the scaled data, one firts needs to select the number of variables to use. This is because including dimentions that do contribute to the separation of your cell types will in the end mask those differences. Another reason for it is because running with all genes/features also will take longer or might be computationally unfeasible. Therefore we will use the scaled data of the highly variable genes.


```r
sce <- runUMAP(sce, exprs_values='logcounts', feature_set = rownames(hvg.out))

#We need to rename it to not overide with other UMAP computations
try(sce@reducedDims$UMAP_on_ScaleData <- NULL)
reducedDimNames(sce)[reducedDimNames(sce)=="UMAP"] <- "UMAP_on_ScaleData"
```

To run tSNE or UMAP on the a graph, we first need to build a graph from the data. In fact, both tSNE and UMAP first build a graph from the data using a specified distance metrix and then optimize the embedding. Since a graph is just a matrix containing distances from cell to cell and as such, you can run either UMAP or tSNE using any other distance metric desired. Euclidean and Correlation are ususally the most commonly used.

### Using a Graph for UMAP


```r
#Build Graph
g <- buildKNNGraph(sce,k=30,use.dimred="PCA",assay.type="RNA")
sce@reducedDims$KNN <- igraph::as_adjacency_matrix(g)


#Run UMAP and rename it for comparisson
# temp <- umap::umap.defaults
# temp$input <- "dist"
sce <- runUMAP(sce,use_dimred = "KNN", ncomponents = 2, input="data")
try(sce@reducedDims$UMAP_on_Graph <- NULL)
reducedDimNames(sce)[reducedDimNames(sce)=="UMAP"] <- "UMAP_on_Graph"
```


We can now plot the UMAP comparing both on PCA vs ScaledSata vs Graph.


```r
plot_grid(ncol = 3,
  plotReducedDim(sce, use_dimred = "UMAP_on_PCA", colour_by = "sample_id",add_ticks = F)+ 
    ggplot2::ggtitle(label ="UMAP_on_PCA"),
  plotReducedDim(sce, use_dimred = "UMAP_on_ScaleData", colour_by = "sample_id",add_ticks = F)+
    ggplot2::ggtitle(label ="UMAP_on_ScaleData"),
  plotReducedDim(sce, use_dimred = "UMAP_on_Graph", colour_by = "sample_id",add_ticks = F)+
    ggplot2::ggtitle(label ="UMAP_on_Graph")
)
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-15-1.png)<!-- -->

## Ploting genes of interest
***


Let's plot some marker genes for different celltypes onto the embedding. Some genes are:

Markers	| Cell Type
--- | ---
CD3E	| T cells
CD3E CD4	| CD4+ T cells
CD3E CD8A	| CD8+ T cells
GNLY, NKG7	| NK cells
MS4A1	| B cells
CD14, LYZ, CST3, MS4A7	| CD14+ Monocytes
FCGR3A, LYZ, CST3, MS4A7	| FCGR3A+  Monocytes
FCER1A, CST3 | DCs


```r
plotlist <- list()
for(i in c("CD3E","CD4","CD8A","NKG7","GNLY","MS4A1","CD14","LYZ","MS4A7","FCGR3A","CST3","FCER1A")){
  plotlist[[i]] <- plotReducedDim(sce,use_dimred = "UMAP_on_PCA",colour_by = i,by_exprs_values = "logcounts",add_ticks = F) +
  scale_fill_gradientn(colours = colorRampPalette(c("grey90","orange3","firebrick","firebrick","red","red" ))(10)) +
  ggtitle(label = i)+ theme(plot.title = element_text(size=20)) }
plot_grid(ncol=3, plotlist = plotlist)
```

![](scater_02_dim_reduction_compiled_files/figure-html/unnamed-chunk-16-1.png)<!-- -->


We can finally save the object for use in future steps.


```r
saveRDS(sce,"data/3pbmc_qc_dm.rds")
```

















