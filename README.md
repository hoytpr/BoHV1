# GeneExpression Analysis of the BoHV1 infected bovine
This is a repository for gene expression analysis of the Ilumina RNA-Seq data from BoHV1 infected bovine TG (if T0-T3 and M) and tonsil (t0-t3 and m).

## RNA-Seq PIPELINE
This pipeline performs the following tasks:
- perform quality control on FastQ files (using FastQC)
- align reads of each sample against reference genome (using tophat2) 
- count reads in features (cufflinks)
- normalize read counts (using cufflinks::cuffdiff)
- calculate RPKMs (using CummeRbund)
- perform DE analysis for standard designs (using CummeRbund)
- Generate Gene Onthology and WikiPathways outputs!

## Download
Use git clone:
```
git clone git@github.com:gabrielachavez2019/BoHV1.git
```

## Installation
Download the RNAseq pipeline.


## Genome files
#### Generate genome indexes or for bovine, just download it from [igenomes](https://support.illumina.com/sequencing/sequencing_software/igenome.html)
Create a directory where you want to store the indexes (e.g. /GENOMES/Bos_taurus/UCSC/bosTau8/).

Collect the following files for your genome:
- Fasta file containing the genome reference sequences
- GTF file containing annotated transcripts

#### Generate genome indexes
Generate genome indexes files for the Viral genomes. 
In this case we are using the Bovine Herpesvirus 1, available at the GenBank:[NC_001847.1](https://www.ncbi.nlm.nih.gov/nuccore/9629818). 
Get the FASTA file from [BHV1.FASTA](https://www.ncbi.nlm.nih.gov/nuccore/NC_001847.1?report=fasta)
The genome indexes are saved to disk and need only be generated once for each genome/annotation combination.

```  
bowtie2-build BHV1.FASTA BHV1
```

## Usage
#### Run pipeline
```bash
sbatch mapping.sh -input [/path/to/rundir] -outputDir [/path/to/outputdirname] -mail [email]
```

## Dependencies
#### Load modules
- fastqc 0.11.7 
- bowtie2 2.3.4.1  
- samtools 1.10
- trimmomatic 0.38
- cufflinks 2.2.1
- tophat 2.1.2 
- R 4.0.3    

#### R packages
- DESeq2 1.6.3
- edgeR 3.8.6
- ggplot2
- gplots
- RColorBrewer
- CummeRbund


## Mapping
Illumina pair-end hight quality reads are aligned first to the BoHV1 genome using minimap, which is better for viral genomes. Followed by mapping them agaist the bovine genome using TOPHAT2, which was designed to specifically address many of the challenges of RNA-seq data mapping, and uses a novel strategy for spliced alignments.

## Read counting
Counting sequencing reads in features (genes/transcripts/exons) is done with cufflinks using the union mode.

## Normalizing read counts
Cufflinks with default options --library-norm-method classic-fpkm (default) for Cufflinks --library-norm-method geometric (default) for cuffdiff.
Cuffnorm will report both FPKM values and normalized, estimates for the number of fragments that originate from each gene, transcript, TSS group, and CDS group. Note that because these counts are already normalized to account for differences in library size, they should not be used with downstream differential expression tools that require raw counts as input.

```
cufflinks --num-threads 32 -o /scratch/gatoo/output_cuff_"$name"  --frag-bias-correct /scratch/gatoo/Bos_taurus/UCSC/bosTau8/Sequence/WholeGenomeFasta/genome.fa  --multi-read-correct -G /scratch/gatoo/Bos_taurus/UCSC/bosTau8/Annotation/Genes/genes.gtf /scratch/gatoo/output_"$name"/accepted_hits.bam
```
This runs in the same mapping file but stops here, because manually you need to create an assemblies file, depending on the comparasions you are interested in, in this example Latent (n=2) vs Uninfected (n=3) would be only t0,m. However multiple comparisons can be made if need it (m,t0,t1,t2,t3).

```
cuffmerge -g /scratch/gatoo/Bos_taurus/UCSC/bosTau8/Annotation/Genes/genes.gtf -s /scratch/gatoo/Bos_taurus/UCSC/bosTau8/Annotation/Genes/genes.fa -p 8 assemblies.txt
```
The assemblies file need to have listed the GTF files from the cufflinks previous count, I usually put all mapping files in my scratch, so:

/scratch/gatoo/output_cuff_tonsil_Lat_1/transcripts.gtf
/scratch/gatoo/output_cuff_tonsil_Lat_2/transcripts.gtf
/scratch/gatoo/output_cuff_tonsil_Uni_1/transcripts.gtf
/scratch/gatoo/output_cuff_tonsil_Uni_2/transcripts.gtf
/scratch/gatoo/output_cuff_tonsil_Uni_3/transcripts.gtf


```
cuffdiff --num-threads 32 -o diff_out -b /scratch/gatoo/Bos_taurus/UCSC/bosTau8/Annotation/Genes/genes.fa -p 8 -L Lat,Uni -u merged_asm/merged.gtf \/scratch/gatoo/output_"$name"/accepted_hits.bam,/scratch/gatoo/output_"$name"/accepted_hits.bam   \/scratch/gatoo/output_"$name"/accepted_hits.bam,/scratch/gatoo/output_"$name"/accepted_hits.bam,/scratch/gatoo/output_"$name"/accepted_hits.bam 

```

## Calculate FPKMs
Cuffdiff calculates the FPKM of each transcript, primary transcript, and gene in each sample. Primary transcript and gene FPKMs are computed by summing the FPKMs of transcripts in each primary transcript group or gene group. The results are output in FPKM tracking files in the format described here. There are four FPKM tracking files:

isoforms.fpkm_tracking	Transcript FPKMs
genes.fpkm_tracking	Gene FPKMs. Tracks the summed FPKM of transcripts sharing each gene_id
cds.fpkm_tracking	Coding sequence FPKMs. Tracks the summed FPKM of transcripts sharing each p_id, independent of tss_id
tss_groups.fpkm_tracking	Primary transcript FPKMs. Tracks the summed FPKM of transcripts sharing each tss_id

## Differential expression analysis
CummeRbund makes managing and querying data easier by loading the data into multiple objects of several different classes and having functions to query them. Because all of this gets stored in an sql database, you can access it quickly without loading everything in to memory.

  readCufflinks- Most important function designed to read all the output files that cuffdiff generates into an R data object (of class CuffSet).
 ```
 cuff_data <- readCufflinks(diff_out)
``` 
CummeRbund has at least 6 classes that it will place different parts of your data in.
Now you can access information using different functions:  gene information using genes(cuff_data), your isoform level output using isoforms(cuff_data), TSS related groups using tss(cuff_data) and so forth

You can explore global statistics on data for quality, dispersion, distribution of gene expression scores etc or Plot things for specific features or genes.

### Create global statistics for Quality Control
Overdispersion is a common problem in RNA-Seq data. To evaluate the quality of the model fitting I used cufflinks v2.0 mean counts, variance, and dispersion to visualize the estimated overdispersion for each sample as a quality control measure.
```
disp<-dispersionPlot(genes(cuff_data)) 
```
![disp](https://github.com/gabrielachavez2019/BoHV1/blob/master/disp.png)

The squared coefficient of variation is a normalized measure of cross-replicate variability that can be useful for evaluating the quality the RNA-seq data. Differences in CV2 can result in lower numbers of differentially expressed genes due to a higher degree of variability between replicate fpkm estimates.
```
genes.scv<-fpkmSCVPlot(genes(cuff_data))
isoforms.scv<-fpkmSCVPlot(isoforms(cuff_data))
```
![genes.scv](https://github.com/gabrielachavez2019/BoHV1/blob/master/genes.scv.png)
![isoforms.scv](https://github.com/gabrielachavez2019/BoHV1/blob/master/isoforms.scv.png)

To assess the distributions of FPKM scores across samples, I used the csDensity plot.
```
densRep<-csDensity(genes(cuff_data),replicates=T)
```
![densRep](https://github.com/gabrielachavez2019/BoHV1/blob/master/densRep.png)

Individual Pairwise comparisons were made by using csScatter between Uninfected (x) vs Latent (y) treatments.
```
sp<-csScatter(genes(cuff_data),"Uni","Lat",smooth=T)
```
![sp](https://github.com/gabrielachavez2019/BoHV1/blob/master/sp.png)

Next, I cluster the Gene Expression data, this allows an openended exploration of the data, without getting lost among the thousands of individual genes. Beyond simple visualization. I used a computational application for gene clustering that is part of the cummeRbund suite.
```
den<-csDendro(genes(cuff_data), replicates=T)
```
![den](https://github.com/gabrielachavez2019/BoHV1/blob/master/den.png)

The volcano plot arrange genes along dimensions of biological and statistical significance. The first (horizontal) dimension is the fold change of Latency vs Uninfected (on a log scale, so that up and down regulation appear symmetric), and the second (vertical) axis represents the p-value for a t-test of differences between samples (most conveniently on a negative log scale ??? so smaller p-values appear higher up). The first axis indicates biological impact of the change; the second indicates the statistical evidence, or reliability of the change, in red are marked all the genes that have a significant p-value (alpha < 0.05).
```
v<-csVolcanoMatrix(genes(cuff_data))
```
![v](https://github.com/gabrielachavez2019/BoHV1/blob/master/v.png)

Later, I computed the graph-based distance matrix for N single gene in expression space for Latent and Uninfected within replicates to get the JS Distance, the more red the more different the more white or close to zero the more similar.  
```
myDistHeat<-csDistHeat(genes(cuff_data),replicates=T)
```
![myDistHeat](https://github.com/gabrielachavez2019/BoHV1/blob/master/myDistHeat.png)
 
 On the same page, is also useful to get a matrix with significant values:
```
mySigMat<-sigMatrix(cuff_data,level='genes',alpha=0.05)
```

To reduce the dimensionality of the data while retaining most of the variation in the data set. I used a mathematical algorithm called  Principal component analysis (PCA) that accomplishes this reduction by identifying directions, called principal components, along which the variation in the data is maximal. By using a few components, each sample can be represented by relatively few numbers instead of by values for thousands of variables. Uninfected and Latency samples were then plotted, making it possible to visually assess similarities and differences between samples and determine whether samples can be grouped. 

![genes.PCA.rep](https://github.com/gabrielachavez2019/BoHV1/blob/master/genes.PCA.rep.png)

To get a table wit the significantly regulated genes:
```
gene_diff_data <- diffData(genes(cuff_data))
sig_gene_data <- subset(gene_diff_data, (significant == 'yes'))
diffGeneIDs <- getSig(cuff_data,level="genes",alpha=0.05)

#gene_short_name values (and corresponding XLOC_* values) can be retrieved from the CuffGeneSet by using:
names<-featureNames(diffGenes)
row.names(names)=names$tracking_id
diffGenesNames<-as.matrix(names)
diffGenesNames<-diffGenesNames[,-1]

# get the data for the significant genes
diffGenesData<-diffData(diffGenes)
row.names(diffGenesData)=diffGenesData$gene_id
diffGenesData<-diffGenesData[,-1]

# merge the two matrices by row names
diffGenesOutput<-merge(diffGenesNames,diffGenesData,by="row.names")

head(diffGenesOutput)
write.table(diffGenesOutput, 'diff_genes.txt', sep='\t',row.names = F, col.names = T, quote = F)
```

To analyze specific gene candidates
```
jones_list <- read.csv("jones_list2.csv", TRUE, ",")
myGeneIds <- (jones_list$wnt0060070)
myGenes2 <- getGenes(cuff_data, myGeneIds)

tail(fpkm(myGenes2))

h<-csHeatmap(myGenes2,cluster='both')
#h<-csHeatmap(myGenes,cluster='both',replicates=T)
h
b<-expressionBarplot(myGenes2)
b
###Finding similar expression
mySimilar<-findSimilar(cuff_data,"XLOC_002340", n=20)
mySimilar.expression<-expressionPlot(mySimilar,logMode=T,showErrorbars=F)
mySimilar.expression

#### Or individual gene
#myGeneId<-"LRP1"
myGene<-getGene(cuff_data,"XLOC_002340")
b<-expressionBarplot(myGene)
b

head(fpkm(isoforms(myGene)))


#h<-csHeatmap(myGenes,cluster='both')
h<-csHeatmap(myGene,cluster='both',replicates=T)
h
b<-expressionBarplot("myGenes", replicates=T)
b

myGene<-getGene(cuff_data,"Stat1")
igb<-expressionBarplot(isoforms(myGene),replicates=T)
igb

myGene<-getGene(cuff_data,"Jak1")
igb<-expressionBarplot(isoforms(myGene),replicates=T)
igb

myGene<-getGene(cuff_data,"Stat3")
igb<-expressionBarplot(isoforms(myGene),replicates=T)
igb
```


#### Additional tools
Please contact Gabriela Toomer (gabriela.toomer@gmail.com) if you want to add additional tools/scripts/options or have any questions.
