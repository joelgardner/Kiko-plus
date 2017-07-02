---
layout: post
title: "A modern JS app (IV): microservices with Seneca.js"
description: "Part 4: In this post, we'll finalize our backend architecture by integrating SenecaJS, which will facilitate our service communication, allowing us to concentrate on business logic.  Our backend will be a web of services, each of which has a single responsibility and will communicate in a uniform manner with one another."
date: 2017-06-30
tags: [javascript, node, microservices, senecajs]
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

#### Directory Structure
At the moment, you might say our directory structure on the backend is in disarray.  We have in `src`:
```
- gateway/
  - resolvers.js
- storage/
  - index.js
- index.js
- schema.graphql
```
> `sum.js` is still there too.  Delete it.

There will certainly be come common utility code (or Flow types) that multiple services will want to use, so we need a way to distinguish this sort of shared, common code with a standalone service.

Let's create a `services` directory under `src`, and it will then contain each service.  The migration begins!  In `server`:

```bash
mkdir src/services
mv src/storage/ src/services/
mv src/gateway/ src/services/
mv src/schema.graphql src/services/gateway/
mv src/index.js src/services/gateway/
rm src/sum.js
```

After running all these commands, our `src` folder should have one item in it: a directory called `services`, which in turn should have two directories: `gateway` and `storage`.  These are two of our microservices.

If we try to run our server application, we'll get a few errors that occur because we just changed a bunch of paths.  Let's fix this.

In `server/package.json`, make the following changes:
- Update the `"main"` property to:

 `"main": "src/services/gateway/index.js",`
- Update the `"scripts"."babel-node"` property to:

 `"babel-node": "./node_modules/.bin/babel-node src/services/gateway/index.js",`

In `server/src/services/gateway/index.js`:
- Remove the import of `connectToStorage` (`import { connectToStorage } from './storage'`)
- Remove the call to `await connectToStorage()`

Additionally, our tests will be broken as well.  Hold tight, we'll fix them in a bit.

#### Seneca
Let's integrate our Storage service with Seneca.  In the `storage` directory, add two files:

`storage-patterns.js`:
```js
import { fetchOne, insertOne, updateOne, deleteOne } from '.'

const ROLE_NAME = 'storage'

/**
  Seneca plugin defining patterns the Storage microservice responds to.
*/
export default function storage(options) {
  this
  .add({ role: ROLE_NAME, cmd: 'fetchOne' }, async (msg, reply) => {
    const result = await fetchOne(msg.type, msg.id)
    reply(null, result.toJSON())
  })

  .add({ role: ROLE_NAME, cmd: 'insertOne' }, async (msg, reply) => {
    const result = await insertOne(msg.type, msg.input)
    reply(null, result.toJSON())
  })

  .add({ role: ROLE_NAME, cmd: 'updateOne' }, async (msg, reply) => {
    const result = await updateOne(msg.type, msg.id, msg.input)
    reply(null, result.toJSON())
  })

  .add({ role: ROLE_NAME, cmd: 'deleteOne' }, async (msg, reply) => {
    const result = await deleteOne(msg.type, msg.id, msg.input)
    reply(null, result.toJSON())
  })
}
```

`storage-listener.js`:
```js
// @flow
import seneca from 'seneca'
import { connectToStorage } from '.'
import { iife } from '../../util'
import storage from './storage-patterns'

/**
  Use an IIFE to initialize a connection to the Mongo store.
  - If successful, start a Seneca microservice listening for patterns defined
      storage-patterns.js.
  - If unsuccessful, end the process, logging the exception.
*/
iife(async () => {
  // connect to Mongo instance
  const connection = await connectToStorage()

  // success
  connection.map(db => {
    seneca()
      .use(storage)
      .listen()
  })

  // failure
  .orElse(e => {
    console.log('Connection to Mongo instance failed:', e)
  })
})
```

We've introduced a few new dependencies here:
 - In `storage-listener.js`, we import something called `iife` from `../../util`.  `iife` is just a very simple wrapper around an `async` function that cleans up our code a bit:

```js
iife(async () => {
  await someAsyncThing()
})
```
vs.
```js
(async () => {
  await someAsyncThing()
})()
```

A subtle difference, and purely aesthetic.  But it conveys intent a bit better (iife = Immediately-Invoked Function Expression).

Let's create `src/util/index.js`, which will house `iife` and other helpful functions.

`src/util/index.js`:

```js
// @flow
import R from 'ramda'
import Result from 'folktale/result'

export async function iife(fn : () => any) {
  await fn()
}

export async function _try(fn : () => any) {
  try {
    let r = await fn()
    return Result.Ok(r)
  }
  catch (e) {
    return Result.Error(e.message)
  }
}
```

