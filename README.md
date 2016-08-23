# API Working Group

The API WG will focus on creating two different specifications. A low-level
JavaScript API and an API/ABI compatible native layer.

There is some overlap with goals of the [Nan](https://github.com/nodejs/nan) 
working group, however this working group is not a replacement for the Nan 
working group.  Its focus is broader and the timeline for the proposed
APIs is relatively long term.  As such the Nan working group will continue its
work/support of the module to maintain an api across node releases for 
the forseable future.


### JS API

Here are some of the goals we will address:

- Well Defined: It should have no ambiguity about how functions operate.
  Everything from explicitly stating when function arguments are coerced and
  when they aren't, to precisely stating when the function will throw.

- Testable: By keeping the API well defined it will be much easier to test.
  This is key to properly creating a compatibility suit that allows other
  vendors to create their own implementation and make sure it still functions
  perfectly well with the existing ecosystem.

- Flexible: Being strict and being well defined are not the same thing. By
  choosing a few key points within the API that give developers more leeway it
  will be easier for them to create new interfaces. While this API may be more
  verbose, it will not be restrictive.

- Performant: The JS API should be not much more than a means to translate a
  developers intent down to the native layer. It should barely show up as a
  blip on any performance benchmarks.

- Safe: Errors will propagate safely to the user as often is possible. If an
  exception is thrown then you'll know something actually went horribly wrong,
  and the application had no idea how to proceed.


### Native API

The goal of the native API is to:

* remove the need to change code between
  Node.js releases (which is also covered by Nan)
* make the API ABI stable in order to remove the need
  to recompile at all (not covered by Nan).  
* explicitly define the C and or C++ functions available os opposed to just
  having all V8 APIs exposed.
* meet the needs of a majority of native modules in a way that does not prevent
  a smaller subset from using the V8 APIs directly.

