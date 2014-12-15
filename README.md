express-redis-cache
===================

Easily cache pages of your app using Express and Redis. *Could be used without Express too.*

# Install

    npm install express-redis-cache
    
`express-redis-cache` ships with a CLI utility you can invoke from the console. In order to use it, install `express-redis-cache` globally (might require super user privileges):

    npm install -g express-redis-cache
    
# Upgrade

Read [this](../master/CHANGELOG.md) if you are upgrading from 0.0.8 to 0.1.0,

# Usage

Just use it as a middleware in the stack of the route you want to cache.

```js
var app = express();
var cache = require('express-redis-cache')();

// replace
app.get('/',
  function (req, res)  { ... });

// by
app.get('/',
  cache.route(),
  function (req, res)  { ... });
```
    
This will check if there is a cache entry for this route. If not. it will cache it and serve the cache next time route is called.

# Redis connexion info

By default, `redis-express-cache` connects to Redis using localhost as host and nothing as port (using Redis default port 6379). To use different port or host, declare them when you require express-redis-cache.

```js
var cache = require('express-redis-cache')({
  host: String, port: Number
  });
```
        
You can pass a Redis client as well:

```js
require('express-redis-cache')({ client: require('redis').createClient() })
```

You can have several clients if you want to serve from more than one Redis server:

```js
var cache = require('express-redis-cache');
var client1 = cache({ host: "...", port: "..." });
var client2 = cache({ host: "...", port: "..." });
...
```

# Name of the cache entry

By default, the cache entry name will be `default prefix`:`name` where name's value defaults to  [req.originalUrl](http://expressjs.com/4x/api.html#req.originalUrl).

```js
app.get('/',
  cache.route(), // cache entry name is `cache.prefix + "/"`
  function (req, res)  { ... });
```

You can specify a custom name like this:

```js
app.get('/',
  cache.route('home'), // cache entry name is now `cache.prefix + "home"`
  function (req, res)  { ... });
```

You can also use the object syntax:

```js
app.get('/',
  cache.route({ name: 'home' }), // cache entry name is `cache.prefix + "home"`
  function (req, res)  { ... });
```

# Prefix

All entry names are prepended by a prefix. Prefix is set when calling the Constructor. 

```js
// Set default prefix to "test". All entry names will begin by "test:"
var cache = require('express-redis-cache')({ prefix: 'test' });
```

To know the prefix:

```js
console.log('prefix', cache.prefix);
```

You can pass a custom prefix when calling `route()`:

```js
app.get('/index.html',
  cache.route('index', { prefix: 'test'  }), // force prefix to be "test", entry name will be "test:index"
  function (req, res)  { ... });
```

You can also choose not to use prefixes:

```js
app.get('/index.html',
  cache.route({ prefix: false  }), // no prefixing, entry name will be "/index.html"
  function (req, res)  { ... });
```

# Expiration

Unless specified otherwise when calling the Constructor, cache entries don't expire. You can specify a default lifetime when calling the constructor:

```js
// Set default lifetime to 60 seconds for all entries
var cache = require('express-redis-cache')({ expire: 60 });
```

You can overwrite the default lifetime when calling `route()`:

```js
app.get('/index.html',
  cache.route({ expire: 5000  }), // cache entry will live 5000 seconds
  function (req, res)  { ... });
    
// You can also use the number sugar syntax
cache.route(5000);
// Or
cache.route('index', 5000);
// Or
cache.route({ prefix: 'test' }, 5000);
```

# Content Type

You can use `express-redis-cache` to cache HTML pages, CSS stylesheets, JSON objects, anything really. Content-types are saved along the cache body and are retrieved using `res._headers['content-type']`. If you want to overwrite that, you can pass a custom type.

```js
app.get('/index.html',
  cache.route({ type: 'text/plain'  }), // force entry type to be "text/plain"
  function (req, res)  { ... });
```

# Events

The module emits the following events:

## error

You can catch errors by adding a listener:

```js
cache.on('error', function (error) {
  throw new Error('Cache error!');
});
```

## message

`express-redis-cache` logs some information at runtime. You can access it like this:

```js
cache.on('message', function (message) {
  // ...
});
```

## connected

Emitted when the client is connected to Redis server

```js
cache.on('connected', function () {
  // ....
});
```

## deprecated

Warning emitted when stumbled upon a deprecated part of the code

