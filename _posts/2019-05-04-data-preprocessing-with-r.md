---
layout: post
title: Data Pre-Processing With R
tags: r machineLearning dataScience
comments: true
---

Before feeding our dataSet into a machine learning algorithm it's absolutely necessary to pre-process the data where we should clean and re-shape our data to get the maximum performance from our machine learning models. 

In this post, I will go through a set of procedures which you can use to pre-process a data set.

you can download the data set that I am going to use in this post from  [Here](https://docs.google.com/spreadsheets/d/15tssQQrC9AnnqUqeHj46eoXCixCvFMhhgf3NcFAyqrY/edit?usp=sharing).
Download the file as `dataSet.csv` to a folder and set it as the working directory in your R editor

The data in this set are completely random and the data set has no real value. as we are only using it in pre-processing, that should not be a problem.


### Loading the Data

Okay, now we have the data file, now we need to read it in order to work with it.

```R
# This will load the SCV file into a data frame
data <- read.csv("dataSet.csv")
```

Now we have the data set loaded into a variable we can go ahead with our pre-processing steps.

## Dealing with Missing Data



As you can see in the data set, it has four cells which don't have any values in `Age` and `Salary` variables.

In both cases, we can replace the `NA` values with the mean of the variable.

to do this we will write a function which will replace the `NA` values with the mean of the specific vector.

```R
# This will fix the NAs in the given vector (column)
fixNa <- function (attribute) {
  ifelse(is.na(attribute), mean(attribute, na.rm = TRUE), attribute)
}

# Now we can re-use the same function to fix NAs in both variables
data$Age <- fixNa(data$Age)
data$Income <- fixNa(data$Income)
```

Okay, now we have got rid of the missing values. if you check the data set now you can see the `NA` cells a are replaced by values `33.3` and `77539.6`

## Dealing With Categorical Variables

In a data set, variables are divided into two types, numerical and categorical. in our data set there are two numerical variables which are `Age` and `Salary` and there are three categorical variables which are `Country`, `Gender` and `Married`. we can't use these categorical variables (string values) as is in a mathematical equation. so we will have to convert or encode them to numbers so we can use them in our algorithms. to do this we can use the `factor` function in R

```R
factorCategories <- function (attr, zeroBased = FALSE) {
  # we can reuse the same result
  uniqueValues = unique(attr)
  factor(attr, uniqueValues, seq(ifelse(zeroBased, 0, 1), 
                                 ifelse(zeroBased, length(uniqueValues) -1, length(uniqueValues))))
}

data$Country <- factorCategories(data$Country)
data$Gender <- factorCategories(data$Gender, TRUE)
data$Married <- factorCategories(data$Married, TRUE)
```

Here again, we use a function to do the processing so we don't have to rewrite the same thing. the `factorCategories` will use the `factor` function to replace the categorical values with factors. with variables like `Gender` and `Married` which has only two possible values, we have used  0 based sequence to include 0.


## Splitting the Model

So, now we have fixed the categorical values and we are ready to split the data set into two sets which are `training set` and `test set`. this step is necessary because, in order to evaluate the performance of our machine learning model, we need a separate set from our training set.

we are going to use the `caTools` R library to do this.

```R
library(caTools) # Load caTools which will help to split the data set
set.seed(12345)

# Creating the input vector which will help the subset function to decide the set of a row
sp <- sample.split(data$Married, SplitRatio = 0.8) # This sets the ratio of the training set

# Now we can subset using split vector
trainingSet <- subset(data, sp == TRUE) # Include if the value is TRUE
testSet <- subset(data, sp == FALSE) # Include if the value is FALSE
```

## Feature Scaling

If you take a look at two numerical variables of the data set you can see that those are not on the same scale. the `Salary` column has much larger values than the `Age` column. this could cause problems in the machine learning model if we don't fix it. 
a lot of machine learning models are based on the Euclidean Distance equation. so if we don't normalize the values, the salary will dominate the Euclidean Distance. 

there are several ways of Feature scaling common ones are 
1. Standardization
2. Normalization

in Standardization, for each observation in each feature, you withdraw the mean value of all the values of the feature and divide it by the standard deviation.

and in Normalization, you subtract your observation feature `x` by the minimum value of all feature values and divide it by the difference between the maximum of your feature values and the minimum of your feature values.

We can use the in-built `scale` function in R to do this.

```R
trainingSet[, c(2,4)] = scale(trainingSet[, c(2,4)])
testSet[, c(2,4)] = scale(testSet[, c(2,4)])
```

we have to select only the features in indexes 2 and 4 which are `Salary` and `Age`. we should not include the features which we converted to factors in the scaling set as they are not considered as numbers in R.


## Wrap up

Okay with feature scaling done, we have a simple and perfect data set that can be used in our machine learning models.

below is the complete script that we built in this article.

```R
# This will load the SCV file into a data frame
data <- read.csv("dataSet.csv")

# This will fix the NAs in the given vector (column)
fixNa <- function (attribute) {
  ifelse(is.na(attribute), mean(attribute, na.rm = TRUE), attribute)
}


# Now we can re-use the same function to fix NAs in both variables
data$Age <- fixNa(data$Age)
data$Income <- fixNa(data$Income)


factorCategories <- function (attr, zeroBased = FALSE) {
  # we can reuse the same result
  uniqueValues = unique(attr)
  factor(attr, uniqueValues, seq(ifelse(zeroBased, 0, 1), 
                                 ifelse(zeroBased, length(uniqueValues) -1, length(uniqueValues))))
}

data$Country <- factorCategories(data$Country)
data$Gender <- factorCategories(data$Gender, TRUE)
data$Married <- factorCategories(data$Married, TRUE)

library(caTools) # Load caTools which will help to split the data set
set.seed(12345)

# Creating the input vector which will help the subset function to decide the set of a row
sp <- sample.split(data$Married, SplitRatio = 0.8) # This sets the ratio of the training set

# Now we can subset using split vector
trainingSet <- subset(data, sp == TRUE) # Include if the value is TRUE
testSet <- subset(data, sp == FALSE) # Include if the value is FALSE


trainingSet[, c(2,4)] = scale(trainingSet[, c(2,4)])
testSet[, c(2,4)] = scale(testSet[, c(2,4)])
```
