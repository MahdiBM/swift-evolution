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
For example, if the function contains a function call, there is an abstract value corresponding to the return value of that call.
At every point in a local variable or `inout` parameter's scope, an abstract value can be determined that represents how the value of the variable at that point was computed.
There are special abstract values representing "computations" such as parameters, initial values of `inout` parameters, final values of `inout` call arguments, and values that are computed differently on different control flow paths.

Every abstract value has a set of lifetime dependencies.
These dependencies are either (1) specific local access scopes or (2) "root" abstract values such as parameters or initial values of `inout` parameters.

The way that an abstract value is computed determines the relationship between its lifetime dependencies and those of the abstract values that it was computed from:
- Abstract values of escapable type, such as `Int`s, generally have no lifetime dependencies, superseding all other rules.
- Abstract values for parameters (including initial values of `inout` parameters) are roots and just have themselves as lifetime dependencies.
- Abstract values for most other computations generally have the same lifetime dependencies as their data dependencies.
  For example, an abstract value representing the read of a non-escapable stored property of a struct value has the same lifetime dependencies as the original struct value.
  Similarly, the result of a control-flow-merge computation carries the union of the dependencies of all of the different possible inputs.
- However, call results (return values and final `inout` argument values) are special: their dependencies are determined based on the dependency signature of the called function.
  This is described in more detail below.
The abstract value's set of lifetime dependencies is the solution of this system of set relationships.

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
This analysis is automatically performed by the compiler for every function and is required for a lot of existing language features and basic optimizations.
Similarly, the primitive value dependencies introduced by things like projecting out property addresses are very important to optimization, and the compiler has extensive support for working with the data dependencies introduced by these operations.
Using these tools, the lifetime dependency set of any particular abstract value can be computed by just finding the fixed point of the system of set inequalities described above for the abstract value.
There's a good reason we started with this model.

[^7]: Compiler developers generally refer to this analysis as putting the function into [static single assignment][SSA] form.
      Swift's "SIL" SSA representation does not normally model the special abstract values described above for `inout` parameters and arguments, though.

(This is not meant to downplay the amount of work that's gone into implementing the current feature.
The compiler does some very impressive things, especially around shrinking and extending scopes in order to avoid unnecessary violations of the model's restrictions.
And it's worth noting that almost all of that work would still be necessary if Swift switched to a different language model.
The analysis parts of it would just get consumed in a different way, and the rewriting parts should be exactly the same.)

### Problems with the value-dependency model

#### Conflation of different scope restrictions

The current value-dependency model does not have the ability to distinguish different kinds of lifetime dependency.
This is unfortunate because values may naturally have scope restrictions for multiple independent reasons.

Consider a `Span` of non-escapable values.
There is a natural scope restriction associated with the borrow of array that the span refers to.
The elements stored in that array also came from some scope, almost certainly a different scope.
So the scope restrictions are almost certainly different.
But in the value-dependency model, the span value must have the union of those dependencies, and so much any element extracted from it.
Putting the value into the span and then taking it out again has unavoidably lost information.
This greatly restricts what can be done with the element.

For example, suppose that you're writing an algorithm that works on a `Span` of values.
Your algorithm naturally wants to filter and rearrange the elements as part of its operation, but it can't modify the original span, and you don't want to pay to copy the whole thing (if you even can).
However, you can efficiently make a scratch array to hold `Ref`s to the interesting elements, since copying around a `Ref`s doesn't require copying its referent.
When you pull those `Ref`s originally out of the `Span`, they start out with the same scope restriction.
But if the scratch array is itself non-escapable --- for example, if it's temporary memory created by `withTemporaryAllocation`, a very efficient choice --- then any `Ref` pulled out of it will also carry a dependency on it.
That means Swift can't let you just return that `Ref` from your algorithm, even though there's no real-world problem with doing so.

There's an especially important special case of this: some values of types that are generally non-escaping actually do not require any dependencies at all.
A global constant can be safely borrowed for a scope that covers the entire duration of the program.
It can be useful to create collections of values like that, like arrays of statically-allocated strings.
If those collections can also be generated as global constants, then great, those collection values need no dependencies.
But if not, and the collection needs to be temporary, then the value-dependency model has no way to separate the global-ness of the elements from that temporary-ness.