```js
cache.on('deprecated', function (deprecated) {
  console.log('deprecated warning', {
      type: deprecated.type,
      name: deprecated.name,
      substitute: deprecated.substitute,
      file: deprecated.file,
      line: deprecated.line
  });
});
```
    
# The Entry Model

This is the object synopsis we use to represent a cache entry:

```js
var entry = {
  body:    String // the content of the cache
  touched: Number // last time cache was set (created or updated) as a Unix timestamp
  expire:  Number // the seconds cache entry lives (-1 if does not expire)
  type: String // the content-type
};
```

# The module

The module exposes a function which instantiates a new instance of a class called [ExpressRedisCache](../master/index.js).

```js
// This
var cache = require('express-redis-cache')({ /* ... */ });
// is the same than
var cache = new (require('express-redis-cache/lib/ExpressRedisCache'))({ /* ... */ });
```

# The constructor

As stated above, call the function exposed by the module to create a new instance of `ExpressRedisCache`,

```js
var cache = require('express-redis-cache')(/** Object | Undefined */ options);
```

Where `options` is an object that has the following properties:

|   Option    |  Type   |  Default  |  Description  |
| ------------- |----------|-------|----------|--------|
| **host**          | `String`    | `undefined` | Redis server host
| **port**      | `Number`     | `undefined` | Redis server port
| **prefix**       | `String`  | `require('express-redis-cache/package.json').config.prefix` | Default prefix (This will be prepended to all entry names) |
| **expire**   | `Number` | `undefined` | Default life time of entries in seconds |
| **client**   | `RedisClient` | `require('redis').createClient({ host: cache.host, port: cache.port })` | A Redis client |

# API

The `route` method is designed to integrate easily with Express. You can also define your own caching logics using the other methos of the API shown below.
    
## `get` Get cache entries
    
```js
cache.get(/** Mixed (optional) */ query, /** Function( Error, [Entry] ) */ callback );
```
    
To get all cache entries:

```js
cache.get(function (error, entries) {
  if ( error ) throw error;
  
  entries.forEach(console.log.bind(console));
});
```

To get a specific entry:

```js
cache.get('home', function (error, entries) {});
```

To get a specific entry for a given prefix:

```js
cache.get({ name: 'home', prefix: 'test' }, function (error, entries) {});
```

You can use wildcard:

```js
cache.get('user*', function (error, entries) {});
```
    
## `add` Add a new cache entry
    
```js
cache.add(/** String */ name, /** String */ body, /** Object (optional) **/ options, /** Function( Error, Entry ) */ callback )
```

Where options is an object that can have the following properties:

- **expire** `Number` (lifetime of entry in seconds)
- **type** `String` (the content-type)
    
Example:

```js
cache.add('user:info', JSON.stringify({ id: 1, email: 'john@doe.com' }), { expire: 60 * 60 * 24, type: 'json' },
    function (error, added) {});
```

## `del` Delete a cache entry
    
```js
cache.del(/** String */ name, /** Function ( Error, Number deletions ) */ callback);
```

You can use wildcard (*) in name.

## `size` Get cache size for all entries
    
```js
cache.size(/** Function ( Error, Number bytes ) */);
```
    
# Command line

We ship with a CLI. You can invoke it like this: `express-redis-cache`

## View cache entries

```bash
express-redis-cache ls
```

## Add cache entry

```bash
express-redis-cache add $name $body $expire --type $type
```

### Examples

```bash
# Cache simple text
express-redis-cache add "test" "This is a test";

# Cache a file
express-redis-cache add "home" "$(cat index.html)";

# Cache a JSON object
express-redis-cache add "user1:location" '{ "lat": 4.7453, "lng": -31.332 }' --type json;

# Cache a text that will expire in one hour
express-redis-cache add "offer" "everything 25% off for the next hour" $(( 60 * 60 ));
```

## Get single cache entry

```bash
express-redis-cache get $name
# Example: express-redis-cache get user1:*
# Output:
```

## Delete cache entry

```bash
express-redis-cache del $name
# Example: express-redis-cache del user1:*
# Output:
```

## Get total cache size

```bash
express-redis-cache size
# Output:
```

# Test

    mocha --host <redis-host> --port <redis-port>

    # or

    npm test --host=<redis-host> --port=<redis-port>

Enjoy!
