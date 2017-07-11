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
- `storage` - a service to interact with our Mongo database to perform C.R.U.D. operations
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
- Update the `"main"` property:

 ```json
 "main": "src/services/gateway/index.js",
 ```
- Update the `"scripts"."babel-node"` property:

 ```json
 "babel-node": "./node_modules/.bin/babel-node src/services/gateway/index.js",
 ```

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
    reply(result.toJSON())
  })

  .add({ role: ROLE_NAME, cmd: 'insertOne' }, async (msg, reply) => {
    const result = await insertOne(msg.type, msg.input)
    reply(result.toJSON())
  })

  .add({ role: ROLE_NAME, cmd: 'updateOne' }, async (msg, reply) => {
    const result = await updateOne(msg.type, msg.id, msg.input)
    reply(result.toJSON())
  })

  .add({ role: ROLE_NAME, cmd: 'deleteOne' }, async (msg, reply) => {
    const result = await deleteOne(msg.type, msg.id, msg.input)
    reply(result.toJSON())
  })
}
```

`storage-listener.js`:
```js
// @flow
import seneca from 'seneca'
import { connectToStorage } from '.'
import { iife } from '../../bnb-book-util'
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
 - In `storage-listener.js`, we import something called `iife` from `../../bnb-book-util`.  `iife` is just a very simple wrapper around an `async` function that cleans up our code a bit:

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

Let's create `src/bnb-book-util/index.js`, which will house `iife` and other helpful functions.

`src/bnb-book-util/index.js`:

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

To facilitate easy communication between services, we need a serializeable way to represent:
 - a successful call with a result value
 - a failed call with an error, describing the failure

#### Enter Folktale
We'll use a library called [Folktale](http://folktale.origamitower.com/) which contains lots of helpful stuff.  In particular, [Result](http://folktale.origamitower.com/api/v2.0.0/en/folktale.result.html) is just what we wished for.  It describes a successful or failed call and wraps the result, be it value or error.  It is easily serializeable via `.toJSON()`/`.fromJSON()`, so it works well across service boundaries.

`npm i --save folktale`

Back to our our new function from above, `_try`.  It executes an asynchronous function, and if successful, wraps and returns the result in `Result.Ok(...)`.  If the call failed (i.e., an exception is thrown), it returns `Result.Error(...)`.

For an example, check out our `storage-listener.js` above.  The value returned from `await connectToStorage()` is a `Result`.

If the `connectToStorage()` call was successful, the `map` handler will execute and pass our DB connection context (the unwrapped value).  In the handler, we start the Seneca listener, and tell it to use the `storage` plugin, which defines which patterns the storage service listens for.

But if `connectToStorage()` failed for any reason (maybe `mongod` isn't running), the `orElse` handler is executed with a string describing the error, which is logged, and then the process dies.  This is exactly what we want.

So given this new information, an overhaul of `storage/index.js` is in order:

```js
// @flow
import { MongoClient } from 'mongodb'
import R from 'ramda'
import { _try } from '../../bnb-book-util'
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

export async function disconnectFromStorage() {
  return _try(async () => {
    await db.close()
    db = null
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

As mentioned in Part #3, while Mongo automatically generates an `_id` property on all objects, it's a long, not-so-user-friendly ID that looks odd in URLs.  We'll use the library `shortid` to generate a URL friendly ID on each object we create, referenced by an `id` property.  This also frees us from juggling between GraphQL expecting (requiring, in fact) an `id` property and Mongo's default `_id`.

`npm i --save shortid`

At this point, our `storage` service is a self-contained, standalone service that will be run as a separate process.  In a separate terminal window:

`./node_modules/.bin/babel-node src/services/storage/storage-listener.js`

You should see something like:

```bash
{"kind":"notice","notice":"hello seneca 4q9l5u9if3pb/1498981616014/8678/3.3.0/-","level":"info","when":1498981616443}
```

Also run `npm run watch` in `server` to start the `gateway` service.

Now if we run our curl commands from earlier:

```bash
curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "mutation CreateUser($input: UserInput) { createUser(input: $input) { id email firstName lastName } }", "args": { "input": {  "email": "dwight@dundermifflin.com", "firstName": "Dwight", "lastName": "Schrute" } } }'
```

We'll get:

```bash
{"data":{"createUser":{"id":"ry7GUNI4Z","email":"dwight@dundermifflin.com","firstName":"Dwight","lastName":"Schrute"}}}
```

Now to fetch (remember to change the ID!):

```bash
curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "query FetchUser($id: ID!) { fetchUser(id: $id) { id email firstName lastName } }", "args": { "id": "ry7GUNI4Z" } }'
```

Returns:

```bash
{"data":{"fetchUser":{"id":"ry7GUNI4Z","email":"dwight@dundermifflin.com","firstName":"Dwight","lastName":"Schrute"}}}
```

Woohoo!  

> For now, we can only query by ID.  Don't worry, we'll add more flexible calls to our API later.  For now, if you need a way to find IDs to query by, you can use the Mongo CLI: just run `mongo`, to open a shell into the Mongo API.  Then `use bnb-book` will switch to our database.  Run `db.getCollection('User').find().toArray()` to get all available `User`s.

Now, let's update our Mongo tests.  Change `__tests__/mongo-tests.js` to the following:

```js
import { connectToStorage, disconnectFromStorage, insertOne, fetchOne, deleteOne, updateOne } from '../src/services/storage'

