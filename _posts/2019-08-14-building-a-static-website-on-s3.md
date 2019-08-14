---
layout: post
title:  Website on S3 From Scratch With SSL, Continues Integration
tags: aws
comments: true
description: In this post, I am creating creating a website and host it on S3 and setup a continues integration pipeline setup using Gitlab
published: false
---

In this post I am going to show you how I used AWS to create a website and use a CI pipeline to continuously update the 
site using GIT. from scratch to updating the site using git commits.

What we can expect from the site that we gonna create ?

* It should be fast
* Should have a proper SSL certificate
* Should be able to update the site right from GIT


### What we are Going to use

Lets see what we are going to use to make this happen, Most of these tools are either free or very cost-effective for a
simple website. So you can give this a try without much worries. 

We are going to spend most of the time in AWS. so if you have not used AWS before, create an account if you want to follow 
alone.

##### A domain name service
I am going to use Godaddy. but you can use any service for this. We use this only to buy the domain.
after adding the name server records, we wont touch this much.

##### Amazon Route 53
We are going to use Amazon Route 53 as the name server for our domain.

##### Amazon Certificate Manager
We will need the Certificate Manager to get a SSL certificate for our website

##### Amazon Cloudfront
We have to use a Cloudfront distribution for our website.

##### Amazon S3
We are going to use S3 to put the files of the website (site itself)

##### Gitlab
We will use Gitlab to host our code and create the CI pipeline. create a new account if you haven't used it before 
if you want to follow alone. its free.



### Getting a Domain Name Ready

If you have a domain name already, skip this step. I am going to go ahead and buy a new domain from GoDaddy so i can use 
that domain throughout the tutorial. You can use any domain name service as you like such as NameCheap, BlueHost.


