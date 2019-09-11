---
layout: post
title:  Let's Write a DAL Generator
tags: programming csharp .net
comments: true
published: false
description: Let's write a code generator that will generate a data access API for .net projects.
---

A data access layer is a code layer in a layered architecture that will enable the above layers to access a database through a simplified  API.
To achieve this, we use ORMs such as Entity Framework most of the time.
For most, the functions such as change tracking is bit magical. 

So in the upcoming posts I am going to write a code generator that will generate this code.
The generator will generate a class library that contains the data access layer.


When it comes to ORMs, there are two approaches, one is the code first approach which we write code first and generate the database according to our code. the other approach is to generate the code using an already existing database. I am going to use the database first approach in the tool that i am going to build.

For now we are going to support only MySQL and MS SQL Server. 

I was doing similar side projects before so I will be taking bits and pieces from those code bases also. I am going to write down every step so you can follow along if you like.

## Structure

Okay, Lets start by creating the basic structure of our project. Our whole code base gonna contain three projects.

01. The core library
02. Generator
03. The CLI interface

The core library will contain the general reusable code that will be available as a separate package so the generator doesn't have to generate the same code again and again.

The Generator contains the the code that will actually generate the code. this should be able using any interface like CLI.

The CLI is a CLI application that uses the Generator and.

Lets start by creating a directory for our project

```sh
mkdir Generator
cd Generator
```

Lets create our the CLI application and the Generator library and bundle inside a solution.


```sh
dotnet new sln
dotnet new classlib --name Generator
dotnet sln add ./Generator/Generator.csproj
dotnet new console --name Generator.CLI
dotnet sln add ./Generator.CLI/Generator.CLI.csproj
cd Generator.CLI/
dotnet add reference ../Generator/Generator.csproj
cd ..
```

Okay now our solution is ready to go. now we can start coding.