This problem also has a huge impact on the ability of higher-order algorithms to usefully work with non-escaping values.
Consider a generic algorithm like the following, which calls a function for each element in a collection and returns the first non-`nil` result:

```swift
extension Collection where Element: ~Copyable & ~Escapable {
  func firstReturning<R>(operation: (borrowing Element) -> R?) -> R?
    where R: ~Copyable & ~Escapable
}
```

Here the algorithm has been generalized to permit a return value of arbitrary type.[^10]
Unfortunately, there's a problem with this.
The `operation` closure is a non-escaping function, as it should be, since `firstReturning` doesn't plan to escape it.
But that means that, when we call it within `firstReturning`, its return value is naturally going to gain a dependency on the closure.
This makes sense: after all, the closure certainly could capture something non-escapable and return a value dependent on it.
The dependency is necessary in order to conservatively model that possibility.
On some level, that's fine; it just means that `firstReturning` has to be declared with a dependency signature that says that the return value gains a dependency on `operation`.
But now clients are really restricted: the closure has to be kept around at least as long as the result of `firstReturning` is being used.
So you could not, for example, use `firstReturning` to implement a function like the following:

```swift
extension Collection where Element: ~Copyable & ~Escapable {
  func firstRefMatching<R>(predicate: (borrowing Element) -> Bool) -> Ref<Element>? {
    // The closure we pass as `operation` is temporary in this function,
    // so when the `Ref` ends up dependent on it, it means we can no longer
    // return it out of this scope.
    firstReturning { element in predicate(element) ? Ref(element) : nil }
  }
}
```

[^10]: Swift's existing `Collection` protocol probably can't be retroactively generalized to support non-`Copyable` or non-`Escapable` elements.
       But we obviously want there to be *some* protocol that can do so, so drop that in instead.
       I'm just using `Collection` for familiarity.

Now, these examples can be changed to work around this problem.
A simple solution would just be to use indexes instead of `Ref`s.
For example, the scratch array of `Ref`s could just use a scratch array of indexes.
And `firstRefMatching` could just use `firstIndex(where:)` and then build a `Ref` directly to that element.
This does require more indexing operations, but that's probably not too high of a burden.
However, it also means that the values no longer stand alone.
For example, if you wanted to sort the indexes by their value, you wouldn't be able to use the standard `sort` function for `Comparable` elements; you'd have to use a comparator that has access to the original span.
Algorithms being generalized to work with non-escapable types would need to be rewritten in a less fluent and more boilerplate-y way, not because the existing implementation actually does anything that might escape the value, but just because the language model of non-escapable types is incapable of proving the correctness of the code.

Another solution would be to extend the current value-dependency model in ways that accommodate multiple sources of source restriction.
Several ideas have been explored for this, generally under the name "nested lifetimes".
Generally, the idea is that a type can declare one or more lifetime members that can be somewhat independent of the overall value's lifetime dependencies.
These approaches differ in their exact treatment of these nested lifetimes.

If the nested lifetime can be concretely constrained to a specific scope, this is essentially adopting a type-based model, at least for the nested lifetimes.
It is somewhat unclear how this would compose with the rules for local values, which would still be following value-dependency rules.
Values constrained by nested lifetimes can often become available as local values and vice-versa, e.g. when they are read out of a collection, and so it would be important to make them interact correctly.
For example, if you gain mutable access to a `MutableSpan` of non-escapable values that's stored in a collection, then inserted something into that span, it could add a dependency to the span.
That would need to be incorporated back into the nested lifetime somehow.
This could get very complicated, both in the theory and in the implementation.
It might be more reasonable to just simply switch wholly to a type-based model.

A different approach would be to make the nested lifetimes simply accumulate their own independent set of dependencies, as if they were separate values.
The dependency signatures of functions would be able to express changes in these dependency sets the same as they can express them for top-level values.
This is probably workable, and it would allow some more programs to be type-checked.
But most of the other problems with the value-dependency model would remain.

#### Abstract propagation of scope restrictions

Another major problem with the value-dependency model is the way that scope restrictions propagate through APIs.
The value-dependency model defaults to being conservative about dependencies.
For example, return values are assumed to depend on all of the parameters to a function unless otherwise annotated.
This is appropriate in some cases; in fact, the type-based model applies essentially the same rule for functions that return a non-escaping type with unbound scope restrictions.
But it is quite over-conservative for common patterns of abstraction over non-escapable values.

