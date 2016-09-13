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

Future is an R package by [Henrik Bengtsson](https://twitter.com/henrikbengtsson), available on [CRAN](https://cran.r-project.org/web/packages/future/index.html).


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
[1] 0.7654462 1.8105259 0.7110902 0.1387135
> 
> Sys.time() - t0
Time difference of 3.010155 secs
```

`%<-%` is non blocking (2/2):
========================================================
Without `%<-%`:

```r
> t0 <- Sys.time()
> a <- slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 3.008813 secs
> 
> a
[1] -0.9071859 -0.5388912
```
***
With `%<-%`:

```r
> t0 <- Sys.time()
> b %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.01002908 secs
> 
> b
[1] -0.2908547  0.1969033
```


`%<-%` is **not** magic:
========================================================
It create a *future*, a variable that will be available in the future.
The task is **not** magicaly optimised, it runs in the background.
When the variable is needed, the R (main) process will wait until the *future* is resolved.

```r
> t0 <- Sys.time()
> b %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.007998943 secs
> 
> b
[1] -0.001421231  1.341619561
> 
> Sys.time() - t0
Time difference of 3.012523 secs
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
Time difference of 6.010204 secs
> 
> list(x1, x2)
[[1]]
[1] 0.01463919 2.02325234

[[2]]
[1] -0.8652523 -0.3555410
> 
> Sys.time() - t0
Time difference of 6.017205 secs
```
***
Assignment with future:

```r
> t0 <- Sys.time()
> x3 %<-% slow_rnorm(2)
> x4 %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.0485909 secs
> 
> list(x3, x4)
[[1]]
[1] -0.1183447 -0.9877792

[[2]]
[1]  0.7939648 -0.9046369
> 
> Sys.time() - t0
Time difference of 3.059174 secs
```

*future* allow easy parallelisation of heterogenous tasks (1/3)
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
  myMean    mySd    myRnorm   myRunif
1    2.5 2.12132 -1.7981085 0.3523992
2    3.5 2.12132 -2.1243283 0.3108422
3    4.5 2.12132 -0.7201249 0.7135452
> 
> Sys.time() - t0
Time difference of 12.03579 secs
```

*future* allow easy parallelisation of heterogenous tasks (2/3)
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
  myMean    mySd   myRnorm   myRunif
1    2.5 2.12132 1.0606803 0.6484825
2    3.5 2.12132 0.4654688 0.5385649
3    4.5 2.12132 0.7534253 0.4539509
> 
> Sys.time() - t0
Time difference of 3.114923 secs
```

*future* allow easy parallelisation of heterogenous tasks (3/3)
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
  myMean    mySd     myRnorm   myRunif
1    2.5 2.12132  0.83677806 0.8214132
2    3.5 2.12132 -1.70406217 0.2800778
3    4.5 2.12132 -0.06966156 0.6290161
> 
> Sys.time() - t0
Time difference of 12.0199 secs
```

A *future_mclapply* draft function (1/2)
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

A *future_mclapply* draft function (1/2)
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
[1] 3.141593
> 
> Sys.time() - t0
Time difference of 3.033974 secs
```

One package, many plans
========================================================
* `plan(multiprocess)`: parallel, non blocking
* `plan(eager)`: non parallel, blocking (default)
* `plan(lazy)`: non parallel, non blocking
* `plan(cluster, workers = c("n1", "n2", "n3"))`: run in a cluster, non blocking
* see also [future.BatchJobs](https://cran.r-project.org/web/packages/future.BatchJobs/index.html) for more cluster plans.

Limiting the number of cores in `plan(multiprocess)`
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
> 
> Sys.time() - t0
Time difference of 0.7439382 secs
> 
> c(x1, x2, x3)
[1]  1.2880858  0.3412981 -0.3605666
> 
> Sys.time() - t0
Time difference of 3.753162 secs
```
***

```r
> 
> 
> 
> 
> plan(multiprocess(workers = 1 + 2)) # 1 main  + 3 background
> t0 <- Sys.time()
> x1 %<-% slow_rnorm(1)
> x2 %<-% slow_rnorm(1)
> x3 %<-% slow_rnorm(1) # no more background process available, blocks the main process
> 
> Sys.time() - t0
Time difference of 3.530033 secs
> 
> c(x1, x2, x3)
[1]  0.9291106 -1.1379026 -1.0312053
> 
> Sys.time() - t0
Time difference of 6.537139 secs
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
* No easy way to stop an unresolved future at the moment. (killing the process?)

Links
========================================================
* The package: [(https://cran.r-project.org/web/packages/future/index.html)](https://cran.r-project.org/web/packages/future/index.html)
* The comprehensive vignette:[https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html](https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html)















