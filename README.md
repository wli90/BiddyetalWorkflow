# CellTag Workflow

This repository will contain our CellTag workflow.

Here is the link to the GEO DataSet which contains all of the sequencing data we generated.

https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE99915

The CellTag libraries are available from AddGene [here](https://www.addgene.org/pooled-library/morris-lab-celltag/). This link also contains whitelists for each CellTag library as well as sequence information for each library.

The CellTag libraries available at AddGene are labeled CellTag-V1, CellTag-V2, and CellTag-V3. These labels correspond to CellTag^MEF^, CellTag^D3^, and CellTag^D13^ respectively.

From our GEO DataSet we will select one timepoint to use as an example of our CellTag processing and analysis.

We will use data collected from Timecourse 1 at Day 15. The link to the SRA for this sample is [here](https://trace.ncbi.nlm.nih.gov/Traces/sra/?run=SRR7347033).

In general this workflow is broken up into three sections.

1. CellTag Extraction and Quantification
2. The Identification of Clones
3. Lineage Visualization

We are using the data from Day 15 because we should be able to call clones using each of the CellTag versions (CellTag^MEF^, CellTag^Day3^, and CellTag^Day13^)

You can follow along with this workflow by cloning or downloading this repository and executing the commands, using the appropriate paths for the data you are processing.

The workflow is a combination of bash and r commands, which will be identified with using comments in the code.

# 1. CellTag Extraction and Quantification

First we will download the BAM file generated by the 10x CellRanger pipeline for this sample.

The link for this BAM file is:

https://sra-download.ncbi.nlm.nih.gov/traces/sra65/SRZ/007347/SRR7347033/hf1.d15.possorted_genome_bam.bam

This BAM file contains all of the reads from the Day 15 timepoint.

This command will download the BAM file into your current working directory. This BAM file is nearly 40 GBs so the download could take some time to complete. 


```bash
#bash
wget https://sra-download.ncbi.nlm.nih.gov/traces/sra65/SRZ/007347/SRR7347033/hf1.d15.possorted_genome_bam.bam

```

Now that we have downloaded the BAM file we can search for any read containing a CellTag motif. First, because the BAM file is binary we need to view the file using a tool such as samtools.
We do this by using the samtools view command. The output from this command is then piped into grep. We use grep and a regular expression to search for CellTag containing reads.
The reads which contain CellTags are then written to an output file. The CellTag motifs for CellTag^MEF^, CellTag^D3^, and CellTag^D13^ are unique. This allows us to distinguish between CellTags originating from different libraries. 


```bash
#bash
#V1
samtools view hf1.d15.possorted_genome_bam.bam | grep -P 'GGT[ACTG]{8}GAATTC' > v1.celltag.reads.out

#V2
samtools view hf1.d15.possorted_genome_bam.bam | grep -P 'GTGATG[ACTG]{8}GAATTC' > v2.celltag.reads.out

#V3
samtools view hf1.d15.possorted_genome_bam.bam | grep -P 'TGTACG[ACTG]{8}GAATTC' > v3.celltag.reads.out

```

With the CellTag reads extracted we use a custom gawk script to parse the file and retain only the information we need.



```bash
#bash
#V1
./scripts/celltag.parse.reads.10x.sh -v tagregex="CCGGT([ACTG]{8})GAATTC" v1.celltag.reads.out > v1.celltag.parsed.tsv

#V2
./scripts/celltag.parse.reads.10x.sh -v tagregex="GTGATG([ACTG]{8})GAATTC" v2.celltag.reads.out > v2.celltag.parsed.tsv

#V3
./scripts/celltag.parse.reads.10x.sh -v tagregex="TGTACG([ACTG]{8})GAATTC" v3.celltag.reads.out > v3.celltag.parsed.tsv

```

Now with the parsed reads we will create a Cell by CellTag Matrix.



```bash
#bash
Rscript ./scripts/matrix.count.celltags.R ./cell.barcodes/hf1.d15.barcodes.tsv v1.celltag.parsed.tsv hf1.d15.v1

```


# 2. Clone Calling

Load functions required for clone calling



```r
library(igraph)
library(proxy)
library(corrplot)
library(data.table)

source("./scripts/CellTagCloneCalling_Function.R")
```


Load CellTag Matrix and create binary matrix.




```r
celltag.mat <- readRDS("./hf1.d15.v1.celltag.matrix.Rds")

rownames(celltag.mat) <- celltag.mat$Cell.BC

celltag.mat <- celltag.mat[, -1]

celltag.bin <- SingleCellDataBinarization(celltag.dat = celltag.mat, 2)
```


Filter CellTag matrix



```r
celltag.filt <- SingleCellDataWhitelist(celltag.dat = celltag.bin, whitels.cell.tag.file = "whitelist/V1.CellTag.Whitelist.csv")

celltag.filt <- MetricBasedFiltering(whitelisted.celltag.data = celltag.filt, cutoff = 20, comparison = "less")

celltag.filt <- MetricBasedFiltering(whitelisted.celltag.data = celltag.filt, cutoff = 2, comparison = "greater")
```


Now Call Clone based on Jaccard Similarity



```r
celltag.jac <- JaccardAnalysis(whitelisted.celltag.data = celltag.filt, plot.corr = FALSE)

clones <- CloneCalling(Jaccard.Matrix = celltag.jac, output.dir = "./", output.filename = "hf1.d15.v1.clones.csv", correlation.cutoff = 0.7)
```


# 3. Lineage and Network Visualization

Load functions required for CellTag lineage network construction and Visualization


```r
# this script require “foreach” and "networkD3" package. Please install them beforehand.
library(foreach)
library(networkD3)

source("./scripts/function_source_for_network_visualization.R")
source("./scripts/function_source_for_network_construction.R")
```

Load Celltag data.


```r
# The called CellTag V1, V2 and V3 will be used to construct Lineage network graph.
# Input Celltag data should be Nx3 matrix. Each row represents cell name (cell barcode) and each column represents Celltag name.
# See the example (./input_data_for_network_construction/)

# load celltag table
celltag_data <- read.table(PATH_TO_YOUR_CELLTAG_TABLE, stringsAsFactors = F)
head(celltag_data)
```
Construct Network.


```r
# please change the celltag name to c(“CellTagV1”, “CellTagV2”, “CellTagV3”) if you are using different name
colnames(celltag_data) <- c("CellTagV1", "CellTagV2", "CellTagV3")

# the first step is to construct linkList, which represents network structure.
# Celltag data will be converted into LinkList. 
#First, clonal population that share the same celltag is combined to make subnetwork. Then, subnewtorks will be combined further if they are originated from same mother.

linkList <- convertCellTagMatrix2LinkList(celltag_data)
```

Make Node list


```r
Nodes <- getNodesfromLinkList(linkList)
head(Nodes)
```
(Optional) Add data to Node list.


```r
# If you want to visualize some information on the network graph, you can add any information to Node list. 
# the rownames of additional_data should be same format as “node_name_unmodified” in Nedes
# See the example (./input_data_for_network_construction/meta_data.txt)

additional_data <- read.table(PATH_TO_YOUR_ADDITIONAL_DATA_TABLE, stringsAsFactors = F)
Nodes <- addData2Nodes(Nodes, additional_data)
head(Nodes)
```

Save results.


```r
# Save results before visualization.
# We provides Rscript for the visualization below, but you can use another software for network visualization. 
# This Nodes and link file can be imported to Cytoscape for the visualization.

write.table(Nodes, file = "./output/Nodes.txt")
write.table(linkList, file = "./output/links.txt")
```

Load network data.


```r
Nodes <- read.table("./output/Nodes.txt", stringsAsFactors = F)
linkList <- read.table("./output/links.txt", stringsAsFactors = F)
head(Nodes)
```

Visualize subnetwork as Force-directed network


```r
# In general, wholw network graph is too large to visualize in R.
# You can visualize sub-network which show the lineage of single clone instead of whole network graph. 
# you can pick up sub-network by selecting CellTag clone id.

# Parameters
# tag: CellTag clone id
# overlay: you can overlay information on network graph
drawSubnet(tag = "CellTagV1_108", overlay = "Reprogramming.Day", linkList = linkList, Nodes = Nodes )
```

