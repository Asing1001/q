[![Build Status](https://secure.travis-ci.org/kriskowal/q.png)](http://travis-ci.org/kriskowal/q)

<a href="http://promises-aplus.github.com/promises-spec">
    <img src="http://promises-aplus.github.com/promises-spec/assets/logo-small.png"
         align="right" alt="Promises/A+ logo" />
</a>

If a function cannot return a value or throw an exception without
blocking, it can return a promise instead.  A promise is an object
that represents the return value or the thrown exception that the
function may eventually provide.  A promise can also be used as a
proxy for a [remote object][Q-Connection] to overcome latency.

[Q-Connection]: https://github.com/kriskowal/q-connection

On the first pass, promises can mitigate the “[Pyramid of
Doom][POD]”: the situation where code marches to the right faster
than it marches forward.

[POD]: http://calculist.org/blog/2011/12/14/why-coroutines-wont-work-on-the-web/

```javascript
step1(function (value1) {
    step2(value1, function(value2) {
        step3(value2, function(value3) {
            step4(value3, function(value4) {
                // Do something with value4
            });
        });
    });
});
```

With a promise library, you can flatten the pyramid.

```javascript
Q.fcall(step1)
.then(step2)
.then(step3)
.then(step4)
.then(function (value4) {
    // Do something with value4
}, function (error) {
    // Handle any error from step1 through step4
})
.done();
```

With this approach, you also get implicit error propagation,
just like ``try``, ``catch``, and ``finally``.  An error in
``step1`` will flow all the way to ``step5``, where it’s
caught and handled.

The callback approach is called an “inversion of control”.
A function that accepts a callback instead of a return value
is saying, “Don’t call me, I’ll call you.”.  Promises
[un-invert][IOC] the inversion, cleanly separating the input
arguments from control flow arguments.  This simplifies the
use and creation of API’s, particularly variadic,
rest and spread arguments.

[IOC]: http://www.slideshare.net/domenicdenicola/callbacks-promises-and-coroutines-oh-my-the-evolution-of-asynchronicity-in-javascript


## Getting Started

The Q module can be loaded as:

-   A ``<script>`` tag (creating a ``Q`` global variable): ~3 KB minified and
    gzipped.
-   A Node.js and CommonJS module, available in [npm](https://npmjs.org/) as
    the [q](https://npmjs.org/package/q) package
-   An AMD module
-   A [component](https://github.com/component/component) as ``microjs/q``
-   Using [bower](http://twitter.github.com/bower/) as ``microjs/q``
-   Using [NuGet](http://nuget.org/) as [Q](https://nuget.org/packages/q)

Q can exchange promises with jQuery, Dojo, When.js, WinJS, and more.
Additionally, there are many libraries that produce and consume Q promises for
everything from file system/database access or RPC to templating. For a list of
some of the more popular ones, see [Libraries][].

Please join the Q-Continuum [mailing list](https://groups.google.com/forum/#!forum/q-continuum).

[Libraries]: https://github.com/kriskowal/q/wiki/Libraries


## Tutorial

Promises have a ``then`` method, which you can use to get the eventual
return value (fulfillment) or thrown exception (rejection).

```javascript
promiseMeSomething()
.then(function (value) {
}, function (reason) {
});
```

If ``promiseMeSomething`` returns a promise that gets fulfilled later
with a return value, the first function (the value handler) will be
called with the value.  However, if the ``promiseMeSomething`` function
gets rejected later by a thrown exception, the second function (the
error handler) will be called with the error.

Note that resolution of a promise is always asynchronous: that is, the
value or error handler will always be called in the next turn of the
event loop (i.e. `process.nextTick` in Node). This gives you a nice
guarantee when mentally tracing the flow of your code, namely that
``then`` will always return before either handler is executed.


### Propagation

The ``then`` method returns a promise, which in this example, I’m
assigning to ``outputPromise``.

```javascript
var outputPromise = getInputPromise()
.then(function (input) {
}, function (reason) {
});
```

The ``outputPromise`` variable becomes a new promise for the return
value of either handler.  Since a function can only either return a
value or throw an exception, only one handler will ever be called and it
will be responsible for resolving ``outputPromise``.

-   If you return a value in a handler, ``outputPromise`` will get
    fulfilled.

-   If you throw an exception in a handler ``outputPromise`` will get
    rejected.

-   If you return a **promise** in a handler, ``outputPromise`` will
    “become” that promise.  Being able to become a new promise is useful
    for managing delays, combining results, or recovering from errors.

If the ``getInputPromise()`` promise gets rejected and you omit the
error handler, the **error** will go to ``outputPromise``:

```javascript
var outputPromise = getInputPromise()
.then(function (value) {
});
```

If the input promise gets fulfilled and you omit the value handler, the
**value** will go to ``outputPromise``:

```javascript
var outputPromise = getInputPromise()
.then(null, function (error) {
});
```

Q promises provide a ``fail`` shorthand for ``then`` when you are only
interested in handling the error:

```javascript
var outputPromise = getInputPromise()
.fail(function (error) {
});
```

If you are writing JavaScript for modern engines only or using
CoffeeScript, you may use `catch` instead of `fail`.

Promises also have a ``fin`` function that is like a ``finally`` clause.
The final handler gets called, with no arguments, when the promise
returned by ``getInputPromise()`` either returns a value or throws an
error.  The value returned or error thrown by ``getInputPromise()``
passes directly to ``outputPromise`` unless the final handler fails, and
may be delayed if the final handler returns a promise.

```javascript
var outputPromise = getInputPromise()
.fin(function () {
    // close files, database connections, stop servers, conclude tests
});
```

-   If the handler returns a value, the value is ignored
-   If the handler throws an error, the error passes to ``outputPromise``
-   If the handler returns a promise, ``outputPromise`` gets postponed.  The
    eventual value or error has the same effect as an immediate return
    value or thrown error: a value would be ignored, an error would be
    forwarded.

If you are writing JavaScript for modern engines only or using
CoffeeScript, you may use `finally` instead of `fin`.

### Chaining

There are two ways to chain promises.  You can chain promises either
inside or outside handlers.  The next two examples are equivalent.

```javascript
return getUsername()
.then(function (username) {
    return getUser(username)
    .then(function (user) {
        // if we get here without an error,
        // the value returned here
        // or the exception thrown here
        // resolves the promise returned
        // by the first line
    })
});
```

```javascript
return getUsername()
.then(function (username) {
    return getUser(username);
})
.then(function (user) {
    // if we get here without an error,
    // the value returned here
    // or the exception thrown here
    // resolves the promise returned
    // by the first line
});
```

The only difference is nesting.  It’s useful to nest handlers if you
need to capture multiple input values in your closure.

```javascript
function authenticate() {
    return getUsername()
    .then(function (username) {
        return getUser(username);
    })
    // chained because we will not need the user name in the next event
    .then(function (user) {
        return getPassword()
        // nested because we need both user and password next
        .then(function (password) {
            if (user.passwordHash !== hash(password)) {
                throw new Error("Can't authenticate");
            }
        });
    });
}
```


### Combination

You can turn an array of promises into a promise for the whole,
fulfilled array using ``all``.

```javascript
return Q.all([
    eventualAdd(2, 2),
    eventualAdd(10, 20)
]);
```

If you have a promise for an array, you can use ``spread`` as a
replacement for ``then``.  The ``spread`` function “spreads” the
values over the arguments of the value handler.  The error handler
will get called at the first sign of failure.  That is, whichever of
the recived promises fails first gets handled by the error handler.

```javascript
function eventualAdd(a, b) {
    return Q.spread([a, b], function (a, b) {
        return a + b;
    })
}
```

But ``spread`` calls ``all`` initially, so you can skip it in chains.

```javascript
return getUsername()
.then(function (username) {
    return [username, getUser(username)];
})
.spread(function (username, user) {
});
```

The ``all`` function returns a promise for an array of values.  If one
of the given promise fails, the whole returned promise fails, not
waiting for the rest of the batch.  If you want to wait for all of the
promises to either be fulfilled or rejected, you can use
``allResolved``.

```javascript
Q.allResolved(promises)
.then(function (promises) {
    promises.forEach(function (promise) {
        if (promise.isFulfilled()) {
            var value = promise.valueOf();
        } else {
            var exception = promise.valueOf().exception;
        }
    })
});
```


### Sequences

If you have a number of promise-producing functions that need
to be run sequentially, you can of course do so manually:

```javascript
return foo(initialVal).then(bar).then(baz).then(qux);
```

However, if you want to run a dynamically constructed sequence of
functions, you'll want something like this:

```javascript
var funcs = [foo, bar, baz, qux];

var result = Q.resolve(initialVal);
funcs.forEach(function (f) {
    result = result.then(f);
});
return result;
```

You can make this slightly more compact using `reduce`:

```javascript
return funcs.reduce(function (soFar, f) {
    return soFar.then(f);
}, Q.resolve(initialVal));
```


### Handling Errors

One sometimes-unintuive aspect of promises is that if you throw an
exception in the value handler, it will not be be caught by the error
handler.

```javascript
return foo()
.then(function (value) {
    throw new Error("Can't bar.");
}, function (error) {
    // We only get here if "foo" fails
});
```

To see why this is, consider the parallel between promises and
``try``/``catch``. We are ``try``-ing to execute ``foo()``: the error
handler represents a ``catch`` for ``foo()``, while the value handler
represents code that happens *after* the ``try``/``catch`` block.
That code then needs its own ``try``/``catch`` block.

In terms of promises, this means chaining your error handler:

```javascript
return foo()
.then(function (value) {
    throw new Error("Can't bar.");
})
.fail(function (error) {
    // We get here with either foo's error or bar's error
});
```

### Progress Notification

It's possible for promises to report their progress, e.g. for tasks that take a
long time like a file upload. Not all promises will implement progress
notifications, but for those that do, you can consume the progress values using
a third parameter to ``then``:

```javascript
return uploadFile()
.then(function () {
    // Success uploading the file
}, function (err) {
    // There was an error, and we get the reason for error
}, function (progress) {
    // We get notified of the upload's progress as it is executed
});
```

Like `fail`, Q also provides a shorthand for progress callbacks
called `progress`:

```javascript
return uploadFile().progress(function (progress) {
    // We get notified of the upload's progress
});
```

### The End

When you get to the end of a chain of promises, you should either
return the last promise or end the chain.  Since handlers catch
errors, it’s an unfortunate pattern that the exceptions can go
unobserved.

So, either return it,

```javascript
return foo()
.then(function () {
    return "bar";
});
```

Or, end it.

```javascript
foo()
.then(function () {
    return "bar";
})
.done();
```

Ending a promise chain makes sure that, if an error doesn’t get
handled before the end, it will get rethrown and reported.

This is a stopgap. We are exploring ways to make unhandled errors
visible without any explicit handling.


### The Beginning

Everything above assumes you get a promise from somewhere else.  This
is the common case.  Every once in a while, you will need to create a
promise from scratch.

#### Using ``Q.fcall``

You can create a promise from a value using ``Q.fcall``.  This returns a
promise for 10.

```javascript
return Q.fcall(function () {
    return 10;
});
```

You can also use ``fcall`` to get a promise for an exception.

```javascript
return Q.fcall(function () {
    throw new Error("Can't do it");
});
```

As the name implies, ``fcall`` can call functions, or even promised
functions.  This uses the ``eventualAdd`` function above to add two
numbers.

```javascript
return Q.fcall(eventualAdd, 2, 2);
```


#### Using Deferreds

If you have to interface with asynchronous functions that are callback-based
instead of promise-based, Q provides a few shortcuts (like ``Q.nfcall`` and
friends). But much of the time, the solution will be to use *deferreds*.

```javascript
var deferred = Q.defer();
FS.readFile("foo.txt", "utf-8", function (error, text) {
    if (error) {
        deferred.reject(new Error(error));
    } else {
        deferred.resolve(text);
    }
});
return deferred.promise;
```

Note that a deferred can be resolved with a value or a promise.  The
``reject`` function is a shorthand for resolving with a rejected
promise.

```javascript
// this:
deferred.reject(new Error("Can't do it"));

// is shorthand for:
var rejection = Q.fcall(function () {
    throw new Error("Can't do it");
});
deferred.resolve(rejection);
```

This is a simplified implementation of ``Q.delay``.

```javascript
function delay(ms) {
    var deferred = Q.defer();
    setTimeout(deferred.resolve, ms);
    return deferred.promise;
}
```

This is a simplified implementation of ``Q.timeout``

```javascript
function timeout(promise, ms) {
    var deferred = Q.defer();
    Q.when(promise, deferred.resolve);
    Q.when(delay(ms), function () {
        deferred.reject(new Error("Timed out"));
    });
    return deferred.promise;
}
```

Finally, you can send a progress notification to the promise with
``deferred.notify``.

For illustration, this is a wrapper for XML HTTP requests in the browser. Note
that a more [thorough][XHR] implementation would be in order in practice.

[XHR]: https://github.com/montagejs/mr/blob/71e8df99bb4f0584985accd6f2801ef3015b9763/browser.js#L29-L73

```javascript
function requestOkText(url) {
    var request = new XMLHttpRequest();
    var deferred = Q.defer();

    request.open("GET", url, true);
    request.onload = onload;
    request.onerror = onerror;
    request.onprogress = onprogress;
    request.send();

    function onload() {
        if (request.status === 200) {
            deferred.resolve(request.responseText);
        } else {
            onerror();
        }
    }

    function onerror() {
        deferred.reject("Can't XHR " + JSON.stringify(url));
    }

    function onprogress(event) {
        deferred.notify(event.loaded / event.total);
    }

    return deferred.promise;
}
```


### The Middle

If you are using a function that may return a promise, but just might
return a value if it doesn’t need to defer, you can use the “static”
methods of the Q library.

The ``when`` function is the static equivalent for ``then``.

```javascript
return Q.when(valueOrPromise, function (value) {
}, function (error) {
});
```

All of the other methods on a promise have static analogs with the
same name.

The following are equivalent:

```javascript
return Q.all([a, b]);
```

```javascript
return Q.fcall(function () {
    return [a, b];
})
.all();
```

When working with promises provided by other libraries, you should
convert it to a Q promise.  Not all promise libraries make the same
guarantees as Q and certainly don’t provide all of the same methods.
Most libraries only provide a partially functional ``then`` method.
This thankfully is all we need to turn them into vibrant Q promises.

```javascript
return Q.when($.ajax(...))
.then(function () {
});
```

If there is any chance that the promise you receive is not a Q promise
as provided by your library, you should wrap it using a Q function.
You can even use ``Q.invoke`` as a shorthand.

```javascript
return Q.invoke($, 'ajax', ...)
.then(function () {
});
```


### Over the Wire

A promise can serve as a proxy for another object, even a remote
object.  There are methods that allow you to optimistically manipulate
properties or call functions.  All of these interactions return
promises, so they can be chained.

```
direct manipulation         using a promise as a proxy
--------------------------  -------------------------------
value.foo                   promise.get("foo")
value.foo = value           promise.put("foo", value)
delete value.foo            promise.del("foo")
value.foo(...args)          promise.post("foo", [args])
value.foo(...args)          promise.invoke("foo", ...args)
value(...args)              promise.fapply([args])
value(...args)              promise.fcall(...args)
```

If the promise is a proxy for a remote object, you can shave
round-trips by using these functions instead of ``then``.  To take
advantage of promises for remote objects, check out [Q-Comm][].

[Q-Comm]: https://github.com/kriskowal/q-comm

Even in the case of non-remote objects, these methods can be used as
shorthand for particularly-simple value handlers. For example, you
can replace

```javascript
return Q.fcall(function () {
    return [{ foo: "bar" }, { foo: "baz" }];
})
.then(function (value) {
    return value[0].foo;
});
```

with

```javascript
return Q.fcall(function () {
    return [{ foo: "bar" }, { foo: "baz" }];
})
.get(0)
.get("foo");
```


### Adapting Node

There is a ``makeNodeResolver`` method on deferreds that is handy for
the NodeJS callback pattern.

```javascript
var deferred = Q.defer();
FS.readFile("foo.txt", "utf-8", deferred.makeNodeResolver());
return deferred.promise;
```

And there are ``Q.nfcall`` and ``Q.ninvoke`` for even shorter
expression.

```javascript
return Q.nfcall(FS.readFile, "foo.txt", "utf-8");
```

```javascript
return Q.ninvoke(FS, "readFile", "foo.txt", "utf-8");
```

There is also a ``Q.nfbind`` function that that creates a reusable
wrapper.

```javascript
var readFile = Q.nfbind(FS.readFile);
return readFile("foo.txt", "utf-8");
```

Note that, since promises are always resolved in the next turn of the
event loop, working with streams [can be tricky][streams]. The
essential problem is that, since Node does not buffer input, it is
necessary to attach your ``"data"`` event listeners immediately,
before this next turn comes around. There are a variety of solutions
to this problem, and even some hope that in future versions of Node it
will [be ameliorated][streamsnext].

[streams]: https://groups.google.com/d/topic/q-continuum/xr8znxc_K5E/discussion
[streamsnext]: http://maxogden.com/node-streams#streams.next

### Long Stack Traces

Q comes with *experimental* support for “long stack traces,” wherein the `stack`
property of `Error` rejection reasons is rewritten to be traced along
asynchronous jumps instead of stopping at the most recent one. As an example:

```js
function theDepthsOfMyProgram() {
  Q.delay(100).done(function explode() {
    throw new Error("boo!");
  });
}

theDepthsOfMyProgram();
```

gives a strack trace of:

```
Error: boo!
    at explode (/path/to/test.js:3:11)
From previous event:
    at theDepthsOfMyProgram (/path/to/test.js:2:16)
    at Object.<anonymous> (/path/to/test.js:7:1)
```

Note how you can see the the function that triggered the async operation in the
stack trace! This is very helpful for debugging, as otherwise you end up getting
only the first line, plus a bunch of Q internals, with no sign of where the
operation started.

This feature comes with some caveats, however. First, it does not (<em>yet!</em>)
stitch together multiple asynchronous steps. You only get the one immediately
prior to the operation that throws. Secondly, it comes with a performance
penalty, and so if you are using Q to create many promises in a
performance-critical situation, you will probably want to turn it off.

To turn it off, set

```js
Q.longStackJumpLimit = 0;
```

Then you stack traces will revert to their usual unhelpful selves:

```
Error: boo!
    at explode (/path/to/test.js:3:11)
    at _fulfilled (/path/to/test.js:q:54)
    at resolvedValue.promiseDispatch.done (/path/to/q.js:823:30)
    at makePromise.promise.promiseDispatch (/path/to/q.js:496:13)
    at pending (/path/to/q.js:397:39)
    at process.startup.processNextTick.process._tickCallback (node.js:244:9)
```

## Reference

A method-by-method [Q API reference][reference] is available on the wiki.

[reference]: https://github.com/kriskowal/q/wiki/API-Reference

## More Examples

A growing [examples gallery][examples] is available on the wiki, showing how Q
can be used to make everything better. From XHR to database access to accessing
the Flickr API, Q is there for you.

[examples]: https://github.com/kriskowal/q/wiki/Examples-Gallery

---

Copyright 2009-2012 Kristopher Michael Kowal
MIT License (enclosed)

