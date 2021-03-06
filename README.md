## Derby 0.6 router

### Getting started

Install: `npm install derby-router`

Add the package into your derby application:

*index.js*
```js
var derby = require('derby');
var app = derby.createApp('app', __filename);

app.use(require('derby-router'));

// named route
app.get('main', '/main', function(page, model, params, next){
  var self = this;
  var items = model.query('items', {});
  items.subscribe(function(){
    items.ref('_page.items');
    page.render('main');
  })
});

// name route using this.model
// and this.render() it's equal page.render('item')
app.get('item', '/item/:id', function(){
  var self = this;
  var item = this.model.at('items.' + this.params.id);
  item.subscribe(function(){
    item.ref('_page.item');
    self.render();
  });
});

// server get route
app.serverGet('time', '/api/time', function(req, res, next){
  res.json(Date.now());
});
```

*main.html*
```html
<index:>
  <h1>Items:</h1>

  {{each _page.items as #item}}

    <!-- view function pathFor return right url-->
    <!-- like /item/4380fefc-001d-493a-aa2b-5e3d512e587d -->
    <a href="{{pathFor('item', #item.id)}}">{{#item.name}}</a>
  {{/each}}

```

## Routes

Derby-router allows to create 'get', 'post', 'put' and 'del' types of derby
application routes, and the same four types of server routes.

```js
// derby-app routes
app.get(...);
app.post(...);
app.put(...);
app.del(...);

// Server routes
app.serverGet(...);
app.serverPost(...);
app.serverPut(...);
app.serverDel(...);
```

### Route names and pathes

First two parameters in routes are `name` and `path`. For example:
```js
app.get('main', '/main');
```
To tell the truth the order is not important, so you can define the route like
this:
```js
app.get('/main', 'main');
```
Or even just:
```js
app.get('main');
```
Path will be extracted automatically from the name: 'main' -> '/main'
Look at the examples:

| name | path |
|------|------|
| main | /main |
| items:item | /items/item |

That is quite convinient for sipmle pathes, also possible to define only `pathes`,
so `names` will be converted automatically. For example you can define the route
like this:

```js
app.get('/main');
```

A few examples:

| path | name |
|------|------|
| /main | main |
| /items/item | items:item |
| /articles/:id | articles:id |
| / | home (special case) |

Of course the best way is to define both `name` and `path` explicitly.

### Path parameters

