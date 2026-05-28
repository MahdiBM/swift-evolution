# Models for scope restrictions for non-escaping types in Swift

## Introduction

### Temporarily available resources

It is common for resources to only be available to a program on a temporary basis.
This can apply to resources up and down the abstraction stack.
A high level resource might be something like an open HTTP connection, which can only be used after the connection is open and before it is closed.
A low level resource might be the right to use a particular piece of memory, which can only be used after the memory is allocated and before it is deallocated.
In any case, there's a point in the program's execution when the resource becomes available to use, and another point when the resource stops being available.

Safe programming languages must eliminate certain safety-implicating bugs[^1] relating to using resources when they're no longer available.
This is widely recognized for low-level resources such as memory; nobody should call a language safe if it is exposes pointers to deallocated memory.
However, it is often the case that the same techniques that work to establish safety for low-level resources can also be used for high-level resources.

[^1]: Or at least strongly discourage them, e.g. by only admitting them when `unsafe` features are used.
      Ideally, the language would encourage such features to at least still be used in patterns that can be proven safe outside the language.

There are many such techniques.
Automatic memory management works well for dynamically-allocated memory.
Non-copyable value ownership generalizes that to cover a host of other dynamically-managed resources.
But neither works for resources that are naturally tied to a scope in the program.

### Scoped resources

The most obvious resource that's naturally tied to a program scope is stack memory.
Small, fixed-size amounts of stack memory can be allocated at no dynamic cost[^2] within the function's ordinary stack frame.
Even when it needs to be done dynamically, stack allocation is also typically significantly faster than heap allocation.
But stack memory can be immediately reused once the current function returns, tying it to the outermost scope in the function.
Any use of the allocation after that point risks arbitrary memory corruption.

[^2]: Well, at the usually negligible cost of increasing the frame size and the ensuing locality loss for other stack allocations.

But there are other scope-associated resources that naturally arise.
These resources need not be as literal as a memory allocation.
Safe languages often use static analysis of the program's control flow to prove that certain things are done correctly.
Often, this takes the form of proving that one thing happens before another.
For example, Swift's [exclusivity rule][SE-0176] says that a variable cannot be mutated while a read of it is active.
Another way to put that is that there is a scope (the duration of the read) during which a resource (the right to read the variable without violating exclusivity) is available.
Since the variable is immutable within the scope, passing around this resource is essentially the same as passing around the value, but without any of the overhead of copying it.
This is an important tool for building efficient abstractions for managing complex operations, like iterating a collection.[^3]
Swift's exclusivity rule also says that a variable cannot be accessed while it is being mutated.
As above, this can be seen as a resource (the right to mutate the variable) that is only available within the scope of the mutation.
And just as above, this is an important tool for building efficient abstractions for managing complex mutating operations.[^4]
Similar ideas can apply to allow abstraction over almost any control-flow-based language rule.

[^3]: This was actually recently added to Swift as the `Ref` type, in [SE-0519][].
[^4]: This was also added in [SE-0519][], as the `MutableRef` type.

Furthermore, scoped resources give rise to more scoped resources.
If a value can only be safely used within a particular scope, then anything built on top of it must also only be safely usable within that scope.
For example, suppose that you have a function that breaks down an array into slices.
If the array you start with comes from a scoped resources, then the slices can be only be used within the original array's scope.
Suppose you instead present your API as an iterator type with a `next()` method that parses the next slice out.
Then not only are the slices restricted, but your iterator also needs to be restricted, since calling `next()` accesses the original array.

### Models of lifetimes

In Swift, we enforce safe access to scoped resources using [non-escapable types][SE-0446].
There's a natural intuition for what it means to have a value of a non-escapable type: the value cannot be used outside of some scope.
That scope is usually clear from context, and intuitive reasoning leads to a set of rules that any reasonable language feature for scoped resources would need to follow.
For example, if you have a parameter of non-escapable type, you cannot assign that value to a global variable, or to any location where you can no longer reason about whether the value has escaped.

