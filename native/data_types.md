### Abstracting Data Types

An initial effort to abstract the data types between JS engine(s) and addon(s).

#### Definitions: (Alphabetical order)

 - Addon or Native Addon : Node.JS modules developed using C,C++.
 - Chakra : A Javascript engine developed by Microsoft
 - ECMAScript : The scripting language standardized by Ecma International in the ECMA-262 
   specification and ISO/IEC 16262. (ref Wikipedia)
 - IO.JS : [Bringing ES6 to the Node Community!](http://iojs.org)
 - JS Engine : Javascript engine that powers Node.JS or it's forks (Chakra, SpiderMonkey,
    V8 ...)
 - JXcore : [Node.JS on mobile and embedded devices](http://jxcore.io/)
 - MDN : Mozilla Developers Network
 - Node : [Node.JS](http://nodejs.org) and it's previously mentioned sub projects
 - SpiderMonkey : A Javascript engine developed by Mozilla.
 - V8 : A Javascript engine developed by Google.


#### JS Types

The latest ECMAScript standard defines seven data types: (ref. MDN and contributors [2015] 
[Online] Available from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures))

    Six data types that are primitives:
        Boolean
        Null
        Undefined
        Number
        String
        Symbol (new in ECMAScript 6)
    and 
        Object

Initial list of the corresponding native types are given below;

```
enum JSTypes {
  Int32, // Number
  Double, // Number
  Boolean, // Boolean
  String, // String
  Object, // Object
  Symbol, // Symbol
  Undefined, // Undefined
  Null, // Null
  Error, // Specialized Object
  Function // Callable Object
};
```

Given list does not provide a definition for `ObjectTemplate` or `FunctionTemplate` V8 types. 
Assumption is that, these sub specializations can be provided via API methods. 

`JS Engine` native type definition and / or behavior is engine and version specific. 
However, regardless from the implementation details, previously mentioned data types are 
available for all the mature JS engines.

Following the **common** `JS engine` behavior, there are two types of variable representation;

 - Persistent / Rooted Variable
 - Scoped / Temp Variable

`Node` is not expected to manage the memory for JS Engine variable for any of the given 
options above. One of the [verified](https://github.com/jxcore/jxcore/blob/master/doc/native/Embedding_Basics.md) 
assumptions show that; a `void ptr` can point to a native `JS Engine` variable without dealing 
with its memory management.

`void *data_;` -> JS Engine Variable

Similar to local C,C++ variables, if `data_` holds a scope variable, its value is not 
expected to be available after function's scope. Persistent or rooted variables are not 
restricted with function scope. However, they need to be freed manually.

V8 API mentions a `weak-able` variable. This type of variable is not expected to be GC'ed 
by `JS engine` until there is no Javascript side variable referencing to it. 

`JS engines` by default do not GC any active variable. As long as there is one another 
(non GC-able) variable holding a reference to the subject variable, it should be reachable. 
However, V8 also doesn't guarantee the `native representation` of non-persistent but 
weakened variable will persist. As a result, from the native side scope/data point of view, 
`weak-able` variables do not bring a new condition or challenge. All the mature `JS Engines` 
provide somehow a similar functionality.

V8 engine (similar to other mature `JS engines`) has a GC event mechanism on C,C++ side to 
tell a weak variable has finally / or about to be garbage collected. Although the timing of 
the GC needs a separate discussion, it has not much to do with data type abstraction subject. 

`JS Engine` uses a thread, context and or scope variable for each interaction with its API. 
`Node` also stores an environment / common object per application or thread instance. `Node` 
and / or `JS Engine` already manages the memory of this global variable. Based on similarly 
verified previous assumption, `void ptr` pointing the underlying global variable may serve 
the same purpose.

`void *env_;` -> Env, commons, Isolate, JSContext ...

Until this point, three variables of the expected interface type is discussed.

```
  void *env_;
  void *data_;
```

Before closing up, some other pieces may require an additional attention.

 - Underlying implementation may require wrapping either `env_` or `data_` prior to 
 delivering it to addon land. ie. engine specific scenarios for specialized type implementations.

 - It may not be clear on the engine side whether a soft wrapped (`void ptr`) data type is 
 rooted/persistent or not.

For the last two items and more, `uintptr_t  flags_` provides an implementation 
specific information for the interface itself.

Initially suggested interface is given below

```
struct JSValue {
  void *env_;
  void *data_;
  uintptr_t  flags_;
};
```

Previously given data structure hides the implementation details behind a flag member and
`void pointers`. One another idea is hiding `JSValue` implementation details. (see `Opaque Pointers`)