let db;
beforeAll(async () => {
  (await connectToStorage())
  .map(_db => {
    db = _db
    //console.log("Connected to Mongo")
  })
  .orElse(e => {
    console.log(e)
    process.exit(1)
  })
})

afterAll(async () => {
  await disconnectFromStorage()
})

test('insertOne creates a document, deleteOne deletes it', async () => {
  const insertResult = await insertOne('testDocuments', { test: 1 })
  const inserted = insertResult.getOrElse('failure')
  expect(inserted).toHaveProperty('test', 1)

  const fetchResult = await fetchOne('testDocuments', inserted.id)
  const fetched = fetchResult.getOrElse('failure')
  expect(fetched).toHaveProperty('test', 1)

  const deleteResult = await deleteOne('testDocuments', fetched.id)
  const deleted = deleteResult.getOrElse('failure')
  expect(deleted).toEqual(fetched)
});

test('updateOne modifies a document', async () => {
  const insertResult = await insertOne('testDocuments', { test: 1 })
  const inserted = insertResult.getOrElse('failure')
  expect(inserted).toHaveProperty('test', 1)

  const updateResult = await updateOne('testDocuments', inserted.id, { newKey: 'boom' })
  const updated = updateResult.getOrElse('failure')
  expect(updated).toHaveProperty('newKey', 'boom')

  const deleteResult = await deleteOne('testDocuments', updated.id)
  const deleted = deleteResult.getOrElse('failure')
  expect(deleted).toEqual(updated)
});
```

#### Embrace the package.json

There's a glaring issue with our architecture right now.  We want each service to be independently deployable, but our `server` folder has a single, over-arching `package.json` file.  We *could* just copy this file into the build folder for each service, but that's a bit ham-fisted: `storage` doesn't need the `express` or `graphql` dependencies, and `gateway` doesn't need the `mongodb` or `shortid` dependencies.  We will instead add a `package.json` for each service.  While a bit annoying, it does make sense: each service should be responsible for tracking its own dependencies.

In `server/src/services/storage`:

`npm init --yes && npm i --save folktale mongodb ramda seneca shortid`

This will create `storage/package.json` and add the service's dependencies.  Additionally, we will remove them from `server/package.json` (which we will keep, as it contains all the Babel stuff we don't need to include in the service-specific `package.json` files).

What about our shared/common code in the `bnb-book-util` folder?  We'll take the easy way and add `bnb-book-util` as a local dependency to `storage/package.json`.  But first, we must add (yes, another) `package.json` file to the `bnb-book-util` folder, because `npm` requires it to install it as a dependency in other packages.  To do so, run the following inside the `bnb-book-util` directory:

`npm init --yes && npm install folktale`

Then, in `storage`:

`npm install --save file:../../bnb-book-util`

> A more proper solution would be to create a private npm repo for `bnb-book-util` and each service would add it as a dependency.  But for now, this will serve us fine.

Let's also package-ize our `gateway` service.  In `server/src/services/gateway`:

`npm init --yes && npm i --save folktale body-parser express graphql seneca`

Repeat the addition of `bnb-book-util` as a local dependency (see above).

#### Service deployment structure

The previous changes will make it possible to deploy our services independently.  When we create a deployment artifact for a service, it will consist of:
 - the microservice directory's contents (transpiled with Babel to basic javascript)
 - the microservice's `package.json`, so that it can `npm install` its dependencies
 - local dependencies, e.g., the `bnb-book-util` directory

For example, let's say we'll deploy the `storage` service via a Docker image.  It's structure will be:

```
package.json
index.js
storage-listener.js
storage-patterns.js
bnb-book-util/
   index.js