Consider an `[Span<Int>]`.
When the programmer uses this value, there will be some kind of scope restriction that the elements of the array are collectively expected to have.
This may be an intersection of different scopes, since different elements may have different sources, but that kind of conservatism is inherent to the problem: we cannot reasonably expect Swift to track more specific associations through the abstract indexing API of `Array`.

In the type-based model, this scope restriction will be set (perhaps implicitly) on the argument type of `Array`.
It will therefore automatically and perfectly transfer by type substitution to everywhere in the `Array` API that refers to its `Element` type parameter.
This works out such that generic code that works with `Element`s does not actually need to reason about the specific scope restrictions of that type.
It can freely copy or move the value around wherever it likes (assuming the other constraints on the type permit that), as long as it doesn't statically erase the type (e.g. by wrapping it up as an `Any`).
This is because the generic function knows that the scope restriction in `Element` cannot refer to any of its own scopes.
That scope restriction was bound into the type by some function up the stack, using scopes meaningful in that function.
That function necessarily calls some function during that scope, or else the use would be in violation of its local rules.
Every function called subsequently, all the way down to the generic function, is therefore wholly contained within that scope, so the data flow of `Element` values no special need to be locally restricted within the function.
Any context that receives a value statically typed as `Element` (or terms of it) will maintain that same knowledge via type substitution of the original scope restriction.
Ultimately that propagates all the way back out to the original function that bound `Element`'s scope restriction to some locally-meaningfully scope.
Whenever that function works with an `Element`, type substitution will replace `Element` with a type containing the actual scope restriction.
Swift then simply needs to check that the value is only used within that locally-meaningful scope.

The result is that values of non-escapable types are only very lightly restricted by their non-escapability within generic functions.
As long as their types aren't erased, and they aren't used in some inherently escaping way (like being captured in an escaping function), they're free to be used exactly like escapable values.
There isn't any way for a function that just works with an `Element` to somehow impose extra scope restrictions on it, because the scope restrictions are bound immutably into the type.

That is not how it works under the value-dependency model.
Dependencies can get conservatively mixed up on essentially any function call.
It is up to the dependency signature of each specific function to only report the dependencies that it actually adds.
Any amount of preemptive caution or generality at any level risks permanently losing information by adding unnecessary dependencies.
This is a somewhat fraught programming model.
It is not in any way unsafe, but any failure to minimize dependencies can bring the whole house of cards down and make it impossible to write algorithms that have no business being forbidden.

The `Iterable` protocol proposed by [SE-0516][] provides an excellent example of this.
The protocol defines an `IterableIterator` associated type which is allowed (actually expected) to be non-escapable.
It also defines an `Element` associated type, which we would also like to allow to be non-escapable in order to support collections of non-escapable values.

Now, the protocol requires an `Iterable` value to have a `makeIterableIterator` method.
Whenever you call this method, you must borrow the collection value, and the iterator it returns should only be used within the scope of that borrow.
The value-dependency model has no problem expressing this scope restriction.
Different calls will be restricted to different borrow scopes.
Nothing about the type of the collection has anything to say about the scope of the iterator, nor should it.

The iterator thus produced is now required to have a `nextSpan` method that returns back a `Span<Element>`, and this is where the model runs into a problem.
What is the scope restriction of the elements of this span?

In the current model, lacking nested lifetimes, there is only one option.
The elements must shared the same dependencies as the span.
Since we want to allow the span to be temporary to the current call to `nextSpan`, the elements are also restricted to that call.
This means that elements cannot safely be persisted across calls to `nextSpan()`.
When the iterator is used for a `for` loop, this means that elements cannot be stashed between iterations of the loop, much less stashed outside of the loop.
This is extremely restrictive and applies even if the element type is known to be copyable.

Essentially, this option is permitting the iterator to not just synthesize a span in temporary memory in the iterator, but to synthesize the elements of that span in some way that depends on temporary memory in the iterator.
This seems admirably general, but it takes some effort to imagine a collection type that could take advantage of the additional flexibility.
A permutation iterator, maybe, where the elements are spans of values.
The cost of this generality is that iteration over anything approaching a normal stored collection of non-escapable values cannot use `Iterable` without facing heavy restrictions.

