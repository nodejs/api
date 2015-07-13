# General Patterns

This document provides an overview of the general architecture and programming
patterns that will be used throughout the JavaScript API. These patterns have
been chosen because of the flexibility they provide in integrating with native
APIs and ease to build new interfaces from, and for their ability to create
performant and more optimizable code.



### Asynchronous Calls

The asynchronous callback pattern is very similar to `setTimeout()`. With the
exception that required arguments must always go first. After the callback, N
arguments can be passed that will propagate to the callback.

Example of pattern:
```js
asyncCall(<arguments>, callback[, ...args])
```

`<arguments>` are the arguments required by the function call.

`callback` is the function that is executed when the asynchronous operation is
complete.

`...args` are the additional values that will propagate to the callback.

Likewise, the callback will always have the context of calling object. Take
this example from the existing node API.
```js
net.connect(8000, function() {
  this.on('data', onData);
});
```

The callback executed after the asynchronous operation uses the common
error-back pattern, in cases where the callback must be called regardless. With
the exception of the additional passed arguments at the end. For example:
```js
function callback(error, <defaults>[, ...args]) { }
```

The following is an example of using this callback pattern to chain together
operations without needing to nest callbacks.
```js
module.exports = function writeAndVerify(path, data, cb) {
  let file = new File(path);

  // Attach callback to created object since it will survive
  // the lifetime of the request.
  file.userCb = cb;

  file.open('wx+', fdOpened, data);
};


function fdOpened(err, data) {
  if (err)
    return this.userCb(err);

  this.write(data, 0, data.length, dataWritten);
}

function dataWritten(err, written, data) {
  if (err) { /* handle error */ }

  // Propagate how many bytes have been written.
  this.close(fdClosed, written);
}

function fdClosed(err, written) {
  if (err) { /* handle error */ }

  // Want to check how large the file is
  this.stat(fileStat, written);
}

function fileStat(err, stats, size) {
  if (err) { /* handle error */ }

  let verifyErr = null;

  // Check if the size of the file is the amount of data written.
  if (stats.size !== size)
    verifyErr = new Error('size verified doesn\'t match');

  this.userCb(verifyErr);
}
```



### Function Signatures

All function parameters before the callback, and including the callback, are
always required. The only optional arguments are those that follow the
callback.



### Asynchronous Operations

Every asynchronous operation returns a request or handle. This includes
something as trivial as a `write()` operation:

```js
net.connect(8000, function(err) {
  let writeReq = this.write('blah');
});
```

Each handle and request already require these objects be created. So it is no
performance disadvantage to return these to the user. While it does offer
greater flexibility in both the amount of information we can communicate and
how it can be used.

For example, as the API evolves a method may be added that returns the number
of bytes processed on the individual request. When data is written the kernel
may only be able to receive so much. Allowing to check how much of the data has
been written out to the kernel. This could be used to inform on the status of a
large upload.

Requests also have their own events. It should be possible to track errors from
individual requests. Along with being notified when the data has been written:
```js
net.connect(8000, function(err) {
  let writeReq = this.write('blah');

  writeReq.onerror(onWriteError);
  writeReq.oncomplete(onWriteComplete);
});
```

Now, why not just use the error-back method in this case? Because it's very
likely developers only want to know when something has gone wrong. Forcing the
callback to be called for every request could be a significant strain on
resources when there is a high volume of requests.

Since the object context always represents the `this` of a request, I have
considered a standard object property that can be accessed to represent the
handle that was used to send the request. It could be diverted around by simply
passing the handle as an argument, but since it's likely operations on the
handle will be necessary from the callback making it available by default in
some form seems appropriate.



### Exception Handling

Exceptions are only thrown when absolutely necessary. In the case of
asynchronous calls, except when critical, the error is passed to the callback.
Though if no callback is passed to the asynchronous call then throwing is
inevitable. Basically, exceptions should only be thrown when there is no other
possible execution path to take.

**TODO:** Consider how to assist the developer to know whether the error
occurred because of a bad argument or because the asynchronous request went
bad.



### Type Coercion

The only coercion allowed is from string to number. This is always done using
the `+` operator. For example:
```
function caller(n) {
  if (typeof n === 'string')
    n = +n;
}
```

Reason for using the `+` operator is because it supports operations that both
`parseInt()` and `parseFloat()` don't support simultaneously.

- `parseFloat('0xff') === 0`

- `parseInt('0.1') === 0`

The `+` operator handles both these cases.

Numeric type coercion is allowed to convert double to (u)int(32|64). Though
range checks should always be in place to ensure values are not allowed to wrap
around. If converting to int `Math.floor()` must be used. Since bitwise ops
don't work properly on negative values.



### Event Emitter

There will be no actual event emitter library. The pattern is as follows:
```
blah.onready(callback[, ...args]);
```

Initially an event can only reference one callback. This is subject to change
in the future, but for the time being it will be easier to implement and test,
especially in cases like error handling.



### Default Callbacks

Allow all callbacks to have a default set for all new instances. For example:
```js
TCP.onready(callback);

let server = new TCP();

// Don't need to set onready() because the default
// has already been set
```

Some of the semantics and edge cases need to be hammered out, but preliminary
testing has shown good performance improvements when many callbacks need to be
set.


### Synchronous Calls

Everything has a place, including synchronous calls. Though the best way to do
this is up for discussion. Either they can be made explicit `fnSync()` or
implicit by doing so automatically when the callback is omitted. Each have
advantages and disadvantages.
