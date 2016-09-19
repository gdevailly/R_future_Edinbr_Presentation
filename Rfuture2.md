R future
========================================================
css: bootstrap.min.css
width: 1440
height: 900

## Non blocking, parallel assignment in R

[EdinbR](http://edinbr.org/), 21/09/2016

Guillaume Devailly, [@G_Devailly](https://twitter.com/G_Devailly)

![The Roslin Institute](The_Roslin_Institute_logo.gif)



R future
========================================================

Future is an R package by [Henrik Bengtsson](https://twitter.com/henrikbengtsson), available on [CRAN](https://cran.r-project.org/package=future).


```r
> install.packages("future")
> library(future)
> plan(multiprocess) # more on that later
```

It introduces yet another assignment operator: `%<-%`

`%<-%` is yet another assignment operator:
========================================================


```r
> a %<-% 42
> a
[1] 42
> b %<-% c(rep(a, 4), 43)
> b
[1] 42 42 42 42 43
```

`%<-%` is non blocking (1/2):
========================================================
First, let's create slow functions:

```r
> slow <- function(myFunc, by_seconds = 3) {
+     return(
+         function(...) {
+             Sys.sleep(by_seconds)
+             myFunc(...)
+         }
+     )
+ }
> 
> slow_rnorm <- slow(rnorm)
> t0 <- Sys.time()
> 
> slow_rnorm(4)
[1]  1.706 -1.586  0.776 -0.139
> 
> Sys.time() - t0
Time difference of 3.01 secs
```

`%<-%` is non blocking (2/2):
========================================================
Without `%<-%`:

```r
> t0 <- Sys.time()
> a <- slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 3.01 secs
> 
> a
[1] -0.0302  0.6574
```
***
With `%<-%`:

```r
> t0 <- Sys.time()
> b %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.016 secs
> 
> b
[1] 0.935 1.681
```


`%<-%` is **not** magic:
========================================================
It create a *future*, a variable that will be available in the future.
The task is **not** magically optimised, it runs in the background.
When the variable is needed, the R (main) process will wait until the *future* is resolved.

```r
> t0 <- Sys.time()
> b %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.00661 secs
> 
> b
[1] -0.115  0.987
> 
> Sys.time() - t0
Time difference of 3.01 secs
```

*futures* can be run in parallel:
========================================================
Standard assignment:

```r
> t0 <- Sys.time()
> x1 <- slow_rnorm(2)
> x2 <- slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 6.01 secs
> 
> list(x1, x2)
[[1]]
[1] 0.0642 1.5622

[[2]]
[1]  2.133 -0.063
> 
> Sys.time() - t0
Time difference of 6.02 secs
```
***
Assignment with future:

```r
> t0 <- Sys.time()
> x3 %<-% slow_rnorm(2)
> x4 %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.0495 secs
> 
> list(x3, x4)
[[1]]
[1]  1.0860 -0.0742

[[2]]
[1] -0.514 -0.370
> 
> Sys.time() - t0
Time difference of 3.06 secs
```

*future* allows easy parallelisation of heterogeneous tasks (1/3)
========================================================

```r
> myMat <- matrix(1:6, ncol = 2)
> slow_apply <- slow(apply)
```

Standard assignment:

```r
> t0 <- Sys.time()
> myMean  <- slow_apply(myMat, 1, mean)
> mySd    <- slow_apply(myMat, 1, sd)
> myRnorm <- slow_rnorm(3)
> myRunif <- slow(runif)(3)
> 
> data.frame(myMean, mySd, myRnorm, myRunif)
  myMean mySd myRnorm myRunif
1    2.5 2.12  -0.964   0.724
2    3.5 2.12  -1.853   0.911
3    4.5 2.12  -0.466   0.559
> 
> Sys.time() - t0
Time difference of 12 secs
```

*future* allows easy parallelisation of heterogenous tasks (2/3)
========================================================
Parallelization with *future*:

```r
> t0 <- Sys.time()
> myMean  %<-% slow_apply(myMat, 1, mean)
> mySd    %<-% slow_apply(myMat, 1, sd)
> myRnorm %<-% slow_rnorm(3)
> myRunif %<-% slow(runif)(3)
> 
> data.frame(myMean, mySd, myRnorm, myRunif)
  myMean mySd myRnorm myRunif
1    2.5 2.12   -1.30   0.811
2    3.5 2.12    1.30   0.511
3    4.5 2.12    1.13   0.433
> 
> Sys.time() - t0
Time difference of 3.11 secs
```

*future* allows easy parallelisation of heterogenous tasks (3/3)
========================================================
Parallelization with parallel:

```r
> library(parallel)
> t0 <- Sys.time()
> myCommands <- c(
+     myMean  = "slow_apply(myMat, 1, mean)",
+     mySd    = "slow_apply(myMat, 1, sd)",
+     myRnorm = "slow_rnorm(3)",
+     myRunif = "slow(runif)(3)"
+ )
> 
> as.data.frame(mclapply( # not parallel on windows, but should work elsewhere
+     myCommands,
+     function(x) eval(parse(text = x))
+     # mc.cores = 4L <- add this on non windows OS
+ ))
  myMean mySd myRnorm myRunif
1    2.5 2.12  -2.648   0.963
2    3.5 2.12   1.488   0.127
3    4.5 2.12  -0.553   0.141
> 
> Sys.time() - t0
Time difference of 12 secs
```

A *future_mclapply* draft function (1/3)
========================================================

```r
> future_mclapply <- function(myList, myFunction) {
+     if(is.vector(myList)) myList <- as.list(myList) # the function will work on vectors to
+     for(i in seq_along(myList)) {
+         command <- paste0(
+             "x",
+             i,
+             " %<-% do.call(",
+             deparse(substitute(myFunction)),
+             ",",
+             deparse(substitute(myList[i])),
+             ")"
+         )
+         eval(parse(text = command))
+     }
+     outputVars <- paste(
+         paste0("x", seq_along(myList)),
+         collapse = ", "
+     )
+     output <- eval(parse(text = paste0("list(", outputVars, ")")))
+     names(output) <- names(myList)
+     return(output)
+ }
```

A *future_mclapply* draft function (2/3)
========================================================

```r
> t0 <- Sys.time()
> 
> future_mclapply(
+     list(1:5, 6:10, pi),
+     slow(mean)
+ )
[[1]]
[1] 3

[[2]]
[1] 8

[[3]]
[1] 3.14
> 
> Sys.time() - t0
Time difference of 3.05 secs
```

A *future_mclapply* function (3/3)
========================================================
Or the hidden function (may change / disappear in future version of the package):

```r
> t0 <- Sys.time()
> 
> future:::flapply(
+     list(1:5, 6:10, pi),
+     slow(mean)
+ )
[[1]]
[1] 3

[[2]]
[1] 8

[[3]]
[1] 3.14
> 
> Sys.time() - t0
Time difference of 3.04 secs
```

One package, many plans
========================================================
* `plan(multiprocess)`: parallel, non blocking
* `plan(eager)`: non parallel, blocking (default)
* `plan(lazy)`: non parallel, non blocking
* `plan(cluster, workers = c("n1", "n2", "n3"))`: run in a cluster, non blocking. Nice gestion of the global variables.
* see also [future.BatchJobs](https://cran.r-project.org/web/packages/future.BatchJobs/index.html) for more cluster plans.
Write package using *future*, users will choose how to run it by changing the `plan`.


Limiting the number of cores in `plan(multiprocess)` (1/2)
========================================================

```r
> availableCores()
system 
     8 
> 
> plan(multiprocess(workers = 1 + 3)) # 1 main  + 3 background
> t0 <- Sys.time()
> x1 %<-% slow_rnorm(1)
> x2 %<-% slow_rnorm(1)
> x3 %<-% slow_rnorm(1)
> 
> 
> Sys.time() - t0
Time difference of 0.728 secs
> 
> c(x1, x2, x3)
[1] -1.021  0.130 -0.263
> 
> Sys.time() - t0
Time difference of 3.74 secs
```
Limiting the number of cores in `plan(multiprocess)` (2/2)
========================================================

```r
> plan(multiprocess(workers = 1 + 2)) # 1 main  + 2 background
> t0 <- Sys.time()
> x1 %<-% slow_rnorm(1)
> x2 %<-% slow_rnorm(1)
> x3 %<-% slow_rnorm(1) # no more background process available, blocks the main process
> 
> Sys.time() - t0
Time difference of 3.49 secs
> 
> c(x1, x2, x3)
[1] -1.816  0.287  1.931
> 
> Sys.time() - t0
Time difference of 6.5 secs
```


Miscellaneous
========================================================
* Nested features are supported, see this [vignette](https://cran.r-project.org/web/packages/future/vignettes/future-3-topologies.html)
* Check if a future is resolved:

```r
> x1 %<-% slow_rnorm(3)
> f <- futureOf(x1)
> resolved(f)
[1] FALSE
```
* No easy way to stop an unresolved future at the moment (killing the process?)

Links
========================================================
* The package: [(https://cran.r-project.org/web/packages/future/index.html)](https://cran.r-project.org/web/packages/future/index.html)
* The comprehensive vignette:[https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html](https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html)
* [slides from useR2016](http://www.jottr.org/2016/07/a-future-for-r-slides-from-user-2016.html) by Henrik Bengtsson.
* [`doFuture` package](https://cran.r-project.org/package=doFuture): alternative to `doMC`, `doParallel`, `doMPI`, and `doSNOW`.















