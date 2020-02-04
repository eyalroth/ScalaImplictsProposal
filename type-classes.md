# Type Classes

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [Overview](#overview)
  - [Goals](#goals)
- [Basic features](#basic-features)
  - [Definition](#definition)
  - [Bounds](#bounds)
  - [Usage](#usage)
  - [Variance](#variance)
- [Advanced features](#advanced-features)
  - [Infix notation](#infix-notation)
  - [Generic type instances](#generic-type-instances)
    - [Induction](#induction)
    - [The diamond problem](#the-diamond-problem)
    - [Derivation](#derivation)
  - [Multiversal equality](#multiversal-equality)

<!-- /code_chunk_output -->

## Overview

A big part of the confusion regarding implicit stems from the attempt to bend their mechanic in order to introduce type classes into the language. This proposal views type classes as "first class citizens" that deserve their own set of constructs and rules that are focused solely on them.

It is not so simple to determine what are the common use-cases for type classes in general, and in Scala in particular. This is even more difficult for me as I did not experience their usage first-hand. I am not part of the functional-programming community; and even though type classes are not inherently related to FP, they were introduced by those languages and are widely used in that community.

### Goals

Despite my lack of experience with type classes, I did gather the following requirements from previous discussions and proposals:

- Multi parameter type classes.
- Orphan instances.
- Infix / unary operations.
- Branching type class inheritance -- aka "the diamond problem".
- Easy resolution of coherence (covered by [lenses](lenses.md#resolving-conflicts)).

This proposal sets itself these requirements as goals that must be achieved.

## Basic features

### Definition

A type class is composed of two parts; interface and implementation. An interface declares abstract functions -- mostly on the enclosed generic type(s) -- while the implementation defines concrete instructions for these functions.

Despite this concept being shared with inheritance and subtyping, type classes are a different concept. A type class interface should not be thought of as a `trait` nor an `abstract class`, and its implementation should not be thought of as an `object` nor an "instance" in the OOP notion.

For the purpose of familiarity, this proposal will use the term `typeclass` for the interface of a type class and `typeinstance` for the implementation. I'll leave the discussion regarding the specific wording outside the scope of this proposal.

Let's start with a simple example of the `Ordering` type class:

```scala
typeclass Ordering[A] {
  def compare(x: A, y: A): Int
}

lens Ordering {
  typeinstance IntOrdering implements Ordering[Int] {
    override def compare(x: Int, y: Int): Int = {
      java.lang.Integer.compare(x, y)
    }
  }
}
```

The structure is quite similar to that of regular classes, however there are a few differences:

- A `typeclass` must have at least one type-parameter.
- A `typeclass` cannot "extend" other `typeclass`es.
- Both `typeclass` and `typeinstance` have no constructor and cannot be instantiated.
- A `typeinstance`, being an "implicit interpretation", must be declared inside a `lens` (see [lenses](lens.md#restrictions)).
- A `typeinstance` is never anonymous.

### Bounds

Type classes are used only in conjunction with other type-parameterized (generic) constructs; functions, classes (constructors) or other type classes. In order to use a type class, one must first require that it will be available for the parameterized type(s):

```scala
def sort[A]<Ordering[A]>(xs: Seq[A]): Seq[A] = ???

trait Set[A]<Eql[A, A]> {
  def contains(x: A): Boolean = ???
}
```

Unlike the old context bounds, this new form of "type class bounds" is not mapped to a parameter, and therefore can be defined anywhere a type-parameter is defined, no matter if that definition accepts parameters or not (`trait`s do not accept parameters). This makes it much more closely related to (upper / lower) type bounds than the old context bounds.

Note that the specific `<>` syntax used in this proposal is not the important part -- [as mentioned earlier](README.md#a-word-about-syntax) -- and was chosen because it is quite distinguishable from the old syntax, which makes it clearer to see the changes introduced in this proposal.

### Usage

After requiring a type class via bounds, type classes can be invoked as if they were a "static" object:

```scala
def sort[A]<Ordering[A]>(xs: Seq[A]): Seq[A] = {
  Ordering.compare(xs.head, xs.tail.head)
  ...
}
```

### Variance

Since type classes and instances are not data types, they need not to consider variance:

```scala
// invalid `+`
typeclass Ordering[+A] { ... }
```

They are never placed in any location that requires variance considerations -- e.g., variable, parameter, return type.

## Advanced features

### Infix notation

Type classes have the unique ability to define infix operations on their enclosed types:

```scala
typeclass Eql[A, B] {
  def areEqual(a: A, b: B): Boolean

  def infix(a: A) {
    def ==(b: B) = areEqual(a, b)
  }

  // alternative syntax
  def infix[B] { b =>
    def ==(a: A) = areEqual(a, b)
  }
}
```

The infix operations will be available on instances of types that were bound to the type class:

```scala
class Set[A]<Eql[A, A]> {
  def contains(elem: A): Boolean = {
    internal.exists(_ == elem)
  }
}
```

### Generic type instances

Type instances may be generic (have type parameters):

```scala
lens Eql {
  typeinstance GenericEql[A, B] implements Eql[A, B] {
    override def areEqual(a: A, b: B): Boolean = a eq b
  }
}
```

This capability is especially useful when the type parameters are bound to a type class, as shown in the following use cases:

#### Induction

Similarly to [LukaJCB's proposal](https://github.com/LukaJCB/typeclass-proposal/blob/master/docs/instances.md#multiple-inductive-instances), we can define type instances that are "deduced" from nested type instances:

```scala
case class Foo[A](value: A)

lens Foo {
  typeinstance FooEql[A]<Eql[A, A]> implements Eql[Foo[A], Foo[A]] {
    override def areEqual(a: Foo[A], b: Foo[A]): Boolean =
      Eql.areEqual(a.value, b.value)
  }

  typeinstance FooOrdering[A]<Ordering[A]> implements Ordering[Foo[A]] {
    override def compare(x: Foo[A], y: Foo[A]): Int =
      Ordering.compare(x.value, y.value)
  }
}
```

#### The diamond problem

One of the problems with encoding type classes via subtyping is that it doesn't easily support branching hierarchies -- aka "the diamond problem" -- which is described [in this article](https://typelevel.org/blog/2016/09/30/subtype-typeclasses.html). The problem can be largely mitigated under the current proposal.

As explained earlier, a "type class bound" can be present wherever a type parameter exists; meaning, we can bind the enclosed type of a type class to another type class, thus establishing a clear connection between the two where one "implies" the other:

```scala
object fanctors {
  typeclass Functor[F[_]] {
    def map[A, B](fa: F[A])(f: A => B): F[B]
  }

  typeclass Applicative[F[_]]<Functor[F]> {
    def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
    def pure[A](a: A): F[A]
  }

  typeclass Traverse[F[_]]<Functor[F]> {
    def traverse[G[_], A, B]<Applicative[G]>(fa: F[A])(f: A => G[B]): G[F[B]]
  }
}
```

We can use genreic type instances to implement the "root" type class `Functor` with either of the branching type classes:

```scala
lens functors {
  typeinstance ApplicativeFunctor[F[_]]<Applicative[F]> implements Functor[F] {
    override def map[A, B](fa: F[A])(f: A => B): F[B] = {
      Applicative.ap(Applicative.pure(f))(fa)
    }
  }

  typeinstance TraverseFunctor[F[_]]<Traverse[F]> implements Functor[F] {
    override def map[A, B](fa: F[A])(f: A => B): F[B] = ???
  }
}
```

When a function binds a type to both of the branching type classes, there seem to be a conflict:

```scala
import functors._

def foo[F[_]]<Applicative[F], Traverse[F]>(f: F[_]): Int = {
  Functor.map(f)(...) // seemingly ambiguous
  ...
}
```

However, since both `typeinstance`s were declared in the same `lens`, there is no ambiguity regarding their order, and it's possible to determine which of them will be used.

In cases where there is an ambiguity, the conflicts can be resolved [via lenses](lens.md#resolving-conflicts).

#### Derivation

The type class derivation mechanism [described in the "contextual abstractions" proposal](https://dotty.epfl.ch/docs/reference/contextual/derivation.html) and [inspired by Haskell](https://www.haskell.org/onlinereport/haskell2010/haskellch11.html) is not inherently supported by this proposal.

The mechanism is split into two parts; the first allows to "annotate" data types in order to automatically generate type instances for them; the second allows to define a new a way of auto generating those instances.

The first part is rather redundant given the ability to define generic `typeinstance`s:

```scala
lens Eql {
  typeinstance MirrorEql[A]<Mirror[A]> implements Eql[A, A] = {
    override def areEqual(a: A, b: A): Boolean = ???
  }
}
```

Generic type instances are more powerful, as they remove the need of a "boilerplate annotating" a data type (`X derives Eql`) in order to gain auto generated type instances.

As for the second part -- implementing the auto generated instances -- there's a need for a certain "compiler magic". Much like in the original proposal, a `Mirror` type class needs to be defined and auto generated by the compiler for the appropriate types (enums, case classes, etc):

```scala
typeclass Mirror[A] {
  type MirroredType
  type MirroredElemTypes
  type MirroredMonoType
  type MirroredLabel <: String
  type MirroredElemLabels <: Tuple
}

object Mirror {
  typeclass Product[A]<Mirror[A]> {
    def fromProduct(p: scala.Product): Mirror.MirroredMonoType
  }

  typeclass Sum[A]<Mirror[A]> {
    def ordinal(x: Mirror.MirroredMonoType): Int
  }
}
```

However, unlike the original proposal, type instances are _not_ actual objects, so this kind of implementation can only be achieved either via macros or reflection:

```scala
lens Eql {
  typeinstance MirrorEql[A]<Mirror.Sum[A]> implements Eql[A, A] = {
    override def areEqual(x: A, y: A): Boolean = {
      val ordx = s.ordinal(x)
      (s.ordinal(y) == ordx) && check(elems(ordx))(x, y)
    }
    // `Eql` is not a data type and cannot be a parameter
    private def check(elem: Eql[_, _])(x: Any, y: Any): Boolean =
      elem.asInstanceOf[Eql[Any, Any]].eqv(x, y)
  }
}
```

Obviously, macros are preferable to reflection due to performance considerations. I'm not fluent enough with macros to provide a full (hypothetical) example, but I suspect that there's a need for two new macro features; (a) representation of type classes and type instances; and (b) a way to look for an instance of a type class associated with a data type in a scope (regardless if it's bound or not):

```scala
// macro features
val eqlOfInt: TypeInstance[Eql[Int, Int]] = summon[Eql[Int, Int]]
def eqlOfType[A]<Type[A]>(): TypeInstance[Eql[A, A]] = summon[Eql[A, A]]
```

This "lens lookup" feature -- aka `summon` -- should not be allowed in runtime. It strongly contradicts the idea that type classes are a compile-time construct by nature. Much like with type-erasure, this information may be available via reflection, but it shouldn't be available by a "first class" construct.

### Multiversal equality

Another topic covered in the "contextual abstractions" proposal is [multiversal type-safe equality](https://dotty.epfl.ch/docs/reference/contextual/multiversal-equality.html), which is also not clearly supported in this proposal.

Not to be mistaken, an equality type class -- aka `Eql` -- is a completely valid use-case, which helps signify that a certain unit requires a special and dedicated care for the equality of its types. Collections are a good example of this:

```scala
trait Set[A]<Eql[A, A]>
```

However, enabling the globally available equality operator `==` is impossible to do with type classes, as they do not work with concrete types, but only with parameterized types that are also bound to them:

```scala
object equality {
  val a: String = "a"
  val b: Int = 2

  // doesn't resolve to `Eql.==` infix method
  a == b

  // resolves correctly
  def eql[A, B]<Eql[A, B]>(a: A, b: B): Boolean = a == b
  eql(a, b)
}

lens equality {
  typeinstance StringIntEql implements Eql[String, Int] { ... }
}
```
