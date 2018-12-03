# CellTag Workflow

This repository contains our CellTag workflow.

Here is the link to the GEO DataSet which contains all of the sequencing data we generated.

https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE99915

The CellTag libraries are available from AddGene [here](https://www.addgene.org/pooled-library/morris-lab-celltag/). This link also contains whitelists for each CellTag library as well as sequence information for each library.

The CellTag libraries available at AddGene are labeled CellTag-V1, CellTag-V2, and CellTag-V3. These labels correspond to CellTag<sup>MEF</sup>, CellTag<sup>D3</sup>, and CellTag<sup>D13</sup> respectively.

From our GEO DataSet we will select one timepoint to use as an example of our CellTag processing and analysis.

We will use data collected from Timecourse 1 at Day 15. The link to the SRA for this sample is [here](https://trace.ncbi.nlm.nih.gov/Traces/sra/?run=SRR7347033).

In general, this workflow is broken up into three sections.

1. CellTag Extraction and Quantification
2. The Identification of Clones
3. Lineage Visualization

We are using the data from Day 15 because we should be able to call clones using each of the CellTag versions (CellTag<sup>MEF</sup>, CellTag<sup>Day3</sup>, and CellTag<sup>Day13</sup>)

You can follow along with this workflow by cloning or downloading this repository and executing the commands, using the appropriate paths for the data you are processing.

The workflow is a combination of bash and r commands, which will be identified with using comments in the code.

# 1. CellTag Extraction and Quantification

First, we will download the BAM file generated by the 10x CellRanger pipeline for this sample.

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
The reads which contain CellTags are then written to an output file. The CellTag motifs for CellTag<sup>MEF</sup>, CellTag<sup>D3</sup>, and CellTag<sup>D13</sup> are unique. This allows us to distinguish between CellTags originating from different libraries. The commands below will identify the CellTags for each CellTag version.

We use the *.possorted_genome_bam.bam output file from the 10x CellRanger pipeline, because each read has been tagged with its associated Cell Barcode, UMI, and aligned genes.


```bash
#bash
#MEF
samtools view hf1.d15.possorted_genome_bam.bam | grep -P 'GGT[ACTG]{8}GAATTC' > v1.celltag.reads.out

#D3
samtools view hf1.d15.possorted_genome_bam.bam | grep -P 'GTGATG[ACTG]{8}GAATTC' > v2.celltag.reads.out

#D13
samtools view hf1.d15.possorted_genome_bam.bam | grep -P 'TGTACG[ACTG]{8}GAATTC' > v3.celltag.reads.out

```

With the CellTag reads extracted we use a custom gawk script to parse the file and retain only the information we need. This scripts identifies and extracts the CellTag, Cell Barcode, and UMI sequences associated with each CellTag read. Furthermore, the read ID, read sequence, and any genes the read aligned to are extracted as well. This allows us to accurately quantify each CellTag and associate the CellTags with the correct cells.
The output of this script is a tab delimited file with the following columns: Read.ID, Read.Seq, Cell.BC, UMI, Cell.Tag, Gene.
We will use this file to quantify and filter the CellTag data. 
Note that the regular expression identifying CellTag<sup>MEF</sup> has two bases added to the beginning. This is a "stricter" CellTag motif which helps filter out some erroneous CellTag reads.

This command calls the script `celltag.parse.reads.10x.sh` from the script directory and passes two arguments to the script. 

1. The variable `tagregex` is set using the `-v tagregex=` option.
  + The `tagregex` variable is the regular expression which defines the CellTag motif of interest.
  + It is important to surround the [ACTG]{8} with parentheses, as this defines the CellTag sequence as a capture group in the regular expression.
  
2. The input file `v1.celltag.reads.out`.
  + This is the output file from the grep command above.
  
Finally, the output from this script is written to the file v1.celltag.parsed.tsv


```bash
#bash
#MEF
./scripts/celltag.parse.reads.10x.sh -v tagregex="CCGGT([ACTG]{8})GAATTC" v1.celltag.reads.out > v1.celltag.parsed.tsv

#D3
./scripts/celltag.parse.reads.10x.sh -v tagregex="GTGATG([ACTG]{8})GAATTC" v2.celltag.reads.out > v2.celltag.parsed.tsv

#D13
./scripts/celltag.parse.reads.10x.sh -v tagregex="TGTACG([ACTG]{8})GAATTC" v3.celltag.reads.out > v3.celltag.parsed.tsv

```

Now that the reads have been parsed we can quantify the CellTag for each cell and create a Cell Barcode x CellTag matrix.
We accomplish this using a custom R script that can be found in the `/scripts/` directory.
This script groups CellTags by Cell Barcodes and then counts the number of unique UMIs for each Cell Barcode x CellTag pair. Any Cell Barcodes not present in the filtered 10x CellRanger results are filtered out. This creates a CellTag UMI count matrix which is then used to identify clones.

The command `Rscript` allows you to run the custom R script `matrix.count.celltags.R` from the command line.
This script requires three arguments:

1. `hf1.d15.barcodes.tsv`
  + This is the output for this timepoint from the 10x CellRanger pipeline. The file contains a list of all cell barcodes identified in the filtered dataset.
  
2. `v1.celltag.parsed.tsv`
  + This is the output file from the previous command. It contains the parsed CellTag read data.
  
3. `hf1.d15.v1`
  + This argument is a prefix that will be used to uniquely identify output files from the script.

This step should be run for each CellTag version independently. 


```bash
#bash

#MEF
Rscript ./scripts/matrix.count.celltags.R ./cell.barcodes/hf1.d15.barcodes.tsv v1.celltag.parsed.tsv hf1.d15.v1

#D3
Rscript ./scripts/matrix.count.celltags.R ./cell.barcodes/hf1.d15.barcodes.tsv v2.celltag.parsed.tsv hf1.d15.v2

#D13
Rscript ./scripts/matrix.count.celltags.R ./cell.barcodes/hf1.d15.barcodes.tsv v3.celltag.parsed.tsv hf1.d15.v3

```

This process of identifying, extracting, and quantifying CellTag information is very similar to the processing necessary for REAP-Seq, CITE-Seq, and Feature Barcoding data. We are currently working on a more general CellTag processing workflow utilizing the tools available for processing these similar data types. 


# 2. Clone Calling

The remaining analysis, Clone Calling and Lineage Visualiztion, will be performed in R. It is necessary to perform the clone calling independently for each of the CellTag versions being analyzed.

First, we must load the required R packages and functions used to perform this analysis. The packages are all available through CRAN.


```r
#R
library(igraph)
library(proxy)
library(corrplot)
library(data.table)

source("./scripts/CellTagCloneCalling_Function.R")
```


Now that the necessary packages are loaded we will load the CellTag matrices for each CellTag version.
This is accomplished by reading the `.Rds` file which is output by the `matrix.count.celltags.R` script.
Once the CellTag matrices are loaded we set the rownames for each matrix to the Cell Barcodes present in the matrix, and then remove the column containing the Cell Barcodes.
Now that the data has been formatted correctly we use the `SingleCellDataBinarization` function to convert the CellTag UMI count matrices into binary matrices.

The `SingleCellDataBinarization` function takes the following arguments:

1. `celltag.dat`
  + This is the object containing the CellTag UMI matrix
2. `tag.cutoff`
  + This argument filters the CellTags based on their UMI counts.
    Any Cell Barcode/CellTag pair with a UMI count less than this cutoff will be disregarded.
      

```r
#R
mef.mat <- as.data.frame(readRDS("./hf1.d15.v1.celltag.matrix.Rds"))

d3.mat <- as.data.frame(readRDS("./hf1.d15.v2.celltag.matrix.Rds"))

d13.mat <- as.data.frame(readRDS("./hf1.d15.v3.celltag.matrix.Rds"))

rownames(mef.mat) <- mef.mat$Cell.BC

rownames(d3.mat) <- d3.mat$Cell.BC

rownames(d13.mat) <- d13.mat$Cell.BC

mef.mat <- mef.mat[,-1]

d3.mat <- d3.mat[,-1]

d13.mat <- d13.mat[,-1]

mef.bin <- SingleCellDataBinarization(celltag.dat = mef.mat, 2)

d3.bin <- SingleCellDataBinarization(celltag.dat = d3.mat, 2)

d13.bin <- SingleCellDataBinarization(celltag.dat = d13.mat, 2)
```


Now we have a binary CellTag matrix for each CellTag version. We will perform a few filtering steps before we identify clones.
First, each CellTag matrix will be filtered based on the CellTag Whitelist for the respective CellTag version.
This is done using the `SingleCellDataWhitelist` function with the following arguments:
1. `celltag.dat`
  + This is the object containing the binary CellTag matrix.
2. `whitels.cell.tag.file`
  + This is the csv file which contains the CellTag whitelist for the correct CellTag version.
  
This filters the CellTag matrix and removes any CellTags which are not present in the CellTag whitelist.
The CellTag whitelists were created by sequencing each CellTag library version. Then CellTags which were in the ninetieth percentile of read counts were included in the whitelist.
This allows us to determine which CellTags are truly present in the CellTag library. It is then possible to identify CellTag sequences which are the result of PCR and sequencing errors, allowing us to filter or correct the CellTag sequence.

We next filter cells based on their number of expressed CellTags. This is done using the function `MetricBasedFiltering`.

1. `whitelisted.celltag.data`
  + This argument is the object which contains the whitelist filtered binary CellTag matrix.
2. `cutoff`
  + This is an argument defining the cutoff for the number of CellTags per cell.
3. `comparison`
  + This argument determines whether cells with a number of CellTags greater than or less than the cutoff are filtered from the dataset.
  
These filtering steps are performed individually for each CellTag version dataset.


```r
#R
mef.filt <- SingleCellDataWhitelist(celltag.dat = mef.bin, whitels.cell.tag.file = "whitelist/V1.CellTag.Whitelist.csv")

d3.filt <- SingleCellDataWhitelist(celltag.dat = d3.bin, whitels.cell.tag.file = "whitelist/V2.CellTag.Whitelist.csv")

d13.filt <- SingleCellDataWhitelist(celltag.dat = d13.bin, whitels.cell.tag.file = "whitelist/V3.CellTag.Whitelist.csv")


mef.filt <- MetricBasedFiltering(whitelisted.celltag.data = mef.filt, cutoff = 20, comparison = "less")

d3.filt <- MetricBasedFiltering(whitelisted.celltag.data = d3.filt, cutoff = 20, comparison = "less")

d13.filt <- MetricBasedFiltering(whitelisted.celltag.data = d13.filt, cutoff = 20, comparison = "less")


mef.filt <- MetricBasedFiltering(whitelisted.celltag.data = mef.filt, cutoff = 2, comparison = "greater")

d3.filt <- MetricBasedFiltering(whitelisted.celltag.data = d3.filt, cutoff = 2, comparison = "greater")

d13.filt <- MetricBasedFiltering(whitelisted.celltag.data = d13.filt, cutoff = 2, comparison = "greater")
```


Now that the CellTag matrices have been filtered we can call clones! Clones are identified based on the Jaccard similarity of each cells CellTag signature.
First, we calculate the Jaccard similarity between each cell, using the `JaccardAnalysis` function.

1. `whitelisted.celltag.data`
  + This is the object containing the filtered binary CellTag matrix.
2. `plot.corr`
  + This is an option to plot the pairwise correlation between each cell.
  + This plot assists in visualizing clones.
  + The plot is saved as a pdf in the current working directory.
  
With the Jaccard similarities between each cell calculated, we can now identify clonally related cells and define clones present in the dataset.
The clone calling is performed using the function `CloneCalling`.

1. `Jaccard.Matrix` 
  + Object containing Jaccard similarities between each cell.
2. `outp.dir` 
  + Directory to save output files in.
3. `output.filename` 
  + Filename to use for output file.
4. `correlation.cutoff` 
  + The minimum Jaccard similarity between clonally related cells.
  
This function takes the Jaccard similarities and identifies clones of cells with a similarity greater than or equal to the cutoff. From our CellTagging validation experiments, we have found a cutoff of 0.7 to work well. Higher values increase the stringency of clone-calling but can lead to related cells being split into multiple clones. Lower values can produce false-positives. The function returns a list of two data frames. One with a list of Cell Barcodes and the clone it belongs to, and the other with a table of the number of cells in each clone. 




```r
#R
mef.sim <- JaccardAnalysis(whitelisted.celltag.data = mef.filt, plot.corr = FALSE, id = "mef")

d3.sim <- JaccardAnalysis(whitelisted.celltag.data = d3.filt, plot.corr = FALSE, id = "d3")

d13.sim <- JaccardAnalysis(whitelisted.celltag.data = d13.filt, plot.corr = FALSE, id = "d13")


mef.clones <- CloneCalling(Jaccard.Matrix = mef.sim, output.dir = "./", output.filename = "hf1.d15.v1.clones.csv", correlation.cutoff = 0.7)

d3.clones <- CloneCalling(Jaccard.Matrix = d3.sim, output.dir = "./", output.filename = "hf1.d15.v2.clones.csv", correlation.cutoff = 0.7)

d13.clones <- CloneCalling(Jaccard.Matrix = d13.sim, output.dir = "./", output.filename = "hf1.d15.v3.clones.csv", correlation.cutoff = 0.7)
```


# 3. Lineage and Network Visualization

Load functions required for CellTag lineage network construction and Visualization


```r
<<<<<<< HEAD
# this script require “foreach” and "networkD3" package. Please install them beforehand.
library(tidyverse)
=======
# this script requires “foreach” and "networkD3" package. Please install them beforehand.
>>>>>>> bd80f244a1325279f2f97ff236b2017774417141
library(foreach)
library(networkD3)

source("./scripts/function_source_for_network_visualization.R")
source("./scripts/function_source_for_network_construction.R")
```

Load CellTag data.


```r
# The called CellTag V1, V2 and V3 will be used to construct Lineage network graph.
# Input CellTag data should be Nx3 matrix. Each row represents cell name (cell barcode) and each column represents CellTag name.
# See the example (./input_data_for_network_construction/)

# load celltag table

mef.clones <- read_csv(file = "hf1.d15.v1.clones.csv")

d3.clones <- read_csv(file = "hf1.d15.v2.clones.csv")
  
d13.clones <- read_csv(file = "hf1.d15.v3.clones.csv")


colnames(mef.clones)[1] <- "CellTagV1"

colnames(d3.clones)[1] <- "CellTagV2"

colnames(d13.clones)[1] <- "CellTagV3"

clone.cells <- unique(c(mef.clones$cell.barcode, d3.clones$cell.barcode, d13.clones$cell.barcode))

celltag_data <- data.frame(clone.cells, row.names = clone.cells)

celltag_data$CellTagV1 <- NA

celltag_data$CellTagV2 <- NA

celltag_data$CellTagV3 <- NA

celltag_data[mef.clones$cell.barcode, "CellTagV1"] <- mef.clones$CellTagV1

celltag_data[d3.clones$cell.barcode, "CellTagV2"] <- d3.clones$CellTagV2

celltag_data[d13.clones$cell.barcode, "CellTagV3"] <- d13.clones$CellTagV3

celltag_data <- celltag_data[, -1]

row.names(celltag_data) <- paste0(rownames(celltag_data), "-1")
```
Construct Network.


```r
# please change the celltag name to c(“CellTagV1”, “CellTagV2”, “CellTagV3”) if you are using different name
colnames(celltag_data) <- c("CellTagV1", "CellTagV2", "CellTagV3")

# the first step is to construct linkList, which represents network structure.
# CellTag data will be converted into LinkList. 
#First, clonal population that share the same celltag is combined to make subnetwork. Then, subnetworks will be combined further if they originate from the same ancestor.

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

additional_data <- data.frame(sample(1:10, size = length(row.names(celltag_data)), replace = TRUE), row.names = row.names(celltag_data))
Nodes <- addData2Nodes(Nodes, additional_data)
colnames(Nodes)[4] <- "Cluster"
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
# you can pick up sub-network by selecting the CellTag clone id.

# Parameters
# tag: CellTag clone id
# overlay: you can overlay information on network graph
drawSubnet(tag = "CellTagV1_2", overlay = "Cluster", linkList = linkList, Nodes = Nodes )
```


Reshape data for ggplot2



```r
bar.data <- celltag_data

bar.data$Cell.BC <- row.names(bar.data)

bar.data <- gather(bar.data, key = "CellTag", value = "Clone", 1:3, na.rm = FALSE)
```


Plot Stacked Bar Chart


```r
ggplot(data = bar.data) + geom_bar(mapping = aes(x = CellTag, fill = factor(Clone)), position = "fill", show.legend = FALSE) + scale_y_continuous(labels = scales::percent_format())
```
