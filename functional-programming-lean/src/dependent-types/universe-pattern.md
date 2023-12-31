# The Universe Design Pattern

In Lean, types such as `Type`, `Type 3`, and `Prop` that classify other types are known as universes.
However, the term _universe_ is also used for a design pattern in which a datatype is used to represent a subset of Lean's types, and a function converts the datatype's constructors into actual types.
The values of this datatype are called _codes_ for their types.

Just like Lean's built-in universes, the universes implemented with this pattern are types that describe some collection of available types, even though the mechanism by which it is done is different.
In Lean, there are types such as `Type`, `Type 3`, and `Prop` that directly describe other types.
This arrangement is referred to as _universes à la Russell_.
The user-defined universes described in this section represent all of their types as _data_, and include an explicit function to interpret these codes into actual honest-to-goodness types.
This arrangement is referred to as _universes à la Tarski_.
While languages such as Lean that are based on dependent type theory almost always use Russell-style universes, Tarski-style universes are a useful pattern for defining APIs in these languages.

Defining a custom universe makes it possible to carve out a closed collection of types that can be used with an API.
Because the collection of types is closed, recursion over the codes allows programs to work for _any_ type in the universe.
One example of a custom universe has the codes `nat`, standing for `Nat`, and `bool`, standing for `Bool`:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean NatOrBool}}
```
Pattern matching on a code allows the type to be refined, just as pattern matching on the constructors of `Vect` allows the expected length to be refined.
For instance, a program that deserializes the types in this universe from a string can be written as follows:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean decode}}
```
Dependent pattern matching on `t` allows the expected result type `t.asType` to be respectively refined to `NatOrBool.nat.asType` and `NatOrBool.bool.asType`, and these compute to the actual types `Nat` and `Bool`.

Like any other data, codes may be recursive.
The type `NestedPairs` codes for any possible nesting of the pair and natural number types:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean NestedPairs}}
```
In this case, the interpretation function `NestedPairs.asType` is recursive.
This means that recursion over codes is required in order to implement `BEq` for the universe:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean NestedPairsbeq}}
```

Even though every type in the `NestedPairs` universe already has a `BEq` instance, type class search does not automatically check every possible case of a datatype in an instance declaration, because there might be infinitely many such cases, as with `NestedPairs`.
Attempting to appeal directly to the `BEq` instances rather than explaining to Lean how to find them by recursion on the codes results in an error:
```lean
{{#example_in Examples/DependentTypes/Finite.lean beqNoCases}}
```
```output error
{{#example_out Examples/DependentTypes/Finite.lean beqNoCases}}
```
The `t` in the error message stands for an unknown value of type `NestedPairs`.

## Type Classes vs Universes

Type classes allow an open-ended collection of types to be used with an API as long as they have implementations of the necessary interfaces.
In most cases, this is preferable.
It is hard to predict all use cases for an API ahead of time, and type classes are a convenient way to allow library code to be used with more types than the original author expected.

A universe à la Tarski, on the other hand, restricts the API to be usable only with a predetermined collection of types.
This is useful in a few situations:
 * When a function should act very differently depending on which type it is passed—it is impossible to pattern match on types themselves, but pattern matching on codes for types is allowed
 * When an external system inherently limits the types of data that may be provided, and extra flexibility is not desired
 * When additional properties of a type are required over and above the implementation of some operations

Type classes are useful in many of the same situations as interfaces in Java or C#, while a universe à la Tarski can be useful in cases where a sealed class might be used, but where an ordinary inductive datatype is not usable.

## A Universe of Finite Types

Restricting the types that can be used with an API to a predetermined collection can enable operations that would be impossible for an open-ended API.
For example, functions can't normally be compared for equality.
Functions should be considered equal when they map the same inputs to the same outputs.
Checking this could take infinite amounts of time, because comparing two functions with type `Nat → Bool` would require checking that the functions returned the same `Bool` for each and every `Nat`.

In other words, a function from an infinite type is itself infinite.
Functions can be viewed as tables, and a function whose argument type is infinite requires infinitely many rows to represent each case.
But functions from finite types require only finitely many rows in their tables, making them finite.
Two functions whose argument type is finite can be checked for equality by enumerating all possible arguments, calling the functions on each of them, and then comparing the results.
Checking higher-order functions for equality requires generating all possible functions of a given type, which additionally requires that the return type is finite so that each element of the argument type can be mapped to each element of the return type.
This is not a _fast_ method, but it does complete in finite time.

One way to represent finite types is by a universe:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean Finite}}
```
In this universe, the constructor `arr` stands for the function type, which is written with an `arr`ow.

Comparing two values from this universe for equality is almost the same as in the `NestedPairs` universe.
The only important difference is the addition of the case for `arr`, which uses a helper called `Finite.enumerate` to generate every value from the type coded for by `t1`, checking that the two functions return equal results for every possible input:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean FiniteBeq}}
```
The standard library function `List.all` checks that the provided function returns `true` on every entry of a list.
This function can be used to compare functions on the Booleans for equality:
```lean
{{#example_in Examples/DependentTypes/Finite.lean arrBoolBoolEq}}
```
```output info
{{#example_out Examples/DependentTypes/Finite.lean arrBoolBoolEq}}
```
It can also be used to compare functions from the standard library:
```lean
{{#example_in Examples/DependentTypes/Finite.lean arrBoolBoolEq2}}
```
```output info
{{#example_out Examples/DependentTypes/Finite.lean arrBoolBoolEq2}}
```
It can even compare functions built using tools such as function composition:
```lean
{{#example_in Examples/DependentTypes/Finite.lean arrBoolBoolEq3}}
```
```output info
{{#example_out Examples/DependentTypes/Finite.lean arrBoolBoolEq3}}
```
This is because the `Finite` universe codes for Lean's _actual_ function type, not a special analogue created by the library.