Now, still under the current model, a refinement of `Iterable` could say that the spans returned by `nextSpan` depend only on the current iterator value.
(This would need an explicit annotation.)
The iterator value will generally only directly depend on the borrow of the collection performed for the call to `makeIterableIterator`.
Thus both the spans and the elements will depend on that borrow scope.
This is a significant generalization in some ways.
Multiple spans can be used at once, and so can elements from different spans.
This permits some amount of element data flow between iterations.
But it's still the case that elements are restricted within the borrow of the collection, even if they could reasonably just be copied around as independent values.
So this refinement still heavily restricts how the element values can be used.

To do better, we would need nested lifetimes.
With these, we can give collections, iterators, and spans a nested lifetime corresponding to the elements.
In the signature for `makeIterableIterator`, we need to say the iterator's element lifetimes depend only on the collection's.
Similarly, in `nextSpan`, we need to say that the the element lifetime of the span depened on the iterator's element lifetime.
Finally, in `Span`, we need to say on element accessors (like the `subscript`) just returns a value that depends on the span's element lifetime.
With this careful set of annotations, and a protocol refinement devoted to the purpose, we can make sure that element lifetimes propagate correctly for this specific API --- when we can statically make use of the refinement.

When I compare this to the simple guarantee afforded automatically to all generic code by type substitution, I feel that this is a significant loss in usability for collections (and other abstractions) with non-escapable elements.
And the best case here already relies on significant extensions (and widespread adoption thereof) beyond what the current value-dependency model is capable of expressing.

#### Lack of explicit local annotations

Another disadvantage of the current value-dependency model is that it provides no direct way to state the expected scope restrictions on a specific value.
A function's dependency signature can say that its return value is dependent on a particular parameter.
However, there is no equivalent way to say within the function that the value in a local variable can only depend on that parameter.
There is no direct translation of such a statement in terms of the model; it would require an additional rule.
This is unfortunate, because while I expect that most programmers will usually be happy to just rely on the implicit inference of scope restrictions, there are some good if situational reasons to sometimes be more explicit.

First, explicit annotations can make it easier for people to learn and teach the scope restriction rules.
When programmers feel comfortable with a language feature, they often really appreciate inference rules and syntactic sugar that make the feature less obtrusive and heavyweight.
But newcomers to a feature often really appreciate the ability to spell these things out.
Some programming teachers even make that a class rule, forcing their students to write out things like type declarations in assignments to help them internalize what the compiler is doing automatically for them.

Second, explicit annotations provide a natural way for tools to communicate inferred scope restrictions back to programmers.
Consider a programmer who's running into a mysterious problem with their code; maybe they've written a `Span` algorithm that's somehow crashing.
It's reasonable for them to ask their IDE what the lifetime of the span actually is.
Many IDEs already have similar features for e.g. spelling out the inferred type annotation when you hover over a reference to a variable.
But those features are generally designed around making things explicit that can actually be written in the language.
Without the ability to express such a restriction in the language, the IDE has to find some other way to communicate it (maybe as a prose comment), and the programmer has no way to double-check that the inferred annotation is actually correct.