Derby-router use [path-to-regexp](https://github.com/pillarjs/path-to-regexp) to
parse pathes. It's express-like parsing engine. Take a look at
[path-to-regexp](https://github.com/pillarjs/path-to-regexp) docs.

Short list of possibilities:

| path | exemple | description |
|------|------|------|
| /:foo/:bar | /test/route | named parameters |
| /:foo/:bar? | /test, /test/route | optional parameter |
| /:foo* | /, /test, /test/test | zero or more |
| /:foo+ | /test, /test/route | one or more |
| /:foo(\\d+) | /123, /abc | custom match parameters |
| /:foo/(.*) | /test/route | unnamed parameters |


### Derby-application routes

Syntax: `app.method(name?, path?, [handler]*, options?)`

`method`: one of 'get', 'post', 'put', 'del'

`name`: string - name of the route, you can use the name to get specific url
using `pathFor`-view function

`path`: string (should starts with '/')

`handler`: one of a list of route-handlers. May be omitted, if so `name`-template
will be rendered. There are two types of handlers:

- handler-function
- array of route-modules (more on this later)

`options` - an object with options. At the moment you can define such options:

- dontRender - boolean - if you don't with to render route by default
- derbyController - function - to override default derby-application route-controller

#### Handler-functions

For derby-application handler-functions accept usual parameters: page, model,
params, and next. For example:

```js
app.get('main', '/main', function(page, model, params, next){
  // ...
});
```

But also `this` in the function is `DerbyRouteController` which provide some
additional functionality:

| property | description |
|----------|-------------|
| this.name | route name |
| this.app | derby-app variable |
| this.page | page - like in parameters |
| this.model | model - like in parameters |
| this.params | params - like in parameters |
| this.path | route-path |
| this.next | next - like in parameters |
| this.render | render-function (by default render template with the name of the route) |
| this.loadModules | function which accept a list of module-names to perform load-function of the modules |
| this.setupModules | the function subscribes to all of the loaded modules data and perform setup-functions of the loaded modules |
| this.addSubscription | the function adds module subscription to the list. Subscribe to the list will be performed after in the setupModules-function |

It's also possible to use a few functions in one route. For example:

```js
function isAdmin(){
  if (this.model.get('_session.user.admin'){
    this.next();
  } else {
    this.next('User should be an admin to get access!');
  }
}

app.get('admin', isAdmin, function(){
  // ...
});
```

#### Router modules

We added to derby-router optional possibility to use so called "router modules".
It allow write routes declarative way and help to construct huge applications.

Look at the example:

```js
app.get('/main', function(page, model, params, next){
  var userId = model.get('_session.userId');
  var user = model.at('users.' + userId);

  model.subscribe(user, function(){
    user.ref('_page.user');
    page.render('main');
  });
});
```

The user-subscription is a frequent task. Often we should do it almost in all
our routes. Router-modules will help us.

First, note - working with subscriptions we often have two part of code: before
subscription and after subscription. We called the parts: load-block and
setup-block.

Let's write the user-module:

```js
app.useRouterModule('user', {
  load: function(){
    var userId = this.model.get('_session.userId');
    this.user = this.model.at('users.' + userId);
    this.addSubscriptions(user);
  },
  setup: function(){
    this.model.ref('_page.user', this.user);
  }
});
```

and use it in our handler:

```js
app.get('/main', function(){
  this.loadModules('user');
  // setup all loaded modules
  this.setupModules(function(){
    // by defauld render use name of the route
    // as a rendered template-name, so
    // it's 'main'
    this.render();
  });
});
```

but there is more convenient way to use router-modules:

```js
app.get('/main', ['user']);
```

Let's imagine more complex example. We should subscribe to the current user and
then to all his friends.

```js
app.get('/main', function(page, model, params, next){
  var userId = model.get('_session.userId');
  var user = model.at('users.' + userId);

  model.subscribe(user, function(){
    user.ref('_page.user');

    var friends = model.query('users', user.path('friendIds'));

    model.subscribe(friends, function(){
      page.render('main');
    };
  });
});
```

Module 'friends' will be something like this:

```js
app.useRouterModule('friends', {
  load: function(user){
    var friends = this.model.query('users', user.user.path('friendIds'));
    this.addSubscriptions(friends);
  }
  // We don't need setup-function here
});
```

Note 'user'-parameter. It's a user-module. After, we can define router like this:

```js
app.get('/main', ['user', 'friends']);
```

Derby-router understand dependencies and make "depencency injections".

Also, we can combine functions with modules, f.e.:

```js
app.get('/main', isAdmin, ['user', 'friends']);
```

### Server routes

Server-routes are like derby-application routes, but without router-modules, and
they have a little bit different API.

Basically, like in derby-application routes we can use four methods for
server-routes: 'get', 'post', 'put', 'del'.

Syntax: `app.serverMethod(name?, path?, [handlers]*, options?)`

`Method`: one of 'Get', 'Post', 'Put', 'Del'

`name`: string - name of the route, you can use the name to get specific url
using `pathFor`-view function

`path`: string (should starts with '/')

`handlers`: a list of handler-functions. May be omitted, if so nothing will
happen

`options` - an object with options. At the moment you can define such options:

- serverController - function - to override default server route-controller

#### Handler-functions

Handler-functions accept usual expressjs parameters: req, res and next. For
example:

```js
app.serverGet('api:time', '/api/time', function(req, res, next){
  res.json(Date.now());
});
```

But also `this` in the function is `ServerRouteController` which provide some
additional functionality:

| property | description |
|----------|-------------|
| this.name | route name |
| this.app | derby-app variable |
| this.model | model - like in parameters |
| this.params | params - like in derby-application routes |
| this.path | route-path |
| this.next | next - like in parameters |

It's also possible to use a few functions in one route. For example:

```js
function isAdmin(){
  if (this.model.get('_session.user.admin'){
    this.next();
  } else {
    this.next('User should be an admin to get access!');
  }
}

app.serverGet('admin', isAdmin, function(){
  // ...
});
```

#### The way to hide server code from browser bundle

Typical pattern is:

```js
var derby = require('derby');

var app = derby.createApp('app', __filename);

var serverRoutes = derby.util.serverRequire(module, './server') || {};

// common derby-routes
app.get(...);
// ...

// Server-routes
app.serverPost('api:time', '/api/time', serverRoutes.time);
app.serverPost('api:foo', '/api/foo', serverRoutes.foo);
app.serverPost('api:bar', '/api/bar', serverRoutes.bar);
```

## View-function `pathFor`

For all routes we can get specific url in templates using view-function
`pathFor`, for example:

```js
app.get('item', '/items/:id', function(){
  // ...
});
```

```html
<a href="{{pathFor('item', #item.id)}}">{{#item.name}}</a>
```

There are two syntax for `pathFor`-function:

Simple syntax: `pathFor(name, [param1, param2, ...])`

- `name` - route-name
- `param1`, `paramN` - route parameters in the order of the params in the
route-path, for example:

```js
app.get('foobar', '/new/:foo/:bar+/(.*)', function(){
  // ...
});
```

```html
<!-- here we get /new/one/two/three/four url -->
<a href="{{pathFor('foobar', 'one', 'two', 'three/four')}}">foobar</a>
```

Object syntax: `pathFor(name, options)`

- `name` - route-name
- `options` - object of named route params, from example:

```js
app.get('foobar', '/new/:foo/:bar+/(.*)', function(){
  // ...
});
```

```html
<!-- here we get /new/one/two/three/four url -->
<a href="{{pathFor('foobar', {foo: 'one', bar: ['two', 'three'], 0: 'four'})}}">
  foobar
</a>
```

Note 0 param. For unnamed params like (.*) we use number-keys.

### Query and hash

Additionally we can pass two special params in the pathFor options:

- $query - query string or query-object
- $hash - hash string

For example:

```js
app.get('main', '/main', function(){
  // ...
});
```

```html
<!-- here we get /main?foo=abc#bar url -->
<a href="{{pathFor('main', {$query: {foo: 'abc'}, $hash: 'bar'})}}">foobar</a>
```
