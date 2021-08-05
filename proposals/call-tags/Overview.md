# Call Tags Proposal

(For historical reference, call tags were briefly known as [*dispatch* tags](https://github.com/WebAssembly/design/issues/1346).)

## Summary

Provides a way to create and call untyped functions (`funcref`) such that:
* calls are efficient (just an additional push, pop, and bitwise equality check compared to a typed function call)
* modules can restrict how their functions can be called indirectly (e.g. ensure no other module can indirectly call a particular function)
* type abstraction and subtyping are respected

Furthermore, in some cases an engine can guarantee that an indirect call will necessarily stay within the same module instance (if it succeeds), enabling various optimizations.

The `func_switch` extension adds the ability to efficiently use the same `funcref` to support many kinds of calls, including calls with different types of arguments and even different numbers of parameters and results.

## Problem

`call_indirect` imposes a number of challenges (each discussed in more detail below), both currently and with respect to how WebAssembly might evolve:
1. Looking at a function's definition alone, it's not apparent if a function can be indirectly called via `call_indirect`.
One has to do some analysis to check if a reference to that function is made.
And once a function reference has been made, it can be very difficult to ensure even something as simple as the function is only accessed by its own module.
This makes it difficult to optimize functions or to ensure they are accessed in a way that respects the module's intended invariants.
2. If subtyping is added, the signature provided by the indirect caller can be compatible with that of the callee function *without* being identical.
Unfortunately, structural comparison of signatures at run time is likely too expensive.
[WebAssembly/function-references#33](https://github.com/WebAssembly/function-references/pull/33) addresses this by requiring signatures to match exactly, but implementers of languages using subtyping have indicated that such a restriction does not suit their needs (e.g. subclasses in Kotlin can refine the signature of superclass methods).
3. To practically support type imports/exports, `call_indirect` needs to compare caller and callee signatures using the "run-time value" of imported/exported types.
This can be exploited to convert imported/exported types to and from the type they are supposed to be abstracting.
Thus `call_indirect` can be used to access information that would otherwise be concealed behind an abstract type, or to create values of an abstract type that would otherwise be unforgeable.

## Solution

One way to implement `call_indirect` is to push all the arguments onto the stack and then to push a signature descriptor onto the stack; the callee then pops that descriptor and compares it for (bitwise) equality with the callee's own signature descriptor.
If the two match, then the invariants guaranteed by the static type-checker ensure the stack has the arguments expected by the function (and similarly that the caller is expecting the values that will be returned by the function), so the function call proceeds.
If they don't match, then it traps.
(This is how SpiderMonkey currently implements `call_indirect`.)

We can solve all the problems above by generalizing this process *call tags*.
That is, a call tag is a generalization of these signature descriptors.

### Defining Call Tags

Somewhere in one of the module's header sections, the module would declare call tags as in `call_tag.new $ct1 : [i32] -> [i32]`.
This declaration establishes a new call tag `$ct1` that only the module has access to (unless it exports the tag), and which has associated signature `[i32] -> [i32]`.
The module could also declare call tags as in `call_tag.canon $ct2 : [i32] -> [i32]` that defines `$ct2` to be the *canonical* call tag for the signature `[i32] -> [i32]`.
This canonical call tag is one that all modules have access to and is uniquely determined by the signature. (Historical note for understanding comments below: `call_tag.canon` was formerly named `call_tag.get`.)

### Associating call tags

When a module defines a function it can specify the call tag(s) that its associated `funcref` should accept.
(By default, specifically the result of `dispatch.canon` on the signature is associated, making this backwards compatible.)
So a module can explicitly specify no tags if it doesn't want the function to be implicitly indirectly callable, or some `new` tag if it only wants it to be indirectly callable with the module's own tag, or multiple tags if it wants to mix-and-match dispatching conventions.
Of course, these associated call tags have to be compatible with the function's signature.

### Calling with call tags

`call_indirect` would then have a variant, say `call_funcref $ct` that specifies what call tag `$ct` to use.
If the call tag `$ct` has associated signature type `[t1*] -> [t2*]`, then `call_funcref $ct` has type `[t1* funcref] -> [t2*]`.
Thus the provided inputs and expected outputs are type-checked to match the signature associated with the call tag.
The current `call_indirect` becomes just a shorthand for first getting the `funcref` from the appropriate table and then using `call_funcref` with the call tag resulting from `call_tag.canon` on the expected signature.

The execution of `call_funcref $ct` on a given `funcref` for some function succeeds when the call tag `$ct` is one of the function's specified call tags.
So if the function specified just one call tag, then a simply bitwise-equality check is done on the two tags at run time.
If a match is found, then this *implies* that the arguments are valid inputs to the function and that the returned values are acceptable outputs from the function.
However, by making call tags explicit, it also becomes clear that compatibility of inputs and outputs *does not imply* the call will succeed because the call tags might (intentionally) not match even if they have the same signature.

## Applications

### Security

To see one way this can be useful for WebAssembly, suppose I am a security-conscious application.
I want to be very conscious about all the entry points for my program. `call_indirect` is an opportunity for an unintended entry point.
I can try to mitigate that by being careful about the function references that escape my program, but leaks happen, and it would be better to just not have these unintended entry points to begin with.

One option is to make it possible to not have any call tags associated with any of my functions.
However, that means even I can't `call_indirect` my own functions, which would be a huge problem for many C/C++ programs.
So this proposal provides the option to use `call_tag.new` for every signature I use and associate each of my functions with the respective new tag.
Then, so long as I don't explicitly export those call tags, I know I'm the only one who can `call_indirect` into my functions. Furthermore, if I have subcomponents of my program that are supposed to be separable, I can even make new tags at a finer granularity for each subcomponent to ensure that a `call_indirect` in one subcomponent cannot call into another subcomponent.
Thus, the proposal makes it easier for me to restrict the scope of `call_indirect`, both from other modules and even within my own module.

### Optimization

If a function has no associated call tags, then it is only directly callable, enabling many optimizations.
Even if a function does have an associated call tag, if that tag is neither imported nor exported than the optimizer can guarantee that the function can only be successfully called from within the current module instance, and furthermore it can only can be indirectly called from occurrences of `call_funcref` using one of the associated call tags.
Similarly, a `call_funcref` using a call tag that is neither imported nor exported eliminates the possibility that a successful call might require an module-instance switch.

### Subtyping

When a function specifies associated call tags, the types of those call tags need only be *compatible* with the functions signature.
Compatibility here can easily incorporate subtyping.
That is, so long as a call tag's input types are subtypes of the input types of the function, and the output types of the function are subtypes of the output types of the call tag, then an indirect call using that call tag is completely sound.
This then supports the surface-level feature of various typed OO languages where a subclass/subinterface can refine the signature of a method to either accept a broader range of inputs or (more often) produce a more precise output.

### Abstraction

Many of the above languages would want modules to be able to export a class `C` without exporting its implementation (or at least its private fields). call tags are designed to support this pattern and yet avoid the problems outlined in [#1343](https://github.com/WebAssembly/design/issues/1343) wherein `call_indirect` can be used to expose and get direct access to a module's concrete implementation of an exported abstract type.

The [Type Imports](https://github.com/WebAssembly/proposal-type-imports/) proposal is still young, so for the sake of conversation suppose that exports are done by (1) specifying a module signature and then (2) specifying how to instantiate the various components of the signature with the module's various definitions.
So part 1 might say there is a type `C_type` with no specifics about the type, and then part 2 might say that `C_type` represents `ref (struct i32 i32)`.

In this setting, suppose some surface-level interface method `foo` defined in the module conceptually takes a `C` and returns an integer.
Using the implementation of interface-method dispatch above, the module would define a call tag `$ct_foo_internal` with signature `[object_impl (ref (struct i32 i32))] -> [i32]` (where the `object_impl` is the `this` pointer).
The module's own implementations of this method are allowed to know the specific implementation of class `C`, which is why this call tag uses the type `(ref (struct i32 i32))` in its signature.

However, although other module's are allowed to provide their own implementations of the surface-level interface method `foo`, they are not allowed to know the implementation of `C`.
So in the module's exports, part 1 would specify a call tag `$ct_foo : [object_impl C_type] -> [i32]`, and then part 2 would instantiate that tag with `$ct_foo_internal`.
Internally, this instantiation is valid because `C_type` was instantiated with `ref (struct i32 i32)`, but externally other modules know nothing about `C_type`.
Nonetheless, they can use `$ct_foo` to invoke `foo` on objects and to provide their own implementations of `foo` just like they can any other interface method.

This pattern enables (controlled) indirect calls across modules, but in a way that respects abstraction.
In particular, unlike in [#1343](https://github.com/WebAssembly/design/issues/1343), this design can prevent using indirect calls to get direct access to a module's implementation of an abstract type or to forge values of an abstract type.
There is just one restriction that needs to be made: `call_tag.canon` must only be allowed for signatures comprised solely of *uninstantiable* types, such as `i32`, `i64`, `f32`, `f64`, and&mdash;per the discussion in [#1343](https://github.com/WebAssembly/design/issues/1343)&mdash;`externref`.
That is, it is the ability to generate canonical call tags from abstract types that violates abstraction.
(This observation seems to extend more generally to processes that would generate canonical values from types where the values are equal only if the types are equal.)

## Extension

In addition to *direct* functions, we could let a module declare a number of *switch* (or indirect?) functions that look like the following:
```
(func_switch $fr
    (on_call_tag $ct1 $f1)
    (on_call_tag $ct2 $f2)
    ...
    (trap)
)
```
This defines a `funcref` (whose index is `$fr`), not a `func`.
An indirect call to `$fr` checks if the call tag provided at run time is (bitwise) equal to any of `$ct1`, ....
If it matches `$ctn`, then a direct call to `$fn` is made (in theory; in practice, this call might be inlined).
If no match is found, then the indirect call traps.
Type-checking simply involves checking that the signature of each `$ctn` is compatible with that of `$f1`.

Given a `func_switch`, one makes a `funcref` via `ref.func $fr`.
That might seem odd because previously `ref.func` took a `func` identifier.
The disconnect is because `func` is currently doing two things: defining a direct function *and* defining a `func_switch` that calls that function on each of the associated call tags.
So every `func` identifier is also a `func_switch` identifier, making this change in perspective still backwards compatible.

## Applications

### Interfaces

At present, `call_indirect` is primarily used for untyped (from wasm's perspective) function calls.
But we can take the concept of call tags further to support more language features.
In particular, this same pattern shows up in interface-method dispatch for languages with multiple inheritance of interfaces.
Interface-method dispatch is a critical feature of various popular languages, and good performance for this feature is often achieved through JITing techniques that WebAssembly is aiming to not rely on.

The `func_switch` extension means that the behavior of a `funcref` can depend on the specific call tag used (beyond just matching versus trapping).
Using this, we can support an efficient non-JITing implementation of interface-method dispatch (and other lesser-known forms of dynamic dispatch) for many languages with multiple inheritance of interfaces.
For context, one way to implement interface-method dispatch is to have every v-table have an array of, say, 19 slots, and to assign to every interface method some slot number.
Unfortunately, it is possible that an object implements multiple interface methods with the same slot number.
So when a interface-method call is made, in addition to the arguments the caller pushes onto the stack the identifier of the interface method (i.e. its call tag) and then calls the function in the matching slot.
That function then switches on that identifier and redirect to the appropriate implementation, just like a `func_switch`.
See [Efficient implementation of Java interfaces: Invokeinterface considered harmless](https://dl.acm.org/doi/10.1145/504282.504291) for more information.

It would be very difficult to implement this pattern without direct support from WebAssembly because two interface methods assigned to the same slot can have completely different signatures, i.e. number and size of arguments.
So call tags enables an important pattern used in practice to support a feature that is critical for many major languages (specifically Java, Kotlin, and Scala come to mind, though not C# due to its decision to support multiple-instantiation inheritance of generic interfaces).

### Dynamic Arity

In functional languages, a value of type `a -> b -> c` (where each letter is a type variable) is a closure of unknown arity.
Due to currying, it could be a closure expecting an `a` that then will return a closure expecting a `b` that then will return a `c`.
Due to first-class functions, the `c` itself could represent a function type, so it could even be the case that this is a ternary closure that has been curried.

A functional language could implement closure application by having each closure specify its arity and by having each caller case on this arity.
Or a functional language could use `func_switch` for a closure's `funcref` and have each caller use a call tag for the arity at hand which then the `func_switch` cases on to provide the appropriate functionality.
The latter is moderately more efficient, but given the frequency of function applications in functional languages, that moderate improvement would likely be notable.

## Forwards-Compatibility

Preliminary investigations suggest that call tags will be compatible with features like parameterized (i.e. generic) interfaces and polymorphic functions/methods as well as existential types to eliminate superfluous casts.
