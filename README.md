<h1><p align="center"><img alt="protobuf.js" src="https://github.com/dcodeIO/protobuf.js/raw/master/pbjs.png" width="120" height="104" /></p></h1>
<p align="center"><a href="https://npmjs.org/package/protobufjs"><img src="https://img.shields.io/npm/v/protobufjs.svg" alt=""></a> <a href="https://travis-ci.org/dcodeIO/protobuf.js"><img src="https://travis-ci.org/dcodeIO/protobuf.js.svg?branch=master" alt=""></a> <a href="https://codeclimate.com/github/dcodeIO/protobuf.js/coverage"><img src="https://codeclimate.com/github/dcodeIO/protobuf.js/badges/coverage.svg" /></a> <a href="https://codeclimate.com/github/dcodeIO/protobuf.js"><img src="https://codeclimate.com/github/dcodeIO/protobuf.js/badges/gpa.svg" /></a> <a href="https://npmjs.org/package/protobufjs"><img src="https://img.shields.io/npm/dm/protobufjs.svg" alt=""></a> <a href="https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=dcode%40dcode.io&item_name=Open%20Source%20Software%20Donation&item_number=dcodeIO%2Fprotobuf.js"><img alt="donate ❤" src="https://img.shields.io/badge/donate-❤-ff2244.svg"></a></p>