But you cannot design language features by enumerating every code pattern that would use the feature and deciding intuitively how each should work.
Instead, you define general rules, which you can then validate against your intuition as part of the design process.
These general rules are called the formal model underlying the feature.
The core ideas of the formal model are predominantly responsible for determining how the feature works in the language.
The rest of the language design of the feature can usually be thought of as relatively superficial tweaks on top of those core ideas.[^5][^6]

[^5]: The rule of exclusivity defined by [SE-0176][] has an example of such a tweak.
      The general model of exclusivity is that, whenever you use a variable, there is a corresponding access scope, and conflicting access scopes (e.g. two mutations of the same variable) cannot overlap.
      But there is a small exception to the model that permits overlapping accesses to different stored properties of the same variable.

[^6]: Another example is the addition of [region-based isolation][SE-0414] to the core model of non-sendable types.
      This was a much deeper change, and one with significantly wider implications to the core feature.
      Even so, it had to be built in order to fit on top of the existing rules.
      Core model replacements are rarely strictly additive in this way, which is a large part of why they are difficult to do retroactively.

It is my contention that the current formal model we use for scope restrictions in Swift is not what it needs to be.
The current interpretation of non-escapable types makes it difficult to correctly handle certain kinds of abstraction over scoped resources because the scope restrictions do not propagate properly through types.
(This has some other knock-on effects.)
I believe that the limitations we've been imposing on non-escapable types up until recently have given us a window in which it still remains acceptable to change the model.
But we will close that window very quickly if we start adding generalizations over non-escapable types to the standard library.
We will regret doing so.

Now, the current model works fine for many simple, useful code patterns.
We do not need to withdraw any of the library features we've already released using non-escapable types.
All we need to do is continue to disallow certain kinds of generalization over non-escapable types until we can introduce the right model for them.

## The value-dependency model of scope restrictions

Swift's current formal model for scope restrictions is based on lifetime dependencies between abstract values.

An abstract value represents a computation that is performed (implicitly or explicitly) as part of evaluating a function.
For example, if the function contains a function call, there is an abstract value corresponding to the result of that call.
At every point in a local variable or `inout` parameter's scope, an abstract value can be determined that represents how the value of the variable at that point was computed.
There are special abstract values representing "computations" such as parameters, initial values of `inout` parameters, final values of `inout` call arguments, and values that are computed differently on different control flow paths.

Every abstract value has a set of root lifetime dependencies.
These dependencies are either (1) specific local access scopes or (2) "root" abstract values such as parameters or initial values of `inout` parameters.
Abstract values of escapable type, such as `Int`s, generally have no lifetime dependencies, superseding all other rules.
Abstract values that represent most "primitive" computations, like projecting the address of a struct stored property, generally just carry the lifetime dependencies of their data dependencies.
For example, the result of reading a stored property of a struct value generally has the same lifetime dependencies as the struct value.
Similarly, the result of a control-flow-merge computation carries the union of the dependencies of all of the different possible inputs.
However, call results and final `inout` call argument values are special: they carry dependencies based on the dependency signature of the called function.

Every function has a dependency signature as part of its type.
This signature describes the dependencies of the results, including the final abstract values of any `inout` call arguments.
Dependencies in a function dependency signature are sets of any of:
- the access scope of a function parameter, if it is a `borrowing` or `inout` parameter,
- the abstract value of a function parameter, if it has a non-escapable type, or
- the initial abstract value of an `inout` parameter, if it has a non-escapable type.
When determining dependencies for a call result or final `inout` call argument argument value, these rules are applied to the lifetime dependencies of the corresponding arguments.

For example, if the dependency signature says that the call result is dependent on the abstract value of parameter #4 and the initial value of `inout` parameter #6, then the lifetime dependencies of the call result abstract value are the union of the lifetime dependencies of the abstract value passed as call argument #4 and the lifetime dependencies of the current abstract value (at the point of the call) of the variable passed as `inout` argument #6.

The formal model then comes down to two restrictions:

1. Every use of a value that has a lifetime dependency on an access scope must occur within that access scope.
   (If the value has dependencies on multiple scopes, the use must occur within all of the scopes.)

