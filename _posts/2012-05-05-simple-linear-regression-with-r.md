---
layout: post
title: Simple Linear Regression With R
comments: true
hasMath: true
---

Regression models are used to predict real values such as salary. Simple linear regression is model of regression which is used to identify the correlation between two variables and possibly predict the dependent variable by using the independent variable.

This will enable us to establish a relationship between two attributes such as Income and Spending and we can use what we know about the relationship to forecast unobserved values. 


There should be two variables when it comes to simple linear regression. which are Dependent variable and Independent variable

| Dependent Variable                    | Independent Variable             |
|---------------------------------------|----------------------------------|
| This is the value we want to forecast | This explains the other variable |
| Value depends on another variable     | Independent of other values      |
| Usually denoted as $$y$$              | Usually denotes as $$x$$         |


In simple linear regression, we use the equation,

$$y = \beta_{0} + \beta_{1}x$$

In which $$\beta_{0}$$ is a constant (y intercept) and $$\beta_{1}$$ is the coefficient or the slope of $$x$$

We call this a linear equation as in will represent a straight line if we were to plot this in a bi-dimensional plain.

So if you were to plot $$y=4+2x$$,

<img src="{{ site.url }}/public/images/2012-05-05-simple-linear-regression-with-r/sample-plot-1.png" style="width:500px">