Third, explicit annotations can help programmers narrow down the cause of a compiler error.
In an ideal world, of course, compiler diagnostics would always point you exactly at the line of code that you need to fix.
In practice, this can be difficult.
Lifetime errors often arise because of a conflict: a value has a lifetime dependency that it shouldn't have at the point where it's used.
The compiler cannot know whether the problem is that the value is being used wrong (it should be okay that it has that dependency) or defined wrong (it shouldn't have that dependency).
By being explicit about their assumptions, programmers can move diagnostics from the use site to the points where the assumption was violated.

For example, consider code like this:

```swift
1  var span: Span<Int>
2  if useGlobalArray {
3    span = globalArray.span
4  } else {
5    span = localArray.span
6  }
7  ...
8  saveSpan(span)
```

Suppose that `saveSpan` requires the span it's passed to be immortal, which is to say, to have no lifetime dependencies.
A span over the global constant `globalArray` fits the bill, but a span over the local variable `localArray` does not.
A perfect diagnostic might report on line 8 that `span` is not necessarily immortal like it's required to be, together with a note on line 5 that it won't be immortal if it's computed this way.
But compilers don't always deliver perfect diagnostics, and it's not hard to imagine that the compiler might sometimes emit this error without the extra note, leaving the programmer to figure out why the span isn't immortal for themselves.
However, if the programmer can add an annotation to `span` saying that it's expected to be immortal, then the compiler will stop reporting the error on line 8.
Instead, it will report an error on line 5, when a non-immortal span is assigned into a variable that requires something immortal.
(Of course, this isn't an excuse for compiler developers to not still try to emit the better diagnostic.)

Finally, explicit annotations can help to enforce correctness when interacting with an unsafe interface.
In safe code, the compiler will analyze both tbe uses of a value and how it's defined.
This creates a complementary balance: the narrower the scope restriction that the compiler infers for the value, the more restricted the uses of the value will be.
But with unsafe code, the compiler often just has to trust one side or the other, eliminating this balance.
An explicit annotation can make sure that the compiler still enforces the assumptions that the unsafe code requires.

For example, a C function might need to be passed a pointer that's valid for a specific duration in order to behave correctly.
It's a C function, so it just takes an unsafe pointer; the lifetime restriction requirement is a documented requirement, not something that's going to be automatically enforced by Swift.
Now suppose that some safe Swift code computes a span with the goal of passing that span to the C function.
Without an annotation, a bug in that computation can result in a span with a narrower than expected scope, silently causing the pointer to not meet the documented restriction.
But an explicit annotation of the required scope of the span prior to extracting the pointer from it will not just document the expectation in source, it will actually enforce it: the compiler will object if a too-narrow span is ever assigned to the explicitly-annotated variable.

#### Flow-sensitive diagnostics for invariant lifetime requirements

This last disadvantage is significant enough to be worthy of inclusion.
I will readily acknowledge that it is less important than the others, though.

The value-dependency model always associates lifetime dependencies with specific abstract values.
When there's a restriction on the lifetime dependencies for some mutable variable, the model is not generally going to enforce that restriction as an invariant on the variable.
Instead, it's going to enforce it specifically on the abstract value of the variable at some specific point.
This is a more control-flow-sensitive rule and can easily lead to diagnostics that seem misplaced.

As an example, consider a function with an `inout` parameter of non-escapable type.
Suppose that the dependency signature of the function says that the function does not add dependencies to the parameter; that is, the final value in the `inout` parameter must have no more dependencies than the initial value.
This is very common: it is in fact the default rule for `inout` parameters.

Now suppose that, within the body of the function, there is a change to the parameter (let's suppose it's an `insert` call) which adds a dependency to its value.
The lifetime checker will not generally be able to diagnose this immediately at the `insert` call, because it does not enforce the restriction on the parameter as an invariant.
Instead, it will just update its internal tracking to record that there's a new dependency and continue onwards.
In some ways, this is arguably good.
The checker is allowing the value in the parameter to subsequently change, and if it changes to a value with the original dependencies (or less), the postcondition on the parameter will be satisfied and there's no reason to diagnose.

But the consequence of not enforcing the restriction as an invariant is that the diagnostic becomes sensitive to control flow.
Assume for the sake of argument that the value *isn't* restored to something with fewer dependencies.
Then a fully precise statement of the error is that *there exists a control flow path leading to an exit from the function which leaves a value in the `inout` parameter with too many dependencies*.
This is fundamentally harder to diagnose than just pointing at the `insert` call, like it could if it were enforcing an invariant on the parameter.
The analysis has to walk the control flow of the function, and it is likely to only detect the problem when that walk actually reaches the exit.
It might reaonably just emit the diagnostic there without even noting the call that added the dependency.
Even if it does point out the `insert` call, it has to also point out the path that led to the exit.
After all, the bug might not be that `insert` was called; it might just be that the value was expected to be reset later.
It is just fundamentally harder to provide a good, concise diagnostic under this rule.

### Advantages of the value-dependency model

None of that is to say that there aren't upsides to the value-dependency model.

Probably the biggest advantage is that it builds relatively directly on top of existing analyses in the compiler.
This is, after all, what has allowed the feature to be delivered so far.
And many use cases of non-escapable values do not run into any of the difficulties around abstraction that I've laid out above.
If you just want to support working with spans of trivial values, with minimal abstraction, you don't need much from the lifetime system.

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

Let's go back to the basic problem.
The core correctness property we'd like to allow to be proven is that certain values are only used within appropriate scopes.
Static type systems are designed to restrict how values can be used.
It's reasonable to ask if these scope restrictions could instead be expressed in types.

Now, we have an existence proof that this can work, because this is exactly what Rust does.
It does have some challenges and complexities, which I'll discuss below.
But it also allows a lot of things to be expressed that we don't know how to express in the value-dependency system, allowing a lot of basic generic expressivity over non-escapable types.
And it's demonstrated a reasonable ability to evolve over time, as Rust's lifetime system has been gradually expanded over the years to allow for more things.
Adopting a type-based model does not require immediately adding every feature of Rust's lifetime types system; Swift can decide where to draw the line, release by release, and if we think a particular generalization is not worth the implementation cost, we don't have to add it.

It also does not require adopting Rust's syntax for lifetime qualifiers.
As I go through the language, I will sketch out what I feel is a workable and more Swift-like design.
Of course, there are many other options.
I am providing a concrete syntax primarily to elucidate the text.

### Concrete scope restrictions

Any given non-escaping type has a set of concrete scope restrictions.
These are the scopes that we need to be able to talk about in order to properly restrict the use of values of the type.

Most non-escapable types have at most one such restriction.
As I'll discuss soon, scope restrictions associated with generic arguments are handled differently.
Therefore, a type only needs a concrete scope restriction if it is unconditionally non-escapable.
It only needs multiple concrete scope restrictions if it has multiple independently-scoped reasons why it's unconditionally non-escapable.

So let's sketch out a syntax for specifying within a value's type that the value is restricted to a specific scope:

```swift
// We'll talk about what goes in the parentheses later.
var span: @scoped(array) Span<Int>
```

Whenever we have a value of such a type, the type must have a scope restriction.
Of course, we would normally want that scope restriction to be inferred:

```swift
// We'll talk about how the `@scoped` restriction can be inferred later.
var span: Span<Int>
```

But it can always be spelled out.

This syntax assumes that there's exactly one concrete scope restriction associated with the type.
That's true for `Span`.
As long it's true, we don't need any way to name different scope restrictions, or to declare names for them in the type.
However, it's not true for all types.
If it were only false for really weird types, we could probably reasonably subset it out of the language, at least to start.
Unfortunately, it does come up quite easily with types that just store multiple non-escapable values, like the following:

```swift
struct SpanPair<T>: ~Escapable {
  let left: Span<T>
  let right: Span<T>
}
```

Stored properties are types of values, so the concrete scope restrictions do need to be given in these types.
We could add a syntax for declaring named scope restrictions, like so:

```swift
struct SpanPair<T>: ~Escapable {
  scope _left
  scope _right

  let left: @scoped(_left) Span<T>
  let right: @scoped(_right) Span<T>
}
```

But I don't think this is actually required.
(It may be necessary to express more complex cases, but it can probably be subsetted out of the language to start.)
Instead, I think we can simply infer anonymous scope restrictions to "fill in" all the unspecified scope restrictions on stored properties.

We do need to extend the `@scoped` attribute (or whatever the syntax ends up being) to allow multiple scope restrictions to be specified:

```swift
var pair: @scoped(left: &array1, right: &array2) SpanPair<Int>
```

Here I've just allowed `@scoped` to take multiple name/scope pairs.
Names can resolve to properties of non-escapable type, which provides a natural way to specify the otherwise-anonymous scope restrictions created for stored properties.
If there's a single "pair", and it doesn't actually have a name, the type needs to only have a single scope restriction.

Note that this model hasn't actually added any syntactic burden to the definition of the non-escapable type so far.
We've just found a reasonable interpretation of the existing code that lets us propagate implicit scope specifiers on types.
We've only gained the option of being more specific.

So how do you specify a scope?
This is definitely a place we can expand over time.
What I think we clearly need at start is at least:

- The name of a variable of non-escapable type, from which we take its concrete scope restriction, e.g. `@scoped(otherSpan) Span<Int>`.

- Some syntax for specifying the scope of a `borrowing` or `inout` parameter (not necessarily of non-escapable type), e.g. `@scoped(&self) Span<Int>`.

- Some syntax for specifying the global scope, e.g. `@scoped(immortal) Span<Int>`.

To this we could gradually add member paths, intersections, scopes of local variables, and so on.
We'll probably need to support most of those in the implementation right away --- they can come up in scope inference very easily --- but we don't necessarily need user-facing syntax for them.

### Bound and unbound non-escapable types

We've said that the types of values must always specify any concrete scope restrictions in the type.
Such a type is said to be *bound*.
But not every place you can write a type is immediately the type of a value.
Forcing every non-escapable type to always be bound to a specific scope restriction would rule out a lot of useful code patterns.

This is straightforward to see with a simple `typealias`:

```swift
typealias ISpan = Span<UInt32>

func slice(span: ISpan) -> ISpan { ... }
```

If all references to a non-escapable type had to bind the type's scope restrictions, every use of `ISpan` would share the same scope restriction.
That's probably not want the programmer wants, though.
If that's what they wanted, they could've written the scope restriction explicitly in the `typealias`, like so:

```swift
typealias ISpan = @scoped(immortal) Span<UInt32>

func slice(span: ISpan) -> ISpan { ... }
```

What they probably want is for writing `ISpan` to behave just like an abbreviation of writing `Span<UInt32>`.
In other words, they want the reference to `Span` in the alias to stay unbound.
When they use `ISpan` in a context that requires scope restrictions to be bound, they can provide the scope restrictions themselves or just allow them to be decided from context exactly as if they'd written `Span<UInt32>`.

This divide between bound and unbound types is very important in the type-based model.
That's especially true for abstract positions like generic parameters and associated types.
With `typealias`es, Swift can really muddle through well enough using any rule; the compiler can always locally choose to look through the `typealias` or throw away scope restrictions if it helps make reasonable code compile.
With the generic positions, Swift really needs to know what the programmer wants, because it's not reasonable for the compiler to do some global analysis of how a generic parameter is used to figure it out.

`Iterable` provides an excellent example of a protocol that requires both:

```swift
protocol Iterable<Element>: ~Copyable, ~Escapable {
  associatedtype IterableIterator: ~Copyable & ~Escapable
  associatedtype Element: ~Copyable & ~Escapable
  where Element == IterableIterator.Element

  func makeIterator() -> IterableIterator
}
```

The `Element` associated type almost certainly ought to be a bound type.
This would rule out some largely theoretical conformances --- types that synthesize non-escapable elements during each call to `nextSpan`  --- while promoting a very clean, unrestricted programming model for standard conformances when they're generalized to support non-escapable elements.
This is because bound types provide a very straightforward generic model.
The bound scope restriction must be derived somehow from the type of `self`, which means the scope in the restriction must always be broader than the current function call.
Any context that expects a value of the bound type will have its own contextually-equivalent understanding of the scope restriction expressed in the element type.
This all means that generic code can usually move and copy values of bound types around freely, subject only to fairly minor restrictions, like not doing truly escaping things like e.g. wrapping them up as an `Any`.
That's a very strong and desirable property for collection elements in generic code.

In contrast, unbound types tend to end up with highly conservative scope restrictions that make them dependent on specific calls and accesses rather than allowing broad data flow limited only by the type.
This is necessary for types like `IterableIterator`, where the protocol really does need clients to infer a different scope restriction for every unique call to `makeIterator`.

Since there are use-cases for both, it's necessary to have a syntax for declaring generic parameters and associated types as either.
At the moment, I believe that the right default is for these positions to expect an bound type.
Unbound types therefore ought to be called out when they're needed.

I'm not sure what the right syntax for unbound types would be.
Considered in full generality, unbound types aren't merely "unbound" as binary flag: there's a whole expected signature for the scope restrictions that need to be applied to them.
This is effectively a very restricted form of higher-kinding in the type system.
I would guess that we don't really need to support more than the simple pattern of a single scope restriction, though.
An attribute like `@unscoped` on the generic parameter or associated type might be fine for that.

### Function signatures

Parameters and results of functions are types of values, so if they have non-escapable type, those types must have concrete scope restrictions.

Function declarations involving non-escaping types will generally have some number of scope parameters.
Usually these will all be implicit, and we can probably start with that.
It's going to be beneficial for the explanation if I can write them out, though, so I'm going to invent a syntax.
I can't think of a better choice offhand than writing it like a generic parameter:

```swift
func returnEither<scope a, scope b>(spanOne: @scoped(a) Span<Int>,
                                    spanTwo: @scoped(b) Span<Int>)
         -> @scoped(a & b) Span<Int>
```

Just like we did with type definitions, we can infer all of this by default:

```swift
func returnEither(spanOne: Span<Int>, spanTwo: Span<Int>) -> Span<Int>
```

The way this works is straightforward.
Since parameters have to have concrete scope restrictions, and the programmer hasn't given us one explicitly for `spanOne` or `spanTwo`, we synthesize new anonymous scope parameters to fill them in.
The compiler then applies heuristics to find a scope restriction for the return type, just like it does today under the value-dependency model.
That ends up being an intersection of the two anonymous parameter scopes.

If the programmer needs to take control, they won't usually have to write out explicit scope parameters.
In most cases, they should be able to just name the value parameters:

```swift
func returnFirst(spanOne: Span<Int>, spanTwo: Span<Int>)
         -> @scoped(spanOne) Span<Int>
```

An explicit scope parameter might be necessary in more complex cases:

```swift
func returnASpan<scope s>(span: Span<@scoped(s) Span<Int>>)
       -> @scoped(s) Span<Int>
```

Note that this is already not expressible in the value-dependency model without losing information through dependency conflation.
I think subsetting this capability at first would probably be fine.

### Inferring scope restrictions

The uses of non-escapable types in a function under the type-based model naturally create a system of scope equalities and inequalities.
Inferring scope restrictions essentially involves solving this system.
A lot of the logic of that should be very similar to the process of solving the dependency-set relationships introduced by the scope-dependency model.
However, there are some significant differences.

The first difference is that, in the value-dependency model, variables in the system are associated 1-1 with specific abstract values.
In the type-based system, variables are introduced mostly for declarations and calls.
A use of a function or type that's generic over scope restrictions essentially "opens" that entity's signature, creating fresh scope variables in the solver.
Explicit scope specifiers on types can immediately resolve some of these variables, but the rest must be inferred through solving.

The second is that, in the type-based model, type substitution can create complex equality relationships between different parts of the system.
However, it is still the case that the system is purely conjunctive.

And finally, the type-based model must reason about scope variance relationships between various types.

### Scope variance

Scopes have natural relationships with each other: some scopes are contained within others.
Two scopes can also always be intersected, although this may produce an empty scope.

There is a closely related subtyping relationship between types that carry concrete scope restrictions.
This system of scope variance preserves a lot of the flexibility of the value-dependency model under the type-based model, because it allows for scopes to be naturally shrunk until they can be merged.
For example, in the scope-dependency model, when you insert spans into an array, the spans are not required to have exactly the same dependencies.
Instead, the array value accumulates dependencies from each of those spans.
In the type-based model, the element type of the array does have to have a consistent scope restriction.
However, that scope restriction is typically a variable that must be inferred.
Whenever a span is inserted into the array, it creates a constraint that the type of that span must be a subtype of the type of the element, and thus that the scope restriction in that span's type must be a superscope of the scope restriction of the element type.
The scope restriction of the element type will thus be inferred to be an intersection of the scope restrictions of the spans, imposing an essentially similar overall constraint as was imposed by the value-dependency model.

In the examples that follow, let `smaller` and `bigger` be two scopes such that `bigger` contains `smaller`.

- Every non-escaping type is covariant with respect to its immediate scope specifiers.
  For example, `@scoped(smaller) Span<Int>` is a subtype of `@scoped(bigger) Span<Int>` because `smaller` is contained within `larger`.

- Types may be covariant, contravariant, or invariant with respect to their concrete `~Escapable` type parameters.
  For example, `Span` is covariant in its type parameter, so `@scoped(x) Span<@scoped(smaller) Span<Int>>` is a subtype of `@scoped(x) Span<@scoped(larger) Span<Int>>`.
  But `MutableSpan` is invariant in its type parameter, so `@scoped(x) MutableSpan<@scoped(smaller) Span<Int>>` is not related to `@scoped(x) MutableSpan<@scoped(larger) Span<Int>>`.
  We would need syntax for declaring this.
  I believe Rust has a defaulting rule for it, but I think it's based on a deep inspection that I'm not sure we want to do.


- higher rank
- scope specifiers on borrowing/inout
- stdlib Scope values for explicit arguments
- captures

## Proposed engineering plan

(to be written)

[SE-0176]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md
[SE-0414]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md
[SE-0446]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0446-non-escapable.md
[SE-0516]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md
[SE-0519]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md
[SSA]: https://en.wikipedia.org/wiki/Static_single-assignment_form
