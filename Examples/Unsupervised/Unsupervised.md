
# Using vtreat with Unsupervised Problems and Non-Y-aware data treatment

## Preliminaries

Nina Zumel and John Mount
September 2019

Note: this is a description of the [`Python` version of `vtreat`](https://github.com/WinVector/pyvtreat), the same example for the [`R` version of `vtreat`](https://github.com/WinVector/vtreat) can be found [here](https://github.com/WinVector/vtreat/blob/master/Examples/Unsupervised/Unsupervised.md).

Load modules/packages.


```python
import pkg_resources
import pandas
import numpy
import numpy.random
import seaborn
import matplotlib.pyplot as plt
import vtreat
import vtreat.util
import wvpy.util
```

Generate example data. 

* `y` is a noisy sinusoidal plus linear function of the variable `x`
* Input `xc` is a categorical variable that represents a discretization of `y`, along with some `NaN`s
* Input `x2` is a pure noise variable with no relationship to the output
* Input `x3` is a constant variable


```python
def make_data(nrows):
    d = pandas.DataFrame({'x':[0.1*i for i in range(500)]})
    d['y'] = numpy.sin(d['x']) + 0.01*d['x'] +  0.1*numpy.random.normal(size=d.shape[0])
    d['xc'] = ['level_' + str(5*numpy.round(yi/5, 1)) for yi in d['y']]
    d['x2'] = numpy.random.normal(size=d.shape[0])
    d['x3'] = 1
    d.loc[d['xc']=='level_-1.0', 'xc'] = numpy.nan # introduce a nan level
    return d

d = make_data(500)

d.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>x</th>
      <th>y</th>
      <th>xc</th>
      <th>x2</th>
      <th>x3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.0</td>
      <td>-0.104856</td>
      <td>level_-0.0</td>
      <td>0.367093</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.1</td>
      <td>0.148332</td>
      <td>level_0.0</td>
      <td>-0.690937</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.2</td>
      <td>0.280450</td>
      <td>level_0.5</td>
      <td>-0.256927</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.3</td>
      <td>0.504849</td>
      <td>level_0.5</td>
      <td>0.188184</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.4</td>
      <td>0.358950</td>
      <td>level_0.5</td>
      <td>0.619167</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>



### Some quick data exploration

Check how many levels `xc` has, and their distribution (including `NaN`)


```python
d['xc'].unique()
```




    array(['level_-0.0', 'level_0.0', 'level_0.5', 'level_1.0', 'level_-0.5',
           nan, 'level_1.5'], dtype=object)




```python
d['xc'].value_counts(dropna=False)
```




    level_-0.5    128
    level_1.0     122
    level_0.5      98
    level_0.0      41
    level_-0.0     40
    NaN            36
    level_1.5      35
    Name: xc, dtype: int64



## Build a transform appropriate for unsupervised (or non-y-aware) problems.

The `vtreat` package is primarily intended for data treatment prior to supervised learning, as detailed in the [Classification](https://github.com/WinVector/pyvtreat/blob/master/Examples/Classification/Classification.ipynb) and [Regression](https://github.com/WinVector/pyvtreat/blob/master/Examples/Regression/Regression.ipynb) examples. In these situations, `vtreat` specifically uses the relationship between the inputs and the outcomes in the training data to create certain types of synthetic variables. We call these more complex synthetic variables *y-aware variables*. 

However, you may also want to use `vtreat` for basic data treatment for unsupervised problems, when there is no outcome variable. Or, you may not want to create any y-aware variables when preparing the data for supervised modeling. For these applications, `vtreat` is a convenient alternative to: `pandas.get_dummies()` or `sklearn.preprocessing.OneHotEncoder()`.

In any case, we still want training data where all the input variables are numeric and have no missing values or `NaN`s.

First create the data treatment transform object, in this case a treatment for an unsupervised problem.


```python
transform = vtreat.UnsupervisedTreatment(
     cols_to_copy = ['y'],          # columns to "carry along" but not treat as input variables
)  
```

Use the training data `d` to fit the transform and the return a treated training set: completely numeric, with no missing values.


```python
d_prepared = transform.fit_transform(d)
```

Now examine the score frame, which gives information about each new variable, including its type and which original variable it is  derived from. Some of the columns of the score frame (`y_aware`, `PearsonR`, `significance` and `recommended`) are not relevant to the unsupervised case; those columns are used by the Regression and Classification transforms.


```python
transform.score_frame_
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
      <th>orig_variable</th>
      <th>treatment</th>
      <th>y_aware</th>
      <th>has_range</th>
      <th>PearsonR</th>
      <th>significance</th>
      <th>recommended</th>
      <th>vcount</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>xc_is_bad</td>
      <td>xc</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>x</td>
      <td>x</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x2</td>
      <td>x2</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>xc_prevalence_code</td>
      <td>xc</td>
      <td>prevalence_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_lev_level_-0.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_lev_level_1.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>xc_lev_level_0.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>xc_lev_level_0.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>xc_lev_level_-0.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>xc_lev__NA_</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>xc_lev_level_1.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
  </tbody>
</table>
</div>



Notice that the variable `xc` has been converted to multiple variables: 

* an indicator variable for each possible level, including `NA` or missing (`xc_lev_level_*`)
* a variable indicating when `xc` was `NaN` in the original data (`xc_is_bad`)
* a variable that returns how prevalent this particular value of `xc` is in the training data (`xc_prevalence_code`)

Any or all of these new variables are available for downstream modeling.

Also note that the variable `x3` did not show up in the score frame, as it had no range (didn't vary), so the unsupervised treatment dropped it.

Let's look at the top of `d_prepared`, which includes all the new variables, plus `y` (and excluding `x3`).


```python
d_prepared.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>y</th>
      <th>xc_is_bad</th>
      <th>x</th>
      <th>x2</th>
      <th>xc_prevalence_code</th>
      <th>xc_lev_level_-0.5</th>
      <th>xc_lev_level_1.0</th>
      <th>xc_lev_level_0.5</th>
      <th>xc_lev_level_0.0</th>
      <th>xc_lev_level_-0.0</th>
      <th>xc_lev__NA_</th>
      <th>xc_lev_level_1.5</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-0.104856</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.367093</td>
      <td>0.080</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.148332</td>
      <td>0.0</td>
      <td>0.1</td>
      <td>-0.690937</td>
      <td>0.082</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.280450</td>
      <td>0.0</td>
      <td>0.2</td>
      <td>-0.256927</td>
      <td>0.196</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.504849</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>0.188184</td>
      <td>0.196</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.358950</td>
      <td>0.0</td>
      <td>0.4</td>
      <td>0.619167</td>
      <td>0.196</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>



## Using the Prepared Data to Model

Of course, what we really want to do with the prepared training data is to model. 

### K-means clustering

Let's start with an unsupervised analysis: clustering.


```python
# don't use y to cluster
not_variables = ['y']
model_vars = [v for v in d_prepared.columns if v not in set(not_variables)]

import sklearn.cluster

d_prepared['clusterID'] = sklearn.cluster.KMeans(n_clusters = 5).fit_predict(d_prepared[model_vars])
d_prepared.clusterID

# colorbrewer Dark2 palette
mypalette = ['#1b9e77', '#d95f02', '#7570b3', '#e7298a', '#66a61e']
ax = seaborn.scatterplot(x = "x", y = "y", hue="clusterID", 
                    data = d_prepared, 
                    palette=mypalette, 
                    legend=False)
ax.set_title("y as a function of x, points colored by (unsupervised) clusterID")
plt.show()
```


![png](output_19_0.png)


### Supervised modeling with non-y-aware variables

Since in this case we have an outcome variable, `y`, we can try fitting a linear regression model to `d_prepared`.


```python
import sklearn.linear_model
import seaborn
import sklearn.metrics
import matplotlib.pyplot

not_variables = ['y', 'prediction', 'clusterID']
model_vars = [v for v in d_prepared.columns if v not in set(not_variables)]
fitter = sklearn.linear_model.LinearRegression()
fitter.fit(d_prepared[model_vars], d_prepared['y'])
print(fitter.intercept_)
{model_vars[i]: fitter.coef_[i] for i in range(len(model_vars))}
```

    0.26687361134312826





    {'xc_is_bad': -0.562356677841655,
     'x': 0.0012639910538561326,
     'x2': 0.001609517233713642,
     'xc_prevalence_code': -0.0044801038835988755,
     'xc_lev_level_-0.5': -0.8087257353814689,
     'xc_lev_level_1.0': 0.7247440547156317,
     'xc_lev_level_0.5': 0.2180424429835109,
     'xc_lev_level_0.0': -0.17839488726269243,
     'xc_lev_level_-0.0': -0.43704784362522275,
     'xc_lev__NA_': -0.5623566778416543,
     'xc_lev_level_1.5': 1.0437386464118965}




```python
# now predict
d_prepared['prediction'] = fitter.predict(d_prepared[model_vars])

# get R-squared
r2 = sklearn.metrics.r2_score(y_true=d_prepared.y, y_pred=d_prepared.prediction)

title = 'Prediction vs. outcome (training data); R-sqr = {:04.2f}'.format(r2)

# compare the predictions to the outcome (on the training data)
ax = seaborn.scatterplot(x='prediction', y='y', data=d_prepared)
matplotlib.pyplot.plot(d_prepared.prediction, d_prepared.prediction, color="darkgray")
ax.set_title(title)
plt.show()
```


![png](output_22_0.png)


Now apply the model to new data.


```python
# create the new data
dtest = make_data(450)

# prepare the new data with vtreat
dtest_prepared = transform.transform(dtest)

# apply the model to the prepared data
dtest_prepared['prediction'] = fitter.predict(dtest_prepared[model_vars])

# get R-squared
r2 = sklearn.metrics.r2_score(y_true=dtest_prepared.y, y_pred=dtest_prepared.prediction)

title = 'Prediction vs. outcome (test data); R-sqr = {:04.2f}'.format(r2)

# compare the predictions to the outcome (on the training data)
ax = seaborn.scatterplot(x='prediction', y='y', data=dtest_prepared)
matplotlib.pyplot.plot(dtest_prepared.prediction, dtest_prepared.prediction, color="darkgray")
ax.set_title(title)
plt.show()
```


![png](output_24_0.png)


## Parameters for `UnsupervisedTreatment`

We've tried to set the defaults for all parameters so that `vtreat` is usable out of the box for most applications. Notice that the parameter object for unsupervised treatment defines a different set of parameters than the parameter object for supervised treatments (`vtreat.vtreat_parameters`).



```python
vtreat.unsupervised_parameters()
```




    {'coders': {'clean_copy',
      'indicator_code',
      'missing_indicator',
      'prevalence_code'},
     'indicator_min_fraction': 0.0,
     'user_transforms': [],
     'sparse_indicators': True}



**coders**: The types of synthetic variables that `vtreat` will (potentially) produce. See *Types of prepared variables* below.

**indicator_min_fraction**: By default, `UnsupervisedTreatment` creates indicators for all possible levels (`indicator_min_fraction=0`). If `indicator_min_fraction` > 0, then indicator variables (type `indicator_code`) are only produced for levels that are present at least `indicator_min_fraction` of the time. A consequence of this is that 1/`indicator_min_fraction` is the maximum number of indicators that will be produced for a given categorical variable. See the Example below.

**user_transforms**: For passing in user-defined transforms for custom data preparation. Won't be needed in most situations, but see [here](https://github.com/WinVector/pyvtreat/blob/master/Examples/UserCoders/UserCoders.ipynb) for an example of applying a GAM transform to input variables.

**sparse_indicators**: When True, use a (Pandas) sparse representation for indicator variables. This representation is compatible with `sklearn`; however, it may not be compatible with other modeling packages. When False, use a dense representation.

### Example: Restrict the number of indicator variables


```python
# calculate the prevalence of each level by hand
d['xc'].value_counts(dropna=False)/d.shape[0]
```




    level_-0.5    0.256
    level_1.0     0.244
    level_0.5     0.196
    level_0.0     0.082
    level_-0.0    0.080
    NaN           0.072
    level_1.5     0.070
    Name: xc, dtype: float64




```python
transform_common = vtreat.UnsupervisedTreatment(
    cols_to_copy = ['y'],          # columns to "carry along" but not treat as input variables
    params = vtreat.unsupervised_parameters({
        'indicator_min_fraction': 0.2 # only make indicators for levels that show up more than 20% of the time
    })
)  

transform_common.fit_transform(d) # fit the transform
transform_common.score_frame_     # examine the score frame
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
      <th>orig_variable</th>
      <th>treatment</th>
      <th>y_aware</th>
      <th>has_range</th>
      <th>PearsonR</th>
      <th>significance</th>
      <th>recommended</th>
      <th>vcount</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>xc_is_bad</td>
      <td>xc</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>x</td>
      <td>x</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x2</td>
      <td>x2</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>xc_prevalence_code</td>
      <td>xc</td>
      <td>prevalence_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_lev_level_-0.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_lev_level_1.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
  </tbody>
</table>
</div>



In this case, the unsupervised treatment only created levels for the two most common levels, which are both present more than 20% of the time. 

In unsupervised situations, this may only be desirable when there are an unworkably large number of possible levels (for example, when using ZIP code as a variable). It is more useful in conjunction with the y-aware variables produced by `NumericOutputTreatment`, `BinomialOutcomeTreatment`, or `MultinomialOutcomeTreatment`.

## Types of prepared variables

**clean_copy**: Produced from numerical variables: a clean numerical variable with no `NaNs` or missing values

**indicator_code**: Produced from categorical variables, one for each level: for each level of the variable, indicates if that level was "on"

**prevalence_code**: Produced from categorical variables: indicates how often each level of the variable was "on"

**missing_indicator**: Produced for both numerical and categorical variables: an indicator variable that marks when the original variable was missing or  `NaN`

### Example: Produce only a subset of variable types

In this example, suppose you only want to use indicators and continuous variables in your model; 
in other words, you only want to use variables of types (`clean_copy`, `missing_indicator`, and `indicator_code`), and no `prevalence_code` variables.


```python
transform_thin = vtreat.UnsupervisedTreatment(
    cols_to_copy = ['y'],          # columns to "carry along" but not treat as input variables
    params = vtreat.unsupervised_parameters({
         'coders': {'clean_copy',
                    'missing_indicator',
                    'indicator_code',
                   }
    })
)  

transform_thin.fit_transform(d) # fit the transform
transform_thin.score_frame_

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
      <th>orig_variable</th>
      <th>treatment</th>
      <th>y_aware</th>
      <th>has_range</th>
      <th>PearsonR</th>
      <th>significance</th>
      <th>recommended</th>
      <th>vcount</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>xc_is_bad</td>
      <td>xc</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>x</td>
      <td>x</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x2</td>
      <td>x2</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>xc_lev_level_-0.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_lev_level_1.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_lev_level_0.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>xc_lev_level_0.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>xc_lev_level_-0.0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>xc_lev__NA_</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>xc_lev_level_1.5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>True</td>
      <td>7.0</td>
    </tr>
  </tbody>
</table>
</div>



## Conclusion

In all cases (classification, regression, unsupervised, and multinomial classification) the intent is that `vtreat` transforms are essentially one liners.

The preparation commands are organized as follows:

 * **Regression**: [`R` regression example](https://github.com/WinVector/vtreat/blob/master/Examples/Regression/Regression.md), [`Python` regression example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Regression/Regression.md).
 * **Classification**: [`R` classification example](https://github.com/WinVector/vtreat/blob/master/Examples/Classification/Classification.md), [`Python` classification  example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Classification/Classification.md).
 * **Unsupervised tasks**: [`R` unsupervised example](https://github.com/WinVector/vtreat/blob/master/Examples/Unsupervised/Unsupervised.md), [`Python` unsupervised example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Unsupervised/Unsupervised.md).
 * **Multinomial classification**: [`R` multinomial classification example](https://github.com/WinVector/vtreat/blob/master/Examples/Multinomial/MultinomialExample.md), [`Python` multinomial classification example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Multinomial/MultinomialExample.md).

These current revisions of the examples are designed to be small, yet complete.  So as a set they have some overlap, but the user can rely mostly on a single example for a single task type.