2. The lifetime dependencies of abstract values corresponding to function results must be a subset of the corresponding dependencies declared in the function's dependency signature.

For example, suppose that the dependency signature for the current function says that the call result is dependent on the abstract value of parameter #4 and the initial value of `inout` parameter #6.
Consider the lifetime dependencies of the abstract value that is returned by the function.
It's okay if these dependencies are the empty set, or just the abstract value of parameter #4.
But if the dependencies include something not in the declared set, that is an error.

### History of the value-dependency model

This model arises naturally as an extension of several things that are already built into Swift.
Almost all of the rules for abstract values defined above fall out automatically from a basic data flow analysis of the function body.[^7]
This analysis is performed unconditionally by the compiler and is required for a lot of existing language features and basic optimizations.
Similarly, the primitive value dependencies introduced by things like projecting out property addresses are very important to optimization, and the compiler has extensive support for inspecting and traversing these operations.
The lifetime dependency set can be computed with a fixed-point analysis on the function's data flow graph.
There's a good reason we started with this model.

[^7]: Compiler developers generally refer to this analysis as putting the function into [static single assignment][SSA] form.
      Swift's "SIL" SSA representation does not normally model the special abstract values described above for `inout` parameters and arguments, though.

(This is not meant to downplay the amount of work that's gone into implementing the current feature.
The compiler does some very impressive things, especially around shrinking and extending scopes in order to avoid unnecessary violations of the model's restrictions.
And it's worth nothing that almost all of that work would still be necessary if Swift switched to a different language model.
The analysis parts of it would just get consumed in a different way, and the rewriting parts should be exactly the same.)

### Problems with the value-dependency model

#### Scope restrictions cannot be expressed on types

The biggest problem with the value-dependency model is that scope restrictions are never carried directly abstractly by types.
Scope restrictions can only ever be applied to specific values, like the parameters or results of a function, and different values of the same type can always have different dependencies.
This makes it impossible to write a scope restriction in an abstract type position, such as a generic argument or an associated type.
And that makes it impossible to express a lot of things, like collections of non-escapable values with a specific scope restriction.
When you try, you end up with lifetime dependencies that are wildly conservative, often uselessly so.
And ultimately that means that many generic abstractions that should be perfectly suitable for non-escapable types end up being impossible to write.

Consider the `Iterable` protocol proposed by [SE-0516][].
The protocol defines an `IterableIterator` associated type which is allowed (actually expected) to be non-escapable.
It also defines an `Element` associated type, which we would also like to allow to be non-escapable in order to support collections of non-escapable values.

Now, the protocol requires an `Iterable` value to have a `makeIterableIterator` method.
Whenever you call this method, you must borrow the collection, and the iterator it returns should only be used within the scope of that borrow.
The value-dependency model has no problem expressing this scope restriction.
Note that different calls will be restricted to different borrow scopes.
Nothing about the type of the collection has anything to say about the scope of the iterator, nor should it.

The iterator thus produced is now required to have a `nextSpan` method that returns back a `Span<Element>`, and this is where the model runs into a problem.
What is the scope restriction of the elements of this span?
The value-dependency model can only express this in terms of the lifetime of something passed in to the method.
Most likely, I expect the scope restriction to be the same as the scope restriction on the elements of the original collection.
But the model has no way to talk about that; there is no concept in the model of the scope restriction on the elements of the collection.
There are only lifetime dependencies on the collection value as a whole.[^8]
The model's natural interpretation of the signature of `nextSpan`, given a non-escapable `Element` type, is that the scope restriction of the element is the same as the scope restriction of the span itself: the scope of the `inout` access made for the call to `nextSpan`.
That is, `nextSpan` is permitted not only to materialize elements into a temporary array, but to actually construct the values in that array with a novel temporary scope restriction.
This means that the caller of `nextSpan` is incredibly constrained.
A `for` loop using this protocol to iterate a collection of non-escapable values — even if they're fully copyable — cannot persist a value between iterations of the loop.

