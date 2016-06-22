# A Node.js and browser Flickr API library

With oauth authentication for Flickr API keys if you're using it server-side (authenticated calls from the browser are too insecure to support for the moment, and will throw an error).

You also get API route proxying so you can call the Flickr methods through your own server and get Flickr responses back for free. Super handy!

# Quick start guide:

You don't want to read this entire README.md, so just find what you want to do in this handy quick start guide right here at the start of the README, and off you go!

## In the browser with client-side credentials:

Script-load the `browser/flickrapi.dev.js` library for development work, and use the `browser/flickrapi.js` library in production.

You can access Flickr by creating an API instance as:

```
var flickr = new Flickr({
  api_key: "1234ABCD1234ABCD1234ABCD1234ABCD"
});
```

Then query Flickr using the API as described over at http://www.flickr.com/services/api - for instance, to search for all photographs that text-match the terms "red panda", you call:

```
flickr.photos.search({
  text: "red+panda"
}, function(err, result) {
  if(err) { throw new Error(err); }
  // do something with result
}
```
All calls are asynchronous, and the callback handling function always has two arguments. The first, if an error occurs, is the error generated by Flickr; the second, if the call succeeds, is the result sent back from Flickr, as plain JavaScript object.

**Note**: this is not secure. People will be able to see your API key, and this is pretty much *the worst idea(tm)*, so you probably want to use this library...

## In the browser, securely via proxied API calls:

Script-load the `browser/flickrapi.dev.js` library for development work, and use the `browser/flickrapi.js` library in production, but don't use your API key. Instead, point to your server as a flickr API proxy:

```
var flickr = new Flickr({
  endpoint: "http//yourhostedplace/services/rest/"
});
```

If you build your app with es6 `module` loader, and you just use like below instead of script-load the `browser/flickrapi.dev.js` in html file:
```
import Flickr from 'flickrapi/browser/flickrapi.dev.js'
// do something that use Flickr name
```

To make this work, have flickapi running on your server with a proxy route enabled, and you'll be able to make use of all the Flickr API calls, without having to put your credentials anywhere in your client-side source code.

