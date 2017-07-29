---
layout: essay
type: essay
title: An HTTP Proxy That Can Handle Authentication and Authorization Using Node JS
date: 2017-07-29
labels:
  - Software Engineering
  - Microservice
  - Authentication
  - Authorization
  - Node JS
---
I Had to implement a service gateway for a microservice backend. I tried various open source API gateways but didn't help with
my requirements. some of my requirements was

* Some endpoints or APIs are secured while others aren't
* For that reason Gateway itself should manage tokens
* One token can be used to access all secured endpoints
* Users should be authenticated using username and password

Then i decided to do it my self without using an existing implementation. as a temporary solution.
Here I Use Node JS to write the proxy . Node JS does a great job handling lot of requests and performance is pretty good.

I use the package [Node HTTP Proxy](https://github.com/nodejitsu/node-http-proxy)  to implement the proxy.
And Redis to store access tokens.

Let's jump into to the code.

<pre style='color:#000000;background:#ffffff;'><span style='color:#808030; '>(</span><span style='color:#800000; font-weight:bold; '>function</span><span style='color:#808030; '>(</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
    <span style='color:#800000; '>"</span><span style='color:#0000e6; '>use strict</span><span style='color:#800000; '>"</span><span style='color:#800080; '>;</span>

    <span style='color:#800000; font-weight:bold; '>var</span> http <span style='color:#808030; '>=</span> require<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>http</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#808030; '>,</span>
        httpProxy <span style='color:#808030; '>=</span> require<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>http-proxy</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#800000; font-weight:bold; '>var</span> redis <span style='color:#808030; '>=</span> require<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>redis</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span> <span style='color:#696969; '>// We use redis to store tokens.</span>
    <span style='color:#800000; font-weight:bold; '>var</span> redisClient <span style='color:#808030; '>=</span> redis<span style='color:#808030; '>.</span>createClient<span style='color:#808030; '>(</span><span style='color:#008c00; '>6379</span><span style='color:#808030; '>,</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>localhost</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span> <span style='color:#696969; '>// Create the redis client.</span>
    <span style='color:#800000; font-weight:bold; '>var</span> helpers <span style='color:#808030; '>=</span> require<span style='color:#808030; '>(</span><span style='color:#800000; '>"</span><span style='color:#0000e6; '>./base/helpers.js</span><span style='color:#800000; '>"</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span> <span style='color:#696969; '>// This module contains some helper functions.</span>
    <span style='color:#800000; font-weight:bold; '>var</span> TokenEndPoint <span style='color:#808030; '>=</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>/token</span><span style='color:#800000; '>'</span><span style='color:#800080; '>;</span> <span style='color:#696969; '>// Endpoint of the token requests.</span>
    <span style='color:#800000; font-weight:bold; '>var</span> settings <span style='color:#808030; '>=</span> <span style='color:#800080; '>{</span> <span style='color:#696969; '>// Settings used in the proxy  (can be from a file or database).</span>
      <span style='color:#800000; '>"</span><span style='color:#0000e6; '>userAuthenticationPath</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>/users/authenticate</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// The endpoint in the authentication API  which actually authenticates the user.</span>
      <span style='color:#800000; '>"</span><span style='color:#0000e6; '>authenticationServiceUrl</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>http://authservice</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Hostname of the authentication service (API). </span>
      <span style='color:#800000; '>"</span><span style='color:#0000e6; '>tokenLifeTime</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#008c00; '>3600</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Lifetime of an access token.</span>
      <span style='color:#800000; '>"</span><span style='color:#0000e6; '>origins</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#808030; '>[</span><span style='color:#800000; '>"</span><span style='color:#0000e6; '>http://abc.com</span><span style='color:#800000; '>"</span><span style='color:#808030; '>]</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Allowed origins.</span>
      <span style='color:#800000; '>"</span><span style='color:#0000e6; '>listen</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#008c00; '>5500</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Port to listen.</span>
      <span style='color:#800000; '>"</span><span style='color:#0000e6; '>services</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#808030; '>[</span><span style='color:#800080; '>{</span>  <span style='color:#696969; '>// List of services (APIs).</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>name</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>Order Service</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Name of the service.</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>path</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>/orders</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Paths belong to this service. </span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>authenticate</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#0f4d75; '>true</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Only authorized users can access this API.</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>allowedMethods</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>GET,POST,PUT,DELETE</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Allowed HTTP verbs.</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>upStream</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>http://orderservice</span><span style='color:#800000; '>"</span> <span style='color:#696969; '>// Destination address of the API.</span>
          <span style='color:#800080; '>}</span><span style='color:#808030; '>,</span>
          <span style='color:#800080; '>{</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>name</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>Item Service</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>path</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>/items</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>authenticate</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#0f4d75; '>false</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Anyone can access this API.</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>allowedMethods</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>GET</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#696969; '>// Only GET is allowed.</span>
              <span style='color:#800000; '>"</span><span style='color:#0000e6; '>upStream</span><span style='color:#800000; '>"</span><span style='color:#800080; '>:</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>http://itemservice</span><span style='color:#800000; '>"</span>
          <span style='color:#800080; '>}</span>
      <span style='color:#808030; '>]</span>
    <span style='color:#800080; '>}</span><span style='color:#800080; '>;</span>

    <span style='color:#696969; '>// This function gives the service for given base path.</span>
    <span style='color:#800000; font-weight:bold; '>function</span> getServiceForPath<span style='color:#808030; '>(</span>path<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        <span style='color:#800000; font-weight:bold; '>var</span> services <span style='color:#808030; '>=</span> settings<span style='color:#808030; '>.</span>services<span style='color:#800080; '>;</span>
        <span style='color:#800000; font-weight:bold; '>for</span> <span style='color:#808030; '>(</span><span style='color:#800000; font-weight:bold; '>var</span> i <span style='color:#808030; '>=</span> <span style='color:#008c00; '>0</span><span style='color:#808030; '>,</span> sl <span style='color:#808030; '>=</span> services<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>length</span><span style='color:#800080; '>;</span> i <span style='color:#808030; '>&lt;</span> sl<span style='color:#800080; '>;</span> i<span style='color:#808030; '>++</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>path<span style='color:#808030; '>.</span>startsWith<span style='color:#808030; '>(</span>services<span style='color:#808030; '>[</span>i<span style='color:#808030; '>]</span><span style='color:#808030; '>.</span>path<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#800000; font-weight:bold; '>return</span> services<span style='color:#808030; '>[</span>i<span style='color:#808030; '>]</span><span style='color:#800080; '>;</span>
            <span style='color:#800080; '>}</span>
        <span style='color:#800080; '>}</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#696969; '>// Creates the proxy server.</span>
    <span style='color:#800000; font-weight:bold; '>var</span> proxy <span style='color:#808030; '>=</span> httpProxy<span style='color:#808030; '>.</span>createProxyServer<span style='color:#808030; '>(</span><span style='color:#800080; '>{</span><span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>

    <span style='color:#696969; '>// Registers a function to call when a response is arrived.</span>
    proxy<span style='color:#808030; '>.</span>on<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>proxyRes</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> <span style='color:#800000; font-weight:bold; '>function</span><span style='color:#808030; '>(</span>proxyReq<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>,</span> options<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        <span style='color:#696969; '>// If the request is a token request (we set this property in the request code), we execute below code.</span>
        <span style='color:#696969; '>// Otherwise the response will be delivered to the requester as is.</span>
        <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>req<span style='color:#808030; '>.</span>tokenRequest <span style='color:#808030; '>&amp;&amp;</span> proxyReq<span style='color:#808030; '>.</span>statusCode <span style='color:#808030; '>===</span> <span style='color:#008c00; '>200</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
            <span style='color:#800000; font-weight:bold; '>var</span> userId <span style='color:#808030; '>=</span> proxyReq<span style='color:#808030; '>.</span>headers<span style='color:#808030; '>.</span>userid<span style='color:#800080; '>;</span>
            <span style='color:#696969; '>// Authentication API should set the userid header if the credentials are correct.</span>
            <span style='color:#696969; '>// If the header userid is not exists, we send a not authorized response the the requester. </span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span><span style='color:#808030; '>!</span>userId<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#800000; font-weight:bold; '>return</span> sendNotAuthorized<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800080; '>}</span>
            <span style='color:#696969; '>// Creates a GUID as an access token.</span>
            <span style='color:#800000; font-weight:bold; '>var</span> accessToken <span style='color:#808030; '>=</span> helpers<span style='color:#808030; '>.</span>guid<span style='color:#808030; '>(</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#696969; '>// Save the access token in Redis.</span>
            redisClient<span style='color:#808030; '>.</span>setex<span style='color:#808030; '>(</span>accessToken<span style='color:#808030; '>,</span> settings<span style='color:#808030; '>.</span>tokenLifeTime<span style='color:#808030; '>,</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>1</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#696969; '>// Set headers required by the requester.</span>
            res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>token</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> accessToken<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>expires_in</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> settings<span style='color:#808030; '>.</span>tokenLifeTime<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>Access-Control-Expose-Headers</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>token,expires_in,userid</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        <span style='color:#800080; '>}</span>
    <span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>

    <span style='color:#696969; '>// proxy the request to the target service.</span>
    <span style='color:#800000; font-weight:bold; '>function</span> sendToService<span style='color:#808030; '>(</span>service<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        <span style='color:#800000; font-weight:bold; '>var</span> url <span style='color:#808030; '>=</span> req<span style='color:#808030; '>.</span>url<span style='color:#800080; '>;</span>
        req<span style='color:#808030; '>.</span>url <span style='color:#808030; '>=</span> url<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>substr</span><span style='color:#808030; '>(</span>service<span style='color:#808030; '>.</span>path<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>length</span><span style='color:#808030; '>,</span> url<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>length</span> <span style='color:#808030; '>-</span> service<span style='color:#808030; '>.</span>path<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>length</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        proxy<span style='color:#808030; '>.</span>web<span style='color:#808030; '>(</span>req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>,</span> <span style='color:#800080; '>{</span> target<span style='color:#800080; '>:</span> service<span style='color:#808030; '>.</span>upStream <span style='color:#800080; '>}</span><span style='color:#808030; '>,</span> <span style='color:#800000; font-weight:bold; '>function</span><span style='color:#808030; '>(</span>e<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span> console<span style='color:#808030; '>.</span>error<span style='color:#808030; '>(</span>e<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span> <span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#800000; font-weight:bold; '>function</span> sendJson<span style='color:#808030; '>(</span>obj<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>"</span><span style='color:#0000e6; '>Content-Type</span><span style='color:#800000; '>"</span><span style='color:#808030; '>,</span> <span style='color:#800000; '>"</span><span style='color:#0000e6; '>application/json</span><span style='color:#800000; '>"</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        res<span style='color:#808030; '>.</span>end<span style='color:#808030; '>(</span>JSON<span style='color:#808030; '>.</span>stringify<span style='color:#808030; '>(</span>obj<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#800000; font-weight:bold; '>function</span> sendNotAuthorized<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        res<span style='color:#808030; '>.</span>statusCode <span style='color:#808030; '>=</span> <span style='color:#008c00; '>401</span><span style='color:#800080; '>;</span>
        sendJson<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>You are not allowed to access this API!</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#800000; font-weight:bold; '>function</span> sendNotFound<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        res<span style='color:#808030; '>.</span>statusCode <span style='color:#808030; '>=</span> <span style='color:#008c00; '>404</span><span style='color:#800080; '>;</span>
        sendJson<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>Could not find any resources for given path.</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#800000; font-weight:bold; '>function</span> sendMethodNotAllowed<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        res<span style='color:#808030; '>.</span>statusCode <span style='color:#808030; '>=</span> <span style='color:#008c00; '>405</span><span style='color:#800080; '>;</span>
        sendJson<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>Method not allowed</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#696969; '>// Check CORS of the request</span>
    <span style='color:#800000; font-weight:bold; '>function</span> CheckCors<span style='color:#808030; '>(</span>service<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        <span style='color:#696969; '>// Set needed headers.</span>
        <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>req<span style='color:#808030; '>.</span>headers<span style='color:#808030; '>.</span>origin<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
            res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>Access-Control-Allow-Origin</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> req<span style='color:#808030; '>.</span>headers<span style='color:#808030; '>.</span>origin<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        <span style='color:#800080; '>}</span>
        res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>Access-Control-Allow-Methods</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> service<span style='color:#808030; '>.</span>allowedMethods<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        res<span style='color:#808030; '>.</span>setHeader<span style='color:#808030; '>(</span><span style='color:#800000; '>'</span><span style='color:#0000e6; '>Access-Control-Allow-Headers</span><span style='color:#800000; '>'</span><span style='color:#808030; '>,</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>Authorization,Content-Type,Accept</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        <span style='color:#696969; '>// Just send the response if the request is an POTIONS request.</span>
        <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>req<span style='color:#808030; '>.</span>method <span style='color:#808030; '>==</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>OPTIONS</span><span style='color:#800000; '>'</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
            res<span style='color:#808030; '>.</span>end<span style='color:#808030; '>(</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800000; font-weight:bold; '>return</span> <span style='color:#0f4d75; '>true</span><span style='color:#800080; '>;</span>
        <span style='color:#800080; '>}</span>
        <span style='color:#800000; font-weight:bold; '>return</span> <span style='color:#0f4d75; '>false</span><span style='color:#800080; '>;</span>
    <span style='color:#800080; '>}</span>

    <span style='color:#800000; font-weight:bold; '>var</span> server <span style='color:#808030; '>=</span> http<span style='color:#808030; '>.</span>createServer<span style='color:#808030; '>(</span><span style='color:#800000; font-weight:bold; '>function</span><span style='color:#808030; '>(</span>req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
        <span style='color:#800000; font-weight:bold; '>var</span> origins <span style='color:#808030; '>=</span> settings<span style='color:#808030; '>.</span>origins<span style='color:#800080; '>;</span>
        <span style='color:#800000; font-weight:bold; '>var</span> requestHeaders <span style='color:#808030; '>=</span> req<span style='color:#808030; '>.</span>headers<span style='color:#800080; '>;</span>
        <span style='color:#800000; font-weight:bold; '>var</span> requestUrl <span style='color:#808030; '>=</span> req<span style='color:#808030; '>.</span>url<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>toLowerCase</span><span style='color:#808030; '>(</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
        <span style='color:#696969; '>// If the allowed origins are declared in the settings.</span>
        <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>origins<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
            <span style='color:#696969; '>// Check the origin of the request and send a not authorized response if the request origin is not allowed.</span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span><span style='color:#808030; '>!</span>requestHeaders<span style='color:#808030; '>.</span>origin <span style='color:#808030; '>||</span> <span style='color:#008c00; '>0</span> <span style='color:#808030; '>></span> origins<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>indexOf</span><span style='color:#808030; '>(</span>requestHeaders<span style='color:#808030; '>.</span>origin<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#800000; font-weight:bold; '>return</span> sendNotAuthorized<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800080; '>}</span>
        <span style='color:#800080; '>}</span>
        <span style='color:#696969; '>// If the request is for getting the token.</span>
        <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>requestUrl<span style='color:#808030; '>.</span>startsWith<span style='color:#808030; '>(</span>TokenEndPoint<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
            <span style='color:#696969; '>// Set request property to check later in the response code.</span>
            req<span style='color:#808030; '>.</span>tokenRequest <span style='color:#808030; '>=</span> <span style='color:#0f4d75; '>true</span><span style='color:#800080; '>;</span>
            <span style='color:#696969; '>// Change destination to the authentication endpoint in the authentication API.</span>
            req<span style='color:#808030; '>.</span>url <span style='color:#808030; '>=</span> settings<span style='color:#808030; '>.</span>userAuthenticationPath<span style='color:#800080; '>;</span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span><span style='color:#808030; '>!</span>CheckCors<span style='color:#808030; '>(</span><span style='color:#800080; '>{</span> allowedMethods<span style='color:#800080; '>:</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>POST</span><span style='color:#800000; '>'</span> <span style='color:#800080; '>}</span><span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#696969; '>// Proxy the request.</span>
                proxy<span style='color:#808030; '>.</span>web<span style='color:#808030; '>(</span>req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>,</span> <span style='color:#800080; '>{</span> target<span style='color:#800080; '>:</span> settings<span style='color:#808030; '>.</span>authenticationServiceUrl <span style='color:#800080; '>}</span><span style='color:#808030; '>,</span> <span style='color:#800000; font-weight:bold; '>function</span><span style='color:#808030; '>(</span>e<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span> console<span style='color:#808030; '>.</span>error<span style='color:#808030; '>(</span>e<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span> <span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800080; '>}</span>
        <span style='color:#800080; '>}</span> <span style='color:#800000; font-weight:bold; '>else</span> <span style='color:#800080; '>{</span>
            <span style='color:#696969; '>// Get the service.</span>
            <span style='color:#800000; font-weight:bold; '>var</span> service <span style='color:#808030; '>=</span> getServiceForPath<span style='color:#808030; '>(</span>requestUrl<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span><span style='color:#808030; '>!</span>service<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#696969; '>// If no service found for the request path, returns a not found.</span>
                <span style='color:#800000; font-weight:bold; '>return</span> sendNotFound<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800080; '>}</span>
            <span style='color:#696969; '>// If the method is not OPTIONS and method is not in the allowed methods list ,</span>
            <span style='color:#696969; '>// returns a not allowed response.</span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>req<span style='color:#808030; '>.</span>method <span style='color:#808030; '>!=</span> <span style='color:#800000; '>'</span><span style='color:#0000e6; '>OPTIONS</span><span style='color:#800000; '>'</span> <span style='color:#808030; '>&amp;&amp;</span> <span style='color:#808030; '>-</span><span style='color:#008c00; '>1</span> <span style='color:#808030; '>></span> service<span style='color:#808030; '>.</span>allowedMethods<span style='color:#808030; '>.</span><span style='color:#800000; font-weight:bold; '>indexOf</span><span style='color:#808030; '>(</span>req<span style='color:#808030; '>.</span>method<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#800000; font-weight:bold; '>return</span> sendMethodNotAllowed<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
            <span style='color:#800080; '>}</span>
            <span style='color:#696969; '>// If the service needs authentication.</span>
            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>service<span style='color:#808030; '>.</span>authenticate<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                <span style='color:#696969; '>// Request must have the the authorization header.</span>
                <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>requestHeaders<span style='color:#808030; '>.</span>authorization<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                   <span style='color:#696969; '>// If Redis has a value for request's authentication header value.</span>
                    redisClient<span style='color:#808030; '>.</span>exists<span style='color:#808030; '>(</span>requestHeaders<span style='color:#808030; '>.</span>authorization<span style='color:#808030; '>,</span> <span style='color:#800000; font-weight:bold; '>function</span><span style='color:#808030; '>(</span>err<span style='color:#808030; '>,</span> reply<span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                        <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span>reply <span style='color:#808030; '>===</span> <span style='color:#008c00; '>1</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                            <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span><span style='color:#808030; '>!</span>CheckCors<span style='color:#808030; '>(</span>service<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                                <span style='color:#696969; '>// Proxy the request to the target service.</span>
                                <span style='color:#800000; font-weight:bold; '>return</span> sendToService<span style='color:#808030; '>(</span>service<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
                            <span style='color:#800080; '>}</span>
                        <span style='color:#800080; '>}</span> <span style='color:#800000; font-weight:bold; '>else</span> <span style='color:#800080; '>{</span>
                            <span style='color:#696969; '>// If not authenticated or token has expired , send a not authorized response.</span>
                            <span style='color:#800000; font-weight:bold; '>return</span> sendNotAuthorized<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
                        <span style='color:#800080; '>}</span>
                    <span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
                <span style='color:#800080; '>}</span> <span style='color:#800000; font-weight:bold; '>else</span> <span style='color:#800080; '>{</span>
                    <span style='color:#696969; '>// If authorization header does not exist , then send a not authorized response.</span>
                    <span style='color:#800000; font-weight:bold; '>return</span> sendNotAuthorized<span style='color:#808030; '>(</span>res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
                <span style='color:#800080; '>}</span>
            <span style='color:#800080; '>}</span> <span style='color:#800000; font-weight:bold; '>else</span> <span style='color:#800080; '>{</span>
                <span style='color:#696969; '>// If the service does not need authentication , proxy the request to the target service.</span>
                <span style='color:#800000; font-weight:bold; '>if</span> <span style='color:#808030; '>(</span><span style='color:#808030; '>!</span>CheckCors<span style='color:#808030; '>(</span>service<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#808030; '>)</span> <span style='color:#800080; '>{</span>
                    <span style='color:#800000; font-weight:bold; '>return</span> sendToService<span style='color:#808030; '>(</span>service<span style='color:#808030; '>,</span> req<span style='color:#808030; '>,</span> res<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
                <span style='color:#800080; '>}</span>
            <span style='color:#800080; '>}</span>
        <span style='color:#800080; '>}</span>
    <span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
    <span style='color:#696969; '>// Listen to the port.</span>
    server<span style='color:#808030; '>.</span>listen<span style='color:#808030; '>(</span>settings<span style='color:#808030; '>.</span>listen<span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
<span style='color:#800080; '>}</span><span style='color:#808030; '>)</span><span style='color:#808030; '>(</span><span style='color:#808030; '>)</span><span style='color:#800080; '>;</span>
</pre>

Simple as that .
This code is not well tested and can be improved . Give me a feedback if you are interested.