[^8]: There has been a small amount of exploration of the idea of having multiple "nested" lifetimes associated with a given abstract value, specifically with the goal of addressing this issue.
      In some cases, that might help.
      It would not help here, because that nested lifetime structure would only be known for a concrete conforming type and cannot be referenced in the abstract protocol requirement.
      So the protocol requirement is stuck making the extremely pessimistic lifetime statement I describe here.

Now, one could argue that this is just a more general signature.
It is possible to imagine an abstract value producer that would benefit from the flexibility to synthesize non-escapable elements bound to the iteration, although it's quite a bit of a stretch.
Perhaps a protocol like `Iterable` really should aim to allow that.
We've already discussed the possible need for refinements of `Iterable` that give stronger lifetime guarantees about the spans returned; maybe this fits into that.

But the same expressivity problem would still affect all of these less-abstract protocols.
Suppose there's a `Container` protocol that represents a concrete, in-memory collection, and it mandates a `ContainerIterator` for which `nextSpan()` returns a `Span` constrained not to the scope of the `nextSpan` call, but to the lifetime of the iterator itself (presumably the lifetime of the borrow of the original collection).
We can still ask, what is the lifetime of the elements?
The protocol is still allowing it to be as narrow as the borrow of the original collection, but that's still over-constrained: the elements are necessarily usable in some broader scope than just this specific borrow of the container holding them.
In theory, the protocol is allowing them to be synthesized as part of the `makeIterator` call, because it has no ability to associate a scope restriction specifically with the element values.[^9]
There's no obvious implementation which could take advantage of that flexibility, since (barring something reference-type-ish) the `makeIterator` call does not have the ability to mutate anything to set that up, but nonetheless, that's all that the protocol would guarantee.

[^9]: It may be possible under a nested lifetime approach to allow the nested lifetime to also be abstracted over in the protocol and named in protocol requirements.
      However, conformances are associated with types, not values.
      This idea would need to be explored further, but I believe it may still require a major model shift towards type-based scope restrictions.

Could we wait to solve that problem?
We could ship `Iterable` over non-escapable elements using this more general, value-dependency-friendly signature, then use a type-based design for the more cncrete `Container` protocols.
Unfortunately, that would come with some very foundational problems.
The `Element` associated type for `Iterable` would have to be an "abstract" non-escapable type, like `IterableIterator` is, with its scope restrictions left to be filled in from context.
But the `Element` associated type for `Container` would be different, carry its scope restrictions explicitly.
It's really unclear how that would work.
Not only are those different types with very different interpretations when used, but they're differently-*kinded* types.
It's very likely that the resulting model would be a huge mess both for users and for the implementation.



<!-- WIP, editing point -->


### Problems with the value-dependency model

In my view, there are several significant problems with the value-dependency model of lifetimes.

The first is that the association between the dependency graph and the correctness property it is upholding is pretty abstract.
Recall that correctness for non-escapable values is generally expressed in terms of scopes: certain values are only safe to use within certain scopes.
When you write a function signature involving non-escapable types, you should be thinking about the relationship between these scopes.
Usually, that will go something like, "The span I'm returning is just a slice of this span parameter, so it's safe to use it for the same scope that it's valid to use the original span in."
But the value-dependency model asks you to instead think about the impact of adding various dependency edges on the lifetime checker.
That is more like, "The span I'm returning is only safe to use within the scope of the span parameter, so I need to add a dependency edge between them."
This is unnecessarily dissociated from the original correctness property, which can lead to confusion and misunderstandings.

This confusion is a major problem for unsafe interactions.
Getting a lifetime signature wrong in fully safe code just means you'll get an error somewhere.[^6]
But non-escapable types are often used for low-level programming, and low-level programming often requires interacting with unsafe subsystems, such as libraries written in C or C++.
Wrapping these subsystems up into safe Swift interfaces requires a contract between the safe and unsafe parts of the code.
It is critical for correctness that the programmer understand that contract clearly, because the unsafe side of it is not going to be automatically checked.[^7]
It is difficult to do this when it requires second-order reasoning about the impact of adding dependencies.
The value-dependency model also doesn't provide a great solution for APIs that need to unsafely construct a safe wrapper type, like an `init` that wraps an `UnsafeBufferPointer` as a `Span`.
In these cases, the `init` call does not have any natural scope dependencies, so the result is treated as maximally unrestricted.
To a certain degree, this is an inherent problem: the compiler does have to just accept that the `Span`'s scope is appropriate for the original pointer.
But it would be better if the code could at least explicitly communicate its expectation about that scope, so that the existence of the call doesn't completely bypass checking.

