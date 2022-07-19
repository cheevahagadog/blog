---
toc: false
layout: post
description: Discovering which features were the most influential
categories: [machine learning]
image: images/shap_example.png
title: Finding out which features contributed to each row's prediction
---

Almost completed with a machine learning project, I was asked by the client if I could include what the reasons where for each prediction. I had done feature importances before, but never was I asked to (or thought of) listing out what the reasons where behind each prediction made by my model.

Here, I’m going to explain how I approached that problem and what worked, and what I had to do.

## Requirements

I recommend using a virtual environment. Here are the packages and the versions I’m using:

- xgboost==0.7 (If `pip install` doesn’t work for you, [these instructions](https://pypi.python.org/pypi/xgboost/) worked the best for me)
- iml==0.4.0
- scikit-learn==0.19.1
- shap==0.11.1
- pandas==0.21.1

## Background
I had developed a customer churn prediction model using XGBoost 0.6. When they requested the prediction breakdown for each row, I searched the XGBoost [documentation](http://xgboost.readthedocs.io/en/latest/python/python_api.html#xgboost.Booster.predict), I found that there was a parameter I could call called `pred_contribs` in the `predict` method. When called, it returned a matrix per each row and could be used for calculating the feature contributions that effected the prediction.

However, while running `xgboost==0.6` the `pred_contribs` wasn’t showing up as an option. After updating to version `0.7` it was available.

## Example from the SHAP package
Some searching led me to the amazing [shap](https://github.com/slundberg/shap) package which helps make machine learning models more visible, even at the row level. Here’s the example posted on their README:

```python
import xgboost
import shap

# load JS visualization code to notebook
shap.initjs() 

# train XGBoost model
X,y = shap.datasets.boston()
bst = xgboost.train({"learning_rate": 0.1}, xgboost.DMatrix(X, label=y), 100)

# explain the model's predictions using SHAP values (use pred_contrib in LightGBM)
shap_values = bst.predict(xgboost.DMatrix(X), pred_contribs=True)

# visualize the first prediction's explaination
shap.force_plot(shap_values[0,:], X.iloc[0,:])
```

Which yields this pretty visualization:
![]({{ site.baseurl }}/image/shap_example.png "SHAP viz")

Awesome! This looked perfect for me…except my output needed to be in a CSV format. I didn’t need the pretty visualizations, I just needed something that an analyst could read and use if they needed to make a phone call to a potentially churning client and say “Hi I am calling because it looks like you haven’t had any X in over 6 months” or something to that effect. Something like this:

![]({{ site.baseurl }}/image/xgboost_factors_row_breakdown.png "SHAP viz")


## Problem
I inspected the `shap_values` to realize it was a numpy matrix but it wasn’t obvious how to use it to get the effects of the features let alone know what the features were. And, for all its visual brilliance, the shap package unfortunately didn’t seem to have the ability to fetch the row-level prediction info into something more maleable, like a pandas DataFrame as shown above.

_NOTE: Definitely checkout the library it has some amazing features that can help make your models more interpretable._

## Solution

I attacked the problem by piecing apart the function that delivered the visualization I wanted in a pandas format, the `force_plot` function. I was hoping to catch some kind of data structure right before it was passed into a graphing function, and I was mostly right, but there were still some hurdles. My function below takes in the same parameters as the `force_plot` function from the SHAP package. I tweaked it to output the top five effects on the prediction, as seen below.

EDIT: it now takes the number of effects as an argument as well as if you want to sort the effects by absolute value or not.

```python
def calculate_top_contributors(shap_values, features=None, feature_names=None, use_abs=False, 
    return_df=False, n_features=5):
    """ Adapted from the SHAP package for visualizing the contributions of features towards a prediction.
        https://github.com/slundberg/shap
        
    Args:
        shap_values: np.array of floats
        features: pandas.core.series.Series, the data with the values
        feature_names: list, all the feature names/ column names
        use_abs: bool, if True, will sort the data by the absolute value of the feature effect
        return_df: bool, if True, will return a pandas dataframe, else will return a list of feature, effect, value
        n_features: int, the number of features to report on. If it equals -1 it will return the entire dataframe

    Returns:
        if return_df is True: returns a pandas dataframe
        if return_df is False: returns a flattened list by name, effect, and value
    """
    assert not type(shap_values) == list, "The shap_values arg looks looks multi output, try shap_values[i]."
    assert len(shap_values.shape) == 1, "Expected just one row. Please only submit one row at a time."

    shap_values = np.reshape(shap_values, (1, len(shap_values)))
    instance = iml.Instance(np.zeros((1, len(feature_names))), features)
    link = iml.links.convert_to_link('identity')

    # explanation obj
    expl = iml.explanations.AdditiveExplanation(
        shap_values[0, -1],                 # base value
        np.sum(shap_values[0, :]),          # this row's prediction value
        shap_values[0, :-1],                # matrix
        None,
        instance,                           # <iml.common.Instance object >
        link,                               # 'identity'
        iml.Model(None, ["output value"]),  # <iml.common.Model object >
        iml.datatypes.DenseData(np.zeros((1, len(feature_names))), list(feature_names))
    )

    # Get the name, effect and value for each feature, if there was an effect
    features_ = {}
    for i in range(len(expl.data.group_names)):
        if expl.effects[i] != 0:
            features_[i] = {
                "effect": ensure_not_numpy(expl.effects[i]),
                "value": ensure_not_numpy(expl.instance.group_display_values[i]),
                "name": expl.data.group_names[i]
            }

    effect_df = pd.DataFrame([v for k, v in features_.items()])

    if use_abs:  # get the absolute value of effect
        effect_df['abs_effect'] = effect_df['effect'].apply(np.abs)
        effect_df.sort_values('abs_effect', ascending=False, inplace=True)
    else:
        effect_df.sort_values('effect', ascending=False, inplace=True)

    if not n_features == -1:
        effect_df = effect_df.head(n_features)
    if return_df:
        return effect_df.reset_index(drop=True)
    else:
        list_of_info = list(zip(effect_df.name, effect_df.effect, effect_df.value))
        effect_list = list(sum(list_of_info, ()))  # flattens the list of tuples
        return effect_list
```

This function is designed to be called for one row at a time and can be looped over to create the explanations for every prediction (see the [full code example](https://gist.github.com/cheevahagadog/756e88fd241cee5e053e170ea7eab1e5)). Here it would return a list for the first row.

```python
first_row_effects_list = calculate_top_contributors(shap_values=contribs[0, :], 
		                                features=final_eval.iloc[0, :],
                                        feature_names=clf.feature_names)
```

## Conclusion
The [shap](https://github.com/slundberg/shap) package should be in your toolbox if you are developing models with XGBoost. It is the perfect companion for a predictive power of the algorithm in delivering stunning and precise visualzations the make your work more transparent. If, like me, you need the information this package offers but in a more ‘dataish’ format, I have a simple function that you can use for each row prediction or call multiple times and create a separate dataset to join back to your original. The possibilities are endless! The full code sample is [here](https://gist.github.com/cheevahagadog/756e88fd241cee5e053e170ea7eab1e5).

Thanks for reading!

