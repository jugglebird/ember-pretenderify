# Ember Pretenderify

[![Build Status](https://travis-ci.org/samselikoff/ember-pretenderify.svg?branch=master)](https://travis-ci.org/samselikoff/ember-pretenderify)
[![npm version](https://badge.fury.io/js/ember-pretenderify.svg)](http://badge.fury.io/js/ember-pretenderify)

Share a single Pretender server across your Ember app's testing and development environments.

----

Are you tired of

- Writing one set of mocks for your tests, and another for development?
- Wiring up tests for each of your apps manually, from scratch?
- Changing lots of files/tests when your API changes?

Ember Pretenderify may be for you! It lets you share your [Pretender](https://github.com/trek/pretender) server in both development and in testing. It only uses Pretender if you're not in production and if you're not proxying to an explicit API server via `ember serve --proxy`.

> Note: this is still experimental. Some shorthands currently only work with AMS-style responses, and some assets will be make it into production, albeit ignored. Eventually these will be fixed.

## Installation

Uninstall `ember-cli-pretender` if you're already using it. Then run

    ember install:addon ember-pretenderify

Finally, add `store` to the `predef` section in your `tests/.jshintrc` file.

## Getting started

Pretenderify splits up your Pretender server into two pieces:

 - **routes**, which define the URLs your server responds to, and
 - the **store**, your server's "database"

Let's add some data to the store. The generator should have created the file `/app/pretender/data/contacts.js`, which exports some data:

```js
// app/pretender/data/contacts.js
export default [
  {
    id: 1,
    name: 'Zelda'
  },
  {
    id: 2,
    name: 'Link'
  },
];
```

Given this file, whenever your Pretender server starts up, this data will be added to its store under the `contacts` key (since that's the name of the file). Add additional data by adding more files under the `/data` folder. All data files should be plural, and export arrays of POJOs.

Now, to return this data from an endpoint, let's create our first route. We'll use the **get** helper method to easily interact with our server's store (which now has these contacts in it).

The generator should have created the file `app/pretender/config.js`, which exports a function. Add a route for `/api/contacts`:

```js
// app/pretender/config.js
export default function() {

  this.get('/api/contacts', 'contacts');

};
```

This is the simplest example of a GET route. Here, we're telling Pretender that whenever our Ember app makes a GET request to `/api/contacts`, respond with all the `contacts` in our store. So the first argument of `get` is the *path* of the route we're defining a handler for, and the second is the *objects in the store* we want to respond with.

As long as all your Pretender routes mutate and read from the store, your user interactions during development will persist. This lets users interact with your app as if it were wired up to a real server.

We can also respond with multiple objects from the store, let's say if our app expects additional data to be sideloaded at this URL:

```js
this.get('/api/contacts', ['contacts', 'addresses']);
```

This will return all the data you added to the `contacts` and `addresses` keys of your store.

Handling POST, PUT and DELETE requests is just as simple. For example, this route lets you create new `users` by POSTing to `/api/users`:

```js
this.post('/api/users', 'user');
```

You can also pass a function in as the second argument in case you want to manipulate the store yourself:

```js
this.post('/api/users', function(store, request) {
  var attrs = JSON.parse(request.requestBody);
  var newContact = store.push('contact', attrs);

  return {
    contact: newContact
  };
});
```

Find the complete documentation for `get`, `post`, `put` and `del` [**below**](#verb-methods).

**Shorthands**

There are many shorthands available to make writing your routes easier. For example, the route

```js
this.get('/api/contacts', 'contacts');
```

can be simplified to

```js
this.get('/api/contacts')
```

since the key of the last URL segment matches the store object we want.

See [the docs below](#verb-shorthands) for all available shorthands.

**Acceptance testing**

During testing, the store always starts off empty; however, all the routes you've defined will still be available (since these are what your app expects, and mostly shouldn't change within your tests). The store is emptied because tests should be atomic, and should not rely on state defined elsewhere.

So for each test, you first define the initial state of the store (i.e. the "server" state your test expects). Then, write your test and assert based on that state.

There's a global helper called `store` to help you with this. Here's an example acceptance test:

```js
// tests/acceptance/index-test.js
test("I can view the models", function() {
  // set up our "server's data"
  store.loadData({
    products = [{
      id: 1,
      name: 'iPhone'
    }]
  });

  visit('/');

  andThen(function() {
    equal(currentRouteName(), 'index');
    equal( find('p').text(), 'iPhone' );
  });
});
```

In the future, we plan on making factories with a simpler API to help you add data to your Pretender server's store. But fundamentally, the point here is that the routes and config you've defined for your Pretender server will be shared across your development and testing environments.

Another benefit of separating your server's routes from its data is that your tests become less brittle. Say you change your API for some route from using links (async model relationships) to having everything sideloaded. The only thing you need to change here is the related route definition, in your `/pretender/config.js` file. You won't need to update your tests, since they just specify what's in the store at the time of the test (which wouldn't change in this case).

-----

That should be enough to get started! Check out the [dummy config](tests/dummy/app/pretender/config.js) in this repo for a simple example, or keep on reading for the full API.

## Configuration

There's some default configuration for your Pretender server which can all be customized. `this.pretender` in your `/pretender/config.js` file refers to the actual [`Pretender`](https://github.com/trek/pretender) instance, so any config options that work there will work here as well.

**prepareBody**

By default, content returned is JSON stringified, so you can just return JS objects. Refer to [Pretender's docs](https://github.com/trek/pretender#mutating-the-body) if you want to change this.

**namespace**

Set the base namespace used for all routes defined with `get`, `post`, `put` or `del`. For example,

```js
// app/pretender/config.js
export default function() {

  this.namespace = '/api';

  // this route will handle the URL '/api/contacts'
  this.get('/contacts', 'contacts');
};
```

**timing**

Set the timing parameter of the response. Default is a 400ms delay. This parameter only applies to non-testing environments so it doesn't slow down your tests. See [Pretender's docs](https://github.com/trek/pretender#timing-parameter) for all possible values.

```js
// app/pretender/config.js
export default function() {

  this.timing = 400; // default

};
```

**Environment options**

*force*

By default, your Pretender server will run in test mode, and in development mode as long as the `--proxy` option isn't passed. You can force your server to run in other environments (e.g. production) with an ENV var:

```js
// config/environment.js
...
ENV['ember-pretenderify'] = {
  force: true
}
```

This is useful to be able to share a working prototype (built with `--environment production`) before a server is ready, for instance.

## API

### store

You interact with the store using the *verb methods* (`get`, `post`, `put` and `del`) in your Pretender routes. You retrieve or modify data from the store, then return what you want for that route. 

Here are the store methods available to you from within your route definitions.

**store.findAll(type)**

Returns the `type` models attached to the store object. Note typekey` is always singular.

For example if you had the following data file named `/app/pretender/data/contacts.js`

```js
export default [
  {id: 1, name: 'Sam'}, {id: 2, name: 'Ryan'}
];
```

then `store.findAll('contact')` would return the `contacts` array.

**store.find(type, id)**

Returns a single model from the store based on the type `type` and id `id`. Note `type` is always singular, and `id` can be an int or a string, but integer ids as strings (e.g. the string `"1"`) will be treated as integers.

For example, given the above contacts in the store, `store.find('contact', 2)` would return `{id: 2, name: 'Ryan'}`.

**store.findQuery(type, query)**

Returns an array of models of type `type` that match the key-value pairs in the `query` object. `query` is a POJO.

For example, given the above contacts in the store, `store.findQuery('contact', {name: 'Ryan'})` would return `[{id: 2, name: 'Ryan'}]`.

**store.push(type, data)**

Creates or updates a model of type `type` in the store. `data` is a POJO. If `data` has an `id`, updates the model in the, otherwise creates a new model.

Returns the new data from the store, which is useful to get the autogenerated id for new models.

**store.remove(type, id)**

Removes a model of type `type` with id `id` from the store.

**store.removeQuery(type, query)**

Removes the models of type `type` that match the key-value pairs in the `query` object from the store. `query` is a POJO.

### Verb methods 

The four verb methods (`get`, `post`, `put` and `del`) are the primary way you define routes and interact with your store. There are many shorthands available to make your server definition more succinct, and ideally most of your config will use them.

You can always fall back to a function and manipulate the data in the store however you need to via the [store's api](#store).

Here's the full definition:

```js
this.verb(path, handler[, responseCode]);
```
where *verb* is `get`, `put`, `post`, or `delete`, and

- **path**: string. The URL you're defining, e.g. `/api/contacts` (or `/contacts` if `namespace` is defined).
- **handler**: function or shorthand.

    As a function, takes two parameters, *store*, your Pretender server's store, and *request*, which is the Pretender request object. Return the data you want as plain JS - it will be stringified and sent as the response body to your request.

    As a shorthand, either a string, an array or undefined. Consult [the shorthand docs](#verb-shorthands) for the various shorthand definitions.
- **responseCode**: number. optional. The response code of the request.

### Verb shorthands

**GET shorthands**

```js
/*
  Return a single collection
*/
// Finds type by singularizing last portion of url
this.get('/contacts');

// Optionally specify the collection as the third param
this.get('/contacts', 'users');

/*
  Return multiple collections
*/
// Note this is everything you have in your store for these models
this.get('/', ['photos', 'articles']);

/*
  Return a single model
*/
// Finds type by singularizing last portion of url before id
this.get('/contacts/:id');
// Optionally specify type as third param
this.get('/contacts/:id', 'user');

/*
  Return a single object with related models
*/
// Make sure you put the owning (singular) model first. Finds `contact`
// by `id`, then finds all `addresses` where `contact_id` matches.
this.get('/contacts/:id', ['contact', 'addresses']);
```

**POST shorthands**

```js
/*
  Create a new object
*/
// The type is found by singularizing the last portion of the url
this.post('/contacts');
// Optionally specify the type of resource to be created as the third param
this.post('/contacts', 'user');
```

**PUT shorthands**

```js
/*
  Update an object
*/
// The type is found by singularizing the last portion of the url
this.put('/contacts/:id');
// Optionally specify the type of resource to be updated as the third param
this.put('/contacts/:id', 'user');
```

**DELETE shorthands**

```js
/*
  Remove a single object
*/
// The type is found by singularizing the last portion of the url.
this.del('/contacts/:id')
// Optionally specify the type of resource to be deleted as the third param
this.del('/contacts/:id', 'user')

/*
  Remove a single object + related models
*/
// Make sure you put the owning (singular) model first. Deletes `contact`
// that matches `id`, then deletes all `addresses` where `contact_id`
// matches `id`.
this.del('/contacts/:id', ['contact', 'addresses']);
```

**Function API**

If you pass a function handler in as the second param of `get`, `post`, `put` or `del`, you can manually interact with your store and return specific data in your response.

```js
this.get('/contacts', function(store, request) {
  // do work, return data
});
```

The handler takes two parameters, *store*, your Pretender server's store, and *request*, which is the Pretender request object (which is the XMLHttpRequest instance that initiated the request, plus some additional data like `params`).

Consult the [store's API](#store) for how to interact with the store, or check out some examples:

```js
/*
  Return a collection
*/
this.get('/contacts', function(store) {
  var contacts = store.findAll('contact');
  return {
    contacts: contacts
  }:
});

/*
  Return a collection with related models
*/
this.get('/contacts', function(store) {
  var contacts = store.findAll('contact');
  var addresses = store.findAll('address');

  // But we only want the related addresses, so...
  var contactIds = contacts
    .map(function(contact) {return contact.id});
  var relatedAddresses = addresses
    .filter(function(addr) {return contactIds.indexOf(addr.contact_id) > -1; });

  return {
    contacts: contacts,
    addresses: relatedAddresses
  }:
});

/*
  Return multiple collections.
*/
this.get('/', function(store, request) {
  var photos = store.findAll('photo');
  var articles = store.findAll('article');

  return {
    photos: photos,
    articles: articles
  }:
});

/*
  Return a single object
*/
this.get('/contacts/:id', function(store, request) {
  var contactId = request.params.id;
  var contact = store.find('contact', contactId);

  return {
    contact: contact
  };
});

/*
  Return a single object with related models
*/
this.get('/contacts/:id', function(store, request) {
  var contactId = request.params.id;
  var contact = store.find('contact', contactId);
  var addresses = store.findAll('address')
    .filterBy('contact_id', contactId);

  return {
    contact: contact,
    addresses: addresses
  };
});

/*
  Create a new object
*/
this.post('/contacts', function(store, request) {
  var attrs = JSON.parse(request.requestBody);
  var newContact = store.push('contact', attrs); // gets an id

  return {
    contact: newContact
  };
});

/*
  Update an object in the store.
*/
this.put('/contacts/:id', function(store, request) {
  var id = request.params.id;
  var attrs = JSON.parse(request.requestBody);
  attrs.id = +id;

  store.push('contact', attrs);
  return {
    contact: attrs
  };
});

/*
  Remove a single object
*/
this.del('/contacts/:id', function(store, request) {
  var contactId = request.params.id;

  store.remove('contact', contactId);

  return {};
});

/*
  Remove a single object + related models
*/
this.del('/contacts/:id', function(store, request) {
  var contactId = request.params.id;

  store.remove('contact', contactId);
  store.removeQuery('address', {contact_id: contactId});

  return {};
});
```

# Known issues

- assets end up in production
- Pretender doesn't handle other-origin requests (e.g. api.twitter.com)

# TODO

- [ ] override route in test to return 404

**stub**
- [ ] shorthand for multiple data relationships? e.g. lesson has many questions, questions has many answers
- [ ] with partial attrs

**Roadmap**
- [ ] Adapter, only works with AMS-style right now.
- [ ] Factories for adding data to store in tests.
- [ ] Option to disable in test so tests hit real backend
