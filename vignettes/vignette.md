treebase tutorial
==========

Here are a few introductory examples to illustrate some of the functionality of the package. Thanks in part to new data deposition requirements at journals such as Evolution, Am Nat, and Sys Bio, and
data management plan requirements from NSF, I hope the package will become increasingly useful for teaching by replicating results and for meta-analyses that can be automatically updated as the repository grows. Additional information and bug-reports welcome via the [treebase page](http://ropensci.org/packages/treebase.html#support).

Basic tree and metadata queries
==========

Downloading trees by different queries: by author, taxa, & study. More options are described in the help file.


```r
install.packages("treebase")
```



```r
library(treebase)
```

```
## Loading required package: ape
```



```r
both <- search_treebase("Ronquist or Hulesenbeck", by = c("author", "author"))
dolphins <- search_treebase("\"Delphinus\"", by = "taxon", max_trees = 5)
studies <- search_treebase("2377", by = "id.study")
```



```r
Near <- search_treebase("Near", "author", branch_lengths = TRUE, max_trees = 3)
Near[1]
```

```
## [[1]]
## 
## Phylogenetic tree with 102 tips and 21 internal nodes.
## 
## Tip labels:
## 	Etheostoma_barrenense_A, Etheostoma_rafinesquei_B, Etheostoma_atripinne_A, Etheostoma_atripinne_Y, Etheostoma_atripinne_B, Etheostoma_atripinne_C, ...
## 
## Unrooted; includes branch lengths.
```


We can query the metadata record directly. For instance, plot the growth of Treebase submissions by publication date


```r
all <- download_metadata("", by = "all")
dates <- sapply(all, function(x) as.numeric(x$date))
library(ggplot2)
qplot(dates, main = "Treebase growth", xlab = "Year", binwidth = 0.5)
```


(The previous query could also take a date range)

How do the weekly's do on submissions to Treebase? We construct this in a way that gives us back the indices of the matches, so we can then grab those trees directly. Run the scripts yourself to see if they've changed!


```r
nature <- sapply(all, function(x) length(grep("Nature", x$publisher)) > 0)
science <- sapply(all, function(x) length(grep("^Science$", x$publisher)) > 
    0)
sum(nature)
sum(science)
```


Replicating results
-------------------

A nice paper by Derryberry et al. appeared in Evolution recently on [diversification in ovenbirds and woodcreepers, 0.1111/j.1558-5646.2011.01374.x](http://www.museum.lsu.edu/brumfield/pubs/furnphylogeny2011.pdf). The article mentions that the tree is on Treebase, so let's see if we can replicate their diversification rate analysis: Let's grab the trees by that author and make sure we have the right one:


```r
tree <- search_treebase("Derryberry", "author")[[1]]
plot(tree)
```

![plot of chunk unnamed-chunk-2](figure/unnamed-chunk-2.png) 


They fit a variety of diversification rate models avialable in the `laser` R package, which they compare using AIC.


```r
library(laser)
tt <- branching.times(tree)
models <-  list(pb = pureBirth(tt),
                bdfit = bd(tt),
                y2r = yule2rate(tt), # yule model with single shift pt
                ddl = DDL(tt), # linear, diversity-dependent
                ddx = DDX(tt), #exponential diversity-dendent
                sv = fitSPVAR(tt), # vary speciation in time
                ev = fitEXVAR(tt), # vary extinction in time
                bv = fitBOTHVAR(tt)# vary both
                )
names(models[[3]])[5] <- "aic"
aics <- sapply(models, "[[", "aic")
# show the winning model
models[which.min(aics)]
```

```
## $y2r
##         LH         r1         r2        st1        aic 
##  5.051e+02  1.319e-01  3.606e-02  1.871e-01 -1.004e+03
```


Yup, their result agrees with our analysis. Using the extensive toolset for diversification rates in R, we could now rather easily check if these results hold up in newer methods such as TreePar, etc.

Meta-Analysis
-------------

Of course one of the more interesting challenges of having an automated interface is the ability to perform meta-analyses across the set of available phylogenies in treebase. As a simple proof-of-principle, let's check all the phylogenies in treebase to see if they fit a birth-death model or yule model better.

We'll create two simple functions to help with this analysis. While these can be provided by the treebase package, I've included them here to illustrate that the real flexibility comes from being able to create custom functions(These are primarily illustrative; I hope users and developers will create their own. In a proper analysis we would want a few additional checks.)


```r
timetree <- function(tree) {
    check.na <- try(sum(is.na(tree$edge.length)) > 0)
    if (is(check.na, "try-error") | check.na) 
        NULL else try(chronoMPL(multi2di(tree)))
}
drop_errors <- function(tr) {
    tt <- tr[!sapply(tr, is.null)]
    tt <- tt[!sapply(tt, function(x) is(x, "try-error"))]
    print(paste("dropped", length(tr) - length(tt), "trees"))
    tt
}
require(laser)
pick_branching_model <- function(tree) {
    m1 <- try(pureBirth(branching.times(tree)))
    m2 <- try(bd(branching.times(tree)))
    as.logical(try(m2$aic < m1$aic))
}
```


Return only treebase trees that have branch lengths. This has to download every tree in treebase, so this will take a while. Good thing we don't have to do that by hand.


```r
all <- search_treebase("Consensus", "type.tree", branch_lengths = TRUE)
tt <- drop_errors(sapply(all, timetree))
is_yule <- sapply(tt, pick_branching_model)
table(is_yule)
```