[^6]: Lifetime signatures are basically logical propositions: theorems that need to be proven for the functions they're attached to.
      The stronger the theorem, the more powerful the guarantees that clients will get when checking themselves, but also the more you will have to satisfy when checking the function definition.

[^7]: For example, we'd like programmers to be able to use annotations on their C APIs to make certain parameters or return types get imported into Swift using `Span` or `Ref`.
      We'd like the C compiler to be able to check that the C function definition satisfies its side of this guarantee, but that's going to take a lot of work.
      In the meantime, the Swift compiler will simply have to accept the annotations as accurate.

A related problem is that the value-dependency model struggles with more abstract uses of lifetimes.
This includes abstractions over non-escapable values, such as collections, adapter types, and callbacks.
There are several different mechanisms behind these struggles.

One mechanism is that the value-dependency model naturally conflates all of the scope restrictions of a value.
A value is a single node in the dependency graph, so it cannot distinguish between different kinds of dependency edge.
But this is unfortunate, because a lot of abstractions have multiple levels of scope restriction.
For example, when working with a span of non-escapable elements, there's a scope restricton associated with the memory in which the elements are stored, but there's also a scope restriction for the element values themselves.
Conflating these means that values read out of the span necessarily pick up dependencies that only really affect the span's backing memory.
Consider the following example:

```swift
  var arrayOfRawSpans: Array<RawSpan> = ...
  let rawSpan = arrayOfRawSpans.span[0]
```

`arrayOfRawSpans.span` has a dependency on a borrow of `arrayOfRawSpans`.
This is important: if we mutate `arrayOfRawSpans`, this span really does
need to become invalid to use.
However, this dependency is preserved onto the result of the subscript because the model does not distinguish different kinds of dependency.
Therefore, `rawSpan` must also become invalid to use if we mutate `arrayOfRawSpans`, even though its scope restriction is naturally independent of the access to `arrayOfRawSpans`.

Solving this under the value-dependency model remains an open problem.
One idea that has been suggested is to add nested lifetimes to the model, which essentially create multiple nodes in the dependency graph associated with a value.
We don't actually even know if that idea is sound, unfortunately; it has not been explored in depth.
And the problem affects essentially every kind of abstraction over non-escapable values.
For example, consider a `map` algorithm that wants to allow the result type to be a non-escapable value.
The element-mapping function passed to `map` is generally a non-escaping closure.
Since the model conflates all of the lifetime dependencies of a value, the result of calling that closure must carry a dependency on the closure.
And this means the result of `map` must also carry that dependency.
This is a problem that type-based lifetime constraints simply do not have, as we will see.

It has also been suggested that the value-dependency model could be augmented, later, with support for type-based lifetime models.
Certainly this cannot be ruled out; with enough compiler effort, we can move mountains.
However, the value-dependency model does not easily extend in this direction, and in some ways it contradicts it.
In type-based lifetime models, a `~Escapable` generic type parameter is usually expected to carry a specific scope constraint along with it when substituted.
But this changes the interpretation of that type parameter in generic code from what you naturally get from the value-dependency model.
Consider a type like this:

```swift
struct Wrapper<T: ~Escapable> {
  var value: T { get { ... } }
}
```

Under the value-dependency model, the `value` getter is understood to produce a `T` that depends only on the borrow of `self`.
This value could be synthesized directly by the getter.
But under a type-based lifetime model, substitution of `T` will carry along a scope restriction determined by the client of the type.
The getter is then expected to produce a value that satisfies that scope restriction.[^8]
This is just a radically different interpretation of what it means to have a non-escapable type parameter.

[^8]: Since the scope is decided by the client at the level of the type, it might be (and almost certainly is) some scope broader than the borrow of `self` for this specific call. An implementation that depended on `self` would be invalid.

