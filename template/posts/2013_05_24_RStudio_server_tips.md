% R RStudio Multicore

Develop in RStudio, run in RScript
==================================

I have been using RStudio Server for a few months now and am finding it a great tool for R development.  The web interface is superb and behaves in almost exactly the same way as the desktop version.  However, I do have one gripe which has forced me to change my working practices slightly - It is really difficult to crash out of a frozen process.  Whereas in Console R, I could just hit Control-D to get out and back to Bash, in RStudio, while you can use the escape key to terminate an operation, if you have a big process going on everything just freezes and you can't do anything.  One way to deal with this is to kill the rstudio process in another terminal, but this kills the whole session, including the editor, and you will lose anything you haven't saved in your scripts.  This problem is exacerbated when I am trying to use run parallel processes using the `Multicore` package, because it takes an age to kill all of the extra forks first.

So, now I use RStudio for development and testing and run my final analysis scripts directly using Rscript.  I have this line of code at the start of my scripts...


```r
require(multicore)
cat(sprintf("Multicore functions running on maximum %d cores",
            ifelse(length(commandArgs(trailingOnly=TRUE)), 
                   cores <- commandArgs(trailingOnly=TRUE)[1],
                   cores <- 1)))
```

```
## Multicore functions running on maximum 1 cores
```


... so when I am testing in Rstudio, cores is set to 1, but when I run as an Rscript, I can specify how many cores I want to use.  I then just add `mc.cores = cores` to all of my `mclapply` calls like this:


```r
# Example processor hungry multicore operation
mats <- mclapply(1:500, 
                 function(x) matrix(rnorm(x*x), 
                                    ncol = x) %*% matrix(rnorm(x*x), 
                                                         ncol = x), 
                 mc.cores = cores)
```


The advantage of this is that, when mc.cores are set to 1, mclapply just calls lapply which is easier to crash out of (try running the above code with cores set to more than 1 to see what I mean. It will hang for a lot longer) and produces more useful error messages:


```r
# Error handling with typos:
# mcapply:
mats <- mclapply(1:500, 
                 function(x) matrix(rnor(x*x), 
                                    ncol = x) %*% matrix(rnorm(x*x), 
                                                         ncol = x), 
                 mc.cores = 2)
```

```
## Warning: all scheduled cores encountered errors in user code
```

```r
# Falls back to lapply with 1 core:
mats <- mclapply(1:500, 
                 function(x) matrix(rnor(x*x), 
                                    ncol = x) %*% matrix(rnorm(x*x), 
                                                         ncol = x), 
                 mc.cores = 1)
```

```
## Error: could not find function "rnor"
```


Running final analysis scripts has these advantages:

* You can much more easily crash out of them if there is a problem.  
* You can run several scripts in parallel without taking up console space
* You can easily redirect the std error and std out from your program to a log file to keep an eye on its progress

Running multicore R scripts in the background with automatic logging
--------------------------------------------------------------------

If you have a bunch of R scripts that might each take hours to run, you don't want to clog up your RStudio console with them. This is a useful command to effectively run a big R analysis script in the background via Rscript.  It should work equally well for Linux and Mac:

```
$ Rscript --vanilla R/myscript.R 12 &> logs/myscript.log &
```

`Rscript` runs the .R file as a standalone script, without going into the R environment.  The --vanilla flag means that you run the script without calling in your .Rprofile (which is typically set up for interactive use) and without prompting you to save a workspace image. I always run the Rscript from the save level to that which is set as the project root for Rstudio to avoid any problems because of relative paths being set up wrongly. The number after the .R file to be run is the number of cores you want to run the parallel stuff on.  Other arguments you may want to pass to R would also go here. the `&>` operator redirects both the stdin and stderror to the file logs/myscript.log (I always set up a logs directory in my R projects for this purpose). The `&` at the end runs the process in the background, so you get your bash prompt back while it is running.  Then if you want to check the progress of your script, just type:

```
$ tail -f logs/myscript.log
```

And you can watch the output of your script in real time. Hit Control-C to get back to your command prompt.  You can also use the `top` command to keep an eye on your processor usage.

If you want to kill your script, either find the PID number associated with your process in `top` and do `kill PIDNUMBER` or, if you are lazy/carefree, type `killall R` to kill any running R processes.  This will not affect your rstudio instance.
