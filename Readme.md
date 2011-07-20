# airbrake

Node.js client for [airbrakeapp.com][], formerly known as [hoptoadapp.com][].

[airbrakeapp.com]: http://airbrakeapp.com/
[hoptoadapp.com]: http://robots.thoughtbot.com/post/7665411707/hoptoad-is-now-airbrake

## Install

``` bash
npm install airbrake
```

## Basic usage

The common use case for this module is to catch all `'uncaughtException'`
events on the `process` object and send them to airbreak:

``` javascript
var airbreak = require('airbrake').createClient("your api key");
airbrake.handleExceptions();

throw new Error('I am an uncaught exception');
```

Please note that the above will re-throw the exception after it has been
successfully delivered to airbreak, caushing your process to exit with status 1.

If you want more control over the delivery of your errors, you can also
manually submit errors to airbreak.

``` javascript
var airbreak = require('airbrake').createClient("your api key");
var err = new Error('Something went terribly wrong');
airbrake.notify(err, function(err, url) {
  if (err) throw err;

  // Error has been delivered, url links to the error in airbreak
});
```

## Screenshot

This screenshot shows an airbrake error send from this module:

![screenshot](https://github.com/felixge/node-airbrake/raw/master/screenshot.png)

## Features

* Automatically add `process.env` information when sending notifications
* Detect and fix circular references in error context information
* Support for all features of the [2.1 notification API][2.1api]
* Support for [long-stack-traces][]
* Optional auto-handler for `uncaughtException` events
* Provides notification url linking to airbrake in `notify()` callback
* Timeout airbrake requests after 30 seconds, you never know

[long-stack-traces]: https://github.com/tlrobinson/long-stack-traces

[2.1api]: http://help.airbrakeapp.com/kb/api-2/notifier-api-version-21

## Adding context to errors

The `notify()` method automatically adds the following context information to
each delivered error:

* **error.class:** (`err.type` string if set, or `'Error'`)
* **error.message:** (`err.message` string)
* **error.backtrace:** (`err.stack` as parsed by [stack-trace][])
* **request.url:** (`err.url` string if set);
* **request.component:** (`err.component` string if set);
* **request.action:** (`err.action` string if set);
* **request.cgi-data:** (`process.env`, merged with `err.env` object if set)
* **request.params:** (`err.params` object if set)
* **request.session:** (`err.session` object if set)
* **server-environment.project-root:** (`airbreak.projectRoot` string if set)
* **server-environment.environment-name:** (`airbreak.env` string, defaults to `process.env.NODE_ENV`)
* **server-environment.app-version:** (`airbreak.appVersion string if set)
* **server-environment.hostname:** (`airbreak.hostname` string if set)

You can add additional context information by modifying the error properties
listed above:

``` javascript
var airbreak = require('airbrake').createClient("your api key");
var http = require('http');

http.createServer(function(req, res) {
  if (req.headers['X-Secret'] !== 'my secret') {
    var err = new Error('403 - Permission denied');
    req.writeHead(403);
    req.end(err.message);

    err.url = req.url;
    err.params = {ip: req.socket.remoteAddress};
    airbrake.notify(err):
  }
});
```

Unfortunately `uncaughtException` events cannot be traced back to particular
requests, so you should still try to handle errors where they occur.

[stack-trace]: https://github.com/felixge/node-stack-trace

## Removing context from errors

In some scenarios you might want to filter some context to never show up in
Airbrake. For example you might have a private key loaded in your environment
memory, or your user has some critical data in his session, and you want to
hide that.

This can be done by hooking into the `'vars'` event like so:

``` javascript
airbrake.on('vars', function(type, vars) {
  if (type === 'cgi-data') {
    delete vars.SECRET;
  }
});
```

## API

### Airbreak.createClient(apiKey, [env])

Returns a new Airbrake instance.

### airbrake.key = null

The API key to use.

### airbrake.env = process.env.NODE_ENV;

The name of the server environment this is running in.

### airbrake.projectRoot = null

The root directory of this project.

### airbrake.appVersion = null

The version of this app. Set to a semantic version number, or leave unset.

### airbrake.protocol = 'http'

The protocol to use.

### airbrake.timeout = 30 * 1000

The timeout after which to give up trying to notify airbrake in ms.

### airbrake.handleExceptions()

Registers a `process.on('uncaughtException')` listener. When an uncaught
exception occurs, the error is send to airbrake, and then re-thrown to
kill the process.

### airbrake.notify(err, [cb])

Sends the given `err` to airbrake.

The callback parameter receives two arguments, `err, url`. `err` is set if
the delivery to airbrake failed.

If no `cb` is given, and the delivery fails, an `error` event is emitted. If
there is no listener for this event, node will kill the process as well. This
is done to avoid silent error delivery failure.

## Alternative modules

This module is meant as a replacement for [hoptoad-notifier][], which does not
support all features of the [2.1 API][2.1api].

[hoptoad-notifier]: https://github.com/tristandunn/node-hoptoad-notifier

## Todo

* Implement `airbrake.deployment()`

## License

airbrake is licensed under the MIT license.
