# Dependency Injection

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [Rationale](#rationale)
- [Terminology](#terminology)
- [Syntax and semantics](#syntax-and-semantics)
  - [Anonymous parameters](#anonymous-parameters)
  - [Direct appliance](#direct-appliance)
  - [Shadowing](#shadowing)
- [Dropped mechanics](#dropped-mechanics)
- [Additional features](#additional-features)

<!-- /code_chunk_output -->

## Rationale

Dependency injection is a technique used to pass dependencies into a unit of work. Unlike regular input, a dependency -- which may be either another unit or pure data -- is not replaced between different invocations of the dependant unit. Dependency injection aims to eliminate the redundancy of providing the same dependencies on each invocation.

A few examples of common generic dependencies are: loggers and monitors, configuration data, database connections, sessions, execution services, etc.

In OOP, dependencies are often injected (given) to an object -- the most basic unit of work -- upon construction or initialization, and are stored inside the object as members; making them available to every method in the object, as they are all provided with a builtin reference to the object's members.

In FP, there are no objects in which one can store members to make them accessible to all of the object's methods. Instead, functions define additional parameters that can be injected to them implicitly; the unit that uses those functions can inject the dependencies once before continuing to invoke the functions, thus removing the need to pass the dependencies to each of them explicitly.

The functional approach may be considered more flexible, as the object-oriented approach involves the construction of an entire new object whenever a dependency changes.

Imagine a unit of work which includes several different functions, and needs to operate within a temporary transaction. In OOP, one would define an object responsible for these functions. The transaction would either be passed explicitly to each function; or be injected to the object upon construction, meaning that a new object will be constructed for each transaction.

Scala brought together the dependency injection techniques of both the OOP and the FP paradigms. One can inject constant dependencies into an object upon construction, while injecting more temporary dependencies via implicit parameters.

In order to retain Scala's approach on dependency injection, this proposal wishes to conserve the core mechanism that enables such approach, while distinguishing it from other mechanisms that are used for other unrelated design patters.

## Terminology

There are several possible ways we could name this ability; applying arguments indirectly; injecting dependencies; implying instances; etc. The important thing to note is that this ability involves two distinct constructs:

  1. The construct in a function's definition that makes a parameter eligible for dependency injection.
  2. The construct in a function's implementation that injects a dependency to all eligible parameters among the functions that are invoked next.

The proposal will use the pair of `implied` and `imply`, but they could be easily replaced with `injected` and `inject`, `indirect` and `apply`, or what not. What matters is that the first is a description, a property, a modifier; the second is an operative, an action, an application.

## Syntax and semantics

`implied` parameters behave similarly to how `implicit` parameters behave nowadays - values are passed to them indirectly, and in-turn they pass themselves indirectly to `implied` parameters of nested functions (propagation).

Any expression can become implied in a certain context by `imply`-ing it:

```scala
imply 2
imply otherVariable
imply function(arguments)
imply new Pokemon { ... }
```

In fact, that is the only way to make a value implied, other than the value being already implied by a function parameter.

### Anonymous parameters

Implied parameters may be anonymous:

```scala
def foo(a: Int)(implied _: String) = ???
```

The sole purpose of this pattern is to allow propagation of parameters to nested function without using them in the parent function.

### Direct appliance

Implied parameters may be applied directly, like any other parameter:

```scala
Future(f)(ExecutionContext.global)
Future(f)(imply ExecutionContext.global) // alternative syntax
Future(f).imply(ExecutionContext.global) // alternative syntax
```

### Shadowing

Whether shadowing of implied values should be allowed or not is undecided in this proposal and is open to discussion:

```scala
def feedAnimal(animal: Animal)(implied food: AnimalFood) = ???
def feedCat(cat: Cat)(implied food: CatFoot) = ???

imply DogFood("anything basically")
imply CatFood("milk") // warning? error?

val cat = Cat("MrMeow")
feedCat(cat)
feedAnimal(cat) // warning? error?
```

## Dropped mechanics

A few implicit mechanics are dropped for implied parameters, as they are replaced by other aspects of this proposal:

- `import` no longer carries over implied values from the imported scope.
- Context bounds no longer relate in any way to `implied` parameters; they are still included as a mechanic of [type classes](type-classes.md#bounds).
- Obtaining an implied parameter via `implicitly` and not via an `implied` parameter is no longer possible.

## Additional features

Other adaptations to existing syntax and additional features -- such as [unison of implicit and non-implicit parameter lists](https://contributors.scala-lang.org/t/proposal-to-revise-implicit-parameters/3044/54), [implicit function types](https://dotty.epfl.ch/docs/reference/contextual/implicit-function-types.html) and [implicit by-name parameters](https://dotty.epfl.ch/docs/reference/other-new-features/implicit-by-name-parameters.html) -- are beyond the scope of this proposal, but should be compatible with the changes it introduces.