Furthermore, a basic goal of type-based models is that they can be used to explicitly specify the scope constraints to values.
For example, you could say that a local variable is required to hold a span with a specific lifetime.
There is no way to express such constraints in terms of value dependencies, and nothing in the lifetime analysis described above would naturally check it.
It requires a different strategy for analysis.

Finally, the value-dependency model turns almost everything about lifetime checking into a flow-sensitive problem.
This can permit more things, but it also makes it harder to generate good diagnostics for certain issues, especially those that arise from lifetime invariance.
Consider a mutable variable of non-escapable type.
If the variable is just a local `var`, it's generally fine to add dependencies to it; the lifetime analysis will simply update its internal state.
But other variables must be invariant in their lifetime restrictions, meaning they cannot gain dependencies.
For example, if the variable is an `inout` parameter, any changes to its dependencies must be stated (or implied) by its signature.
Ideally, if you write code that transparently violates these restrictions, it will be diagnosed at the call site.
However, the flow-sensitive analysis allows these calls to occur.
After all, there might be more changes, later in the function, that will restore the final value in the variable to something that satisfies the constraints.
As a result, the analysis is naturally prone to diagnosing these violations as state conflicts when the end of the function is reached.
It is not in any way impossible to make sure the violation is diagnosed at the problematic call site, but it is more difficult.

### Advantages of the value-dependency model

None of that is to say that there aren't upsides to the value-dependency model.

Probably the biggest advantage is that it builds relatively directly on top of existing analyses in the compiler.
This is, after all, what has allowed the feature to be delivered so far.
And many use cases of non-escapable values do not run into any of the difficulties around abstraction that I've laid out above.
If you just want to support working with spans of trivial values, with minimal abstraction you don't need much from the lifetime system.

A more fundamental advantage arises from flow-sensitivity.
In type-based lifetime models, it is still generally true that any given variable has a single type for its duration.
The lifetime constraints in this type can be inferred "globally" within the function, as discussed later, but ultimately the scopes will be picked and applied everywhere the variable is used.
This means that the checking ends up being very conservative if the lifetime restrictions of the variable significantly change over the course of the function.
For example, suppose that you have an array of spans:

```swift
  var spans = [RawSpan]()
  use1(spans) // might require an unrestricted array
  spans.append(longTermArray.span)
  use2(spans) // might require a broadly-restricted array
  spans.append(shortTermArray.span)
  use3(spans) // must work with a narrowly-restricted array
```

In this example, the lifetime constraint on the elements of `spans` gets progressively narrower as the function executes.
It's possible that `use1` and `use2` could take advantage of the broader lifetime restrictions that still hold at the time of those calls, allowing this code to pass lifetime checking when a type-based analysis would reject it.
However, this is unlikely.
Most code that builds up a collection this way does not have interleaved uses of the collection; there tend to be distinct phases.
Moreover, the uses are likely to be uniform, meaning that if they work with the narrow restriction at the end, they would also work with an artificially narrow restriction at the beginning.
And if the programmer really needs this, they can assign the array to a new variable, allowing the compiler to infer a broad scope for the first variable and a narrow scope from the second.


## The type-based model of scope restrictions

Okay, let's step back for a moment.
The core correctness property we're trying to achieve with non-escapable values is that certain values must only be used within certain scopes.
We have a system in Swift for restricting how values are used: the static type system.
It's natural to ask if we can make the type of a value reflect the scope restrictions on it.
Doing so lets us immediately take advantage of normal type system operations, like generic argument substitution, to propagate those restrictions through abstractions and signatures.
And since it aligns the language model directly with the core correctness property, it makes it straightforward to reason as programmers about how those restrictions need to play out in code.

Abstractly, this model closely resembles Rust's approach to scope restrictions.
That does not require us to make the same syntactic decisions that Rust does, however.
I believe there is a workable syntax that largely (perhaps even completely) avoids the need for named lifetime parameters.
And, just like Rust, I think we can avoid requiring the simple dependency rules that dominate most use patterns to be written explicitly.

