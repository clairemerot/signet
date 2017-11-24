# signet: Selection Inference in Gene NETworks

## General principle

The `signet` R package implements a method to detect selection in biological 
pathways. The general idea is to search for gene subnetworks within biological 
pathways that present unusual features, using a heuristic approach 
(simulated annealing).

The general idea is simple: we consider a gene list with prealably defined
scores (e.g. a differentiation measure like the Fst) and we want to find
gene networks presenting a score higher than expected under the null hypothesis.

To do so, we will use biological pathways databases converted as gene networks
and search in these graphs for high-scoring subnetworks.

Details about the algorithm can be found in
<a href="https://doi.org/10.1093/nar/gkx626">Gouy et al. (2017)</a>.

## Citation

Please cite this paper if you use `signet` for your project:

* Gouy A, Daub JT & Excoffier L (2017) Detecting gene subnetworks under 
selection in biological pathways. Nucleic Acids Research 45(16): e149

## Walkthrough example

### Installation

There is no official release of the `signet` package at the moment. 
But you can install the development version on GitHub using the `devtools` 
package (`Rtools` must also be installed and properly configured):

```{r}
## install devtools and signet dependencies
# install.packages('devtools')
# source("http://bioconductor.org/biocLite.R")
# biocLite("graph")
# biocLite("RBGL")
# install.packages("igraph")

devtools::install_github('CMPG/signet')
library(signet)
```

### Input

`signet` takes as input a data frame of gene scores. The first column must
correspond to the gene ID (e.g. Entrez) and the second columns is the gene 
score (a single value per gene).

The other input is a list of biological pathways (gene networks) in the
`graphNEL` format. We advise to use the package `graphite` to get the
pathway data:

```{r}
library(signet)
library(graphite)

# pathwayDatabases() #to have a look at pathways and species available
paths <- pathways("hsapiens", "kegg") #get the pathway list
graphs <- lapply(paths, pathwayGraph) #convert pathways to graphs
```

Note that gene identifiers must be the same between the gene scores data frame
and the pathway list (e.g. entrez). `graphite` provides a function to convert
gene identifiers.

A example dataset from Daub et al. (2013, MBE) as well as human KEGG 
pathways are provided:

```{r}
data(daub13)
head(scores) #gene scores
head(kegg_human) #pathways
```

### Workflow

We first have to search for high-scoring subnetworks within the
provided biological pathways, using simulated annealing:

```{r}
#Run simulated annealing on the first 10 pathways:
HSS <- searchSubnet(kegg_human[1:10],
                    scores,
                    iterations = 5000)
```

This function returns, for each pathway, the highest-scoring subnetwork found,
its score and other information about the simulated annealing run.

Then, to test the significance of the high-scoring subnetworks, we
generate a null distribution of high-scores:

```{r}
#Generate the null distribution
null <- nullDist(kegg_human,
                 scores,
                 n = 1000)
```

Note that the `null` object is a simple vector of null high-scores (here, 1000).
Therefore, you can run other iterations afterwards and concatenate the output
with the previous vector if you want to compute more precise p-values.

This distribution is finally used to compute p-values and update the
`signet` object:

```{r}
HSS <- testSubnet(HSS,null)
```

### Interpretation of the results

When p-values have been computed, you can generate a summary table
(one row per pathway):

```{r}
# Results: generate a summary table
tab <- summary(HSS)
head(tab)

# you can write the summary table as follow:
# write.table(tab,
#             file = "signet_output.tsv",
#             sep = "\t",
#             quote = FALSE,
#             row.names = FALSE)

```

Note that searching for high-scoring subnetworks and generating the null
distribution can take a few hours. However, these steps are easy
to parallelize on a cluster as different iterations are independent from each
other.

### Plot the results using Cytoscape

Cytoscape (www.cytoscape.org) is an external software dedicated to network 
visualization. `signet` allows to generate an XGMML file to be loaded in 
Cytoscape (File > Import > Network > File...). 
This file can be written in your working directory thanks to the 
function `writeXGMML`.

1. Plot a single pathway and its highest-scoring subnetwork

If the input of the function is a single `signet` object, the whole pathway will
be represented and nodes belonging to the highest-scoring subnetwork (HSS) 
will be highlighted in red.

```{r}
writeXGMML(HSS[[1]], filename = "cytoscape_input.xgmml")
```

2. Merge all the significant subnetworks and plot the resulting graph

If a list of pathways (signetList) is provided, all subnetworks with a p-value 
below a given threshold (default: 0.01) are merged and represented. Note that 
in this case, only the nodes belonging to HSS are kept for representation.

```{r}
writeXGMML(HSS, filename = "cytoscape_input.xgmml", threshold = 0.01)
```

The representation can then be finely customised in Cytoscape.
