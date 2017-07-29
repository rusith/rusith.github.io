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

# An HTTP Proxy That Can Handle Authentication and Authorization Using Node JS

I Had to implement a service gateway for a microservice backend. I tried various open source API gateways but didn't help with
my requirements. some of my requirements was

* Some endpoints or APIs are secured while others aren't
* For the letter reason Gateway itself should manage tokens
* One token can be used to access all non secured endpoints
* Users should be authenticated using username and password

Then i decided to do it my self without using an existing implementation. as a temporary solution.
Here I Use Node JS to write the proxy . Node JS does a great job handling lot of requests and performance is pretty good.

I use the package Node HTTP Proxy https://github.com/nodejitsu/node-http-proxy to implement the proxy.
And Redis to store access tokens.

Let's jump into to the code.

```JS
(function() {
    "use strict";

    var http = require('http'),
        httpProxy = require('http-proxy');
    var redis = require('redis'); // We use redis to store tokens.
    var redisClient = redis.createClient(6379, 'localhost'); // Create the redis client.
    var helpers = require("./base/helpers.js"); // This module contains some helper functions.
    var TokenEndPoint = '/token'; // Endpoint of the token requests.
    var settings = { // Settings used in the proxy  (can be from a file or database).
      "userAuthenticationPath": "/users/authenticate", // The endpoint in the authentication API  which actually authenticates the user.
      "authenticationServiceUrl": "http://authservice", // Hostname of the authentication service (API). 
      "tokenLifeTime": 3600, // Lifetime of an access token.
      "origins": ["http://abc.com"], // Allowed origins.
      "listen": 5500, // Port to listen.
      "services": [{  // List of services (APIs).
              "name": "Order Service", // Name of the service.
              "path": "/orders", // Paths belong to this service. 
              "authenticate": true, // Only authorized users can access this API.
              "allowedMethods": "GET,POST,PUT,DELETE", // Allowed HTTP verbs.
              "upStream": "http://orderservice" // Destination address of the API.
          },
          {
              "name": "Item Service",
              "path": "/items",
              "authenticate": false, // Anyone can access this API.
              "allowedMethods": "GET", // Only GET is allowed.
              "upStream": "http://itemservice"
          }
      ]
    };

    // This function gives the service for given base path.
    function getServiceForPath(path) {
        var services = settings.services;
        for (var i = 0, sl = services.length; i < sl; i++) {
            if (path.startsWith(services[i].path)) {
                return services[i];
            }
        }
    }

    // Creates the proxy server.
    var proxy = httpProxy.createProxyServer({});

    // Registers a function to call when a response is arrived.
    proxy.on('proxyRes', function(proxyReq, req, res, options) {
        // If the request is a token request (we set this property in the request code), we execute below code.
        // Otherwise the response will be delivered to the requester as is.
        if (req.tokenRequest && proxyReq.statusCode === 200) {
            var userId = proxyReq.headers.userid;
            // Authentication API should set the userid header if the credentials are correct.
            // If the header userid is not exists, we send a not authorized response the the requester. 
            if (!userId) {
                return sendNotAuthorized(res);
            }
            // Creates a GUID as an access token.
            var accessToken = helpers.guid();
            // Save the access token in Redis.
            redisClient.setex(accessToken, settings.tokenLifeTime, '1');
            // Set headers required by the requester.
            res.setHeader('token', accessToken);
            res.setHeader('expires_in', settings.tokenLifeTime);
            res.setHeader('Access-Control-Expose-Headers', 'token,expires_in,userid');
        }
    });

    // proxy the request to the target service.
    function sendToService(service, req, res) {
        var url = req.url;
        req.url = url.substr(service.path.length, url.length - service.path.length);
        proxy.web(req, res, { target: service.upStream }, function(e) { console.error(e); });
    }

    function sendJson(obj, res) {
        res.setHeader("Content-Type", "application/json");
        res.end(JSON.stringify(obj));
    }

    function sendNotAuthorized(res) {
        res.statusCode = 401;
        sendJson('You are not allowed to access this API!', res);
    }

    function sendNotFound(res) {
        res.statusCode = 404;
        sendJson('Could not find any resources for given path.', res);
    }

    function sendMethodNotAllowed(res) {
        res.statusCode = 405;
        sendJson('Method not allowed', res);
    }

    // Check CORS of the request
    function CheckCors(service, req, res) {
        // Set needed headers.
        if (req.headers.origin) {
            res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
        }
        res.setHeader('Access-Control-Allow-Methods', service.allowedMethods);
        res.setHeader('Access-Control-Allow-Headers', 'Authorization,Content-Type,Accept');
        // Just send the response if the request is an POTIONS request.
        if (req.method == 'OPTIONS') {
            res.end();
            return true;
        }
        return false;
    }

    var server = http.createServer(function(req, res) {
        var origins = settings.origins;
        var requestHeaders = req.headers;
        var requestUrl = req.url.toLowerCase();
        // If the allowed origins are declared in the settings.
        if (origins) {
            // Check the origin of the request and send a not authorized response if the request origin is not allowed.
            if (!requestHeaders.origin || 0 > origins.indexOf(requestHeaders.origin)) {
                return sendNotAuthorized(res);
            }
        }
        // If the request is for getting the token.
        if (requestUrl.startsWith(TokenEndPoint)) {
            // Set request property to check later in the response code.
            req.tokenRequest = true;
            // Change destination to the authentication endpoint in the authentication API.
            req.url = settings.userAuthenticationPath;
            if (!CheckCors({ allowedMethods: 'POST' }, req, res)) {
                // Proxy the request.
                proxy.web(req, res, { target: settings.authenticationServiceUrl }, function(e) { console.error(e); });
            }
        } else {
            // Get the service.
            var service = getServiceForPath(requestUrl);
            if (!service) {
                // If no service found for the request path, returns a not found.
                return sendNotFound(res);
            }
            // If the method is not OPTIONS and method is not in the allowed methods list ,
            // returns a not allowed response.
            if (req.method != 'OPTIONS' && -1 > service.allowedMethods.indexOf(req.method)) {
                return sendMethodNotAllowed(res);
            }
            // If the service needs authentication.
            if (service.authenticate) {
                // Request must have the the authorization header.
                if (requestHeaders.authorization) {
                   // If Redis has a value for request's authentication header value.
                    redisClient.exists(requestHeaders.authorization, function(err, reply) {
                        if (reply === 1) {
                            if (!CheckCors(service, req, res)) {
                                // Proxy the request to the target service.
                                return sendToService(service, req, res);
                            }
                        } else {
                            // If not authenticated or token has expired , send a not authorized response.
                            return sendNotAuthorized(res);
                        }
                    });
                } else {
                    // If authorization header does not exist , then send a not authorized response.
                    return sendNotAuthorized(res);
                }
            } else {
                // If the service does not need authentication , proxy the request to the target service.
                if (!CheckCors(service, req, res)) {
                    return sendToService(service, req, res);
                }
            }
        }
    });
    // Listen to the port.
    server.listen(settings.listen);
})();
```
Simple as that .
This code is not well tested and can be improved . Give me a feedback if you are interested.