I am presenting this models as if it were *the* alternative to the value-dependency model.
I don't have a formal argument for why that would be true; there may be other compelling options.
But we know that this is a sound and workable model that supports a reasonable base of abstraction and generalization in Rust libraries.

### Basic concepts

I have to apologize for this section, because there's going to be quite a lot of forward-reference here.
Please bear with me.

There are two fundamental ideas in the type-based model:

1. Every unconditionally non-escapable type has one or more *scope bindings* representing its scope restrictions.

2. After *scope reconstruction*, the type structure of every unconditionally non-escapable type in a *concrete position* always includes *scope specifiers* corresponding 1-1 to its scope bindings.

#### Scope bindings

We would probably want scope bindings to be writable explicitly.
But most types will only have one binding, and it might be sensible to just infer a default one with a standard name.

Placeholder syntax:

```swift
struct Span<Element> : ~Escapable {
  scope memory
}
```

Builtin unconditionally non-escapable types, like non-`@escaping` function types, would behave as if they had an implicit such declaration with some name TBD.

I don't think there's any way that a type can ever evolve its set of scope bindings; it's fixed at first release.
But maybe I'm missing something.


#### Scope specifiers

Scope specifiers are typically inferred by scope reconstruction (basically, lifetime analysis).
But they do also need to be writable explicitly in source for a variety of reasons:
- to specify relationships in the signatures of functions and properties;
- to let programmers verify their understanding of what's being inferred, especially to guide diagnostics; and
- to let programmers force a specific scope, especially around unsafe code.

Placeholder syntax:

```swift
@scoped(memory: x) Span<Int>
```

The label must be that of a scope binding in the type. But there's no good reason to require the label when there's only one scope binding in the type, which would be predominantly true, so usually this would just be

```swift
@scoped(x) Span<Int>
```

I think we'd generally not want it to be written in a generic argument position.

The argument of the specifier must somehow specify a scope, and I'll talk more about syntax for that later.

Scope specifiers are part of the structure of a type and are therefore carried by type substitution.

Note that *conditionally* non-escapable types don't get scope bindings or specifiers.
The specifiers get applied to their generic arguments instead: `Optional<@scoped(x) Span<Int>>`, not `@scoped(x) Optional<Span<Int>>`.

#### Concrete positions

A *concrete position* is a position that requires a scope-applied type.
The intuition is: any place you would write a type that directly describes the type of a value, or that might after generic substitution.
Types of variables, properties, parameters, results, case payloads, and tuple elements are always concrete positions.
`~Escapable` type parameters and associated types may be declared to not be concrete positions; more on this later.
Generic arguments and associated type witnesses are concrete positions if the corresponding parameter or associated type declaration is.

#### Rule of scope specifiers after reconstruction

Every non-escapable type (conditional or not) in a concrete position (and thus: the type of every non-escapable value) must have, somewhere in its type structure, either:
- a `~Escapable` type parameter or associated type thereof in a concrete position, or
- an unconditionally non-escapable type in a concrete position, which must therefore have scope specifiers after scope reconstruction.

This rule actually just falls out naturally from a sensible restriction on conditional `Escapable` conformances: that they can only be conditional on the escapability of a concrete generic argument of the type.
I believe this is not currently true, both because the concept of concrete type parameters is novel to this document and because we allow these conformances to be conditioned on certain other marker protocols.
But I don't think fixing that would be a noticeable loss.
Note that this rule is preserved by generic substitution: when you replace a dependent type, either the replacement is escapable (eliminating whatever contribution this use of the parameter made to the non-escapability of the overall type) or it obeys the rule (and therefore satisfies the rule for the overall type).

### Scope variance

Scopes are partially ordered by the containment relation.
Two scopes can always be intersected; this may produce an empty scope.
There is a global "immortal" scope that contains all other scopes.
Intersection with the immortal scope is the identity operation.

There is also a scope subtyping relation between types.
In the examples that follow, let `smaller` and `bigger` be two scopes such that `bigger` contains `smaller`.

