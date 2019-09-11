---
layout: post
title:  Let's Write a DAL Generator
tags: programming csharp .net
comments: true
published: false
description: Lets's write a code generator that will generate a data access layer for our .net project.
---

A data access layer is a code layer in a layerd architecture that will enable the above layers to access a database through a simplified  API.
To achieve this, we use ORMs such as Entity Framework most of the time.
For most, the functions such as change tracking is bit magical. 

So let's build a code generator to generate an API that will help us to access a database. this will be a simple one and will not implement existing interfaces like `IQueryable`. and we will try to implement change tracking and we will build this using the repository and unit of work pattern.

We will not build the SQL data to model mapping (ORM) part of this tool as there is a good open souce mini ORM that we can use which is Dapper.

So we are going to use Dapper source code to map the SQL results to objects. we will implement all functionlities above mapping from scratch.

