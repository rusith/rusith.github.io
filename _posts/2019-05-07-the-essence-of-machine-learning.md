---
layout: post
title: The Essence of Machine Learning
tags: machineLearning
comments: true
hasMath: true
description: Machine learning can significantly improve the performance of a system. This could be very beneficial for businesses or any other application. So if someone comes to you and suggest you a business idea that involves machine learning, how do you decide that a machine learning implementation is feasible with the particular idea?
dateCreated: 2019-05-07
datePublished: 2019-05-07
about: Basic understanding of machine learning. what is the essence of machine learning
---

Machine learning can significantly improve the performance of a system. This could be very beneficial for businesses or any other application.

So if someone comes to you and suggest you a business idea that involves machine learning, how do you decide that a machine learning implementation is feasible with the particular idea?

Essentially, there are three things to consider when it comes to determining the feasibility of machine learning


## You Should Have Data

The first thing is you should have data. if you don't have enough data. you can't use machine learning in that problem. Machine learning is learning from data, so what you can do with machine learning if you have no data to teach the computer? this is an essential requirement that you can't avoid.


## A Pattern Should Exist

In order to apply machine learning on a data set, there should be some pattern in data as an example, A rating on a movie is related to the previous ratings of the same person and the ratings of the other movies are related to the authors of those ratings. so there is a pattern to be discovered. simply, we can answer the question How a user might rate a movie by using the patterns in the data.

This would be not possible if the data had no pattern. or the relationship between the attributes.


## Should not be Able to  pin it Down Mathematically

If you can find the pattern mathematically you won't need machine learning in the first place. the fact that a pattern exists and we cannot pin it down mathematically is the reason to go for a machine learning approach.


## Components of Learning

Think of a scenario which a bank tries to use a machine learning system to determine a customer is creditworthy or not. They have data from old customers and the customer that they are gonna evaluate each time. so what are the components in this learning scenario?

### Input ($$\boldsymbol{x}$$)

In this case, the credit application is the input for the machine learning system. This is a vector that will include data such as income and age that are related to the expected output which is creditworthiness of the applicant. They don't necessarily directly determine it,  but they are related.

### Output (*y*)

In this case, the output has two possibilities (the customer is creditworthy or not)

### Target Function

$$f : \boldsymbol{x}  \rightarrow  y$$

In this case, this is the ideal credit approval formula which is unknown to us.

### Data 

$$(\boldsymbol{x}_1, y_{1}), (\boldsymbol{x}_2, y_{2}), (\boldsymbol{x}_3, y_{3}), ... , (\boldsymbol{x}_n, y_{n})$$

The historical records that can be used to train the model. this includes the attributes ($$boldsymbol{x}$$) and the output they produced ($$y$$) in this case.


## Hypothesis

$$g: x \rightarrow y$$

This is the function we create that will estimate *f*, that's the goal of learning.