The implementation of `enumerate` is also by recursion on the codes from `Finite`.
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteAll}}
```
In the case for `Unit`, there is only a single value.
In the case for `Bool`, there are two values to return (`true` and `false`).
In the case for pairs, the result should be the Cartesian product of the values for the type coded for by `t1` and the values for the type coded for by `t2`.
In other words, every value from `t1` should be paired with every value from `t2`.
The helper function `List.product` can certainly be written with an ordinary recursive function, but here it is defined using `for` in the identity monad:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean ListProduct}}
```
Finally, the case of `Finite.enumerate` for functions delegates to a helper called `Finite.functions` that takes a list of all of the return values to target as an argument.

Generally speaking, generating all of the functions from some finite type to a collection of result values can be thought of as generating the functions' tables.
Each function assigns an output to each input, which means that a given function has \\( k \\) rows in its table when there are \\( k \\) possible arguments.
Because each row of the table could select any of \\( n \\) possible outputs, there are \\( n ^ k \\) potential functions to generate.

Once again, generating the functions from a finite type to some list of values is recursive on the code that describes the finite type:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteFunctionSigStart}}
```

The table for functions from `Unit` contains one row, because the function can't pick different results based on which input it is provided.
This means that one function is generated for each potential input.
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteFunctionUnit}}
```
There are \\( n^2 \\) functions from `Bool` when there are \\( n \\) result values, because each individual function of type `Bool → α` uses the `Bool` to select between two particular `α`s:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteFunctionBool}}
```
Generating the functions from pairs can be achieved by taking advantage of currying.
A function from a pair can be transformed into a function that takes the first element of the pair and returns a function that's waiting for the second element of the pair.
Doing this allows `Finite.functions` to be used recursively in this case:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteFunctionPair}}
```

Generating higher-order functions is a bit of a brain bender.
Each higher-order function takes a function as its argument.
This argument function can be distinguished from other functions based on its input/output behavior.
In general, the higher-order function can apply the argument function to every possible argument, and it can then carry out any possible behavior based on the result of applying the argument function.
This suggests a means of constructing the higher-order functions:
 * Begin with a list of all possible arguments to the function that is itself an argument.
 * For each possible argument, construct all possible behaviors that can result from the observation of applying the argument function to the possible argument. This can be done using `Finite.functions` and recursion over the rest of the possible arguments, because the result of the recursion represents the functions based on the observations of the rest of the possible arguments. `Finite.functions` constructs all the ways of achieving these based on the observation for the current argument.
 * For potential behavior in response to these observations, construct a higher-order function that applies the argument function to the current possible argument. The result of this is then passed to the observation behavior.
 * The base case of the recursion is a higher-order function that observes nothing for each result value—it ignores the argument function and simply returns the result value.

Defining this recursive function directly causes Lean to be unable to prove that the whole function terminates.
However, using a simpler form of recursion called a _right fold_ can be used to make it clear to the termination checker that the function terminates.
A right fold takes three arguments: a step function that combines the head of the list with the result of the recursion over the tail, a default value to return when the list is empty, and the list being processed.
It then analyzes the list, essentially replacing each `::` in the list with a call to the step function and replacing `[]` with the default value:
```lean
{{#example_decl Examples/DependentTypes/Finite.lean foldr}}
```
Finding the sum of the `Nat`s in a list can be done with `foldr`:
```lean
{{#example_eval Examples/DependentTypes/Finite.lean foldrSum}}
```

With `foldr`, the higher-order functions can be created as follows:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteFunctionArr}}
```
The complete definition of `Finite.Functions` is:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:FiniteFunctions}}
```



Because `Finite.enumerate` and `Finite.functions` call each other, they must be defined in a `mutual` block.
In other words, right before the definition of `Finite.enumerate` is the `mutual` keyword:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:MutualStart}}
```
and right after the definition of `Finite.functions` is the `end` keyword:
```lean
{{#include ../../../examples/Examples/DependentTypes/Finite.lean:MutualEnd}}
```

This algorithm for comparing functions is not particularly practical.
The number of cases to check grows exponentially; even a simple type like `((Bool × Bool) → Bool) → Bool` describes {{#example_out Examples/DependentTypes/Finite.lean nestedFunLength}} distinct functions.
Why are there so many?
Based on the reasoning above, and using \\( \\left| T \\right| \\) to represent the number of values described by the type \\( T \\), we should expect that
\\[ \\left| \\left( \\left( \\mathtt{Bool} \\times \\mathtt{Bool} \\right) \\rightarrow \\mathtt{Bool} \\right) \\rightarrow \\mathtt{Bool} \\right| \\]
is 
\\[ \\left|\\mathrm{Bool}\\right|^{\\left| \\left( \\mathtt{Bool} \\times \\mathtt{Bool} \\right) \\rightarrow \\mathtt{Bool} \\right| }, \\]
which is
\\[ 2^{2^{\\left| \\mathtt{Bool} \\times \\mathtt{Bool} \\right| }}, \\]
which is
\\[ 2^{2^4} \\]
or 65536.
Nested exponentials grow quickly, and there are many higher-order functions.


## Exercises

 * Write a function that converts any value from a type coded for by `Finite` into a string. Functions should be represented as their tables.
 * Add the empty type `Empty` to `Finite` and `Finite.beq`.
 * Add `Option` to `Finite` and `Finite.beq`.

