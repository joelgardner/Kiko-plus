---
layout: post
title: "A modern JS app (IV): backend architecture"
description: "Part 4: In this post, we'll finalize our backend architecture by integrating SenecaJS, which will facilitate our service communication, allowing us to concentrate on business logic.  Our backend will be a web of services, each of which has a single responsibility and will communicate in a uniform manner with one another."
date: 2017-06-30
tags: [javascript, node, microservices, seneca]
comments: true
share: false
---

## A modern React / Node.js application

### Part 4: Finalizing the Backend Architecture with Microservices

#### Microservices

Before we get to the nitty-gritty, let's take a moment to think about our server architecture.  We'll break our application into microservices.  We will use the [Seneca.js](http://senecajs.org/) library to facilitate our services.  Let's think about the services we need:
- `gateway` - we definitely need a service that receives client requests and sends back a response
- `booking` - we need a service to perform our booking logic
- `inventory` - a service to maintain our room inventory
- `storage` - a service to interact with our Postgres database to perform C.R.U.D. operations
- `image` - we'll use this service to interact with Amazon S3 to store the images associated with each room.  

> You could argue `image` should be part of the `inventory` service.  But what if we want to implement a feature allowing guests to post pictures they've taken on their trips?  This way, we'll have a service that basically does what we want already, without having to extract such logic from `inventory`.

Don't let the term *microservice* scare you.  It's a fancy way to describe a class or function (or set of functions).  A microservice should be independently deployable.  One microservice handles one aspect of our business logic.  The opposite of a microservice architecture is a monolith, in which all code is a huge, single project wherein a single part cannot be broken out independently.  

As with everything in software, there are tradeoffs.  In a monolith, a service invoking another service is probably just a nice, easy function call.  In a microservices architecture, the same two services don't necessarily live inside the same process.  Thus, invocation must be done over some other transport method: http, ipc, tcp, etc(p).

This is where Seneca comes in.  It takes care of the inter-service communication, and all we need to think about is the arguments and result each service sends and receives.

`npm i --save seneca`
