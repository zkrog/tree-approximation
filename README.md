# tree-approximation

When a predictive model is complex, approximating the model can be a
useful strategy to understand the model’s behavior. Using only a small
subset of predictors to describe overall model behavior is an attractive
means of “decoding” black box models such as neural networks. This is
illustrated in “Regression Modeling Strategies”<sup>\[1\]</sup> and
“Understanding neural networks using regression trees: an application to
multiple myeloma survival data”<sup>\[2\]</sup>.

As an example, sale price of homes in Ames, Iowa will be
predicted<sup>\[3\]</sup> . Tidymodels and Keras are used for modeling.

``` r
library(keras)
library(tidyverse)
library(tidymodels)
```

``` r
set.seed(1)

# initial split of data
ames_split <- initial_split(ames, prop = 0.75, strata = Sale_Price)
ames_train <- training(ames_split)
ames_test <- testing(ames_split)

# normalizing numeric X, creating dummy vars, log transforming y
ames_data_preproc <- 
  recipe(Sale_Price ~ ., data = ames_train) |> 
  step_normalize(all_numeric_predictors()) |> 
  step_dummy(all_nominal_predictors()) |> 
  step_log(Sale_Price)

# assigning training and validation rows
processed <- 
  bake(prep(ames_data_preproc), new_data = NULL) |> 
  mutate(set = sample(c('train', 'val'),
                      size = nrow(ames_train),
                      replace = TRUE,
                      prob = c(0.75, 0.25)))

# preparing data for Keras
xtrain <- 
  processed |> 
  filter(set == 'train') |> 
  select(!c(Sale_Price, set)) |> 
  as.matrix()

xval <- 
  processed |> 
  filter(set == 'val') |> 
  select(!c(Sale_Price, set)) |> 
  as.matrix()

ytrain <- 
  processed |> 
  filter(set == 'train') |> 
  pull(Sale_Price)

yval <- 
  processed |> 
  filter(set == 'val') |> 
  pull(Sale_Price)
```

Fitting a neural network to serve as a complex model.

``` r
# NN with 2 hidden layers, loss is MSE
tensorflow::set_random_seed(1)

model <- 
  keras_model_sequential() |> 
  layer_dense(64, activation = 'relu') |> 
  layer_dense(64, activation = 'relu') |> 
  layer_dense(1)

# training for 100 epochs
history <- 
  model |> 
  compile(optimizer = optimizer_rmsprop(0.00005),
          loss = 'mse') |> 
          
  fit(x = xtrain,
      y = ytrain, 
      validation_data = list(xval, yval),
      epochs = 100,
      batch_size = 32)

# trace plot
plot(history)
```

![](README_files/figure-commonmark/unnamed-chunk-3-1.png)

``` r
# model diagram
plot(model)
```

![](README_files/figure-commonmark/unnamed-chunk-3-2.png)

Taking a look at actual vs. predicted plots:

``` r
predtrain <- predict(model, xtrain)
```

    51/51 - 0s - 93ms/epoch - 2ms/step

``` r
predval <- predict(model, xval)
```

    19/19 - 0s - 19ms/epoch - 1ms/step

``` r
bind_rows(data.frame(act = ytrain, pred = predtrain,
                     set = 'train', scale = 'log'),
          data.frame(act = yval, pred = predval,
                     set = 'val', scale = 'log'),
          data.frame(act = exp(ytrain), pred = exp(predtrain),
                     set = 'train', scale = 'orig'),
          data.frame(act = exp(yval), pred = exp(predval),
                     set = 'val', scale = 'orig')) |> 

  ggplot(aes(x = pred, y = act, color = set)) +
  geom_point() +
  geom_abline(slope = 1) +
  facet_wrap(~scale, scales = 'free')
```

![](README_files/figure-commonmark/unnamed-chunk-4-1.png)

Decision tree approximation of neural network. Trees of varying depths
are fit to the neural network’s predictions.

“You need to set cp=0 in the controls the complexity of the tree. If
cp\>0 the tree won’t grown unless the increase in tree size doesn’t
improve the performance of the tree by at least cp.”
https://stats.stackexchange.com/questions/164042/setting-different-depth-in-rpart-but-didnt-actually-change

``` r
# combining training and validation data and generating predictions
xames <- rbind(xtrain, xval)
predames <- predict(model, xames)
```

    69/69 - 0s - 68ms/epoch - 979us/step

``` r
# combining processed data and predictions
xpredames <- 
  xames |> 
  as.data.frame() |> 
  mutate(pred = as.numeric(predames))
  
# specify a regression tree with varying tree depths
at_spec <- 
  decision_tree(mode = 'regression',
                cost_complexity = 0,
                tree_depth = tune(),
                min_n = 2)

at_recipe <- 
  recipe(pred ~ .,
         data = xpredames)
  
at_wf <- 
  workflow() |> 
  add_model(at_spec) |> 
  add_recipe(at_recipe)

at_grid <- 
  data.frame(tree_depth = 1:12)

# fit trees of varying complexity to the NN predictions
at_fits <-
  tune_grid(at_wf,
            grid = at_grid,
            resamples = apparent(xpredames),
            metrics = metric_set(rsq),
            control = control_grid(save_pred = TRUE))
```

Plot of decision tree complexity vs. model fit.

``` r
at_fits |> 
  collect_predictions() |> 
  
  ggplot(aes(x = log(.pred), y = log(pred))) +
  geom_point() +
  geom_abline(slope = 1, color = 'blue') +
  facet_wrap(~tree_depth)
```

![](README_files/figure-commonmark/unnamed-chunk-6-1.png)

Plot of R2 vs. tree depth. A decision tree with a depth of 3 can
describe 60% of the variation of the neural network’s output.

``` r
at_fits |> 
  collect_metrics() |> 
  
  ggplot(aes(x = tree_depth, y = mean)) +
  geom_point()
```

![](README_files/figure-commonmark/unnamed-chunk-7-1.png)

Taking a look at this tree, some of the trends of the neural network’s
behavior can be interpreted. Although the price predictions are on the
log scale and the variables are normalized, the directionality of the
most influential variables appear near the root of the tree. For
example:

- newer homes tend to be more expensive
- having central air is more expensive
- bigger garages are more expensive

``` r
at <- 
  at_fits |> 
  collect_metrics() |> 
  filter(tree_depth == 3) |> 
  finalize_workflow(x = at_wf) |> 
  fit(xpredames) |> 
  extract_fit_engine()

at |> 
  rpart.plot::rpart.plot(type = 4,
                         extra = 0,
                         clip.right.labs = FALSE, 
                         box.palette = "BlGnYl",
                         branch = .3,
                         digits = -4)
```

![](README_files/figure-commonmark/unnamed-chunk-8-1.png)

References:

- \[1\] https://hbiostat.org/rmsc/validate#sec-val-approx
- \[2\] https://pubmed.ncbi.nlm.nih.gov/11568952/
- \[3\] https://jse.amstat.org/v19n3/decock.pdf
