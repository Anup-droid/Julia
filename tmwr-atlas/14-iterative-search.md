

# Iterative Search {#iterative-search}

Chapter \@ref(grid-search) demonstrated how grid search takes a pre-defined set of candidate values, evaluates them, then chooses the best settings. Iterative search methods pursue a different strategy. During the search process, they predict which values to test next. 

:::rmdnote
When grid search is infeasible or inefficient, iterative methods are a sensible approach for optimizing tuning parameters. 
:::

This chapter outlines two search methods. First, we discuss _Bayesian optimization_, which uses a statistical model to predict better parameter settings. After that, the chapter describes a global search method called _simulated annealing_. 

We use the same data on cell characteristics as the previous chapter for illustration, but change the model. This chapter uses a support vector machine model because it provides nice two-dimensional visualizations of the search processes. 

## A Support Vector Machine Model {#svm}

We once again use the cell segmentation data, described in Chapter \@ref(grid-search), for modeling, with a support vector machine (SVM) model to demonstrate sequential tuning methods. See @apm for more information on this model. The two tuning parameters to optimize are the SVM cost value and the radial basis function kernel parameter $\sigma$. Both parameters can have a profound effect on the model complexity and performance. 

The SVM model uses a dot product and, for this reason, it is necessary to center and scale the predictors. Like the multilayer perceptron model, this model would benefit from the use of PCA feature extraction. However, we will not use this third tuning parameter in this chapter so that we can visualize the search process in two dimensions.

Along with the previously used objects (shown in the summary of Chapter \@ref(grid-search)), the tidymodels objects `svm_rec`, `svm_spec`, and `svm_wflow` define the model process: 


```r
library(tidymodels)
tidymodels_prefer()

svm_rec <- 
  recipe(class ~ ., data = cells) %>%
  step_YeoJohnson(all_numeric_predictors()) %>%
  step_normalize(all_numeric_predictors())

svm_spec <- 
  svm_rbf(cost = tune(), rbf_sigma = tune()) %>% 
  set_engine("kernlab") %>% 
  set_mode("classification")

svm_wflow <- 
  workflow() %>% 
  add_model(svm_spec) %>% 
  add_recipe(svm_rec)
```

The default parameter ranges for the two tuning parameters `cost` and `rbf_sigma` are: 


```r
cost()
#> Cost (quantitative)
#> Transformer:  log-2 
#> Range (transformed scale): [-10, 5]
rbf_sigma()
#> Radial Basis Function sigma (quantitative)
#> Transformer:  log-10 
#> Range (transformed scale): [-10, 0]
```

For illustration, let's slightly change the kernel parameter range, to improve the visualizations of the search: 


```r
svm_param <- 
  svm_wflow %>% 
  extract_parameter_set_dials() %>% 
  update(rbf_sigma = rbf_sigma(c(-7, -1)))
```

Before discussing specific details about iterative search and how it works, let's explore the relationship between the two SVM tuning parameters and the area under the ROC curve for this specific data set. We constructed a very large regular grid, comprised of 2,500 candidate values, and evaluated the grid using resampling. This is obviously impractical in regular data analysis and tremendously inefficient. However, it elucidates the path that the search process should take and where the numerically optimal value(s) occur. 

Figure \@ref(fig:roc-surface) shows the results of evaluating this grid, with lighter color corresponding to higher (better) model performance. There is a large swath in the lower diagonal of the parameter space that is relatively flat with poor performance. A ridge of best performance occurs in the upper right portion of the space. The black dot indicates the best settings. The transition from the plateau of poor results to the ridge of best performance is very sharp. There is also a sharp drop in the area under the ROC curve just to the right of the ridge.

