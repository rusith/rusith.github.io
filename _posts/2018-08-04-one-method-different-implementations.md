---
layout: post
title: One Method, Different Implementations
tags: programming csharp
comments: true
description: We can set Func<>s as properties, so people can call them just like methods, and we can change the function from the constructor according to the user's needs. in this example, I am creating a class which has few methods to download data, the user of the class can decide whether they should cache the results or not
dateCreated: 2018-08-04
datePublished: 2018-08-04
dependencies: .Net
about: Changing implementation of methods of a class at runtime in C#
---
Well, these are not actually methods, we set Func<>s as properties, so people can call them just like methods, and we can change the function from the constructor according to the user's needs. in this example, I am creating a class which has few methods to download data, the user of the class can decide whether they should cache the results or not.
This may have limitations. but this could be useful in some scenarios.
Here's the code

```cs
using System;
using System.Collections.Generic;
using System.Net;
using Newtonsoft.Json;
using Xunit;
using static Xunit.Assert;

namespace MyApp.Tests.Experiments
{
    public class Model
    {
        public string JsonFor { get; set; }
    }

    /// <summary>
    /// This thing can download things (synchronous [bad, but who cares]) 
    /// </summary>
    public sealed class Downloader
    {
        /// <summary>
        /// This function takes a function and returns a function which caches  the results of the given function
        /// </summary>
        /// <typeparam name="TIn">Input type</typeparam>
        /// <typeparam name="TOut">Output type</typeparam>
        /// <param name="func">function</param>
        /// <returns></returns>
        private static Func<TIn, TOut> Cache<TIn, TOut>(Func<TIn, TOut> func)
        {
            var cache = new Dictionary<TIn, TOut>();
            return key => cache.ContainsKey(key) ? cache[key] : cache[key] = func(key);
        }

        public Downloader(bool cached)
        {
            if (!cached) return;

            /* If the user wants this to be cached, we send the properties through the Cache function */
            DownloadWebPage = Cache(DownloadWebPage);
            DownloadJson = Cache(DownloadJson);
        }

        /// <summary>
        /// Download HTML from a URI (not tested)
        /// </summary>
        public Func<string, string> DownloadWebPage { get; } = (uri) =>
        {
            using (var wc = new WebClient())
            {
                return wc.DownloadString(uri);
            }
        };

        /// <summary>
        /// Download JSON (fake because i didn't have an internet connection when writing this)
        /// </summary>
        public Func<string, Model> DownloadJson { get; } = uri => JsonConvert.DeserializeObject<Model>(
            "{'jsonFor': '+ " + url + " +', 'data': {}}".Replace('\'', '"'));
    }


    /// <summary>
    /// Some tests 
    /// </summary>
    public class DownloaderTests
    {
        [Fact]
        public void DownloadJson_Should_Download_Json()
        {
            var downloader = new Downloader(false);
            var obj = downloader.DownloadJson("URLTOJSON");
            Equal("URLTOJSON", obj.JsonFor);
        }

        [Fact]
        public void DownloadJson_Should_Cache_Object_If_Cached()
        {
            var downloader = new Downloader(true);
            var objOne = downloader.DownloadJson("URLTOJSON");
            var objTwo = downloader.DownloadJson("URLTOJSON");
            var objThree = downloader.DownloadJson("URLTOJSON");
            var obj4 = downloader.DownloadJson("URLTOJSON1");

            Equal(objOne, objTwo);
            Equal(objTwo, objThree);
            NotEqual(objThree, obj4); // Should be in a seperate test
        }
    }
}
```