**Protocol Buffers** are a language-neutral, platform-neutral, extensible way of serializing structured data for use in communications protocols, data storage, and more, originally designed at Google ([see](https://developers.google.com/protocol-buffers/)).

**protobuf.js** is a pure JavaScript implementation with [TypeScript](https://www.typescriptlang.org) support for [node.js](https://nodejs.org) and the browser. It's super easy to use, blazingly fast and works out of the box with [.proto](https://developers.google.com/protocol-buffers/docs/proto) files!

Contents
--------

* [Installation](#installation)<br />
  How to include protobuf.js in your project.

* [Usage](#usage)<br />
  A brief introduction to using the toolset.

* [Examples](#examples)<br />
  A few examples to get you started.

  * [Using .proto files](#using-proto-files)
  * [Using JSON descriptors](#using-json-descriptors)
  * [Using reflection only](#using-reflection-only)
  * [Using custom classes](#using-custom-classes)
  * [Using services](#using-services)
  * [Usage with TypeScript](#usage-with-typescript)

* [Command line](#command-line)<br />
  How to use the command line utility.

  * [pbjs for JavaScript](#pbjs-for-javascript)
  * [pbts for TypeScript](#pbts-for-typescript)
  * [Reflection vs. static code](#reflection-vs-static-code)
  * [Command line API](#command-line-api)

* [Additional documentation](#additional-documentation)<br />
  A list of available documentation resources.

* [Performance](#performance)<br />
  A few internals and a benchmark on performance.

* [Compatibility](#compatibility)<br />
  Notes on compatibility regarding browsers and optional libraries.

* [Building](#building)<br />
  How to build the library and its components yourself.

Installation
---------------

### node.js

```
$> npm install protobufjs [--save --save-prefix=~]
```

```js
var protobuf = require("protobufjs");
```

### Browsers

Development:
```
<script src="//cdn.rawgit.com/dcodeIO/protobuf.js/6.X.X/dist/protobuf.js"></script>
```

Production:
```
<script src="//cdn.rawgit.com/dcodeIO/protobuf.js/6.X.X/dist/protobuf.min.js"></script>
```

**NOTE:** Remember to replace the version tag with the exact [release](https://github.com/dcodeIO/protobuf.js/tags) your project depends upon.

The `protobuf` namespace will always be available globally / also supports AMD loaders.

### Distributions

The library supports both reflection-based and code-based use cases:

1. Parsing protocol buffer definitions (.proto files) to reflection
2. Loading JSON descriptors to reflection
3. Generating static code without any reflection features

Where bundle size is a factor, there is a suitable distribution for each of these:

|         | Gzipped | Downloads                    | How to require                  | Description
|---------|---------|------------------------------|---------------------------------|-------------
| full    | 18.5kb  | [dist][dist-full]            | `require("protobufjs")`         | All features. Works with everything.
| light   | 15.5kb  | [dist/light][dist-light]     | `require("protobufjs/light")`   | All features except tokenizer, parser and bundled common types. Works with JSON definitions, pure reflection and static code.
| minimal | 6.0kb+  | [dist/minimal][dist-minimal] | `require("protobufjs/minimal")` | Just enough to run static code. No reflection.

In case of doubt it is safe to just use the full library.

[dist-full]: https://github.com/dcodeIO/protobuf.js/tree/master/dist
[dist-light]: https://github.com/dcodeIO/protobuf.js/tree/master/dist/light
[dist-minimal]: https://github.com/dcodeIO/protobuf.js/tree/master/dist/minimal

Usage
-----

Each message type provides a set of methods with each method doing just one thing. This allows to avoid unnecessary operations where [performance](#performance) is a concern but also forces a user to perform verification explicitly where necessary - for example when dealing with user input.

Note that **Message** below refers to any message type. See the next section for the definition of a [valid message](#valid-message).

* **Message.verify**(message: `Object`): `null|string`<br />
  explicitly performs verification prior to encoding a plain object. Instead of throwing, it returns the error message as a string, if any.

  ```js
  var payload = "invalid (not an object)";
  var err = AwesomeMessage.verify(payload);
  if (err)
    throw Error(err);
  ```

* **Message.encode**(message: `Message|Object` [, writer: `Writer`]): `Writer`<br />
  is an automatically generated message specific encoder expecting a valid message or plain object. Note that this method does not implicitly verify the message and that it's up to the user to make sure that the data can actually be encoded properly.

  ```js
  var buffer = AwesomeMessage.encode(message).finish();
  ```

* **Message.encodeDelimited**(message: `Message|Object` [, writer: `Writer`]): `Writer`<br />
  works like `Message.encode` but additionally prepends the length of the message as a varint.

* **Message.decode**(reader: `Reader|Uint8Array`): `Message`<br />
  is an automatically generated message specific decoder. If required fields are missing, it throws a `util.ProtocolError` with an `instance` property set to the so far decoded message. If the wire format is invalid, it throws an `Error`. The result is a runtime message.

  ```js
  try {
    var decodedMessage = AwesomeMessage.decode(buffer);
  } catch (e) {
      if (e instanceof protobuf.util.ProtocolError) {
        // e.instance holds the so far decoded message with missing required fields
      } else {
        // wire format is invalid
      }
  }
  ```

* **Message.decodeDelimited**(reader: `Reader|Uint8Array`): `Message`<br />
  works like `Message.decode` but additionally reads the length of the message prepended as a varint.

* **Message.create**(properties: `Object`): `Message`<br />
  quickly creates a new runtime message from known to be valid properties without any conversion being performed. Where applicable, it is recommended to prefer `Message.create` over `Message.fromObject`.

  ```js
  var message = AwesomeMessage.create({ awesomeField: "AwesomeString" });
  ```

* **Message.fromObject**(object: `Object`): `Message`<br />
  converts any plain object to a runtime message. Tries to convert whatever is specified (use `Message.verify` before if necessary).

  ```js
  var message = AwesomeMessage.fromObject({ awesomeField: 42 });
  // converts awesomeField to a string
  ```

* **Message.toObject**(message: `Message` [, options: `ConversionOptions`]): `Object`<br />
  converts a runtime message to a plain object.

  ```js
  var object = AwesomeMessage.toObject(message, {
    enums: String,  // enums as string names
    longs: String,  // longs as strings (requires long.js)
    bytes: String,  // bytes as base64 encoded strings
    defaults: true, // includes default values
    arrays: true,   // populates empty arrays (repeated fields) even if defaults=false
    objects: true,  // populates empty objects (map fields) even if defaults=false
    oneofs: true    // includes virtual oneof fields set to the present field's name
  });
  ```

  See also: [ConversionOptions](http://dcode.io/protobuf.js/global.html#ConversionOptions)

### Valid message

A valid message is an object not missing any required fields and exclusively using JS types for its fields that are understood by the wire format writer.

* Calling `Message.verify` with any object returns `null` if the object can be encoded as-is and otherwise the error as a string.
* Calling `Message.create` or `Message.encode` must be called with a valid message.
* Calling `Message.fromObject` with any object naively converts all values to the optimal JS type.

| Field type | Expected JS type (create, encode) | Naive conversion (fromObject)
|------------|-----------------------------------|------------------------------
| s-/u-/int32<br />s-/fixed32 | `Number` (32 bit integer) | `value | 0` if signed<br /> `value >>> 0` if unsigned
| s-/u-/int64<br />s-/fixed64 | `Long`-like (optimal)<br />`Number` (53 bit integer) | `Long.fromValue(value)` with long.js<br />`parseInt(value, 10)` otherwise
| float<br />double | `Number` | `Number(value)`
| bool | `Boolean` | `Boolean(value)`
| string | `String` | `String(value)`
| bytes | `Uint8Array` (optimal)<br />`Buffer` (optimal under node)<br />`Array.<Number>` (8 bit integers)<br />`String` (base64) | `base64.decode(value)` if a String<br />`Object` with non-zero `.length` is kept
| enum | `Number` (32 bit integer) | Looks up the numeric id if a string
| message | Valid message | `Message.fromObject(value)`

* Explicit `undefined` and `null` are considered as not set when optional.
* Repeated fields are `Array.<T>`.
* Map fields are `Object.<string,T>` with the key being the string representation of the respective value or an 8 characters long binary hash string for `Long`-likes.
* `String` refers to both objects and values while `Number` refers to values only.
* Types marked as *optimal* provide the best performance because no conversion step (i.e. number to low and high bits or base64 string to buffer) is required.

Examples
--------

### Using .proto files

It is possible to load existing .proto files using the full library, which parses and compiles the definitions to ready to use (reflection-based) message classes:

```protobuf
// awesome.proto
package awesomepackage;
syntax = "proto3";

message AwesomeMessage {
    string awesome_field = 1; // becomes awesomeField
}
```

```js
protobuf.load("awesome.proto", function(err, root) {
    if (err)
        throw err;

    // Obtain a message type
    var AwesomeMessage = root.lookupType("awesomepackage.AwesomeMessage");

    // Exemplary payload
    var payload = { awesomeField: "AwesomeString" };

    // Verify the payload if necessary (i.e. when possibly incomplete or invalid)
    var errMsg = AwesomeMessage.verify(payload);
    if (errMsg)
        throw Error(errMsg);

    // Create a new message
    var message = AwesomeMessage.fromObject(payload); // or use .create if payload is already known to be valid

    // Encode a message to an Uint8Array (browser) or Buffer (node)
    var buffer = AwesomeMessage.encode(message).finish();
    // ... do something with buffer

    // Decode an Uint8Array (browser) or Buffer (node) to a message
    var message = AwesomeMessage.decode(buffer);
    // ... do something with message

    // If the application uses length-delimited buffers, there is also encodeDelimited and decodeDelimited.

    // Maybe convert the message back to a plain object
    var object = AwesomeMessage.toObject(message, {
        longs: String,
        enums: String,
        bytes: String,
        // see ConversionOptions
    });
});
```

Additionally, promise syntax can be used by omitting the callback, if preferred:

```js
protobuf.load("awesome.proto")
    .then(function(root) {
       ...
    });
```

### Using JSON descriptors

The library utilizes JSON descriptors that are equivalent to a .proto definition. For example, the following is identical to the .proto definition seen above:

```json
// awesome.json
{
  "nested": {
    "AwesomeMessage": {
      "fields": {
        "awesomeField": {
          "type": "string",
          "id": 1
        }
      }
    }
  }
}
```

JSON descriptors closely resemble the internal reflection structure:

| Type (T)           | Extends            | Type-specific properties
|--------------------|--------------------|-------------------------
| *ReflectionObject* |                    | options
| *Namespace*        | *ReflectionObject* | nested
| Root               | *Namespace*        | **nested**
| Type               | *Namespace*        | **fields**
| Enum               | *ReflectionObject* | **values**
| Field              | *ReflectionObject* | rule, **type**, **id**
| MapField           | Field              | **keyType**
| OneOf              | *ReflectionObject* | **oneof** (array of field names)
| Service            | *Namespace*        | **methods**
| Method             | *ReflectionObject* | type, **requestType**, **responseType**, requestStream, responseStream

* **Bold properties** are required. *Italic types* are abstract.
* `T.fromJSON(name, json)` creates the respective reflection object from a JSON descriptor
* `T#toJSON()` creates a JSON descriptor from the respective reflection object (its name is used as the key within the parent)

Exclusively using JSON descriptors instead of .proto files enables the use of just the light library (the parser isn't required in this case).

A JSON descriptor can either be loaded the usual way:

```js
protobuf.load("awesome.json", function(err, root) {
    if (err) throw err;

    // Continue at "Obtain a message type" above
});
```

Or it can be loaded inline:

```js
var jsonDescriptor = require("./awesome.json"); // exemplary for node

var root = protobuf.Root.fromJSON(jsonDescriptor);

// Continue at "Obtain a message type" above
```

### Using reflection only

Both the full and the light library include full reflection support. One could, for example, define the .proto definitions seen in the examples above using just reflection:

```js
...
var Root  = protobuf.Root,
    Type  = protobuf.Type,
    Field = protobuf.Field;

var AwesomeMessage = new Type("AwesomeMessage").add(new Field("awesomeField", 1, "string"));

var root = new Root().define("awesomepackage").add(AwesomeMessage);

// Continue at "Create a new message" above
...
```

Detailed information on the reflection structure is available within the [API documentation](#additional-documentation).

### Using custom classes

Runtime message classes can also be extended with custom functionality and it is also possible to register a custom constructor with a reflected message type:

```js
...

// Define a custom constructor
function AwesomeMessage(properties) {
    // custom initialization code
    ...
}

// Register the custom constructor with its reflected type (*)
root.lookupType("awesomepackage.AwesomeMessage").ctor = AwesomeMessage;

// Define custom functionality
AwesomeMessage.customStaticMethod = function() { ... };
AwesomeMessage.prototype.customInstanceMethod = function() { ... };

// Continue at "Create a new message" above
```

(*) Besides referencing its reflected type through `AwesomeMessage.$type` and `AwesomeMesage#$type`, the respective custom class is automatically populated with:

* `AwesomeMessage.create`
* `AwesomeMessage.encode` and `AwesomeMessage.encodeDelimited`
* `AwesomeMessage.decode` and `AwesomeMessage.decodeDelimited`
* `AwesomeMessage.verify`
* `AwesomeMessage.fromObject`, `AwesomeMessage.toObject`, `AwesomeMessage#toObject` and `AwesomeMessage#toJSON`

Afterwards, decoded messages of this type are `instanceof AwesomeMessage`.

Alternatively, it is also possible to reuse and extend the internal constructor if custom initialization code is not required:

```js
...

// Reuse the internal constructor
var AwesomeMessage = root.lookupType("awesomepackage.AwesomeMessage").ctor;

// Define custom functionality
AwesomeMessage.customStaticMethod = function() { ... };
AwesomeMessage.prototype.customInstanceMethod = function() { ... };

// Continue at "Create a new message" above
```

### Using services

The library also supports consuming services but it doesn't make any assumptions about the actual transport channel. Instead, a user must provide a suitable RPC implementation, which is an asynchronous function that takes the reflected service method, the binary request and a node-style callback as its parameters:

```js
function rpcImpl(method, requestData, callback) {
    // perform the request using an HTTP request or a WebSocket for example
    var responseData = ...;
    // and call the callback with the binary response afterwards:
    callback(null, responseData);
}
```

Example:

```protobuf
// greeter.proto
syntax = "proto3";

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}
```

```js
...
var Greeter = root.lookup("Greeter");
var greeter = Greeter.create(/* see above */ rpcImpl, /* request delimited? */ false, /* response delimited? */ false);

greeter.sayHello({ name: 'you' }, function(err, response) {
    console.log('Greeting:', response.message);
});
```

Services also support promises:

```js
greeter.sayHello({ name: 'you' })
    .then(function(response) {
        console.log('Greeting:', response.message);
    });
```

There is also an [example for streaming RPC](https://github.com/dcodeIO/protobuf.js/blob/master/examples/streaming-rpc.js).

Note that the service API is meant for clients. Implementing a server-side endpoint pretty much always requires transport channel (i.e. http, websocket, etc.) specific code with the only common denominator being that it decodes and encodes messages.

### Usage with TypeScript

The library ships with its own [type definitions](https://github.com/dcodeIO/protobuf.js/blob/master/index.d.ts) and modern editors like [Visual Studio Code](https://code.visualstudio.com/) should automatically detect and use them for code completion when following this pattern:

```ts
// node.js
import * as protobuf from "protobufjs";
import * as Long from "long"; // optional

// browser only (alternatively)
// import * as protobuf from "./node_modules/protobufjs/index.js";
// import * as Long from "./node_modules/long/dist/long.js"; // optional

protobuf.load("awesome.proto", function(err, root) {
  if (err)
    throw err;

  // example code
  const AwesomeMessage = root.lookupType("awesomepackage.AwesomeMessage");

  let message = AwesomeMessage.create({ awesomeField: "hello" });
  console.log(`message = ${JSON.stringify(message)}`);

  let buffer = AwesomeMessage.encode(message).finish();
  console.log(`buffer = ${Array.prototype.toString.call(buffer)}`);

  let decoded = AwesomeMessage.decode(buffer);
  console.log(`decoded = ${JSON.stringify(decoded)}`);
});
```

**Note:** Dynamically generated runtime message classes cannot be typed, technically, so you must either access its fields using `message["awesomeField"]` notation or you can utilize [typings of its static counterpart](#pbts-for-typescript) for full typings support.

If you generated static code to `bundle.js` using the CLI and its type definitions to `bundle.d.ts` instead, then you can just do:

```ts
import * as root from "./bundle.js";

// example code
var AwesomeMessage = root.AwesomeMessage;
var message = AwesomeMessage.create({ awesomeField: "hello" });
var buffer = AwesomeMessage.encode(message).finish();
...
```

**Note:** When using [long.js](https://github.com/dcodeIO/long.js), make sure to add [@types/long](https://www.npmjs.com/package/@types/long) to your project's dependencies. Likewise, when building for node, make sure to add [@types/node](https://www.npmjs.com/package/@types/node). Alternatively, if you are not using one of these, there are two stubs available that can be referenced instead of the respective full type definition:

```ts
/// <reference path="./node_modules/protobufjs/stub-long.d.ts" />
/// <reference path="./node_modules/protobufjs/stub-node.d.ts" />
```

Command line
------------

**Note** that moving the CLI to [its own package](./cli) is a work in progress. At the moment, it's still part of the main package.

The command line interface (CLI) can be used to translate between file formats and to generate static code as well as TypeScript definitions.

### pbjs for JavaScript

```
Translates between file formats and generates static code.

  -t, --target     Specifies the target format. Also accepts a path to require a custom target.

                  json          JSON representation
                  json-module   JSON representation as a module
                  proto2        Protocol Buffers, Version 2
                  proto3        Protocol Buffers, Version 3
                  static        Static code without reflection (non-functional on its own)
                  static-module Static code without reflection as a module

  -p, --path       Adds a directory to the include path.

  -o, --out        Saves to a file instead of writing to stdout.

  --sparse         Exports only those types referenced from a main file (experimental).

   Module targets only:

  -w, --wrap       Specifies the wrapper to use. Also accepts a path to require a custom wrapper.

                   default   Default wrapper supporting both CommonJS and AMD
                   commonjs  CommonJS wrapper
                   amd       AMD wrapper
                   es6       ES6 wrapper (implies --es6)

  -r, --root       Specifies an alternative protobuf.roots name.

  -l, --lint       Linter configuration. Defaults to protobuf.js-compatible rules:

                   eslint-disable block-scoped-var, no-redeclare, no-control-regex, no-prototype-builtins

  --es6            Enables ES6 syntax (const/let instead of var)

   Proto sources only:

  --keep-case      Keeps field casing instead of converting to camel case.

   Static targets only:

  --no-create      Does not generate create functions used for reflection compatibility.
  --no-encode      Does not generate encode functions.
  --no-decode      Does not generate decode functions.
  --no-verify      Does not generate verify functions.
  --no-convert     Does not generate convert functions like from/toObject
  --no-delimited   Does not generate delimited encode/decode functions.
  --no-beautify    Does not beautify generated code.
  --no-comments    Does not output any JSDoc comments.

  --force-long     Enfores the use of 'Long' for s-/u-/int64 and s-/fixed64 fields.
  --force-message  Enfores the use of runtime messages instead of plain objects.

usage: pbjs [options] file1.proto file2.json ...  (or)  other | pbjs [options] -
```

For production environments it is recommended to bundle all your .proto files to a single .json file, which minimizes the number of network requests and avoids any parser overhead (hint: works with just the **light** library):

```
$> pbjs -t json file1.proto file2.proto > bundle.json
```

Now, either include this file in your final bundle:

```js
var root = protobuf.Root.fromJSON(require("./bundle.json"));
```

or load it the usual way:

```js
protobuf.load("bundle.json", function(err, root) {
    ...
});
```

Generated static code, on the other hand, works with just the **minimal** library. For example

```
$> pbjs -t static-module -w commonjs -o compiled.js file1.proto file2.proto
```

will generate static code for definitions within `file1.proto` and `file2.proto` to a CommonJS module `compiled.js`.

**ProTip!** Documenting your .proto files with `/** ... */`-blocks or (trailing) `/// ...` lines translates to generated static code.


### pbts for TypeScript

```
Generates TypeScript definitions from annotated JavaScript files.

  -o, --out       Saves to a file instead of writing to stdout.

  -g, --global    Name of the global object in browser environments, if any.

  --no-comments   Does not output any JSDoc comments.

  Internal flags:

  -n, --name      Wraps everything in a module of the specified name.

  -m, --main      Whether building the main library without any imports.

usage: pbts [options] file1.js file2.js ...  (or)  other | pbts [options] -
```

Picking up on the example above, the following not just generates static code to a CommonJS module `compiled.js` but also its respective TypeScript definitions to `compiled.d.ts`:

```
$> pbjs -t static-module -w commonjs -o compiled.js file1.proto file2.proto
$> pbts -o compiled.d.ts compiled.js
```

Additionally, TypeScript definitions of static modules are compatible with their reflection-based counterparts (i.e. as exported by JSON modules), as long as the following conditions are met:

1. Instead of using `new SomeMessage(...)`, always use `SomeMessage.create(...)` because reflection objects do not provide a constructor.
2. Types, services and enums must start with an uppercase letter to become available as properties of the reflected types as well (i.e. to be able to use `MyMessage.MyEnum` instead of `root.lookup("MyMessage.MyEnum")`).

For example, the following generates a JSON module `bundle.js` and a `bundle.d.ts`, but no static code:

```
$> pbjs -t json-module -w commonjs -o bundle.js file1.proto file2.proto
$> pbjs -t static-module file1.proto file2.proto | pbts -o bundle.d.ts -
```

### Reflection vs. static code

While using .proto files directly requires the full library respectively pure reflection/JSON the light library, pretty much all code but the relatively short descriptors is shared.

Static code, on the other hand, requires just the minimal library, but generates additional, albeit editable, source code without any reflection features.

There is no significant difference performance-wise as the code generated statically is pretty much the same as generated at runtime and both are largely interchangeable as seen in the previous section.

### Command line API

Both utilities can be used programmatically by providing command line arguments and a callback to their respective `main` functions:

```js
var pbjs = require("protobufjs/cli/pbjs"); // or require("protobufjs/cli").pbjs / .pbts

pbjs.main([ "--target", "json-module", "path/to/myproto.proto" ], function(err, output) {
    if (err)
        throw err;
    // do something with output
});
```

Additional documentation
------------------------

#### Protocol Buffers
* [Google's Developer Guide](https://developers.google.com/protocol-buffers/docs/overview)

#### protobuf.js
* [API Documentation](http://dcode.io/protobuf.js)
* [CHANGELOG](https://github.com/dcodeIO/protobuf.js/blob/master/CHANGELOG.md)
* [Frequently asked questions](https://github.com/dcodeIO/protobuf.js/wiki) on our wiki

#### Community
* [Questions and answers](http://stackoverflow.com/search?tab=newest&q=protobuf.js) on StackOverflow

Performance
-----------
The package includes a benchmark that tries to compare performance to native JSON as far as this is possible. On an i7-2600K running node 6.9.1 it yields:

```
benchmarking encoding performance ...

Type.encode to buffer x 547,361 ops/sec ±0.27% (94 runs sampled)
JSON.stringify to string x 310,848 ops/sec ±0.73% (92 runs sampled)
JSON.stringify to buffer x 173,608 ops/sec ±1.51% (86 runs sampled)

      Type.encode to buffer was fastest
   JSON.stringify to string was 43.5% slower
   JSON.stringify to buffer was 68.7% slower

benchmarking decoding performance ...

Type.decode from buffer x 1,294,378 ops/sec ±0.86% (90 runs sampled)
JSON.parse from string x 291,944 ops/sec ±0.72% (92 runs sampled)
JSON.parse from buffer x 256,325 ops/sec ±1.50% (90 runs sampled)

    Type.decode from buffer was fastest
     JSON.parse from string was 77.4% slower
     JSON.parse from buffer was 80.3% slower

benchmarking combined performance ...

Type to/from buffer x 254,126 ops/sec ±1.13% (91 runs sampled)
JSON to/from string x 122,896 ops/sec ±1.29% (90 runs sampled)
JSON to/from buffer x 88,005 ops/sec ±0.87% (89 runs sampled)

        Type to/from buffer was fastest
        JSON to/from string was 51.7% slower
        JSON to/from buffer was 65.3% slower

benchmarking verifying performance ...

Type.verify x 6,246,765 ops/sec ±2.00% (87 runs sampled)

benchmarking message from object performance ...

Type.fromObject x 2,892,973 ops/sec ±0.70% (92 runs sampled)

benchmarking message to object performance ...

Type.toObject x 3,601,738 ops/sec ±0.72% (93 runs sampled)
```

Note that JSON is a native binding nowadays and as such is about as fast as it possibly can get. So, how can protobuf.js be faster?

* The benchmark is [somewhat flawed](https://github.com/dcodeIO/protobuf.js/blob/master/bench/index.js).
* Reader and writer interfaces configure themselves according to the environment to eliminate redundant conditionals.
* Node-specific reader and writer subclasses benefit from node's buffer binding.
* Reflection has built-in code generation that builds type-specific encoders, decoders and verifiers at runtime.
* Encoders and decoders do not implicitly call `verify` on messages to avoid unnecessary overhead where messages are already known to be valid. It's up to the user to call `verify` where necessary.
* Quite a bit of V8-specific profiling is accountable for everything else.

You can also run [the benchmark](https://github.com/dcodeIO/protobuf.js/blob/master/bench/index.js) ...

```
$> npm run bench
```

and [the profiler](https://github.com/dcodeIO/protobuf.js/blob/master/bench/prof.js) yourself (the latter requires a recent version of node):

```
$> npm run prof <encode|decode|encode-browser|decode-browser> [iterations=10000000]
```

Note that as of this writing, the benchmark suite performs significantly slower on node 7.2.0 compared to 6.9.1 because moths.

Compatibility
-------------

[![Sauce Test Status](https://saucelabs.com/browser-matrix/protobuf.svg)](https://saucelabs.com/u/protobuf)

* Because the internals of this package do not rely on `google/protobuf/descriptor.proto`, options are parsed and presented literally.
* If typed arrays are not supported by the environment, plain arrays will be used instead.
* Support for pre-ES5 environments (except IE8) can be achieved by [using a polyfill](https://github.com/dcodeIO/protobuf.js/blob/master/scripts/polyfill.js).
* Support for [Content Security Policy](https://w3c.github.io/webappsec-csp/)-restricted environments (like Chrome extensions without [unsafe-eval](https://developer.chrome.com/extensions/contentSecurityPolicy#relaxing-eval)) can be achieved by generating and using static code instead.
* If a proper way to work with 64 bit values (uint64, int64 etc.) is required, just install [long.js](https://github.com/dcodeIO/long.js) alongside this library. All 64 bit numbers will then be returned as a `Long` instance instead of a possibly unsafe JavaScript number ([see](https://github.com/dcodeIO/long.js)).

Building
--------

To build the library or its components yourself, clone it from GitHub and install the development dependencies:

```
$> git clone https://github.com/dcodeIO/protobuf.js.git
$> cd protobuf.js
$> npm install
```

Building the respective development and production versions with their respective source maps to `dist/`:

```
$> npm run build
```

Building the documentation to `docs/`:

```
$> npm run docs
```

Building the TypeScript definition to `index.d.ts`:

```
$> npm run types
```

### Browserify integration

By default, protobuf.js integrates into any browserify build-process without requiring any optional modules. Hence:

* If int64 support is required, explicitly require the `long` module somewhere in your project as it will be excluded otherwise. This assumes that a global `require` function is present that protobuf.js can call to obtain the long module.

  If there is no global `require` function present after bundling, it's also possible to assign the long module programmatically:

  ```js
  var Long = ...;
  
  protobuf.util.Long = Long;
  protobuf.configure();
  ```

* If you have any special requirements, there is [the bundler](https://github.com/dcodeIO/protobuf.js/blob/master/scripts/bundle.js) for reference.

**License:** [BSD 3-Clause License](https://opensource.org/licenses/BSD-3-Clause)