<div class="figure" style="text-align: center">
<img src="premade/roc_surface.png" alt="A heatmap of the mean area under the ROC curve for a high density grid of tuning parameter values. The best point is a solid dot in the upper right corner. The surface has a ridge of high performance moving to the lower right. " width="80%" />
<p class="caption">(\#fig:roc-surface)Heatmap of the mean area under the ROC curve for a high density grid of tuning parameter values. The best point is a solid dot in the upper right corner. </p>
</div>

The following search procedures require at least some resampled performance statistics before proceeding. For this purpose, the following code creates a small regular grid that resides in the flat portion of the parameter space. The `tune_grid()` function resamples this grid:  


```r
set.seed(1401)
start_grid <- 
  svm_param %>% 
  update(
    cost = cost(c(-6, 1)),
    rbf_sigma = rbf_sigma(c(-6, -4))
  ) %>% 
  grid_regular(levels = 2)

set.seed(1402)
svm_initial <- 
  svm_wflow %>% 
  tune_grid(resamples = cell_folds, grid = start_grid, metrics = roc_res)

collect_metrics(svm_initial)
#> # A tibble: 4 × 8
#>     cost rbf_sigma .metric .estimator  mean     n std_err .config             
#>    <dbl>     <dbl> <chr>   <chr>      <dbl> <int>   <dbl> <chr>               
#> 1 0.0156  0.000001 roc_auc binary     0.864    10 0.00864 Preprocessor1_Model1
#> 2 2       0.000001 roc_auc binary     0.863    10 0.00867 Preprocessor1_Model2
#> 3 0.0156  0.0001   roc_auc binary     0.863    10 0.00862 Preprocessor1_Model3
#> 4 2       0.0001   roc_auc binary     0.866    10 0.00855 Preprocessor1_Model4
```

This initial grid shows fairly equivalent results, with no individual point much better than any of the others. These results can be ingested by the iterative tuning functions we discuss in the following sections to be used as initial values. 

## Bayesian Optimization

Bayesian optimization techniques analyze the current resampling results and create a predictive model to suggest tuning parameter values that have yet to be evaluated. The suggested parameter combination is then resampled. These results are then used in another predictive model that recommends more candidate values for testing, and so on. The process proceeds for a set number of iterations or until no further improvements occur. @Shahriari and @frazier2018tutorial are good introductions to Bayesian optimization. 

When using Bayesian optimization, the primary concerns are how to create the model and how to select parameters recommended by that model. First, let's consider the technique most commonly used for Bayesian optimization, the Gaussian process model. 

### A Gaussian process model

Gaussian process (GP) [@SCHULZ20181] models are well-known statistical techniques that have a history in spatial statistics (under the name of _kriging methods_). They can be derived in multiple ways, including as a Bayesian model; see @RaWi06 for an excellent reference. 

Mathematically, a GP is a collection of random variables whose joint probability distribution is  multivariate Gaussian. In the  context of our application, this collection is the collection of performance metrics for the tuning parameter candidate values. For the previous initial grid of four samples, the realization of these four random variables were 0.8639, 0.8625, 0.8627, and 0.8659. These are assumed to be distributed as multivariate Gaussian. The inputs that  define the independent variables/predictors for the GP model are the corresponding tuning parameter values (shown in Table \@ref(tab:initial-gp-data)).


Table: (\#tab:initial-gp-data)Resampling statistics used as the initial substrate to the Gaussian process model.

|ROC    |cost    |rbf_sigma |
|:------|:-------|:---------|
|0.8639 |0.01562 |0.000001  |
|0.8625 |2.00000 |0.000001  |
|0.8627 |0.01562 |0.000100  |
|0.8659 |2.00000 |0.000100  |

Gaussian process models are specified by their mean and covariance functions, although the latter has the most effect on the nature of the GP model. The covariance function is often parameterized in terms of the input values (denoted as $x$). As an example, a commonly used covariance function is the squared exponential^[This equation is also the same as the _radial basis function_ used in kernel methods, such as the SVM model that is currently being used. This is a coincidence; this covariance function is unrelated to the SVM tuning parameter that we are using. ] function: 

$$\operatorname{cov}(\boldsymbol{x}_i, \boldsymbol{x}_j) = \exp\left(-\frac{1}{2}|\boldsymbol{x}_i - \boldsymbol{x}_j|^2\right) + \sigma^2_{ij}$$
where $\sigma^2_{ij}$ is a constant error variance term that is zero when $i=j$. This equation translates to: 

> As the distance between two tuning parameter combinations increases, the covariance between the performance metrics increase exponentially. 

The nature of the equation also implies that the variation of the outcome metric is minimized at the points that have already been observed (i.e., when $|\boldsymbol{x}_i - \boldsymbol{x}_j|^2$ is zero). 

The nature of this covariance function allows the Gaussian process to represent highly nonlinear relationships between model performance and the tuning parameters even when only a small amount of data exists. 

:::rmdwarning
However, fitting these models can be difficult in some cases and the model becomes more computationally expensive as the number of tuning parameter combinations increases. 
:::

An important virtue of this model is that, since a full probability model is specified, the predictions for new inputs can reflect the entire distribution of the outcome. In other words, new performance statistics can be predicted in terms of both mean and variance. 

Suppose that two new tuning parameters were under consideration. In Table \@ref(tab:tuning-candidates), candidate _A_ has a slightly better mean ROC value than candidate _B_ (the current best is 0.8659). However, its variance is four-fold larger than _B_. Is this good or bad? Choosing option _A_ is riskier but has potentially higher return. The increase in variance also reflects that this new value is further away from the existing data than _B_. The next section considers these aspects of GP predictions for Bayesian optimization in more detail.


Table: (\#tab:tuning-candidates)Two example tuning parameters considered for further sampling.

|candidate |mean |variance |
|:---------|:----|:--------|
|A         |0.90 |0.000400 |
|B         |0.89 |0.000025 |

:::rmdnote
Bayesian optimization is an iterative process. 
:::

Based on the initial grid of four results, the GP model is fit, candidates are predicted, and a fifth tuning parameter combination is selected. We compute performance estimates for the new configuration, the GP is refit with the five existing results (and so on). 

### Acquisition functions

Once the Gaussian process is fit to the current data, how is it used? Our goal is to choose the next tuning parameter combination that is most likely to have "better results" than the current best. One approach to do this is to create a large candidate set (perhaps using a space-filling design) and then make mean and variance predictions on each.  Using this information, we choose the most advantageous tuning parameter value. 

A class of objective functions, called _acquisition functions_, facilitate the trade-off between mean and variance. Recall that the predicted variance of the GP models are mostly driven by how far away they are from the existing data. The trade-off between the predicted mean and variance for new candidates is frequently viewed through the lens of exploration and exploitation: 

* _Exploration_ biases the selection towards regions where there are fewer (if any) observed candidate models. This tends to give more weight to candidates with higher variance and focuses on finding new results. 

* _Exploitation_ principally relies on the mean prediction to find the best (mean) value. It focuses on existing results. 



To demonstrate, let's look at a toy example with a single parameter that has values between [0, 1] and the performance metric is $R^2$. The true function is shown in Figure \@ref(fig:performance-profile), along with 5 candidate values that have existing results as points.

<div class="figure" style="text-align: center">
<img src="figures/performance-profile-1.png" alt="A hypothetical true performance profile over an arbitrary tuning parameter. Five estimated points are also shown. The profile is highly nonlinear with a peak in-between two of the observed points."  />
<p class="caption">(\#fig:performance-profile)Hypothetical true performance profile over an arbitrary tuning parameter, with five estimated points.</p>
</div>

For these data, the GP model fit is shown in Figure \@ref(fig:estimated-profile). The shaded region indicates the mean $\pm$ 1 standard error. The two vertical lines indicate two candidate points that are examined in more detail later.

The shaded confidence region demonstrates the squared exponential variance function; it becomes very large between points and converges to zero at the existing data points. 

<div class="figure" style="text-align: center">
<img src="figures/estimated-profile-1.png" alt="The estimated performance profile generated by the Gaussian process model. The shaded region shows one-standard error bounds. Two vertical lines show potential points to be sampled in the next iteration."  />
<p class="caption">(\#fig:estimated-profile)Estimated performance profile generated by the Gaussian process model. The shaded region shows one-standard error bounds.</p>
</div>

This nonlinear trend passes through each observed point but the model is not perfect. There are no observed points near the true optimum setting and, in this region, the fit could be much better. Despite this, the GP model can effectively point us in the right direction. 

From a pure exploitation standpoint, the best choice would select the parameter value that has the best mean prediction. Here, this would be a value of 0.106, just to the right of the existing best observed point at 0.09. 

As a way to encourage exploration, a simple (but not often used) approach is to find the tuning parameter associated with the largest confidence interval. For example, by using a single standard deviation for the $R^2$ confidence bound, the next point to sample would be 0.236. This is slightly more into the region with no observed results. Increasing the number of standard deviations used in the upper bound would push the selection further into empty regions. 

One of the most commonly used acquisition functions is _expected improvement_. The notion of improvement requires a value for the current best results (unlike the confidence bound approach). Since the GP can describe a new candidate point using a distribution, we can weight the parts of the distribution that show improvement using the probability of the improvement occurring. 

For example, consider two candidate parameter values of 0.10 and 0.25 (indicated by the vertical lines in Figure \@ref(fig:estimated-profile)). Using the fitted GP model, their predicted $R^2$ distributions are shown in Figure \@ref(fig:two-candidates) along with a reference line for the current best results.

<div class="figure" style="text-align: center">
<img src="figures/two-candidates-1.png" alt="Predicted performance distributions for two sampled tuning parameter values. For one, the distribution is slightly better than the current value with a small spread. The other parameter value is slightly worse but has a very wide spread." width="80%" />
<p class="caption">(\#fig:two-candidates)Predicted performance distributions for two sampled tuning parameter values.</p>
</div>

When only considering the mean $R^2$ prediction, a parameter value of 0.10 is the better choice (see Table \@ref(tab:two-exp-improve)). The tuning parameter recommendation for 0.25 is, on average, predicted to be worse than the current best. However, since it has higher variance, it has more overall probability area above the current best. As a result, it has a larger expected improvement of the two:


Table: (\#tab:two-exp-improve)Expected improvement for the two candidate tuning parameters.

|Parameter Value |Mean   |Std Dev   |Expected Improvment |
|:---------------|:------|:---------|:-------------------|
|0.10            |0.8679 |0.0004317 |0.000190            |
|0.25            |0.8671 |0.0039301 |0.001216            |

When expected improvement is computed across the range of the tuning parameter, the recommended point to sample is much closer to 0.25 than 0.10, as shown in Figure \@ref(fig:expected-improvement).

<div class="figure" style="text-align: center">
<img src="figures/expected-improvement-1.png" alt="The estimated performance profile generated by the Gaussian process model (top panel) and the expected improvement (bottom panel). The vertical line indicates the point of maximum improvement where estimated performance is high and the predicted variation is also large."  />
<p class="caption">(\#fig:expected-improvement)The estimated performance profile generated by the Gaussian process model (top panel) and the expected improvement (bottom panel). The vertical line indicates the point of maximum improvement.</p>
</div>


Numerous acquisition functions have been proposed and discussed; in tidymodels, expected improvement is the default. 



### The `tune_bayes()` function {#tune-bayes}

To implement iterative search via Bayesian optimization, use the `tune_bayes()` function. It has syntax that is very similar to `tune_grid()` but with several additional arguments: 

* `iter` is the maximum number of search iterations. 

* `initial` can be either an integer, an object produced using `tune_grid()`, or one of the racing functions. Using an integer specifies the size of a space-filling design that is sampled prior to the first GP model. 

* `objective` is an argument for which acquisition function should be used. The <span class="pkg">tune</span> package contains functions to pass here, such as `exp_improve()` or `conf_bound()`. 

* The `param_info` argument, in this case, specifies the range of the parameters as well as any transformations that are used. These are used to define the search space. In situations where the default parameter objects are insufficient, `param_info` is used to override the defaults. 

The `control` argument now uses the results of `control_bayes()`. Some helpful arguments there are: 

* `no_improve` is an integer that will stop the search if improved parameters are not discovered within `no_improve` iterations. 

* `uncertain` is also an integer (or `Inf`) that will take an _uncertainty sample_ if there is no improvement within `uncertain` iterations. This will select the next candidate that has large variation. It has the effect of pure exploration since it does not consider the mean prediction. 

* `verbose` is a logical that will print logging information as the search proceeds. 

Let's use the first SVM results from the beginning of this chapter as the initial substrate for the Gaussian process model. Recall that, for this application, we want to maximize the area under the ROC curve. Our code is: 


```r
ctrl <- control_bayes(verbose = TRUE)

set.seed(1403)
svm_bo <-
  svm_wflow %>%
  tune_bayes(
    resamples = cell_folds,
    metrics = roc_res,
    initial = svm_initial,
    param_info = svm_param,
    iter = 25,
    control = ctrl
  )
```




The search process starts with an initial best value of 0.8659 for the area under the ROC curve. A Gaussian process model uses these 4 statistics to create a model. The large candidate set is automatically generated and scored using the expected improvement acquisition function. The first iteration failed to improve the outcome with an ROC value of 0.86315. After fitting another Gaussian process model with the new outcome value, the second iteration also failed to yield an improvement.

The log of the first two iterations, produced by the `verbose` option, was: 


```
#> Optimizing roc_auc using the expected improvement
#> 
#> ── Iteration 1 ──────────────────────────────────────────────────────────────────────
#> 
#> i Current best:		roc_auc=0.8659 (@iter 0)
#> i Gaussian process model
#> ✓ Gaussian process model
#> i Generating 5000 candidates
#> i Predicted candidates
#> i cost=0.386, rbf_sigma=0.000266
#> i Estimating performance
#> ✓ Estimating performance
#> ⓧ Newest results:	roc_auc=0.8631 (+/-0.00866)
#> 
#> ── Iteration 2 ──────────────────────────────────────────────────────────────────────
#> 
#> i Current best:		roc_auc=0.8659 (@iter 0)
#> i Gaussian process model
#> ✓ Gaussian process model
#> i Generating 5000 candidates
#> i Predicted candidates
#> i cost=13.8, rbf_sigma=7.83e-07
#> i Estimating performance
#> ✓ Estimating performance
#> ⓧ Newest results:	roc_auc=0.8624 (+/-0.00865)
```

The search continues. There were a total of 9 improvements in the outcome along the way at iterations 3, 4, 5, 6, 8, 13, 22, 23, and 24. The best result occurred at iteration 24 with an area under the ROC curve of 0.8986. 


```
#> ── Iteration 24 ─────────────────────────────────────────────────────────────────────
#> 
#> i Current best:		roc_auc=0.8986 (@iter 23)
#> i Gaussian process model
#> ✓ Gaussian process model
#> i Generating 5000 candidates
#> i Predicted candidates
#> i cost=31.8, rbf_sigma=0.0016
#> i Estimating performance
#> ✓ Estimating performance
#> ♥ Newest results:	roc_auc=0.8986 (+/-0.00785)
```
The last step was: 


```
#> ── Iteration 25 ─────────────────────────────────────────────────────────────────────
#> 
#> i Current best:		roc_auc=0.8986 (@iter 24)
#> i Gaussian process model
#> ✓ Gaussian process model
#> i Generating 5000 candidates
#> i Predicted candidates
#> i cost=20, rbf_sigma=0.00188
#> i Estimating performance
#> ✓ Estimating performance
#> ⓧ Newest results:	roc_auc=0.8982 (+/-0.00781)
```

The functions that are used to interrogate the results are the same as those used for grid search (e.g., `collect_metrics()`, etc.). For example: 


```r
show_best(svm_bo)
#> # A tibble: 5 × 9
#>    cost rbf_sigma .metric .estimator  mean     n std_err .config .iter
#>   <dbl>     <dbl> <chr>   <chr>      <dbl> <int>   <dbl> <chr>   <int>
#> 1  31.8   0.00160 roc_auc binary     0.899    10 0.00785 Iter24     24
#> 2  30.8   0.00191 roc_auc binary     0.899    10 0.00791 Iter23     23
#> 3  31.4   0.00166 roc_auc binary     0.899    10 0.00784 Iter22     22
#> 4  31.8   0.00153 roc_auc binary     0.899    10 0.00783 Iter13     13
#> 5  30.8   0.00163 roc_auc binary     0.899    10 0.00782 Iter15     15
```

The `autoplot()` function has several options for iterative search methods. Figure \@ref(fig:progress-plot) shows how the outcome changed over the search by using `autoplot(svm_bo, type = "performance")`.

<div class="figure" style="text-align: center">
<img src="figures/progress-plot-1.png" alt="The progress of the Bayesian optimization produced when the `autoplot()` method is used with `type = 'performance'`. The plot shows the estimated performance on the y axis versus the iteration number on the x axis. Confidence intervals are shown for the points."  />
<p class="caption">(\#fig:progress-plot)The progress of the Bayesian optimization produced when the `autoplot()` method is used with `type = "performance"`.</p>
</div>

An additional type of plot uses `type = "parameters"`  which shows the parameter values over iterations.

<<<<<<< HEAD


Figure \@ref(fig:bo-surfaces) shows the surfaces of the mean, variance, and expected improvement surfaces estimated by the GP after 11 iterations. The panel on the right shows a ridge of best estimated improvement along the right side of the candidate space.

<div class="figure" style="text-align: center">
<img src="figures/bo-surfaces-1.png" alt="Heat maps of the predicted mean RMSE (left), variance of RMSE (middle), and the expected improvement (right) after 11 search iterations. The means surface correctly reflects that the best results are near the upper right of the parameter space. The variance patterns show low variance at existing parameter combinations. The expected improvement surface, at this point, is a narrow ridge going form high to low in the cost dimension along higher levels of the kernel function parameter." width="100%" />
<p class="caption">(\#fig:bo-surfaces)Heat maps of the predicted mean RMSE (left), variance of RMSE (middle), and the expected improvement (right) after 11 search iterations.</p>
</div>

Figure \@ref(fig:bo-search) shows the search process at three different points in the optimization.

<div class="figure" style="text-align: center">
<img src="figures/bo-search-1.png" alt="The Bayesian optimization search path after 1, 11, and 25 iterations. Initially the search goes in a poor direction before approaching the region of best results. By eleven iterations, the search has focused on the location of the truly optimal results and has probed more extremest directions. By the end, the search focuses on the best area or probes outlying areas, especially at the bounds of the parameter space." width="100%" />
<p class="caption">(\#fig:bo-search)The Bayesian optimization search path after 1, 11, and 25 iterations.</p>
</div>

The first five iterations initially moved in a poor direction but quickly moved closer to better results.  The middle panel shows the first eleven iterations where the process investigates the region of true optimal results with a short foray to the bottom right boundary of the candidate space. The remaining iterations shown in the panel on the left switch between the region of best results and the far borders of the search space. 

While the best tuning parameter combination is on the boundary of the parameter space, Bayesian optimization will often choose new points on other sides of the boundary. While we can adjust the ratio of exploration and exploitation, the search tends to sample boundary points early on.

:::rmdnote
If the search is seeded with an initial grid, a space-filling design would probably be a better choice than a regular design.  It samples more unique values of the parameter space and would improve the predictions of the standard deviation in the early iterations. 
:::

Finally, if the user interrupts the `tune_bayes()` computations, the function returns the current results (instead of resulting in an error).  

## Simulated Annealing 

_Simulated annealing_ (SA)  [@kirkpatrick1983optimization;@van1987simulated] is a general nonlinear search routine inspired by the process in which metal cools. It is a global search method that can effectively navigate many different types of search landscapes, including discontinuous functions. Unlike most gradient-based optimization routines, simulated annealing can reassess previous solutions. 

### Simulated annealing search process

The process of using simulated annealing starts with an initial value and embarks on a controlled random walk through the parameter space. Each new candidate parameter value is a small perturbation of the previous value that keeps the new point within a local neighborhood. 

The candidate point is resampled to obtain its corresponding performance value. If this achieves better results than the previous parameters, it is accepted as the new best and the process continues. If the results are worse than the previous value the search procedure may still use this parameter to define further steps. This depends on two factors. First, the likelihood of accepting a bad result decreases as performance becomes worse. In other words, a slightly worse result has a better chance of acceptance than one with a large drop in performance. The other factor is the number of search iterations. Simulated annealing wants to accept fewer suboptimal values as the search proceeds. From these two factors, the _acceptance probability_  for a bad result can be formalized as:

$$\operatorname{Pr}[\text{accept suboptimal parameters at iteration } i] = \exp(c\times D_i \times i)$$

where $i$ is the iteration number, $c$ is a user-specified constant, and $D_i$ is the percent difference between the old and new values (where negative values imply worse results). For a bad result, we determine the acceptance probability and compare it to a random uniform number. If the random number is greater than the probability value, the search discards the current parameters and the next iteration creates its candidate value in the neighborhood of the previous value. Otherwise, the next iteration forms the next set of parameters based on the current (suboptimal) values. 

:::rmdnote
The acceptance probabilities of simulated annealing allow the search to proceed in the wrong direction, at least for the short term, with the potential to find a much better region of the parameter space in the long run. 
:::

How are the acceptance probabilities influenced? The heatmap in Figure \@ref(fig:acceptance-prob) shows how the acceptance probability can change over iterations, performance, and the user-specified coefficient.

<div class="figure" style="text-align: center">
<img src="figures/acceptance-prob-1.png" alt="A heatmap of the simulated annealing acceptance probabilities for different coefficient values. The probabilities are affected by the both the iteration number as well as how far off the the performance is from the current best." width="80%" />
<p class="caption">(\#fig:acceptance-prob)Heatmap of the simulated annealing acceptance probabilities for different coefficient values.</p>
</div>

The user can adjust the coefficients to find a probability profile that suits their needs. In `finetune::control_sim_anneal()`, the default for this `cooling_coef` argument is 0.02. Decreasing this coefficient will encourage the search to be more forgiving of poor results.

This process continues for a set amount of iterations but can halt if no globally best results occur within a pre-determined number of iterations. However, it can be very helpful to set a _restart threshold_. If there are a string of failures, this feature revisits the last globally best parameter settings and starts anew.  

The main important detail is to define how to perturb the tuning parameters from iteration to iteration. There are a variety of methods in the literature for this. We follow the method given in @gsa called _generalized simulated annealing_. For continuous tuning parameters, we define a small radius to specify the local "neighborhood". For example, suppose there are two tuning parameters and each is bounded by zero and one. The simulated annealing process generates random values on the surrounding radius and randomly chooses one to be the current candidate value.

In our implementation, the neighborhood is determined by scaling the current candidate to be between zero and one based on the range of the parameter object, so radius values between 0.05 and 0.15 seem reasonable. For these values, the fastest that the search could go from one side of the parameter space to the other is about 10 iterations. The size of the radius controls how quickly the search explores the parameter space. In our implementation, a range of radii is specified so different magnitudes of "local" define the new candidate values. 

To illustrate, we'll use the two main <span class="pkg">glmnet</span> tuning parameters: 

* The amount of total regularization (`penalty`). The default range for this parameter is $10^{-10}$ to $10^{0}$. It is typical to use a log (base 10) transformation for this parameter.  

* The proportion of the lasso penalty (`mixture`). This is bounded at zero and one with no transformation.



The process starts with initial values of `penalty = 0.025` and `mixture = 0.050`. Using a radius that randomly fluctuates between 0.050 and 0.015, the data are appropriately scaled, random values are generated on radii around the initial point, then one is randomly chosen as the candidate. For illustration, we will assume that all candidate values are improvements. Using the new value, a set of new random neighbors are generated, one is chosen, and so on. Figure \@ref(fig:iterative-neighborhood) shows 6 iterations as the search proceeds toward the upper left corner.

<div class="figure" style="text-align: center">
<img src="figures/iterative-neighborhood-1.png" alt="An illustration of how simulated annealing determines what is the local neighborhood for two numeric tuning parameters. The clouds of points show possible next values where one would be selected at random. The candidate points are small circular clouds surrounding the current best point." width="80%" />
<p class="caption">(\#fig:iterative-neighborhood)An illustration of how simulated annealing determines what is the local neighborhood for two numeric tuning parameters. The clouds of points show possible next values where one would be selected at random. </p>
</div>

Note that, during some iterations, the candidate sets along the radius exclude points outside of the parameter boundaries. Also, our implementation biases the choice of the next tuning parameter configurations _away_ from new values that are very similar to previous configurations. 

For non-numeric parameters, we assign a probability for how often the parameter value changes. 

### The `tune_sim_anneal()` function {#tune-sim-anneal}

To implement iterative search via simulated annealing, use the `tune_sim_anneal()` function. The syntax for this function is nearly identical to `tune_bayes()`. There are no options for acquisition functions or uncertainty sampling. The `control_sim_anneal()` function has some details that define the local neighborhood and the cooling schedule:

* `no_improve`, for simulated annealing, is an integer that will stop the search if no global best or improved results are discovered within `no_improve` iterations. Accepted suboptimal or discarded parameters count as "no improvement".

* `restart` is the number of iterations with no new best results before starting from the previous best results. 

* `radius` is a numeric vector on (0, 1) that defines the minimum and maximum radius of the local neighborhood around the initial point. 

* `flip` is a probability value that defines the chances of altering the value of categorical or integer parameters. 

* `cooling_coef` is the $c$ coefficient in $\exp(c\times D_i \times i)$ that modulates how quickly the acceptance probability decreases over iterations. Larger values of `cooling_coef` decrease the probability of accepting a suboptimal parameter setting.

For the cell segmentation data, the syntax is very consistent with the previously used functions: 



```r
ctrl_sa <- control_sim_anneal(verbose = TRUE, no_improve = 10L)

set.seed(1404)
svm_sa <-
  svm_wflow %>%
  tune_sim_anneal(
    resamples = cell_folds,
    metrics = roc_res,
    initial = svm_initial,
    param_info = svm_param,
    iter = 50,
    control = ctrl_sa
  )
```





The simulated annealing process discovered new global optimums at 4 different iterations. The earliest improvement was at iteration 5 and the final optimum occured at iteration 27. The best overall results occured at iteration 27 with a mean area under the ROC curve of 0.8985 (compared to an initial best of 0.8659). There were 4 restarts at iterations 13, 21, 35, and 43 as well as 12 discarded candidates during the process.

The `verbose` option prints details of the search process. The output for the first five iterations was: 


```
#> Optimizing roc_auc
#> Initial best: 0.86594
#>  1 ◯ accept suboptimal  roc_auc=0.86351	(+/-0.008642)
#>  2 ◯ accept suboptimal  roc_auc=0.86233	(+/-0.008657)
#>  3 + better suboptimal  roc_auc=0.86233	(+/-0.008661)
#>  4 + better suboptimal  roc_auc=0.86492	(+/-0.008504)
#>  5 ♥ new best           roc_auc=0.87247	(+/-0.008232)
```

The output for last ten iterations was: 


```
#> 40 ◯ accept suboptimal  roc_auc=0.89606	(+/-0.008203)
#> 41 ─ discard suboptimal roc_auc=0.87556	(+/-0.009272)
#> 42 ─ discard suboptimal roc_auc=0.87198	(+/-0.009301)
#> 43 x restart from best  roc_auc=0.89801	(+/-0.008224)
#> 44 ◯ accept suboptimal  roc_auc=0.89006	(+/-0.008789)
#> 45 + better suboptimal  roc_auc=0.89781	(+/-0.008104)
#> 46 ◯ accept suboptimal  roc_auc=0.89563	(+/-0.008601)
#> 47 ─ discard suboptimal roc_auc=0.88527	(+/-0.008766)
#> 48 ◯ accept suboptimal  roc_auc=0.8922	(+/-0.008891)
#> 49 ─ discard suboptimal roc_auc=0.87691	(+/-0.008352)
#> 50 ◯ accept suboptimal  roc_auc=0.88803	(+/-0.008728)
```

As with the other `tune_*()` functions, the corresponding `autoplot()` function produces visual assessments of the results. Using `autoplot(svm_sa, type = "performance")` shows the performance over iterations (Figure \@ref(fig:sa-iterations)) while `autoplot(svm_sa, type = "parameters")` plots performance versus specific tuning parameter values (Figure \@ref(fig:sa-parameters)).

<div class="figure" style="text-align: center">
<img src="figures/sa-iterations-1.png" alt="The progress of the simulated annealing process shown when the `autoplot()` method is used with `type = 'performance'`. The plot shows the estimated performance on the y axis versus the iteration number on the x axis. Confidence intervals are shown for the points."  />
<p class="caption">(\#fig:sa-iterations)Progress of the simulated annealing process shown when the `autoplot()` method is used with `type = "performance"`.</p>
</div>

<div class="figure" style="text-align: center">
<img src="figures/sa-parameters-1.png" alt="A visualization of performance versus tuning parameter values when the `autoplot()` method is used with `type = 'parameters'`. The plot shows different panels for each tuning parameter in their transformed units."  />
<p class="caption">(\#fig:sa-parameters)Performance versus tuning parameter values when the `autoplot()` method is used with `type = "parameters"`.</p>
</div>

Like `tune_bayes()`, manually stopping execution will return the completed iterations. 

A visualization of the search path helps to understand where the search process did well and where it went astray. Figure \@ref(fig:sa-plot) illustrates several "phases" of the optimization; these are separated by a restart of the process at the last best results. 

<div class="figure" style="text-align: center">
<img src="figures/sa-plot-1.png" alt="A visualization of different phases of the simulated annealing search. Each portion of the search has many 'dead end paths' that either have immediate poor results or have several iterations before a restart is required. After four restarts, the search finds itself in a region of optimal results." width="90%" />
<p class="caption">(\#fig:sa-plot)A visualization of different phases of the simulated annealing search.</p>
</div>

In the first phase, the search initially finds two new global optima (shown with the solid points). From these, there are several settings that are immediately discarded (light gray lines) while others are suboptimal but acceptable. After a set number of failures, it restarts at the last solid point. The other phases show a slow improvement in global optima with many discarded settings along the way. The process eventually finds its way to the region of optimal results as it exhausts the total number of allowed iterations.

## Chapter Summary {#iterative-summary}

This chapter described two iterative search methods for optimizing tuning parameters. Bayes optimization uses a predictive model trained on existing resampling results to suggest tuning parameter values, while simulated annealing walks through the hyperparameter space to find good values. Both can be effective at finding good values alone or as a follow-up method that is used after an initial grid search to further <span class="pkg">finetune</span> performance. 


