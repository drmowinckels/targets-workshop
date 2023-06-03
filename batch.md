---
title: 'Batch and Parallel Processing'
teaching: 10
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- How can we specify many targets without typing everything out?
- How can we build targets in parallel?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Be able to specify targets using branching
- Be able to build targets in parallel

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: instructor

Episode summary: Show how to use branching and parallel processing (technically separate topics, but they go well together)

:::::::::::::::::::::::::::::::::::::



## Why batch and parallel processing?

One of the major strengths of `targets` is the ability to define many targets from a single line of code (batch processing).
This not only saves you typing, it also **reduces the risk of errors** since there is less chance of making a typo.
Furthermore, it is related to another powerful feature: `targets` can run multiple analyses in parallel (at the same time), thereby **making your analysis finish sooner**.

## Types of branching

Batching in `targets` is called "branching."
There are two types of branching, **dynamic branching** and **static branching**.
"Branching" refers to the idea that you can provide a single specification for how to make targets (the "pattern"), and `targets` generates multiple targets from it ("branches").
"Dynamic" means that the branches that result from the pattern do not have to be defined ahead of time---they are a dynamic result of the code.

In this workshop, we will only cover dynamic branching since it is generally easier to write (static branching requires use of [meta-programming](https://books.ropensci.org/targets/static.html#metaprogramming), an advanced topic). For more information about each and when you might want to use one or the other (or some combination of the two), [see the `targets` package manual](https://books.ropensci.org/targets/dynamic.html).

## Example without branching

To see how this works, let's continue our analysis of the `palmerpenguins` dataset.

**Our hypothesis is that bill depth decreases with bill length.**
We will test this hypothesis with a linear model.

For example, this is a model of bill depth dependent on bill length:


```r
lm(bill_depth_mm ~ bill_length_mm, data = penguin_data)
```

We can add this to our pipeline. We will call it the `combined_model` because it combines all the species together without distinction:


```r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build model
  combined_model = lm(
    bill_depth_mm ~ bill_length_mm, data = penguins_data)
)
```


```{.output}
✔ skip target penguins_data_raw_file
✔ skip target penguins_data_raw
✔ skip target penguins_data
• start target combined_model
• built target combined_model [0.027 seconds]
• end pipeline [0.117 seconds]
```

Let's have a look at the model. We will use the `glance()` function from the `broom` package. Unlike base R `summary()`, this function returns output as a tibble (the tidyverse equivalent of a dataframe), which as we will see later is quite useful for downstream analyses.


```r
tar_load(combined_model)
broom::glance(combined_model)
```


```{.output}
# A tibble: 1 × 12
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1    0.0552        0.0525  1.92      19.9 0.0000112     1  -708. 1422. 1433.    1256.         340   342
```

Notice the small *P*-value.
This seems to indicate that the model is highly significant.

But wait a moment... is this really an appropriate model? Recall that there are three species of penguins in the dataset. It is possible that the relationship between bill depth and length **varies by species**.

We should probably test some alternative models.
These could include models that add a parameter for species, or add an interaction effect between species and bill length.

Now our workflow is getting more complicated. This is what a workflow for such an analysis might look like **without branching** (make sure to add `library(broom)` to `packages.R`):


```r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build models
  combined_model = lm(
    bill_depth_mm ~ bill_length_mm, data = penguins_data),
  species_model = lm(
    bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
  interaction_model = lm(
    bill_depth_mm ~ bill_length_mm * species, data = penguins_data),
  # Get model summaries
  combined_summary = glance(combined_model),
  species_summary = glance(species_model),
  interaction_summary = glance(interaction_model)
)
```


```{.output}
✔ skip target penguins_data_raw_file
✔ skip target penguins_data_raw
✔ skip target penguins_data
✔ skip target combined_model
• start target interaction_model
• built target interaction_model [0.006 seconds]
• start target species_model
• built target species_model [0.001 seconds]
• start target combined_summary
• built target combined_summary [0.05 seconds]
• start target interaction_summary
• built target interaction_summary [0.003 seconds]
• start target species_summary
• built target species_summary [0.003 seconds]
• end pipeline [0.182 seconds]
```

Let's look at the summary of one of the models:


```r
tar_read(species_summary)
```


```{.output}
# A tibble: 1 × 12
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1     0.769         0.767 0.953      375. 3.65e-107     3  -467.  944.  963.     307.         338   342
```

So this way of writing the pipeline works, but is repetitive: we have to call `glance()` each time we want to obtain summary statistics for each model.
Furthermore, each summary target (`combined_summary`, etc.) is explicitly named and typed out manually.
It would be fairly easy to make a typo and end up with the wrong model being summarized.

## Example with branching

### First attempt

Let's see how to write the same plan using **dynamic branching**:


```r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build models
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
  # Get model summaries
  tar_target(
    model_summaries,
    glance(models[[1]]),
    pattern = map(models)
  )
)
```

What is going on here?

First, let's look at the messages provided by `tar_make()`.


```{.output}
✔ skip target penguins_data_raw_file
✔ skip target penguins_data_raw
✔ skip target penguins_data
• start target models
• built target models [0.008 seconds]
• start branch model_summaries_5ad4cec5
• built branch model_summaries_5ad4cec5 [0.009 seconds]
• start branch model_summaries_c73912d5
• built branch model_summaries_c73912d5 [0.046 seconds]
• start branch model_summaries_91696941
• built branch model_summaries_91696941 [0.003 seconds]
• built pattern model_summaries
• end pipeline [0.189 seconds]
```

There is a series of smaller targets (branches) that are each named like model_summaries_5ad4cec5, then one overall `model_summaries` target.
That is the result of specifying targets using branching: each of the smaller targets are the "branches" that comprise the overall target.
Since `targets` has no way of knowing ahead of time how many branches there will be or what they represent, it names each one using this series of numbers and letters (the "hash").
`targets` builds each branch one at a time, then combines them into the overall target.

Next, let's look in more detail about how the workflow is set up, starting with how we defined the models:


```r
# Build models
models <- list(
  combined_model = lm(
    bill_depth_mm ~ bill_length_mm, data = penguins_data),
  species_model = lm(
    bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
  interaction_model = lm(
    bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
)
```

Unlike the non-branching version, we defined the models **in a list** (instead of one target per model).
This is because dynamic branching is similar to the `base::apply()` or [`purrrr::map()`](https://purrr.tidyverse.org/reference/map.html) method of looping: it applies a function to each element of a list.
So we need to prepare the input for looping as a list.

Next, take a look at the command to build the target `model_summaries`.


```r
# Get model summaries
tar_target(
  model_summaries,
  glance(models[[1]]),
  pattern = map(models)
)
```

As before, the first argument is the name of the target to build, and the second is the command to build it.

Here, we apply the `glance()` function to each element of `models` (the `[[1]]` is necessary because when the function gets applied, each element is actually a nested list, and we need to remove one layer of nesting).

Finally, there is an argument we haven't seen before, `pattern`, which indicates that this target should be built using dynamic branching.
`map` means to apply the command to each element of the input list (`models`) sequentially.

Now that we understand how the branching workflow is constructed, let's inspect the output:


```r
tar_read(model_summaries)
```


```{.output}
# A tibble: 3 × 12
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1    0.0552        0.0525 1.92       19.9 1.12e-  5     1  -708. 1422. 1433.    1256.         340   342
2    0.769         0.767  0.953     375.  3.65e-107     3  -467.  944.  963.     307.         338   342
3    0.770         0.766  0.955     225.  8.52e-105     5  -466.  947.  974.     306.         336   342
```

The model summary statistics are all included in a single dataframe.

But there's one problem: **we can't tell which row came from which model!** It would be unwise to assume that they are in the same order as the list of models.

This is due to the way dynamic branching works: by default, there is no information about the provenance of each target preserved in the output.

How can we fix this?

### Second attempt

The key to obtaining useful output from branching pipelines is to include the necessary information in the output of each individual branch.
Here, we want to know the kind of model that corresponds to each row of the model summaries.
To do that, we need to write a **custom function**.
You will need to write custom functions frequently when using `targets`, so it's good to get used to it!

Here is the function. Save this in `R/functions.R`:


```r
glance_with_mod_name <- function(model_in_list) {
  model_name <- names(model_in_list)
  model <- model_in_list[[1]]
  broom::glance(model) %>%
    mutate(model_name = model_name)
}
```

Our new pipeline looks almost the same as before, but this time we use the custom function instead of `broom::glance()`.


```r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build models
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
  # Get model summaries
  tar_target(
    model_summaries,
    glance_with_mod_name(models),
    pattern = map(models)
  )
)
```


```{.output}
✔ skip target penguins_data_raw_file
✔ skip target penguins_data_raw
✔ skip target penguins_data
✔ skip target models
• start branch model_summaries_5ad4cec5
• built branch model_summaries_5ad4cec5 [0.017 seconds]
• start branch model_summaries_c73912d5
• built branch model_summaries_c73912d5 [0.009 seconds]
• start branch model_summaries_91696941
• built branch model_summaries_91696941 [0.005 seconds]
• built pattern model_summaries
• end pipeline [0.193 seconds]
```

And this time, when we load the `model_summaries`, we can tell which model corresponds to which row (you may need to scroll to the right to see it).


```{.output}
# A tibble: 3 × 13
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs model_name       
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int> <chr>            
1    0.0552        0.0525 1.92       19.9 1.12e-  5     1  -708. 1422. 1433.    1256.         340   342 combined_model   
2    0.769         0.767  0.953     375.  3.65e-107     3  -467.  944.  963.     307.         338   342 species_model    
3    0.770         0.766  0.955     225.  8.52e-105     5  -466.  947.  974.     306.         336   342 interaction_model
```

::::::::::::::::::::::::::::::::::::: {.callout}

## Best practices for branching

Dynamic branching is designed to work well with **dataframes** (tibbles).

So if possible, write your custom functions to accept dataframes as input and return them as output, and always include any necessary metadata as a column or columns.

:::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: {.challenge}

## Challenge: What other kinds of patterns are there?

So far, we have only used a single function in conjunction with the `pattern` argument, `map()`, which applies the function to each element of its input in sequence.

Can you think of any other ways you might want to apply a branching pattern?

:::::::::::::::::::::::::::::::::: {.solution}

Some other ways of applying branching patterns include:

- crossing: one branch per combination of elements (`cross()` function)
- slicing: one branch for each of a manually selected set of elements (`slice()` function)
- sampling: one branch for each of a randomly selected set of elements (`sample()` function)

You can [find out more about different branching patterns in the `targets` manual](https://books.ropensci.org/targets/dynamic.html#patterns).

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

## Parallel processing

Once a pipeline starts to include many targets, you may want to think about parallel processing.
This takes advantage of multiple processors in your computer to build multiple targets at the same time.

::::::::::::::::::::::::::::::::::::: {.callout}

## When to use parallel computing

Parallel computing should only be used if your workflow has a structure such that it makes sense---if your workflow only consists of a linear sequence of targets, then there is nothing to parallelize.

:::::::::::::::::::::::::::::::::::::

`targets` includes support for high-performance computing, cloud computing, and various parallel backends.
Here, we assume you are running this analysis on a laptop and so will use a relatively simple backend.
If you are interested in high-performance computing, [see the `targets` manual](https://books.ropensci.org/targets/hpc.html).

### Install R packages for parallel computing

For this demo, we will use the [`future` backend](https://github.com/HenrikBengtsson/future).

::::::::::::::::::::::::::::::::::::: {.prereq}

### Install required packages

You will need to install several packages to use the `future` backend:


```r
install.packages("future")
install.packages("future.batchtools")
install.packages("future.callr")
```

:::::::::::::::::::::::::::::::::::::

### Set up workflow

There are a few things you need to change to enable parallel processing with `future`:

- Load the `future` and `future.callr` packages
- Add a line with `plan(callr)`
- When you run the pipeline, use `tar_make_future(workers = 2)` instead of `tar_make()`

Here, `workers = 2` is the number of processes to run in parallel. You may increase this up to the number of cores available on your machine.

To show how this works we will simulate a long(ish) running analysis with the `Sys.sleep()` function, which just tells the computer to wait some number of seconds.


```r
library(targets)
library(tarchetypes)
library(future)
library(future.callr)

plan(callr)

long_square <- function(data) {
  Sys.sleep(3)
  data^2
}

tar_plan(
  some_data = c(1, 2, 3, 4),
  tar_target(
    data_squared,
    long_square(some_data),
    pattern = map(some_data)
  )
)
```

Here is the output when running with `tar_make_future(workers = 2)`:


```{.output}
• start target some_data
• built target some_data [0.164 seconds]
• start branch data_squared_3ba31302
• start branch data_squared_880e1e2e
• built branch data_squared_3ba31302 [3.168 seconds]
• start branch data_squared_552eb2cc
• built branch data_squared_880e1e2e [3.167 seconds]
• start branch data_squared_92b840e1
• built branch data_squared_552eb2cc [3.173 seconds]
• built branch data_squared_92b840e1 [3.17 seconds]
• built pattern data_squared
• end pipeline [10.187 seconds]
```

Notice that although the time required to build each individual target is about 3 seconds, the total time to run the entire workflow is less than the sum of the individual target times! That is proof that processes are running in parallel **and saving you time**.

The unique and powerful thing about targets is that **we did not need to change our custom function to run it in parallel**. We only adjusted *the workflow*. This means it is relatively easy to refactor (modify) a workflow for running sequentially locally or running in parallel in a high-performance context.

We can see this by applying parallel processing to the penguins analysis.
Add two more packages to `packages.R`, `library(future)` and `library(future.callr)`.
Also, add the line `plan(callr)` to `_targets.R`:


```r
source("R/packages.R")
source("R/functions.R")

plan(callr)

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build models
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
  # Get model summaries
  tar_target(
    model_summaries,
    glance_with_mod_name(models),
    pattern = map(models)
  )
)
```

Finally, run the pipeline with `tar_make_future()` (you may need to run `tar_invalidate(everything())` to reset the pipeline first).


```{.output}
• start target penguins_data_raw_file
• built target penguins_data_raw_file [0.745 seconds]
• start target penguins_data_raw
• built target penguins_data_raw [0.906 seconds]
• start target penguins_data
• built target penguins_data [0.753 seconds]
• start target models
• built target models [0.751 seconds]
• start branch model_summaries_5ad4cec5
• start branch model_summaries_c73912d5
• built branch model_summaries_5ad4cec5 [0.795 seconds]
• start branch model_summaries_91696941
• built branch model_summaries_c73912d5 [0.798 seconds]
• built branch model_summaries_91696941 [0.76 seconds]
• built pattern model_summaries
• end pipeline [11.812 seconds]
```

You won't notice much difference since these computations run so quickly, but this demonstrates how easy it is to make massive gains in efficiency with your own real analysis by using parallel computing.

::::::::::::::::::::::::::::::::::::: keypoints 

- Dynamic branching creates multiple targets with a single command
- You usually need to write custom functions so that the output of the branches includes necessary metadata 
- Parallel computing works at the level of the workflow, not the function

::::::::::::::::::::::::::::::::::::::::::::::::