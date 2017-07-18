---
layout: post
title: "A modern JS app (V): Building our frontend with React, Redux, and Redux-Saga"
description: "Part 5: In this post, we'll take a deep dive into the React(/Redux) ecosystem with libraries such as Redux, Redux Saga, and Redux Little Router to build our booking site's frontend"
date: 2017-07-13
tags: [javascript, react, react-redux, redux, redux-saga]
comments: true
share: false
---

## A modern React / Node.js application

### Part 5: Building our frontend with React, Redux, and Redux-Saga

#### React ecosystem
Currently our frontend is pretty sparse.  We've used `create-react-app` to bootstrap our UI with React, and that's about it.  Now, we'll install and use a few libraries to take care of stuff like state management, data-flow, routing, and app behavior.
 - [redux](http://redux.js.org/): a wonderful and simple library that pairs nicely with React (it wasn't written specifically *for* React; it can be used with most any other UI library).  It's a thin state container that focuses on one-way data flow and was inspired by earlier iterations of the Flux pattern.  It has become the go-to state management library for React apps.  It provides a flexible middleware mechanism, which has resulted in all sorts of plugins written to work with Redux.
 - [redux-saga](https://github.com/redux-saga/redux-saga): a Redux middleware that facilitates side-effects (such as AJAX requests) in an easy-to-read, testable, declarative way.  It removes us from callback hell and nicely describes what is happening when actions are triggered by the user (or internally by the app).
 - [redux-little-router](https://github.com/FormidableLabs/redux-little-router): another Redux middleware that treats the browser URL as state, and as such, is accessible via the Redux store's state tree.
 - [apollo-client](https://github.com/apollographql/apollo-client): a wonderful client-side GraphQL framework that provides flexible usage of queries/mutations, and best of all, it uses Redux state to cache normalized responses so we don't have to worry about re-fetching data we've already fetched.  Below, you'll see what I mean.
 - [immutable](https://facebook.github.io/immutable-js): yet another Facebook library that provides Clojure-like data-structures to wrap javascript's regular Array and Map objects.  Each Immutable object will track its changes and return a new reference.  This helps React and Redux determine whether or not to re-render components.  Admittedly, for such a simple app, it's a bit overkill.  But I wanted to demonstrate its utility.

> Note, we're actually *not* going to use [`react-apollo`](https://github.com/apollographql/react-apollo), the React bindings for the Apollo framework.  The reason is that I prefer to keep my React views very thin: they shouldn't care where their data came from, they should only be concerned with displaying it.  Still, check out the bindings and how they work, because they're very interesting way to fetch/mutate data, and you may actually prefer a "fatter" view.

> Facebook's [Relay](https://github.com/facebook/relay) is a similar library.  It is perhaps a bit more powerful but the learning curve is steeper.  There are many articles online that compare the two.  You can't really go wrong with either, I simply chose Apollo for this post due to its lower barrier of entry.

#### Configure our Redux store

#### Routes
Let's define the routes we will expose:
 - `/` will display a list of properties
 - `/property/:propertyId` will display a list of the property's rooms in a grid view.  Each room will be represented by an image, a short description, and the price per night.
 - `/room/:roomId` will display the details of a property's room: images and a description.  It will also allow a user to book the room for a specified date range.
 - `/manage` will display an "owner's dashboard", allowing property managers to add properties, rooms, and view bookings.
 - `/manage/:propertyId` will allow a property manager to edit property details.
 - `/manage/room/:roomId` will allow a property manager to edit room details.

The last 3 routes will require a user to login to access them.

#### Directory structure
We're building a Redux app, so let's add a few folders to `client/src`.  `In client:`

`mkdir src/reducers src/actions src/components src/containers src/sagas`

Every directory we created, except for `components`, is Redux-specific.  One critique of Redux is that it's very boilerplate heavy.  That can be true ([it doesn't have to be](http://blog.isquaredsoftware.com/2017/05/idiomatic-redux-tao-of-redux-part-1/)).  It's a tradeoff I'm willing to make: we add a little more boilerplate, but gain much more declarative, readable, and... reason-about-able code.

 - `reducers` will contain our functions that mutate our application state on actions
 - `actions` will contain our actions/action-creators that describe everything that can happen in our app
 - `containers` will contain our Redux containers, which map application state to component props and callbacks for events
 - `sagas` will contain functions that describe app behavior and perform the side-effects required for our app to work
 - `components` will contain our React components

#### Clientside API
We will use the awesome GraphQL framework [`apollo`](http://dev.apollodata.com/react), which allows us to declaratively specify the data that each component expects and the queries and fragments that retrieve that data.  It actually uses Redux under the hood, and thus will integrate nicely with our setup.  Install it now:

`npm i --save apollo-client graphql-tag`

We'll also install `graphql-anywhere` and `prop-types`:

`npm i --save graphql-anywhere prop-types`

Let's add our initial call to retrieve a list of properties from the server.



#### Building our state

>If you haven't, please read the Redux documentation.

Our initial state is simple:

```json
{
  "properties": [],
  "token": null
}
```

The idea is that when the app initializes, we will have an empty `properties` array and a null token (we assume the user to be anonymous).

This is a good time to build our reducer.  Add a `reducers` directory to

#### New API calls
Right now, all we can do is create and fetch/edit/delete an entity by ID.  These are all useful, but how do we get the ID from the server to the client if we don't know it?  We'll need to add a few more API calls:
 - `listProperties` will return a list of `Property`s
 - `listRooms` will return a list of `Room`s associated with a `Property`
 - `bookRoomByDates` will take a date-range and attempt to reserve the room for the user.

 This will do for now.

 #### GraphQL schema changes
 The above API calls bring up a question of search and pagination.  It'd obviously be nice to be able to search by keyword or (especially) dates and location.  
