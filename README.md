# Scala implicits proposal

This is a proposal to revise Scala's implicits in order to simplify them while retaining their usability.

It is a direct follow-up on [the "contextual abstractions" proposal](https://contributors.scala-lang.org/t/updated-proposal-revisiting-implicits/3821), which mostly shares the same motivation as this proposal, but heavily differentiates in its design and rationale.

This proposal draws much inspiration from [LukaJCB's proposal](https://github.com/LukaJCB/typeclass-proposal) on how to approach and design type classes.

## Rationale

Implicits are a powerful tool that combines multiple techniques and design patterns; dependency injection, type classes, extension methods and type conversions.

It's not only that all of these use te same [polysemic](https://en.wikipedia.org/wiki/Polysemy) construct -- `implicit` -- but that the language views them as part of the same abstraction -- be it "contextual abstractions" or "term inference".  

This abstraction is the major cause of confusion with implicits, as it is not always easy to grasp and is often not necessary for the everyday developer.

At its core, this proposal wishes to break this abstraction and grant each feature its own unique set of constructs, completely separated / orthogonal to the others. Call a spade a spade; or as we say in Hebrew, call a child by his name.

### Context

One major fallacy I've noted in the previous discussions on revisioning implicits is the use of the term _context_ for two very distinct and unrelated concepts:

 1. Compile-time constraint that bound a generic type to a specific type class, as in [Context Bound](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html) and as in [Haskell's terminology](https://www.haskell.org/tutorial/classes.html).
 2. Ephemeral run-time data that is shared among multiple units that are composed and interact in a tree-like structure, as in [React's Context](https://reactjs.org/docs/context.html).

In this proposal, _context_ bears no special meaning or does not relate to any particular concept.

Moreover, this gives further reasoning to separate between dependency injection and type classes; type classes are _constraints_, not parameters. This proposal wishes to remove some of the old `implicit` mechanics and hone it solely for the purpose of dependency injection.

## Details

The details of the proposal are described in the following documents:

1. [Dependency injection](dependency-injection.md).
2. [Lenses](lens.md).
3. [Type classes](type-classes.md).
4. [Extensions (extension methods)](extensions.md).
5. [Type conversions](type-conversions.md).
6. [Comparison with Scala 2 implicits](vs-scala2.md).

### A word about syntax

Please note that this proposal does not concern itself too heavily with the exact syntax that will eventually express the concepts it introduces. There are many small syntactic details and variations that change very little the general notions and ideas of the proposal, and therefore they will not be discussed in depth.
