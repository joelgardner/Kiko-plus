---
layout: post
title: "A modern JS app (III): backend architecture"
description: "Part 3: In this post, we'll continue setting up our backend by bringing GraphQL and Mongo into the mix. At the end, our client will be speaking GraphQL and our backend will perform C.R.U.D. operations at a 5th grade level."
date: 2017-06-19
tags: [javascript, node, graphql, mongo, jest]
comments: true
share: true
---

## A modern React / Node.js application

### Part 3: Setting up our Backend... continued

> NOTE: You'll need to be running Node version >= 8.0 for the following.  So if you don't have that one installed, do so.  Check out the [nvm](https://github.com/creationix/nvm) package if you'd like some assistance.

#### Concurrently
You might've been bothered by the fact that to run our application, we have to run two commands inside two different terminal windows: in `client`, we run `npm start` and in `server`, we run `npm run watch`.

We'll use [Concurrently](https://www.npmjs.com/package/concurrently) to fix this by running everything with a single command.  

In our top level directory, run:

`npm init --yes && npm i --save-dev concurrently`

This creates a `package.json` and installs the Concurrently tool.  We will add the following line to the `scripts` key in `package.json`:

`"start": "./node_modules/.bin/concurrently \"cd client && npm start\" \"cd server && npm run watch\""`

Kill the `server`/`client` processes if necessary, and then run `npm start` in the top-level folder.  This will start both the client and server applications with a single command.  Much more convenient!

#### Real world example
Our calculator app is marvelous, but let's build something a bit more realistic.  We'll create a simple booking application for B&Bs.  The basic features we'll build are:
- a way to manage room inventory.  Rooms will have a name, price, description, and an image
- a way for guests to browse and book rooms by date
- a way for B&B owners to sign up for our app (guests however can book anonymously)

#### Backend business logic / models
As stated earlier, we'll use GraphQL to serve our API.  GraphQL provides a language that describes our API and doubles as a way to automatically build the schema as well.  Let's define our models in a new file,  `src/schema.graphql`:

```
type User {
  id: ID!
  email: String!
  firstName: String
  lastName: String
}

input UserInput {
  email: String
  firstName: String
  lastName: String
}

type LocalAuth {
  id: ID!
  user: User!
  password: String!
}

type Property {
  id: ID!
  owner: User!
  address: Address
  rooms: [Room]
}

type Address {
  street1: String!
  street2: String
  city: String!
  state: String!
}

type Room {
  id: ID!
  name: String!
  price: Float
  description: String
  image: File
}

type File {
  id: ID!
  url: String!
}
```

This is a very basic schema that will allow us to build our B&B Booking app.  It simply creates some types that define the data we need to manage and book properties.  

>As far as schemas go, this is about as simple as you can get in GraphQL, which is capable of much more than what we'll use for this simple application.  Go ahead and read the GraphQL [documentation](http://graphql.org/learn/) to get a feel for the possibilities.

Now let's install the GraphQL library:

`npm i --save graphql`

#### Creating Users

Let's implement an API call to create users, and then some inventory.  Add the following to our `schema.graphql` file:

```
type Query {
  fetchUser(id: ID!): User
}

type Mutation {
  createUser(input: UserInput): User
  updateUser(id: ID!, input: UserInput): User
  deleteUser(id: ID!): User
}
```
Create a `gateway` folder in `src`, and inside it, add a `resolvers.js` with contents:

```js
import R from 'ramda'

const users = [
  { id: 0, email: 'toby@dundermifflin.com' },
  { id: 1, email: 'jim@dundermifflin.com' },
  { id: 2, email: 'pam@dundermifflin.com' },
  { id: 3, email: 'dwight@dundermifflin.com' },
  { id: 4, email: 'michael@dundermifflin.com' },
  { id: 5, email: 'andy@dundermifflin.com' },
]

export function fetchUser({ id }, context) {
  return users[id]
}

export function createUser({ input }, context) {
  users.push({ id: users.length, email: input.email, firstName: input.firstName, lastName: input.lastName })
  return R.last(users)
}
```

Finally, change our `src/index.js` file to look like the following:

```js
// @flow
'use strict'
import express from 'express'
import bodyParser from 'body-parser'
import { graphql, buildSchema } from 'graphql'
import fs from 'fs'
import { promisify } from 'util'
import * as Root from './gateway/resolvers'

init()

async function init() {
  const app = express()
  app.use(bodyParser.json())
  app.use(bodyParser.urlencoded({ extended: true }))

  // read in the schema.graphql file, and build our schema with it
  const readFile : (string, string) => Promise<string> = promisify(fs.readFile)
  const gql : string = await readFile(`${__dirname}/schema.graphql`, 'utf8')
  const schema : Object = buildSchema(gql)
  app.post('/graphql', async (req, res) => {
    const { query, args } = req.body
    const result : Object = await graphql(schema, query, Root, { user: 'Bill' }, args)
    res.send(result)
  })

  app.listen(3001, () => {
    /* eslint-disable no-console */
    console.log('Listening on port 3001.')
    /* eslint-enable no-console */
  })
}
```

We're now listening at `/graphql` for requests.  Let's test our endpoint.  Run the following curl request:

`curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "query FetchUser($id: ID!) { fetchUser(id: $id) { id email } }", "args": { "id": "3" } }'`

You should be getting the response:

`{"data":{"getUser":{"id":"3","email":"dwight@dundermifflin.com"}}}`

Woohoo!  Let's create a new user:

`curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "mutation CreateUser($input: UserInput) { createUser(input: $input) { id email } }", "args": { "input": { "email": "kevin@dundermifflin.com", "firstName": "Kevin", "lastName": "Malone" } } }'`

This should return:

`{"data":{"createUser":{"id":"6","email":"kevin@dundermifflin.com"}}}`

It's good to include Kevin.  If you run the above GraphQL *query* (as opposed to the *mutation*), with the `id` changed to `6`, we'll also get Kevin back, proving he's now in our "database."

I used quotes around the word *database* because as I'm sure you've noticed, we're simply using an in-memory array to store our users.  Let's change that.

#### Enter Mongo
If you haven't already, [install Mongo](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-os-x/).  

Done?  Created your `/data/db` directory?  Run the `mongod` command to make sure everything's set.  *CTRL-C* out of it and let's add that to our top-level `package.json`'s `start` command:

`"start": "./node_modules/.bin/concurrently \"cd client && npm start\" \"cd server && npm run watch\" \"mongod\""`

Now when we run `npm start` in our base directory, we'll kick off 3 processes: the client and server apps, and Mongo.

> Alternatively, you may prefer just to maintain an extra (permanent) terminal window to run the Mongo server from, without adding the `mongod` command to our `npm start`, as it adds a bit of startup time.

Let's now install the Node.js mongodb driver.  In `server/`:

`npm install mongodb --save`

Let's add a `storage` folder under `server`.  It will house the files we use for persistence to our Mongo instance:

`mkdir storage && touch storage/index.js`

To start, we'll use Mongo's singular C.R.U.D. operations: `insertOne`, `findOne`, `findOneAndUpdate`, and `findOneAndDelete`.  They map nicely to the initial [mutations](http://graphql.org/learn/queries/#mutations) we'll create for our GraphQL schema.

We'll also use a library called `shortid` to generate URL-friendly IDs for our entities.  Mongo generates a long alphanumeric ID that is fine, but it's good practice not to expose internal IDs.  Additionally, it allows us to not have to juggle between GraphQL expecting an `id` property and Mongo's `_id`.

In `storage/index.js`, we'll have the following:

```js
// @flow
import 'babel-polyfill'
import { MongoClient } from 'mongodb'
import R from 'ramda'
import shortid from 'shortid'

//** URL where Mongo server is listening
const url = 'mongodb://localhost:27017/bnb-book'

/**
  Variable that holds the connection to Database.
*/
let db

export async function connectToStorage() {
  try {
    db = await MongoClient.connect(url)
    return db
  }
  catch (e) {
    console.log(e)
    throw e
  }
}

export async function disconnectFromStorage() {
  try {
    await db.close()
    db = null
  }
  catch (e) {
    console.log(e)
    throw e
  }
}

export async function insertOne(collection : string, item : Object) {
  try {
    const itemWithId = R.assoc('id', shortid.generate(), item)
    const result = await db.collection(collection).insert(itemWithId)
    return result.ops[0]
  }
  catch (e) {
    console.log(e)
    throw e
  }
}

export async function fetchOne(collection : string, id : string) : Object {
  try {
    return await db.collection(collection).findOne({ id })
  }
  catch (e) {
    console.log(e)
    throw e
  }
}

export async function updateOne(collection : string, id : string, input : Object) : Object {
  try {
    let result = await db.collection(collection).findOneAndUpdate(
      { id },
      { $set: input },
      { returnOriginal: false }
    )
    return result.value
  }
  catch (e) {
    console.log(e)
    throw e
  }
}

export async function deleteOne(collection : string, id : String) : Object {
  try {
    let result = await db.collection(collection).findOneAndDelete({ id })
    return result.value
  }
  catch (e) {
    console.log(e)
    throw e
  }
}
```

> Note that we're defining a `const url` to hold our Mongo server's URL.  Eventually, we'll need to be more robust with this and use a proper configuration library.  But for now, this is fine.

Most methods are self explanatory.  `connectToStorage` simply creates a connection to the Mongo server and returns the context if successful.  Collections are strings that reference the document-type we're dealing with: `User`s, `Room`s, `File`s, etc.  

We're using `async`/`await` here so that we can use a simple `try/catch` to handle any errors that popup when interacting with Mongo.  For now we'll just log and rethrow the `Error` object passed to the `catch`.

Let's not forget our tests!  We'll write some simple tests in a new file, `__tests__/mongo-tests.js`:

```js
import { connectToStorage, insert, select, remove, update } from '../src/storage'

beforeAll(async () => {
  let db = await connectToStorage()
})

// technically not exactly a "unit" test but will do for now
test('insertOne creates a document, updateOne updates it, fetchOne retrieves it, deleteOne removes it', async () => {
  let result = await insertOne('testDocuments', { test: 1 })
  expect(result).toHaveProperty('test', 1)

  let updated = await updateOne('testDocuments', result.id, { test2: 2 })
  expect(updated).toHaveProperty('test2', 2)

  let fetched = await fetchOne('testDocuments', result.id)
  expect(fetched.id).toBe(result.id)
  expect(fetched).toHaveProperty('test2', 2)

  let deleted = await deleteOne('testDocuments', fetched.id)
  expect(deleted.id).toBe(result.id)
  expect(deleted).toHaveProperty('test2', 2)
});
```

In addition to verifying that everything's running smoothly, these tests provide a way to see how our `storage` service will work.

One more thing.  We need to have our main script call `connectToStorage` when the app loads:  we'll just add these two lines after the `app.use(bodyParser.urlencoded({ extended: true }))` line in `src/index.js`:

```js
// mongo setup
await connectToStorage()
```

We just wait for the `connectToStorage` function to finish, and discard the return value (we don't need it).

Now that our Mongo server is creating/reading/updating/deleting, we can start to build our queries and mutations.

But wait, there's more!  We need to set up some basic data-integrity checks in our Mongo schema.  For example, we don't want two  `User`s with the same email, or two `LocalAuth`s with the same usernames, of course.

Let's introduce another tool called `db-migrate`, which will help us in these situations where we need to tell our Mongo instance how to behave.

`npm i --save-dev db-migrate db-migrate-mongodb`

Now let's create a `scripts` under our `server` directory to hold these database scripts.  In `server`:

`mkdir -p scripts/database/migrations && touch scripts/database/database.json`

We also created a `database.json` file which we'll populate with these contents:

```json
{
  "dev": {
    "driver": "mongodb",
    "database": "bnb-book",
    "host": "localhost"
  }
}
```

To make working with `db-migrate` a bit more convenient, let's add some commands to our server's `package.json`'s `scripts` property:

```bash
"db-migrate-up": "./node_modules/.bin/db-migrate up --config scripts/database/database.json --migrations-dir scripts/database/migrations/",
"db-migrate-down": "./node_modules/.bin/db-migrate down --config scripts/database/database.json --migrations-dir scripts/database/migrations/",
"db-migrate-create": "./node_modules/.bin/db-migrate create --config scripts/database/database.json --migrations-dir scripts/database/migrations/",
"db-migrate-dropdb": "read -p \"*** Are you sure? Entering 'yes' or 'Y' will remove bnb-book DB from your local Mongo instance.\n> \" choice && case \"$choice\" in yes|Y ) ./node_modules/.bin/db-migrate db:drop bnb-book --config scripts/database/database.json;; * ) echo \"Nothing to do.\";; esac",
"db-migrate-createdb": "./node_modules/.bin/db-migrate db:create bnb-book --config scripts/database/database.json"
```

Let's go over what these commands do:
- `npm run db-migrate-up` will run all of our migrations' `up` functions.
- `npm run db-migrate-down` will run a single migration's `down` function (this is a `db-migrate` default).  To run more than one, you can run `npm run db-migrate-down -- -c 2`, the *2* represents the number of down migrations you want to run (from the current state).
- `npm run db-migrate-create -- createStuff` will create a new migration file, e.g., `20170630070556-createStuff.js`.
- `npm run db-migrate-dropdb` will drop our *bnb-book* database in our Mongo instance.  It guards against an accidental execution with a prompt.
- `npm run db-migrate-createdb` will create an empty *bnb-book* database in our Mongo instance.

Then, we'll create our first migration, which will create the collections we need:

`npm run db-migrate-create -- createCollections`

This will create a new file in `server/scripts/database/migrations` that looks like `20170629172134-createCollections.js`.  Your numbers will be different; they just represent the current date.  The file contains two exports called `up` and `down`.  These functions will be used to perform a migration (`up`) and rollback a migration (`down`).  Let's complete both here:

```js
exports.up = function(db) {
  // create collections
  return Promise.all([
    db.createCollection('User'),
    db.createCollection('LocalAuth'),
    db.createCollection('Property'),
    db.createCollection('Address'),
    db.createCollection('Room'),
    db.createCollection('File')
  ])
  // create unique indices
  .then(() => Promise.all([
    db.addIndex('User', 'idx_User_email', ['email'], true),
    db.addIndex('LocalAuth', 'idx_LocalAuth_username', ['username'], true)
  ]))
};

exports.down = function(db) {
  // drop all collections
  return Promise.all([
    db.dropCollection('User'),
    db.dropCollection('LocalAuth'),
    db.dropCollection('Property'),
    db.dropCollection('Address'),
    db.dropCollection('Room'),
    db.dropCollection('File')
  ])
};
```

We're simply creating a collection for each of our entities, and adding a unique index on `User.email` and `LocalAuth.username` (technically, we'll be using the email *as* the username -- but with this schema, we have a choice of using emails or regular, non-email usernames).

To test our migration, execute `npm run db-migrate-up`.  If you see the following, we're good to go:

```bash
[INFO] Processed migration 20170629172134-createCollections
[INFO] Done
```

> You can also test our `db-migrate-down` command.

Whew!  Lots of DB boilerplate, but that's all done.  Let's create some objects!  Look at `gateway/resolvers.js`.  It's using an in-memory array to track our users.  Let's change this and use our shiny new Mongo instance instead.  Change the contents of `resolvers.js` to:

```js
import { select, insert } from '../storage'

export async function getUser({ id }, context) {
  try {
    const results = await select('User', { _id: id })
    return Object.assign(results[0], { id : results[0]._id })
  }
  catch (e) {
    console.log(e)
    throw e
  }
}

export async function createUser(user, context) {
  try {
    return await insert('User', user)
  }
  catch (e) {
    console.log(e)
    throw e
  }
}
```

> Note that in the `catch` of `createUser` and `getUser`, we're logging the exception and simply re-throwing.  This will cause the API to return the exception message down the wire to the client.  This is at best, user-unfriendly, and at worst, a security issue.  Later, we'll add logic to return safe and user-friendly errors back to the client.

Let's test our changes.  Run our curl command from above:

`curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "mutation CreateUser($input: UserInput) { createUser(input: $input) { id email } }", "args": { "input": {  "email": "dwight@dundermifflin.com" } } }'`

We should get back `{"data":{"createUser":{"id":"5955fad170cc03b92a1c6199","email":"kevin@dundermifflin.com"}}}` the first time it's run (your ID will be different).  If we run the same command a second time, we should get back:

`{"errors":[{"message":"E11000 duplicate key error collection: bnb-book.User index: idx_User_email dup key: { : \"kevin@dundermifflin.com\" }","locations":[{"line":1,"column":40}],"path":["createUser"]}],"data":{"createUser":null}}`

Excellent!  Our response contains an `errors` array that contains a message describing the problem: we already added "kevin@dundermifflin.com" as a user.

Let's try our `getUser` function (change the ID here to the one you received above!):

`curl -X POST localhost:3000/graphql -H "content-type: application/json" -d '{ "query": "query GetUserById($id: ID!) { getUser(id: $id) { id email } }", "args": { "id": "5955fad170cc03b92a1c6199" } }'`

This should return:

`{"data":{"getUser":{"id":"5955fad170cc03b92a1c6199","email":"kevin@dundermifflin.com"}}}`

Excellent.  Two basic operations, but enough of a basis to start sending requests from the frontend.

#### Back to the front
While we're not done fleshing out our API, we can return to the frontend a bit.  Let's send a GraphQL request.  Change `client/src/App.js` to:

```js
import React, { Component } from 'react'
import './App.css'
import * as api from './api'

class App extends Component {
  constructor() {
    super()
    this.state = {
      email: '',
      id: null
    }
  }

  render() {
    return (
      <div className="App">
        <div>Create user:</div>
        <div>
          <span>Email</span>
          <input type="text" defaultValue={this.state.email} onChange={e => this.setState({ email: e.target.value })} />
        </div>
        <div>
          <button onClick={() => this.handleClick(this.state.email)}>Create User</button>
        </div>
        <span>{this.state.id === null ? '' : `The user's ID is ${this.state.id}`}</span>
        <span style={{ color:'#f00' }}>{this.state.error}</span>
      </div>
    )
  }

  async handleClick(email) {
    const result = await api.createUser(email)
    if (result.errors) {
      this.setState({ error: result.errors[0].message, id: null })
    }
    else {
      this.setState({ email: result.data.createUser.email, id: result.data.createUser.id, error: null })
    }
  }
}

export default App
```

And `client/src/api.js` to:

```js
export function createUser(email) {
  return graphql(
    `mutation CreateUser($input: UserInput!) {
      createUser(input: $input) {
        id
        email
      }
    }`,
    { input: { email } }
  )
}

function graphql(query, args) {
  return send(`/graphql`, {
    method: 'POST',
    headers: {
      'content-type': 'application/json'
    },
    body: JSON.stringify({ query, args })
  })
}

async function send(url, options) {
  const res = await fetch(url, options)
  if (!res.ok) {
    const err = new Error(await res.json())
    return Promise.reject(err)
  }
  return res.json()
}
```

And play around with our new form that creates users.  It displays the user's ID if successful, and the error if not.  Obviously we've got a long way to go for our app, but this is a start!

Read on, as we will fully defined our backend's architecture in the next post!
