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
[1]  0.242154279  1.012279949 -1.338579594  0.003491644
> 
> Sys.time() - t0
Time difference of 3.012811 secs
```

`%<-%` is non blocking (2/2):
========================================================
Without `%<-%`:

```r
> t0 <- Sys.time()
> a <- slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 3.008517 secs
> 
> a
[1] -0.05181082 -0.66309328
```
***
With `%<-%`:

```r
> t0 <- Sys.time()
> b %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.009992123 secs
> 
> b
[1] -0.3755324  0.6766176
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
Time difference of 0.006513119 secs
> 
> b
[1]  0.7877939 -0.6519630
> 
> Sys.time() - t0
Time difference of 3.012473 secs
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
Time difference of 6.019479 secs
> 
> list(x1, x2)
[[1]]
[1] -1.605941 -2.310211

[[2]]
[1] 0.6198926 1.4842387
> 
> Sys.time() - t0
Time difference of 6.026482 secs
```
***
Assignment with future:

```r
> t0 <- Sys.time()
> x3 %<-% slow_rnorm(2)
> x4 %<-% slow_rnorm(2)
> 
> Sys.time() - t0
Time difference of 0.05050611 secs
> 
> list(x3, x4)
[[1]]
[1] 1.3221904 0.0371681

[[2]]
[1]  0.3116354 -1.8248445
> 
> Sys.time() - t0
Time difference of 3.061399 secs
```

*future* allows easy parallelisation of heterogenous tasks (1/3)
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
1    2.5 2.12132 -0.6964756 0.8142671
2    3.5 2.12132  1.9235974 0.3604851
3    4.5 2.12132 -0.8084506 0.8266054
> 
> Sys.time() - t0
Time difference of 12.03554 secs
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
  myMean    mySd      myRnorm   myRunif
1    2.5 2.12132  0.005661826 0.1218853
2    3.5 2.12132 -0.354909042 0.3147438
3    4.5 2.12132  0.375833827 0.1155356
> 
> Sys.time() - t0
Time difference of 3.115655 secs
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
  myMean    mySd    myRnorm   myRunif
1    2.5 2.12132 -2.6652296 0.9493826
2    3.5 2.12132  0.3308004 0.1741118
3    4.5 2.12132  1.9009600 0.4970288
> 
> Sys.time() - t0
Time difference of 12.02489 secs
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

A *future_mclapply* draft function (2/2)
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
Time difference of 3.02435 secs
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
Time difference of 0.7332718 secs
> 
> c(x1, x2, x3)
[1]  0.1950557  0.9782678 -0.9215404
> 
> Sys.time() - t0
Time difference of 3.744617 secs
```
***

```r
> 
> 
> 
> 
> plan(multiprocess(workers = 1 + 2)) # 1 main  + 2 background
> t0 <- Sys.time()
> x1 %<-% slow_rnorm(1)
> x2 %<-% slow_rnorm(1)
> x3 %<-% slow_rnorm(1) # no more background process available, blocks the main process
> 
> Sys.time() - t0
Time difference of 3.485722 secs
> 
> c(x1, x2, x3)
[1] -0.4970261  0.6928258 -0.3332621
> 
> Sys.time() - t0
Time difference of 6.490151 secs
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