Proxy mode is explained [below](#Flickr_API_proxying_for_connect/express_apps), but is essentially a one-liner add to your regular connect/express app.


## As a module in a (larger) Node.js program

Install like any other package:
```
$> npm install flickrapi --save
```
After that, you have two choices, based on whether you want to authenticate or not. Both approaches require an API key, but using OAuth2 authentication means you get access to the full API, rather than only the public API.

To suppress the progress bars in stdout you can include a `progress` attribute when initializing:

```
var flickr = new Flickr({
    api_key: "1234ABCD1234ABCD1234ABCD1234ABCD",
    progress: false
});
```

### No authentication, public API only

```
var Flickr = require("flickrapi"),
    flickrOptions = {
      api_key: "API key that you get from Flickr",
      secret: "API key secret that you get from Flickr"
    };

Flickr.tokenOnly(flickrOptions, function(error, flickr) {
  // we can now use "flickr" as our API object,
  // but we can only call public methods and access public data
});
```

### Authenticated, full Flickr API access

```
var Flickr = require("flickrapi"),
    flickrOptions = {
      api_key: "API key that you get from Flickr",
      secret: "API key secret that you get from Flickr"
    };

Flickr.authenticate(flickrOptions, function(error, flickr) {
  // we can now use "flickr" as our API object
});
```

# The end.

That's it, that's all the quickstart guiding you need. For more detailed information, keep reading. If you just wanted to get up and running, then the preceding text should have gotten you there!

---

# What to do now that you have the library loaded


## Call some API functions

calling API functions is then a matter of calling the functions as they are listed on http://www.flickr.com/services/api, so if you wish to get all your own photos, you would call:

```
flickr.photos.search({
  user_id: flickr.options.user_id,
  page: 1,
  per_page: 500
}, function(err, result) {
  // result is Flickr's response
});
```
## Call functions that don't *require* authentication, authenticated anyway

Simply add an `authenticated: true` pair to your function call. Compare:

```
flickr.people.getPhotos({
  api_key: ...
  user_id: <your own ID>
  page: 1,
  per_page: 100
}, function(err, result) {
  /*
    This will give public results only, even if we used
    Flickr.authenticate(), because the function does not
    *require* authentication to run. It just runs with
    fewer permissions.
  */
});
```

To:

```
flickr.people.getPhotos({
  api_key: ...
  user_id: <your own ID>
  authenticated: true,
  page: 1,
  per_page: 100
}, function(err, result) {
  /*
    This will now give all public and private results,
    because we explicitly ran this as an authenticated call
  */ 
});
```


## Proxy the Flickr API

If your app is a [connect](https://github.com/senchalabs/connect) or [express](http://expressjs.com) app, you get Flickr API proxying
for free.

Simply use the `.proxy()` function to set everything up and then call your own API route in the same way you would call the Flickr API, minus the security credentials, since the servers side Flickr api object already has those baked in.

As an example, the test.js script for node-flickrapi uses the following code to set up the local API route:

```
var express = require("express");

Flickr.authenticate(FlickrOptions, function(error, flickr) {
  var app = express();
  app.configure(function() {
    ...
    flickr.proxy(app, "/service/rest");
    ...
  });
  ...
});
```

This turns the `/service/rest` route into a full Flickr API proxy, which the browser library can talk to, using `POST` operations.

To verify your proxy route works, simply use `cURL` in the following fashion:

```
curl -X POST -H "Content-Type: application/json"
             -d '{"method":"flickr.photos.search", "text":"red+pandas"}'
             http://127.0.0.1:3000/service/rest/
```

Note that the proxy is "open" in that there is no explicit user management. If you want to make sure only "logged in users" get to use your API proxy route, you can pass an authentication middleware function as third argument to the `.proxy` function:

```
function authenticator(req, res, next) {
  // assuming your session management uses req.session:
  if(req.session.authenticated) {
    return next();
  }
  next({status:403, message: "not authorised to call API methods"});
}

flickr.proxy(app, "/service/rest/", authenticator);
```


## Upload photos to Flickr

If you're running the code server-side, and you've authenticated with Flickr already, you can use the `Flickr.upload` function to upload individual photos, or batches of photos, to your own account (or, the account that is tied to the API key that you're using).

```
Flickr.authenticate(FlickrOptions, function(error, flickr) {
  var uploadOptions = {
    photos: [{
      title: "test",
      tags: [
        "happy fox",
        "test 1"
      ],
      photo: __dirname + "/test.jpg"
    },{
      title: "test2",
      tags: "happy fox image \"test 2\" separate tags",
      photo: __dirname + "/test.jpg"
    }]
  };

  Flickr.upload(uploadOptions, FlickrOptions, function(err, result) {
    if(err) {
      return console.error(error);
    }
    console.log("photos uploaded", result);
  });
});
```

For the list of available upload properties, see the [Flickr Upload API](https://www.flickr.com/services/api/upload.api.html) page.

## Download data from Flickr

You can use this module to very easily download all your Flickr content, using the built in `downsync` function:

```
var Flickr = require("flickrapi"),
    flickrOptions = { ... };
Flickr.authenticate(flickrOptions, flickrapi.downsync());
```

That's all you need to run. The package will generate a data directory with your images in `./data/images` (in several sizes), and the information architecture (metadata, sets, collections, etc) in `./data/ia`.

If you want this in a different directory, you can pass the dir as an argument to the downsync function:

```
var Flickr = require("flickrapi"),
    flickrOptions = { ... };
Flickr.authenticate(flickrOptions, flickrapi.downsync("userdata/me"));
```

This will now create a `./data` for the flickr API information, but also a `./userdata/me/` directory that contains the `images` and `ia` dirs with your personal data.

### Downloading shortcut: one-step downsyncing

If you just want to immediately downsync all your data right now, simply use the `test.js` application with the --downsync runtime argument: add your Flickr API key information to the `.env` file and then run:

`$> node test --downsync`

Run through the authentication procedure, and then just wait for it to finish. Once it's done, you should have a local mirror of all your Flickr data.

#### (Re)syncing with Flickr in your code

(Re)syncing is a mostly a matter or running the downsync function again. This will update anything that was updated or added on Flickr, but will not delete anything from your local mirror that was deleted from Flickr unless specifically told to do so, by passing a second argument (internally known as the "removeDeleted" flag in the code) to the `downsync` function call:

```
var Flickr = require("flickrapi"),
    flickrOptions = { ... };
Flickr.authenticate(flickrOptions, flickrapi.downsync("userdata/me", true));
```

If `true`, this will delete local files that were removed on Flickr (e.g. photos that you didn't like anymore, etc). If `false`, or omitted, no pruning of the local mirror will be performed.

## Use your Flickr data in another application

If you downloaded all your Flickr data, you can use these in your own node apps by "dry loading" Flickr:

```
var Flickr = require("flickrapi"),
    flickrData = Flickr.loadLocally();
```

This will give you an object with the following structure:

```
{
  photos: [photo objects],
  photo_keys: [photo.id array, sorted on publish date],
  photosets: [set objects],
  photoset_keys: [set.id array, sorted on creation date],
  collections: [collection objects],
  collection_keys: [collection.id array, sorted on title],
}
```

Not sure what these objects look like? head over to your `./data/ia` directory and just open a .json file in your favourite text editor.

The `loadLocally` function can take two arguments, namely a location where the ia data can be found, and an options object. If you want to pass in an options object you *must* supply a location, too.

```
flickrData = Flickr.loadLocally("./userdata", {
  loadPrivate: false
});
```

Currently the options object only has one meaningful property, `loadPrivate`, which determines whether or not photos and photosets that are marked "not public" in Flickr show up in the `photo_keys` and `photoset_keys` lists.


# An example of a first run

## Fetching Flickr's most up to date API definitions

On first run, the package will fetch all known methods from Flickr, and cache them for future use. This can take a bit, as there are a fair number of methods, but is inconsequential on subsequent package loading.

## Authenticate your API key with Flickr

On first run, the authentication function will notice that there are no `access_token` and `access_token_secret` values set, and will negotiate these with Flickr using their oauth API, based on the permissions you request for your API key.

By default, the only permissions are "read" permissions, but you can override this by adding a `permissions` property to the options object:

* `permissions: "read"` will give the app read-only access (default)
* `permissions: "write"` will give it read + write access
* `permissions: "delete"` will give it read, write and delete access

Note that you cannot make use of the upload functions unless you authenticate with `write` or `delete` permissions.

### An example run

Running the app will show output such as the following block:

```
$> node app
{ oauth_callback_confirmed: 'true',
  oauth_token: '...',
  oauth_token_secret: '...' }
prompt: oauth_verifier: _
```

Once the app reaches this point it will open a browser, allowing you to consent to the app accessing your most private of private parts. On Flickr, at least. If you agree to authorize it, you will get an authorisation code that you need to pass so that the flickrapi can negotiate access tokens with Flickr.

Doing so continues the program:

```
$> node app
{ oauth_callback_confirmed: 'true',
  oauth_token: '...',
  oauth_token_secret: '...' }
prompt: oauth_verifier: 123-456-789

Add the following variables to your environment:

export FLICKR_USER_ID="12345678%40N12"
export FLICKR_ACCESS_TOKEN="72157634942121673-3e02b190b9720d7d"
export FLICKR_ACCESS_TOKEN_SECRET="99c038c9fc77673e"
```

These are namespaced environment variables, which works really well with env packages like [habitat](https://www.npmjs.com/package/habitat),
so if you're going to use a namespace-aware enviroment parser, simply add these variables
to your environment, or put them in an `.env` file and then parse them in.

If you would prefer to use plain `process.env` consulting, remove the `FLICKR_` namespace
prefix, and then pass `process.env` as options object.

Alternatively, if you don't mind hardcoding values (but **be careful never to check that code
in, because github gets mined by bots for credentials**) you can put them straight into your
source code:

```
var FlickrOptions = {
      api_key: "your API key",
      secret: "your API key secret",
      user_id: "...",
      access_token: "...",
      access_token_secret: "..."
    }
```

The flickrapi package will now be able to authenticate with Flickr without constantly needing to ask you for permission to access data.

## Using your own OAuth callback endpoint

By default the oauth callback is set to "out-of-band". You can see this in the `.env` file as the `FLICK_CALLBACK="oob"` parameter, but if this is omitted the code falls back to oob automatically. For automated processes, or if you don't want your uers to have to type anything in a console, you can override this by setting your own oauth callback endpoint URL.

Using a custom callback endpoint, the oauth procedure will contact the indicated endpoint with the authentication information, rather than requiring your users to manually copy/paste the authentication values.

**Note** your users will still need to authenticate the app from a browser!

To use a custom endpoint, add the URL to the options as the `callback` property:

```
var options = ...;
options.callback: "http://.../...";
Flickr.authenticate(options, function(error, flickr) {
  ...
}
```

You can make your life easier by using an environment variable in the `.env` file rather than hardcoding your endpoint url:

```
export FLICKR_CALLBACK="http://..."
```

The callback URL handler will at its minimum need to implement the following middleware function:

```
function(req, res) {
  res.write("");
  options.exchange(req.query);
}
```

However, having the response tell the user that authorisation was received and that they can safely close this window/tab is generally a good idea.

If you wish to call the exchange function manually, the object expected by `options.exchange` looks like this:

```
{
  oauth_token: "...",
  oauth_verifier: "..."
}
```

# Advanced topics

If all you wanted to know was how to use the flickrapi library, you can stop reading. However, there's some more magic built into the library that you might be interested in, in which case you should totally keep reading.

## Custom authentication: browserless, noAPI, and silent

There are a number of special options that can be set to effect different authentication procedures. Calling the authenticate function with an options object means the following options can also be passed:

```
options = {
  ...

  // console.logs the auth URL instead of opening a browser for it.
  nobrowser: true,

  // only performs authentication, without building the Flickr API.
  noAPI: true,

  // suppress the default console logging on successful authentication.
  silent: true,

  // suppress writing progress bars to stdout
  progress: false

  ...
}
```

If you use the `noAPI` option, the authentication credentials can be extracted from the options object inside the callback function that you pass along. The `options.access_token` and `options.access_token_secret` will contain the result of the authentication procedure.


## (Re)compiling the client-side library

If, for some reason, you want to (re)compile the client-side library, you can run the
```
$> node compile
```
command to (re)generate a flickrapi.js client-side library, saved to the `browser` directory. This generates a sparse library that will let you call all public methods (but currently not any method that requires read-private, write, or delete permissions), but will not tell you what's wrong when errors occur.

If you need the extended information, for instance in a dev setting, use

```
$> node compile dev
```
to generate a flickrapi.dev.js library that has all the information needed for developing work; simply use this during development and use the flickrapi.js library in production.

Note that no `min` version is generated; For development there is no sense in using one, and the savings on the production version are too small to matter (it's only 10kb smaller). If your server can serve content gzipped, the minification will have no effect on the gzipped size anyway (using gzip, the plain library is ~4.5kb, with the dev version being ~30kb).

## The options object

Once you have a Flickr API object in the form if the `flickr` variable, the options can be found as `flickr.options` so you don't need to pass those on all the time. This object may contain any of the following values (some are quite required, others are entirely optional, and some are automatically generated as you make Flickr API calls):

### api_key
your API key.

### secret
your API key secret.

###user_id
your user id, based on your first-time authorisation.

###access_token
the preauthorised Flickr access token.

###access_token_secret
its corresponding secret.

###oauth_timestamp
the timestamp for the last flickr API call.

###oauth_nonce
the cryptographic nonce that request used.

###force_auth

true or false (defaults to false) to indicate whether to force oauth signing for functions that can be called both key-only and authenticated for additional data access (like the photo search function)

###retry_queries
if used, Flickr queries will be retried if they fail.

###afterDownsync
optional; you can bind an arg-less callback function here that is called after a downsync() call finishes.

###permissions
optional 'read', 'write', or 'delete'. defaults to 'read'.

###nobrowser

optional boolean, console.logs the auth URL instead of opening a browser window for it.

###noAPI
optional boolean, performs authentication without building the Flickr API object.

###silent
optional boolean, suppresses the default console logging on successful authentication.

###progress
optional boolean, suppresses writing progress bars to stdout if set to `false`
