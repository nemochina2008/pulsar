<!-- README.md is generated from README.Rmd. Please edit that file -->
<!--pandoc
s:
t: markdown_github
bibliography: inst/gstars.bib
o: README.md
-->
pulsar: Parallelized Utilities for Lambda Selection Along a Regularization path
===============================================================================

[![Build Status](https://travis-ci.org/zdk123/pulsar.svg?branch=master)](https://travis-ci.org/zdk123/pulsar)

Table of Contents
-----------------

-   [Introduction](#introduction)
-   [Installation](#installation)
-   [Basic usage](#basic-usage)
-   [Custom methods](#using-a-custom-graphical-model-method)
-   [Graphlet stability](#graphlet-stability)
-   [Batch Mode](#batch-mode)
-   [Notes](#a-few-notes-on-batch-mode-pulsar)
-   [References](#references)

Introduction
------------

Probabilistic graphical models (Lauritzen (1996)) have become an important scientific tool for finding and describing patterns in high-dimensional data. Learning a graphical model from data requires a simultaneous estimation of the graph and of the probability distribution that factorizes according to this graph. In the Gaussian case, the underlying graph is determined by the non-zero entries of the precision matrix (the inverse of the population covariance matrix). Gaussian graphical models have become popular after the advent of computationally tractable estimators, such as neighborhood selection (Meinshausen and Bühlmann (2010)) and sparse inverse covariance estimation (Banerjee, El Ghaoui, and D’Aspremont (2008), Yuan and Lin (2007)). State-of-the-art solvers are the Graphical Lasso (GLASSO) (Friedman, Hastie, and Tibshirani (2008)) and the QUadratic approximation for sparse Inverse Covariance estimation (QUIC) method (Hsieh et al. (2014)).

Any neighborhood selection and inverse covariance estimation method requires a careful calibration of a regularization parameter "λ" (lambda) because the actual model complexity is not known a priori. State-of-the- art tuning parameter calibration schemes include cross-validation, (extended) information criteria (IC) such as Akaike IC and Bayesian IC (Yuan and Lin (2007); Foygel and Drton (2010)), and the Stability Approach to Regularization Selection (StARS) (H. Liu, Roeder, and Wasserman (2010)). The StARS method is particularly appealing because it shows superior empirical performance on synthetic and real-world test cases and has a clear interpretation: StARS seeks the minimum amount of regularization that results in a sparse graph whose edge set is reproducible under random subsampling of the data at a fixed proportion β (Zhao, Liu, and Roeder (2012)). Regularization parameter selection is thus determined by the concept of stability rather than regularization strength.

Two major shortcomings in StARS are computational cost and optimal setting of beta. StARS must repeatedly solve costly global optimization problems (neighborhood or sparse inverse covariance selection) over the entire regularization path for N sets of subsamples (where the choice of N is user-defined). Also, there may be no universally optimal setting of β as edge stability is strongly influenced by the underlying unknown topology of the graph (Ravikumar et al. (2011)).

We alleviated both of these shortcomings (see Müller, Bonneau, and Kurtz (2016)). Firstly, we speed up StARS by proposing β-dependent lower and upper bounds on the regularization path from as few as N = 2 subsamples (Bounded StARS (B-StARS)). This implies that the lower part of regularization path (resulting in dense graphs and hence computationally expensive optimization) does not need to be explored for future samples without compromising selection quality. Secondly, we generalized the concept of edge stability to induced subgraph (graphlet) stability. We use the graphlet correlation distance (gcd) (Yaveroglu et al. (2014)) as a novel variability measure for small induced subgraphs across graph estimates. Requiring simultaneously edge and graphlet stability (gcd+StARS or G-StARS) leads to superior regularization parameter selection on synthetic benchmarks and real-world data.

The pulsar package comprises **p**arallelized **u**tilities for **l**ambda **s**election **a**long a **r**egularization path using the StARS methodology and our recent extensions. Pulsar includes additional options for speedups, parallelizations and complementary metrics (e.g., graphlet stability (gcd), natural connectivity, ...). This R package uses function passing to allow you to use your favorite graphical model learning method (e.g, GLASSO, neighborhood selection, QUIC, clime, etc). Since the tools are quite generic, pulsar can be used for other regularized problem formulations, e.g., for regularized regression (with the LASSO) or by using other sparsity-inducing norms (e.g., SCAD, MCP, ...)

The option to find lower/upper bounds on the StARS-selected lambda from N=2 subsamples works particularly well when the underlying target graphs are sparse or when the dimensionality is high (above 20 variables or so). The bounds greatly reduce computational burden even when running in embarrassingly parallel (batch) mode.

In batch computing systems, we use the [BatchJobs](http://cran.r-project.org/package=BatchJobs) Map/Reduce strategy (for batch computing systems such as Torque, LSF, SLURM or SGE) which can significantly reduce the computation and memory burdens for StARS. This is useful for hpc users, when the number of processors available on a multicore machine only allows modest parallelization.

Please see the paper preprint on [arXiv](http://arxiv.org/abs/1605.07072).

Installation
------------

The most recent version of pulsar is on [github](http://github.com/zdk123/pulsar) and installation requires the devtools package. For the purposes of this tutorial suggested but not-imported packages will be prompted as needed (e.g. `huge`, `orca`, ...).

``` r
library(devtools)
install_github("zdk123/pulsar")
library(pulsar)
```

Basic usage
-----------

For this readme, we will use synthetic data generated from the `huge` package.

``` r
library(huge)
set.seed(10010)
p <- 40 ; n <- 3000
dat  <- huge.generator(n, p, "hub", verbose=FALSE, v=.1, u=.5)
lmax <- getMaxCov(dat$sigmahat)
lams <- getLamPath(lmax, lmax*.05, len=40)
```

You can use the `pulsar` package to run StARS, serially, as a drop-in replacement for the `huge.select` function in the `huge` package. Pulsar differs in that we run the model selection step first and then refit using arguments stored in the original call. Remove the `seed` argument for real data (this seeds the pseudo-random number generator to fix subsampling for reproducing test code).

``` r
hugeargs <- list(lambda=lams, verbose=FALSE)
time1    <- system.time(
out.p    <- pulsar(dat$data, fun=huge, fargs=hugeargs, rep.num=20,
                   criterion='stars', seed=10010)
            )
fit.p    <- refit(out.p)
```

Inspect the output:

``` r
out.p
# Mode: serial
# Path length: 40 
# Subsamples:  20 
# Graph dim:   40 
# Criterion:
#   stars... opt: index 12, lambda 0.105
fit.p
# Pulsar-selected refit of huge 
# Path length: 40 
# Graph dim:   40 
# Criterion:
#   stars... sparsity 0.025
```

Including the lower bound option `lb.stars` and upper bound option `ub.stars` can improve runtime for same StARS result (referred to as B-StARS in Mueller et al., 2016).

``` r
time2 <- system.time(
out.b <- pulsar(dat$data, fun=huge, fargs=hugeargs, rep.num=20, criterion='stars',
                lb.stars=TRUE, ub.stars=TRUE, seed=10010)
               )
```

Compare runtimes and StARS-selected lambda index for each method.

``` r
time2[[3]] < time1[[3]]
# [1] TRUE
opt.index(out.p, 'stars') == opt.index(out.b, 'stars')
# [1] TRUE
```

Using a custom graphical model method
-------------------------------------

You can pass in an arbitrary graphical model estimation function to `fun`. The function has some requirements: the first argument must be the nxp data matrix, and one argument must be named `lambda`, which should be a decreasing numeric vector containing the lambda path. The output should be a list of adjacency matrices (which can be of sparse representation from the `Matrix` package to save memory). Here is an example from `QUIC`.

``` r
library(QUIC)
quicr <- function(data, lambda) {
    S    <- cov(data)
    est  <- QUIC::QUIC(S, rho=1, path=lambda, msg=0, tol=1e-2)
    path <-  lapply(seq(length(lambda)), function(i) {
                tmp <- est$X[,,i]; diag(tmp) <- 0
                as(tmp!=0, "lgCMatrix")
    })
    est$path <- path
    est
}
```

We can use `pulsar` with a similar call. We can also parallelize this a bit for multi-processor machines by specifying `ncores` (which wraps `mclapply` in the parallel package).

``` r
quicargs <- list(lambda=lams)
nc    <- if (.Platform$OS.type == 'unix') 2 else 1
out.q <- pulsar(dat$data, fun=quicr, fargs=quicargs, rep.num=100, criterion='stars',
                lb.stars=TRUE, ub.stars=TRUE, ncores=nc, seed=10010)
```

Graphlet stability
------------------

We can use the graphlet correlation distance (gcd) as an additional stability criterion (G-StARS). We could call `pulsar` again with a new criterion, or simply `update` the arguments for model we already used. Then, we can use our default approach for selecting the optimal index, based on the gcd+StARS criterion: choose the minimum gcd summary statistic between the upper and lower StARS bounds.

``` r
out.q2 <- update(out.q, criterion=c('stars', 'gcd'))
opt.index(out.q2, 'gcd') <- get.opt.index(out.q2, 'gcd')
fit.q2 <- refit(out.q2)
```

Compare model error by relative Hamming distances between refit adjacency matrices and the true graph and visualize the results:

``` r
plot(out.q2, scale=TRUE)
```

![plot of chunk unnamed-chunk-13](http://i.imgur.com/QY4nfZV.png)

``` r
starserr <- sum(fit.q2$refit$stars != dat$theta)/p^2
gcderr   <- sum(fit.q2$refit$gcd   != dat$theta)/p^2
gcderr < starserr
# [1] TRUE

## install.packages('network')
library(network)
truenet  <- network(dat$theta)
starsnet <- network(summary(fit.q2$refit$stars))
gcdnet   <- network(summary(fit.q2$refit$gcd))
par(mfrow=c(1,3))
coords <- plot(truenet, usearrows=FALSE, main="TRUE")
plot(starsnet, coord=coords, usearrows=FALSE, main="StARS")
plot(gcdnet, coord=coords, usearrows=FALSE, main="gcd+StARS")
```

![plot of chunk unnamed-chunk-14](http://i.imgur.com/JBEe4Z3.png)

Batch Mode
----------

For large graphs, we could reduce `pulsar` run time by running each subsampled dataset in parallel (i.e., each run as an independent job). This is a natural choice since we want to infer an independent graphical model for each subsampled dataset.

Enter the BatchJobs. This package lets us invoke the queuing system in a high performance computing (hpc) environment so that we don't have to worry about any of the job-handling procedures in R. `pulsar` has only been tested for Torque so far, but should work without too much effort for LSF, SLURM, SGE and probably others.

We also potentially gain efficiency in memory usage. Even for memory efficient representations of sparse graphs, for a lambda path of size L and for number N subsamples we must hold `L*N` `p*p`- sized adjacency matrices in memory to compute the summary statistic. BatchJobs lets us use a MapReduce strategy, so that only one `p*p` graph and one `p*p` aggregation matrix needs to be held in memory at any time. For large p, it can be more efficient to read data off the disk.

This also means we will need access to a writable directory to write intermediate files (where the BatchJobs registry is stored). These will be automatically generated R scripts, error and output files and sqlite files so that BatchJobs can keep track of everything (although a different database can be used). Please see that package's documentation for more information. By default, `pulsar` will create the registry directory under R's (platform-dependent) tmp directory but this should be overridden if these need to be retained long term (`regdir` argument to `batch.pulsar`).

For generating BatchJobs, we need a configuration file (supply a path string to `conffile` argument, a good choice is the working directory) and a template file (which is best put in the same directory as the config file). Example config (BatchJobsTorque.R) and PBS template file (simpletorque.tml) for Torque can be found in the inst/extdata/ subdirectory of this github. See the BatchJobs homepage for creating templates for other systems.

For this README I suggest using BatchJobs's convenient serial mode to get things up and running. Download the files in a browser (if https is not supported) or directly in an R session:

``` r
url <- "https://raw.githubusercontent.com/zdk123/pulsar/master/inst/extdata"
url <- paste(url, "BatchJobsSerialTest.R", sep="/") # Serial mode
download.file(url, destfile=".BatchJobs.R")
```

Since BatchJobs is not imported by `pulsar`, it needs to be loaded. Uncomment `cleanup=TRUE` to remove registry directory (useful if running through this readme multiple times).

``` r
## uncomment below if BatchJobs is not already installed
# install.packages('BatchJobs')
library(BatchJobs)
out.batch <- batch.pulsar(dat$data, fun=quicr, fargs=quicargs, rep.num=100,
                          criterion='stars', seed=10010
                          #, cleanup=TRUE
                         )
```

Check that we get the same result from batch mode pulsar:

``` r
opt.index(out.q, 'stars') == opt.index(out.batch, 'stars')
# [1] TRUE
```

It is also possible to run B-StARS in batch. The first two jobs (representing the first 2 subsamples) will run to completion before the final N-2 are run. This serializes the batch mode a bit but is, in general, faster whenever individual jobs require costly computation outside the lambda bounds, e.g., when some of the provided lambda values along the regularization path induce dense graph estimates.

To keep the initial two jobs separate from the rest, the string provided to the `init` argument ("subtwo" by default) is concatenated to the basename of `regdir`. The registry/id is returned but named `init.reg` and `init.id` from `batch.pulsar`.

``` r
out.bbatch <- update(out.batch, criterion=c('stars', 'gcd'),
                     lb.stars=TRUE, ub.stars=TRUE)
```

Check that we get the same result from bounded/batch mode pulsar:

``` r
opt.index(out.bbatch, 'stars') == opt.index(out.batch, 'stars')
# [1] TRUE
```

### Notes on batch mode

-   In real applications, on an hpc, it is important to specify the `res.list` argument, which is a **named** list of PBS resources that matches the template file. For example if using the simpletorque.tml file provided [here](https://raw.githubusercontent.com/zdk123/pulsar/master/inst/extdata/simpletorque.tml) one would provide `res.list=list(walltime="4:00:00", nodes="1", memory="1gb")` to give the PBS script 4 hours and 1GB of memory and 1 node to the resource list for `qsub`.

-   Gains in efficiency and run time (especially when paired with lower/upper bound mode) will largely depend on your hpc setup. E.g - do you have sufficient priority in the queue to run your 100 jobs in perfect parallel? Because settings vary so widely, we cannot provide support for unexpected hpc problems or make specific recommendations about requesting the appropriate resource requirements for jobs.

-   One final note: we assume that a small number of jobs could fail at random. If jobs fail to complete, a warning will be given, but `pulsar` will complete the run and summary statistics will be computed only over the successful jobs (with normalization constants appropriately adjusted). It is up to the user to re-start `pulsar` if there is a sampling-dependent reason for job failure, e.g., when an outlier data point increases computation time or graph density and insufficient resources are allocated.

References
----------

Banerjee, Onureena, Laurent El Ghaoui, and Alexandre D’Aspremont. 2008. “Model selection through sparse maximum likelihood estimation for multivariate gaussian or binary data.” *The Journal of Machine …* 9: 485–516. <http://dl.acm.org/citation.cfm?id=1390696>.

Foygel, Rina, and Mathias Drton. 2010. “Extended Bayesian Information Criteria for Gaussian Graphical Models.” *Arxiv Preprint*, 1–14. <http://arxiv.org/abs/1011.6640>.

Friedman, Jerome, Trevor Hastie, and Robert Tibshirani. 2008. “Sparse inverse covariance estimation with the graphical lasso.” *Biostatistics (Oxford, England)* 9 (3). Department of Statistics, Stanford University, CA 94305, USA.: 432–41.

Hsieh, Cho-Jui, Mátyás A Sustik, Inderjit S Dhillon, and Pradeep Ravikumar. 2014. “QUIC: Quadratic Approximation for Sparse Inverse Covariance Estimation.” *Journal of Machine Learning Research* 15: 2911–47. <http://jmlr.org/papers/v15/hsieh14a.html>.

Lauritzen, Steffen L. 1996. *Graphical models*. Oxford University Press.

Liu, Han, Kathryn Roeder, and Larry Wasserman. 2010. “Stability approach to regularization selection (stars) for high dimensional graphical models.” *Proceedings of the Twenty-Third Annual Conference on Neural Information Processing Systems (NIPS)*, 1–14. <https://papers.nips.cc/paper/3966-stability-approach-to-regularization-selection-stars-for-high-dimensional-graphical-models.pdf>.

Meinshausen, N, and P Bühlmann. 2010. “Stability selection.” *J. R. Stat. Soc. Ser. B Stat. Methodol.* 72 (4). Blackwell Publishing Ltd: 417–73. doi:[10.1111/j.1467-9868.2010.00740.x](https://doi.org/10.1111/j.1467-9868.2010.00740.x).

Müller, Christian L., Richard Bonneau, and Zachary Kurtz. 2016. “Generalized Stability Approach for Regularized Graphical Models.”

Ravikumar, Pradeep, Martin J. Wainwright, Garvesh Raskutti, and Bin Yu. 2011. “High-dimensional covariance estimation by minimizing L1 -penalized log-determinant divergence.” *Electronic Journal of Statistics* 5 (January 2010): 935–80. doi:[10.1214/11-EJS631](https://doi.org/10.1214/11-EJS631).

Yaveroglu, Ömer Nebil, Noël Malod-Dognin, Darren Davis, Zoran Levnajic, Vuk Janjic, Rasa Karapandza, Aleksandar Stojmirovic, and Nataša Pržulj. 2014. “Revealing the hidden language of complex networks.” *Scientific Reports* 4 (January): 4547. doi:[10.1038/srep04547](https://doi.org/10.1038/srep04547).

Yuan, Ming, and Yi Lin. 2007. “Model selection and estimation in the Gaussian graphical model.” *Biometrika* 94 (1): 19–35. doi:[10.1093/biomet/asm018](https://doi.org/10.1093/biomet/asm018).

Zhao, Tuo, H Liu, and Kathryn Roeder. 2012. “The huge package for high-dimensional undirected graph estimation in r.” *The Journal of Machine Learning Research* 13: 1059–62. <http://dl.acm.org/citation.cfm?id=2343681>.