- Every non-escaping type is covariant with respect to its immediate scope specifiers.
  For example, `@scoped(smaller) Span<Int>` is a scope subtype of `@scoped(bigger) Span<Int>` because `smaller` is contained within `larger`.

- Types may be covariant, contravariant, or invariant with respect to their concrete `~Escapable` type parameters.
  For example, `Span` is covariant in its type parameter, so `@scoped(x) Span<@scoped(smaller) Span<Int>>` is a scope subtype of `@scoped(x) Span<@scoped(larger) Span<Int>>`.
  But `MutableSpan` is invariant in its type parameter, so `@scoped(x) MutableSpan<@scoped(smaller) Span<Int>>` is not related to `@scoped(x) MutableSpan<@scoped(larger) Span<Int>>`.
  We would need syntax for this.
  I believe Rust has a defaulting rule for it, but it's based on a deep inspection that I'm not sure we want to do.

- If two types are scope-subtypes, they are also subtypes for the normal subtype relationship, meaning that values can be converted between them.

### Type checking and scope reconstruction

Scope reconstruction is the process of either inferring scope specifiers in all positions where they are required or diagnosing why inference is impossible.
The basic checking model for scope reconstruction is as follows:
- Scope specifiers are placed in types where required by the standard type-checker, but are left expressed in terms of unresolved scope variables.
- The standard type-checking passes assume that unresolved scope variables are always related, which should mean that the specifiers have no effect on the passes when there are no explicit annotations.[^10]
  It may or may not be a good idea of consider explicit annotations at this stage.
- The lifetime passes attempt to resolve all of the unresolved scope variables, repairing and extending scopes as necessary to find a valid assignment.

[^10]: It may be a good idea to stage the insertion of unresolved scope variables after constraint solving.

These passes will use many of the same basic dependency-inspection analyses as the current passes do.
However, since the scope variables can create scope relationships between dependency-unrelated values in the function, resolving variables may require a more holistic algorithm within function bodies: essentially, it must solve a system of scope inequalities.
Fortunately, the system is a pure conjunction, which means the solver doesn't need to be nearly as complicated as the typechecker's constraint solver is.

The passes will need to be able to determine the required inequality by inspecting SIL.
One approach to this would be to encode scope specifiers in SIL, which would require some kind of scope-converting instruction.
The passes would then just search the function for these conversions and determine the inequalities from the types involved.
However, that would require the specifiers to be make explicit in SIL, and then probably stripped therefater.
An alternative approach would be to determine the inequalities from call sites in the function, similar to how the current analysis works.

### Syntaxes for scope specifiers

(to be written)

### Non-concrete type positions

(to be written)



[SE-0176]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md
[SE-0414]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md
[SE-0446]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0446-non-escapable.md
[SE-0447]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md
[SE-0467]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0467-MutableSpan.md
[SE-0516]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md
[SE-0519]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md
[SSA]: https://en.wikipedia.org/wiki/Static_single-assignment_form








Swift currently implements a straightforward model for non-escapable types based on value dependencies.
This is sufficient to allow a lot of basic types to be defined, like the `Span` and `MutableSpan` types added by [SE-0447][] and [SE-0467][], and these types have seen substantial practical use.
It is, of course, good that we've been able to deliver these features, and the discussion about fundamental models that follows is not meant to suggest that we've made any serious missteps there.
The most important use patterns of non-escapable types, for most developers, fit within a relatively simple subset that doesn't look too different in different models of lifetimes.
But I do think there are major problems with the value-dependency model, problems that we are running into more and more as we try to build out abstractions over non-escapable types, like allowing the element types of pointers and [iterable containers][SE-0516] to be `~Escapable`.
And I am particularly concerned that the model makes it too easy to misunderstand the generalizations we're looking at, potentially leading us to make the *wrong* generalizations.

My concrete proposal in this document is that Swift should switch to a type-based model for the lifetime restrictions on non-escapable types. The third section of the document lays out the basic requirements on the model as I see them. I believe that we can design a syntax for describing lifetime relationships on top of that model that avoids the weaknesses of the Rust syntax while still permitting a straightforward and high-performance dynamic erasure semantics.