```

But we're not out of the woods yet.  With structure outlined above, our services will be attempting to load `bnb-book-util` from `../../bnb-book-util`, which will (attempt to) escape the root of the image.  This leads us to a new script we will write: `scripts/build-service.sh`:

```bash
# capture the first parameter as $SERVICE
SERVICE=$1

# check that service is not empty
if [ -z "$SERVICE" ]; then
  echo "build-service.sh requires a single argument: the name of a directory src/services"
  exit
fi

# check that the service exists
if [ ! -d src/services/$SERVICE ]; then
  echo "Invalid service: src/services/$SERVICE does not exist!"
  exit
fi

echo "**********************************"
echo "Building src/services/$SERVICE"
echo "**********************************"

# delete and recreate a build/{serviceName} directory
rm -rf build/$SERVICE

# copy service directory contents (excluding node_modules)
rsync -r --exclude=node_modules src/services/$SERVICE build

# copy bnb-book-util directory
rsync -r --exclude=node_modules src/bnb-book-util build/$SERVICE

# build babel-transformed javascript in out/
./node_modules/.bin/babel build/$SERVICE --out-dir build/$SERVICE

# update the package.json's dependencies.bnb-book-util path
./node_modules/.bin/json -I -f build/$SERVICE/package.json -e 'this.dependencies["bnb-book-util"]="file:./bnb-book-util"'
./node_modules/.bin/json -I -f build/$SERVICE/package-lock.json -e 'this.dependencies["bnb-book-util"].version="file:./bnb-book-util"'
```

This is a very basic bash script which takes a service name as a parameter, and builds a deployable service.  Here are the steps it takes:
 - Validates that the service passed in is non-empty and matches a directory in `src/services`
 - Copies the directory contents into a build folder of the same name
 - Copies the `bnb-book-util` package/directory into the build folder
 - Uses Babel to transpile the ES* to valid javascript, runnable with plain Node.js without Babel
 - Edits `package.json` (and `package-lock.json`) dependency path for `bnb-book-util`, so that Node doesn't attempt to escape the root of the directory

Now we can run `./scripts/build-scripts.sh storage` and we'll have a deployable `storage` service in `build/storage`.

One last thing: our use of `async/await` requires an npm module named `regenerator-runtime`, which we get for free when running our services locally via `babel-node`.  But since we'll run our deployed services via just Node, we must:
 - Run `npm install --save regenerator-runtime` in each of our service directories
 - Add an `import regenerator-runtime/runtime` statement to each `services/*/index.js` file

Without these changes, our deployed services will incur a `regeneratorRuntime is not defined` error on startup.

#### Convert the `gateway` to a Seneca service

Our `gateway` service should be a Seneca microservice as well, for the following reasons:
 - Consistency: each of our services will have an identical structure (namely the `*-listener/patterns.js` convention)
 - We can further take advantage of the capabilities that Seneca provides automatically (logging, easy changing of communication mechanism, etc.)
 - Decouples our API from our API technology.  If we want to switch from Express to Hapi, it's much easier: simply use the Seneca-Hapi plugin instead of the Seneca-Express plugin.

Seneca provides a web plugin that integrates with Express, so our changes won't be too drastic.  In `services/gateway`, install the needed modules:

`npm i --save seneca-web seneca-web-adapter-express`

Now we'll split `gateway/index.js` into `gateway-listener.js` and `gateway-patterns.js`:

`gateway-listener.js`:

```js
// @flow
import 'regenerator-runtime/runtime'
import Seneca from 'seneca'
import SenecaWeb from 'seneca-web'
import Express from 'express'
import SenecaWebExpress from 'seneca-web-adapter-express'
import BodyParser from 'body-parser'
import gateway from './gateway-patterns'

const Router = Express.Router
const context = new Router()
const senecaWebConfig = {
  context: context,
  adapter: SenecaWebExpress,
  options: { parseBody: false } // so we can use body-parser
}

const app = Express()
  .use(BodyParser.json())
  .use(context)
  .listen(3001)

Seneca()
  .use(SenecaWeb, senecaWebConfig)
  .use(gateway)
  .client({ type:'tcp', pin: 'role:gateway' })
```

`gateway-patterns.js`:

```js
// @flow
import { graphql, buildSchema } from 'graphql'
import fs from 'fs'
import { promisify } from 'util'
import Root from './resolvers'

/**
  Seneca plugin for our API gateway
*/
export default async function gateway(options) {

  this
  .add('role:gateway, path:graphql', async (msg, reply) => {
    const { query, args } = msg.args.body
    const result : Object = await graphql(schema, query, Root, { user: 'Bill' }, args)
    reply(result)
  })

  .add('init:gateway', (msg, reply) => {
    this.act('role:web', {
      routes: {
        //prefix: 'v0',
        pin: 'role:gateway, path:*',
        map: {
          graphql: {
            POST: true
          }
        }
      }
    }, reply)
  })

  const readFile : (string, string) => Promise<string> = promisify(fs.readFile)
  const gql : string = await readFile(`${__dirname}/schema.graphql`, 'utf8')
  const schema : Object = buildSchema(gql)
}
```

The `gateway-patterns.js` file defines two Seneca patterns via `.add(...)`:
 - `'role:gateway, path:graphql'` is called by Seneca on a `POST /graphql`.
 - `'init:gateway'` is called by Seneca at service startup (because of our `.client({ type:'tcp', pin: 'role:gateway' })` line in `storage-listener.js`).  Its handler sends a message via the special `role:web` pin which contains the routes to listen on and contains other configuration.  Seneca-Web uses this message's contents to configure the Express context.

> Now, we can delete `gateway/index.js`.

> If we really wanted to run with this pattern, we could even split our GraphQL logic off entirely into its own service.  But for now, this is maybe a bit of over-architecting, so we'll keep it simple.

#### Service `package.json` updates and running locally
For convenience, let's add some commands to help us when running locally.

In `storage/package.json`, update the `scripts` node like so:

```json
"start": "node --require babel-register storage-listener.js",
"watch": "nodemon --exec \"npm start\"",
```

And in `gateway/package.json`:

```json
"start": "node --require babel-register gateway-listener.js",
"watch": "nodemon --exec \"npm start\"",
```

Finally, let's alter our `server/package.json` file to make it easy to get our services up and running.  Add this to the `scripts` node:

```json
"services": "find src/services/ -type d -maxdepth 1 -mindepth 1 -exec echo '\"cd {}; npm run watch;\"' \\; | xargs ../node_modules/.bin/concurrently || true",
```

Then, `npm run services` will start up any service that has a `*-listener.js` file via Concurrently.  So an `npm start` in the `client` directory, and an `npm run services` in `server` would have the entire app up and running, which leads us to a final update of our top-level `package.json`.  Change the `scripts.start` command:

```json
"start": "./node_modules/.bin/concurrently \"cd client && npm start\" \"cd server && npm run services\""
```

#### Wrapping up

We've built out the main pattern we'll use for our backend.  While we've still got a ways to go to whip our backend into production shape, this will serve us well for now as we can finally get down to some business logic and UI!  Read on to see how we'll structure our frontend so we can start booking some rooms!
