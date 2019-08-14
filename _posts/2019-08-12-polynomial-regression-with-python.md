---
layout: post
title: Polynomial Regression With Python
tags: machineLearning dataScience python
comments: true
hasMath: true
description: A basic introduction to polynomial regression using Python.
published: false
---

<div class="post-links-box">
  <span class="also-read">also read</span>
  <p><a target="_blank" href="{{ site.url }}/2019/05/05/simple-linear-regression-with-r/">Simple Linear Regression</a></p>
  <p><a target="_blank" href="{{ site.url }}/2019/06/12/multiple-linear-regression-python/">Multiple Linear Regression</a></p>
</div>


By now you may know the basics of linear regression. if not, check the above articles to get an understanding about linear
regression.

Before we talk about polynomial regression, Lets take a data set and try to apply linear regression and see how it fits.

<dataset>

Okay now we have the database, let's run simple regression on the dataset and try to plot a regression line.

<regression line with dataset>

As you can see the line doesn't fit with the data set. this line is under-fitted because the dataset is not a linear one.

Linear regression as the name implies requires the the relationship between the independent and dependent variable
to be linear. so it is not possible to use linear regression model with data sets with non linear relationships with the
dependent and independent variables.

To fix the under-fitting of, one thing we can do is to increase the complexity of the model.

we can add powers of the features as new features to generate a higher order equation

<model change>

But this is still a linear model because the coefficients associated with the features are linear. X^2 is only a feature. However the curve that we are fitting is quadratic in nature.
To convert the original features into their higher order terms we will use the PolynomialFeatures class provided by scikit-learn. Next, we train the model using Linear Regression.


