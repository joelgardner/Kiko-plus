---
layout: post
title: "A modern JS app (V): Building our frontend with React, Redux, and Redux-Saga"
description: "Part 5: In this post, we'll take a deep dive into the React/Redux ecosystem with libraries such as Redux Saga and Redux Little Router to build our booking site's frontend.  We'll use Apollo to facilitate our API's GraphQL requests, and we'll  check use Immutable data-structures as our app state."
date: 2017-07-13
tags: [javascript, react, react-redux, redux, redux-saga, redux-little-router, immutable, apollo]
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
 - [immutable](https://facebook.github.io/immutable-js): yet another Facebook library that provides Clojure-like data-structures to wrap javascript's regular Array and Map objects.  Each Immutable object will track its changes and return a new reference if something actually changed.  This helps React and Redux determine whether or not to re-render components.  Admittedly, for such a simple app, it's a bit overkill.  But I wanted to demonstrate its utility.

> Note, we're actually *not* going to use [`react-apollo`](https://github.com/apollographql/react-apollo), the React bindings for the Apollo framework.  The reason is that I prefer to keep my React views very thin: they shouldn't care where their data came from, they should only be concerned with displaying it.  Still, check out the bindings and how they work, because they're very interesting way to fetch/mutate data, and you may actually prefer a "fatter" view.

> Facebook's [Relay](https://github.com/facebook/relay) is a similar library.  It is perhaps a bit more powerful but the learning curve is steeper.  There are many articles online that compare the two.  You can't really go wrong with either, I simply chose Apollo for this post due to its lower barrier of entry.

#### Configure our Redux store
Our app integrates a few different Redux plugins, so our configuration will be pretty hairy.  Here is the new `src/index.js`:

```js
import React from 'react'
import ReactDOM from 'react-dom'
import App from './Components/App/App'
import registerServiceWorker from './registerServiceWorker'
import { createStore, applyMiddleware, combineReducers, compose } from 'redux'
import { Provider } from 'react-redux'
import createSagaMiddleware from 'redux-saga'
import entitiesReducer from './Reducers/Entities'
import rootSaga from './Sagas/RootSaga'
import './index.css'
import { apolloClient } from './Api/ApolloProxy'
import { routerForBrowser, initializeCurrentLocation } from 'redux-little-router';
import routes from './Routes'

// initialize our router
const {
  reducer     : routerReducer,
  middleware  : routerMiddleware,
  enhancer    : routerEnhancer
} = routerForBrowser({ routes })

// build our store
const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  combineReducers({
    app     : entitiesReducer,
    router  : routerReducer,
    apollo  : apolloClient.reducer()
  }),
  {}, // initial state
  compose(
    routerEnhancer,
    applyMiddleware(sagaMiddleware, routerMiddleware, apolloClient.middleware()),
    (typeof window.__REDUX_DEVTOOLS_EXTENSION__ !== 'undefined') ? window.__REDUX_DEVTOOLS_EXTENSION__() : f => f,
  )
);

// kick off rootSaga
sagaMiddleware.run(rootSaga)

// render app
ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
registerServiceWorker()

// hot-reloading
if (module.hot) {
  module.hot.accept('./Components/App/App', () => {
    const NextApp = require('./Components/App/App').default
    ReactDOM.render(
      <Provider store={store}>
        <NextApp />
      </Provider>,
      document.getElementById('root')
    )
  })
}

// bootstrap the router with initial location
const initialLocation = store.getState().router;
if (initialLocation) {
  store.dispatch(initializeCurrentLocation(initialLocation));
}
```

This is much different from our original `index.js`.  Let's go over the changes:
- We're importing lots more functions; we see the expected stuff from `redux-saga`, `redux-little-router`, etc., but there are a few we haven't seen:
  - `entitiesReducer` will mutate our app's state as it fetches and displays entities (e.g., `Property`s or `Room`s) from the API
  - `rootSaga` will be our base saga, which for now, simply defines what should happen when a route changes
  - `apolloClient` is an Apollo [Client](http://dev.apollodata.com/core/apollo-client-api.html#apollo-client) object; we're using it as our API proxy, but the only reason we need to import now is to integrate it with our Redux store by calling its `.reducer()` function
  - `routes` is imported from a `./Routes.js` file we will soon create.  It defines each route and the saga that needs to be executed upon navigating to that route
- We initialize our `redux-little-router` with our `routes` object.
- We configure our Redux store to hold a state object that has three properties: `app`, `router`, `apollo`.  We'll mainly concern ourselves with `app`, because `router` and `apollo` are used by our router and API.  We're running the Redux middlewares for `redux-little-router` and `redux-saga`.  We're also telling Redux to use the [Redux DevTools](https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd?hl=en) extension, if available.  We then kick off the `rootSaga` (which simply listens for route-change events and runs the appropriate saga for the new route).
- The next part is familiar, just rendering the `App` component and setting up hot-reloading.
- Finally, we dispatch an `initializeCurrentLocation` action -- which comes from `redux-little-router` -- to kick everything off!

#### Routes
Let's define the routes we will expose:
- `/` will display a list of properties
- `/property/:id` will display a list of the property's rooms in a grid view.  Each room will be represented by an image, a short description, and the price per night.
- `/room/:id` will display the details of a property's room: images and a description.  It will also allow a user to book the room for a specified date range.
- `/manage` will display an "owner's dashboard", allowing property managers to add properties, rooms, and view bookings.
- `/manage/:id` will allow a property manager to edit property details.
- `/manage/room/:id` will allow a property manager to edit room details.

The last 3 routes will require a user to login to access them.

Let's create our `src/Routes.js` file:

```js
import homeSaga from './Sagas/RouteSagas/HomeSaga'
import propertyDetailsSaga from './Sagas/RouteSagas/PropertyDetailsSaga'

const routes = {
  '/': {
    title: 'Properties',
    saga: homeSaga
  },
  '/property/:id' : {
    title: 'Property Details',
    saga: propertyDetailsSaga
  },
  '/room/:id': {
    title: 'Room details'
  },
  '/manage': {
    title: 'Manage Properties',
    '/:id': {
      title: 'Manage Property'
    },
    '/room/:id': {
      title: 'Manage Room'
    }
  }
}

export default routes
```

Here we're importing two sagas: the `homeSaga` and `propertyDetailsSaga`.  We'll import more, but for now we'll concentrate on the `/` and `/property/:id` routes.  

The exported object is a simple map from the route to an object that is attached to the `ROUTER_LOCATION_CHANGED` action fired by `redux-little-router` on a route change.  

There's nothing special about the keys (e.g., `title`, `saga`) inside each object, save for the ones that start with `/`, as they signify a nested route.  The `/manage` route has two nested routes: `/:id` and `/room/:id`, which mean `/manage/:id` will be where a manager updates a `Property`, and `/manage/room/:id` will be where a manager updates a `Room`.

#### Directory structure
Let's add a few folders to `client/src`.  In `client:`

`mkdir src/Reducers src/Actions src/Components src/Sagas`

Every directory we created, except for `Components`, is Redux-specific.  One critique of Redux is that it's very boilerplate heavy.  That can be true ([it doesn't have to be](http://blog.isquaredsoftware.com/2017/05/idiomatic-redux-tao-of-redux-part-1/)).  It's a tradeoff I'm willing to make: we add a little more boilerplate, but gain much more declarative, readable, and... reason-about-able code.

- `Reducers` will contain functions that mutate our application state based on triggered actions
- `Actions` will contain our actions, which describe everything that can happen in our app
- `Components` will contain our Redux view-containers which map application state to component props and UI events to actions.  It will also hold our React components.  Each component-group will live in its own directory, e.g., `Components/PropertyList` will contain both `PropertyListContainer.js` and `PropertyList.js`
- `Sagas` will contain functions that describe app business logic and performing the side-effects required for our app to work

#### The Reducer
In our `Reducer` folder, let's add a file called `Entities.js`:

```js
import { combineReducers } from 'redux'
import { List, Map } from 'immutable'
import { FETCH_LIMIT } from '../Constants'

const initialPropertyState = Map({
  selectedItem: null,
  showing: -1,
  batches: List(),
  args: Map(),
  searchParameters: Map({
    sortKey: 'id',
    sortAsc: true,
    searchText: '',
    first: FETCH_LIMIT,
    skip: 0
  })
})

function Property(state = initialPropertyState, action) {
  switch(action.type) {
    case 'FETCH_ENTITIES':
      return state.withMutations(st => {
        st.update('showing', showing => showing + 1)
          .update('searchParameters', searchParameters => searchParameters.merge(action.searchParameters))
          .update('args', args => args.merge(action.args))
      })
    case 'FETCH_ENTITIES_SUCCESS':
      return state.update('batches', batches => batches.set(action.batchIndex, List(action.entities)))
    case 'FETCH_ENTITY_DETAILS_SUCCESS':
      return state.set('selectedItem', Map(action.entity))
    default:
      return state
  }
}

export default combineReducers({
  Property
})
```

This file defines an `initialState` object that is an `immutable` data-structure.  Let's go over the object's properties:
- `selectedItem` will point to the `Property` object used for the `/property/:id` route.  It's initial value is `null`.
- `showing` is a piece of metadata that determines how many batches of properties to display on the `/` (home) route, which will be infinitely-scrollable, where we pre-fetch the next batch of properties each time the user scrolls to the bottom.  More on this later (including why the initial value is `-1`)
- `batches` is an empty Immutable.List that will contain other Immutable.Lists of `properties`.  A `PropertyList` component will display the properties.
- `args` is an empty Immutable.Map that will hold any values used to fetch an item.
- `searchParameters` is an Immutable.Map that will determine things like sort order, search string, number of items to fetch, and how many to skip (i.e., an offset).  This is important for our infinite scrolling (or any pagination control).

Then we have our `Property` reducer, which takes the *current state* and an *action*, and returns the *new state*.
- On `FETCH_ENTITIES`, we increment `showing` and update our `args` and `searchParameters` (which are passed in by our Saga as you will see).
- On `FETCH_ENTITIES_SUCCESS`, we set the `batch` by the `batchIndex`. `batchIndex` is required because it is possible we have two requests going at once, and if request A takes longer than request B, we want to maintain the correct order, so we must make sure the batches from request B are slotted into the right index in our `batches` array (rather than simply `push()`ing onto the end).
- On `FETCH_ENTITY_DETAILS_SUCCESS`, we simply set `selectedItem` to the result from our API call.

#### Clientside API
As mentioned above, we'll use the [`apollo`](http://dev.apollodata.com/react) framework to make API requests.  It allows us to do things like define re-useable GraphQL fragments, so we can declaratively specify the data that each request expects.  As we saw earlier, it uses Redux under the hood, and thus we simply integrated with our own Redux store.  Install it now:

`npm i --save apollo-client `

We'll also install `graphql-tag` which allows us to represent GraphQL queries as interpolated strings:

`npm i --save graphql-tag`

Let's add our initial call to retrieve a list of properties from the server.  We'll create an `Api` folder and it will contain two files:

`src/Api/ApolloProxy.js`:

```js
import ApolloClient, { createNetworkInterface } from 'apollo-client'
import gql from 'graphql-tag'
import * as Fragments from './Fragments'

/**
  The API wraps an Apollo client, which provides query/mutation execution
  as well as fragment caching.
*/
export const apolloClient = new ApolloClient({
  networkInterface: createNetworkInterface({ uri: 'http://localhost:3000/graphql' })
})

/**
  @param {Object} args - Map of Property attribute names to values.
  @returns {Promise<Property>} Uses Apollo to fulfill fetchProperty query.
*/
export const fetchProperty = ({ id }) => apolloClient.query({
  query: gql`
    query FetchProperty($id: ID!) {
      fetchProperty(id: $id) {
        ... PropertyAttributes
        rooms {
          ... RoomAttributes
        }
      }
    }
    ${Fragments.Property.attributes}
    ${Fragments.Room.attributes}
  `,
  variables: {
    id
  }
})


/**
  @param {Object} args - Map of Property attribute names to values.
  @param {Object} search - Map of search parameters.
  @returns {Promise<List<Property>>} Uses Apollo to fulfill listProperties query.
*/
export const listProperties = (args, search) => apolloClient.query({
  query: gql`
    query ListProperties($args: PropertyInput, $search: SearchParameters) {
      listProperties(args: $args, search: $search) {
        ... PropertyAttributes
      }
    }
    ${Fragments.Property.attributes}
  `,
  variables: {
    args,
    search
  }
})

```

`src/Api/Fragments.js`:

```js
import gql from 'graphql-tag'

export const Property = {
  attributes: gql`
    fragment PropertyAttributes on Property {
      id
      street1
      street2
      city
      state
    }
  `,
  rooms: gql`
    fragment PropertyRooms on Property {
      rooms {
        ... RoomAttributes
      }
    }
  `
}

export const Room = {
  attributes: gql`
    fragment RoomAttributes on Room {
      id
      name
      price
      description
    }
  `
}
```

As you can see, `ApolloProxy.js` contains two methods -- `listProperties` and `fetchProperty` -- each of which send a GraphQL query of the same name to our server.  They both use the query fragments we've defined in `Fragments.js`, which allows us to not litter our API with multiple copies of lists of attributes for each API call.

#### What the hell is a Saga?
If you're asking this question, don't fret: a Saga is merely a way to describe business logic in a declarative way.  

Let's define our first saga: the `rootSaga`.  In `src/Sagas`, add `RootSaga.js`:

```js
import { all, takeLatest } from 'redux-saga/effects'
import navigationSaga from './NavigationSaga'

export default function* rootSaga() {
  yield takeLatest('ROUTER_LOCATION_CHANGED', navigationSaga)
}
```

It's very simple.  We are simply listening for the latest `ROUTER_LOCATION_CHANGED` action (which is triggered by our router), and when we see it, we execute the `navigationSaga`, which is defined in `NavigationSaga.js`:

```js
import { call } from 'redux-saga/effects'
import invalidRouteSaga from './RouteSagas/InvalidRouteSaga'

export default function* navigationSaga(action) {
  const location = action.payload
  const saga = location.result.saga || invalidRouteSaga
  yield call(saga, location)
}
```

This Navigation Saga takes the payload from the `ROUTER_LOCATION_CHANGED` message, and executes another saga.  Where does this magic "other" saga come from?  Back in `Routes.js`, we defined a `saga` property for our first two routes.  That's the saga being executed here.  Those two sagas were `homeSaga.js` and `propertyDetailsSaga`, which we will define in `Sagas/RouteSagas`:

`Sagas/RouteSagas/HomeSaga.js`:

```js
import { all, put, takeEvery, select } from 'redux-saga/effects'
import { fetchEntities } from '../../Actions'
import fetchEntitiesSaga from '../FetchEntitiesSaga'

const TYPE_NAME = 'Property'
const API_ACTION = 'listProperties'

export default function* homeSaga(location) {
  yield takeEvery('FETCH_ENTITIES', fetchEntitiesSaga(TYPE_NAME, API_ACTION))
  const showing = yield select(s => s.app.Property.get('showing'))
  if (showing === -1) {
    yield all([
      put(fetchEntities(TYPE_NAME, API_ACTION)),
      put(fetchEntities(TYPE_NAME, API_ACTION))
    ])
  }
}
```

This one gets a little more complex.  Remember that we're using an infinitely scrollable list of Properties on this route.  When we scroll to the bottom, we must do both of the following:
- Display the next batch of properties immediately
- Prefetch the next batch of properties

To fulfill these requirements, we must fetch two batches on page-load: the first of which will be displayed immediately, and the second will be displayed when the user scrolls to the bottom.  This is why we defined `showing` to be `-1` in our initial state:  each time we trigger a `FETCH_ENTITIES` action, `showing` is incremented by our reducer, but we only want to show the first batch of the initial two requests.  So if we started it at `0`, *both* initial batches would be displayed.  This "stutter-step" allows us to provide an illusion of very fast loading for the user.

So back to this saga, it is doing in code what we just described: if this is the initial page-load (i.e., `showing` is `-1`), it triggers two `fetchEntities` actions.  It also listens for *every* instance of `FETCH_ENTITIES`, and executes yet another saga: the `fetchEntitiesSaga`.

Create `Sagas/FetchEntitiesSaga.js`:

```js
import { call, put, select  } from 'redux-saga/effects'
import * as api from '../Api/ApolloProxy'
import { FETCH_LIMIT } from '../Constants'
import {
  fetchEntitiesSuccess,
  fetchEntitiesError
} from '../Actions'
import R from 'ramda'

export default function fetchEntitiesSaga(entityName, apiAction) {
  return function* (action) {
    const batchIndex = yield select(st => st.app[entityName].get('showing'))
    try {
      const result = yield call(
        api[apiAction],
        action.args,
        R.merge(action.searchParameters, { skip: FETCH_LIMIT * batchIndex })
      )
      yield put(fetchEntitiesSuccess(
        action.entityName,
        result.data[apiAction],
        batchIndex
      ))
    }
    catch (e) {
      yield put(fetchEntitiesError(
        action.entityName,
        e.message,
        batchIndex
      ))
    }
  }
}
```

This Saga takes care of the logic for (you guessed it) fetching entities.  The actual generator function is wrapped in a normal function that provides the entity name and corresponding API action to call (in this case, `Property` and `listProperties`, respectively).

The logic is pretty simple.  It reads from current app state `showing`, which will be the `batchIndex` for the subsequent request.  It then tries calling the API method with the appropriate parameters, but with a modification to `searchParameters`: it defines `skip` so that the server returns the correct batch of entities.

If the call was successful, we trigger a `FETCH_ENTITIES_SUCCESS` action that contains the resulting list and the `batchIndex` (from above, we know that the reducer will set the entities to app state at this `batchIndex`).

If the call fails, we trigger a `FETCH_ENTITIES_ERROR` action, which would do something like display a big red banner.

Let's move on to the next route, `/property/:id`.  Just like we defined a Saga for the home route, we'll define a `PropertyDetailsSaga` for this one.

`Sagas/RouteSagas/PropertyDetailsSaga.js`:

```js
import { put, takeLatest } from 'redux-saga/effects'
import fetchEntityDetailsSaga from '../FetchEntityDetailsSaga'
import { fetchEntityDetails } from '../../Actions'

export default function* propertyDetailsSaga(location) {
  yield takeLatest('FETCH_ENTITY_DETAILS', fetchEntityDetailsSaga('Property', 'fetchProperty'))
  yield put(fetchEntityDetails('Property', 'fetchProperty', { id: location.params.id }))
}
```

Another simple one.  Listen for the latest `FETCH_ENTITY_DETAILS`, and trigger a `fetchEntityDetailsSaga`:

`Sagas/FetchEntityDetailsSaga.js`:

```js
import { call, put  } from 'redux-saga/effects'
import * as api from '../Api/ApolloProxy'
import {
  fetchEntityDetailsSuccess,
  fetchEntityDetailsError
} from '../Actions'

export default function fetchEntityDetailsSaga(entityName, apiAction) {
  return function* (action) {
    try {
      const result = yield call(api[apiAction], action.args)
      yield put(fetchEntityDetailsSuccess(
        action.entityName,
        result.data[apiAction],
        action.args
      ))
    }
    catch (e) {
      yield put(fetchEntityDetailsError(
        action.entityName,
        e.message,
        action.args
      ))
    }
  }
}
```

This one is basically the same as `fetchEntitiesSaga` but without the `batchIndex` logic.

Whew... that's about enough Saga fun for now!

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