You'll notice we added another function: `_try`.  This leads us to how our services will communicate with eachother.  Up until now we've made heavy use of traditional `try/catch`.  Because we've been running our "services" inside the same process, a `try/catch` works fine.  But our microservices will now run in separate processes (or even machines), and the `gateway` service can't just `catch` an exception from the `storage` service in such an environment.

To facilitate easy communication between services, we need a way to represent:
 - a successful call with a result value
 - a failed call with an error, describing the failure


#### Enter Folktale
We'll use a library called [Folktale](http://folktale.origamitower.com/) which contains lots of helpful stuff.  In particular, [Result](http://folktale.origamitower.com/api/v2.0.0/en/folktale.result.html) is just what we wished for.  It describes a successful or failed call and wraps the result, be it value or error.  It is easily serializeable via `.toJSON()`/`fromJSON()`, so it works well across service boundaries.

Back to our our new function from above, `_try`.  It executes an asynchronous function, and if successful, wraps and returns the result in `Result.Ok(...)`.  If the call failed, it returns `Result.Error(...)`.

For an example, check out our `storage-listener.js` above.  The value returned from `await connectToStorage()` is a `Result`.  

If the call was successful, the `map` handler will execute and pass our DB connection context.  In the handler, we start the Seneca listener, and tell it to use the storage plugin, which defines which patterns the storage service listens for.

But if the call to `connectToStorage` failed for any reason (maybe `mongod` isn't running), the `orElse` handler is executed with a string describing the error, which is logged, and then the process dies.  This is exactly what we want.

So given this new information, an overhaul of `storage/index.js` is in order:

```js
// @flow
import { MongoClient } from 'mongodb'
import R from 'ramda'
import { _try } from '../../util'
import shortid from 'shortid'

//** URL where Mongo server is listening
const url = 'mongodb://localhost:27017/bnb-book'

// Variable that holds the connection to Database.
let db

export async function connectToStorage() {
  return _try(async () => {
    db = await MongoClient.connect(url)
    return db
  })
}

export async function insertOne(collection : string, item : Object) {
  return _try(async () => {
    const itemWithId = R.assoc('id', shortid.generate(), item)
    const result = await db.collection(collection).insert(itemWithId)
    return result.ops[0]
  })
}

export async function fetchOne(collection : string, id : string) : Object {
  return _try(async () => {
    return await db.collection(collection).findOne({ id })
  })
}

export async function updateOne(collection : string, id : string, input : Object) : Object {
  return _try(async () => {
    let result = await db.collection(collection).findOneAndUpdate(
      { id },
      { $set: input },
      { returnOriginal: false }
    )
    return result.value
  })
}

export async function deleteOne(collection : string, id : String) : Object {
  return _try(async () => {
    let result = await db.collection(collection).findOneAndDelete({ id })
    return result.value
  })
}
```

One more thing.  While Mongo automatically generates an `_id` property on all objects, it's a long, not-so-user-friendly ID that looks odd in URLs.  We'll actually use the library `shortid` to generate a URL friendly ID on each object we create, referenced by an `id` property.  This also frees us from juggling between GraphQL expecting (requiring, in fact) an `id` property and Mongo's default `_id`.

`npm i --save shortid`

At this point, our `storage` service is a self-contained, standalone service that will be run as a separate process.  In a separate terminal window:

`./node_modules/.bin/babel-node src/services/storage/storage-listener.js`

You should see something like:

```bash
{"kind":"notice","notice":"hello seneca 4q9l5u9if3pb/1498981616014/8678/3.3.0/-","level":"info","when":1498981616443}
```

Also run `npm run watch` in `server` to start the `gateway` service.

Now if we run our curl commands from earlier:

`curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "mutation CreateUser($input: UserInput) { createUser(input: $input) { id email firstName lastName } }", "args": { "input": {  "email": "kevin@dundermifflin.com", "firstName": "Kevin", "lastName": "Malone" } } }'`

We'll get:

`{"data":{"createUser":{"id":"ry7GUNI4Z","email":"kevin@dundermifflin.com","firstName":"Kevin","lastName":"Malone"}}}`

Now to fetch:

`curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "query FetchUser($id: ID!) { fetchUser(id: $id) { id email firstName lastName } }", "args": { "id": "ry7GUNI4Z" } }'`

Returns:

`{"data":{"fetchUser":{"id":"ry7GUNI4Z","email":"kevin@dundermifflin.com","firstName":"Kevin","lastName":"Malone"}}}`

Woohoo!
