# Lenses

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [Rationale](#rationale)
  - [Better origins](#better-origins)
  - [The problem of ambiguity](#the-problem-of-ambiguity)
    - [Conflict resolution](#conflict-resolution)
    - [Global coherence](#global-coherence)
- [Syntax and semantics](#syntax-and-semantics)
  - [Restrictions](#restrictions)
  - [Companion lens](#companion-lens)
  - [Package lens](#package-lens)
  - [Including lens](#including-lens)
    - [Encapsulation](#encapsulation)
  - [Resolving conflicts](#resolving-conflicts)
    - [Shadowing](#shadowing)
    - [Including resolutions](#including-resolutions)

<!-- /code_chunk_output -->

## Rationale

### Better origins

One complaint about implicits that might play a large role in the confusion surrounding them is the lack of visibility regarding the origin of an implicit in a given scope.

Implicits are automatically applied whenever they are `import`ed into a scope. This may be regarded as a side-effect, as it is both unseen and doesn't exist for any other construct. It may be confusing for newcomers who expect `import` to act as a name-resolution mechanic and nothing more.

One of the aims of this part of the proposal is to better help developers identify the origins of implicits in their scope -- preferably by mere static analysis of the source code -- and to distinguish between name resolution and the appliance of implicit interpretations.

### The problem of ambiguity

In programming, ambiguity arises whenever a compiler (or interpreter) cannot infer the meaning of a piece of code because it has more than one valid interpretation.

Name collisions are one example of an ambiguity, and in Scala there are a few yet sufficient ways for dealing with them. Function overloading is another area where ambiguity might creep in, but it's usually easy to detect and resolve.

On the other hand, deduction of implicits is an area where ambiguity is often impossible to resolve, because the conflicting interpretations are defined externally and because there are no means provided to resolve these conflicts locally:

```scala
package a {
  implicit object IntOrderingA extends Ordering[Int] {
    override def compare(x: Int, y: Int): Int = 1
  }
  implicit class IntExtensionsA(i: Int) {
    def foo() =  i + 1
  }
  implicit def intToStringA(i: Int): String = i.toString() + "a"
}

package b {
  implicit object IntOrderingB extends Ordering[Int] {
    override def compare(x: Int, y: Int): Int = -1
  }
  implicit class IntExtensionsB(i: Int) {
    def foo() =  i + 2
  }
  implicit def intToStringB(i: Int): String = i.toString() + "b"
}

object local {
  import a._
  import b._

  implicitly[Ordering[Int]] // ambiguity (type class)
  1.foo()  // ambiguity (extension method)
  val s: String = 1 // ambiguity (type conversion)
}
```

Despite this proposal's approach on splitting the abstraction connecting the various implicit techniques, it does acknowledge this as a shared problem between them; that is, except for dependency injection, where everything is defined locally.

#### Conflict resolution

If we look at name conflicts, there are two ways to resolve them. The first is renaming the symbol on `import`, which resolves the conflicts for the entire scope. The second is to specify the fully qualified name of the symbol, which resolves it for that location and that location only.

This proposal wishes to enable conflict resolution for implicits via the "scope resolution" rather than "call-site resolution", and for two reasons:

1. Implicits are all about less verbose code, and call-site resolution is by far a more verbose option.
2. Call-site resolution requires invention of new syntax that needs to integrate with already existing language construct, making it a hefty task to say the least.

Additionally, unlike name resolution, implicit resolution could be extended to support larger scopes than just a single source-code file, by specifying resolutions inside a package object and thus applying these resolutions to the entire package.

#### Global coherence

Coherence is the property of a program which determines whether its meaning can be interpreted deterministically; i.e, with no ambiguity.

Coherence can be attempted via global coherence or "global uniqueness", which states that no ambiguous interpretations should be allowed to co-exist across the entire program -- including third party dependencies -- no matter if they are being used or not.

For instance, in the case of type classes, global coherence means that no two type instance definitions for the same type class and type may exist in the entire program.

Global coherence is extremely problematic, and for two reasons:

1. It is hard to implement, as the JVM compiler needs to scan the entire classpath. Moreover, the classpath of a JVM program may change at runtime, preventing any complete validation of global coherence by the compiler.

2. It cripples designs that encourage developers to enrich third party types and type classes -- for instance, with "orphan instances" -- as global coherence is expected to be violated in a program that utilizes the many uncoordinated endeavors of different third party developers.

For these reasons, this proposal completely disregards any attempts at achieving global coherence. It may be worth noting that global coherence is not attempted with other aspects that may introduce ambiguity, such as name resolution, and perhaps for the same exact reasons.

## Syntax and semantics

Lenses are `object`-like modular components that introduce possible interpretations for implicit constraints:

```scala
lens IntLens {
  typeinstance IntOrdering implements Ordering[Int] { ... }
  
  extension IntExtensions extends Int {
    def plusOne(): Int = this + 1
  }
  
  conversion intToString(i: Int): Double = i.toString()
}
```

They are named "lenses" as they apply interpretations to a scope, much like optical lenses can be mounted over an optical scope and thus modify the information that passes through the scope.

### Restrictions

There are two restrictions regarding lenses and implicit interpretations:

1. `lens` cannot include anything other than implicit interpretations -- not `def`s, `class`es, `object`s, etc.

2. Implicit interpretations can be declared only inside a `lens`.

### Companion lens

Companion lens are similar to companion objects in the sense that they accompany another class, but their purpose is different. They are the most basic form of introducing implicit interpretation into a scope -- that of their companion class or object:

```scala
case class Foo(bar: Int)

class FooSorter {
  def sort(foos: Seq[Foo]): Seq[Foo] = foos.sorted() // `FooOrdering` inferred
}

object FooSorter {
  val ZERO: Int = Foo(0) // `fooToInt` inferred
}

lens FooSorter {
  typeinstance FooOrdering implements Ordering[Foo] { ... }
  
  conversion fooToInt(foo: Foo): Int = foo.bar
}
```

Note that there is no need to `import` any of the interpretations; they are automatically applied in the companion class and object.

### Package lens

Once more, similarly to objects, lenses may come in the form of package lenses:

```scala
object core {
  1.plusOne() // `IntExtensions` inferred

  class Foo {
    def bar(i: Int) = i.plusOne() // `IntExtensions` inferred
  }
}

lens core {
  extension IntExtensions extends Int {
    def plusOne(): Int = this + 1
  }
}
```

### Including lens

Inclusion of interpretations from other packages is possible by including said packages' lenses into the companion lens or package lens of the relevant scope:

```scala
object scala {
  object math {
    typeclass Ordering[A] { ... }

    lens Ordering {
      typeinstance Int implements Ordering[Int] { ... }
    }
  }
}

import scala.math.Ordering

class Sorter {
  def sort(xs: Seq[Int]): Seq[Int] = {
    xs.sorted() // `scala.math.Ordering.Int` inferred
  }
}

lens Sorter includes Ordering
```

Note that `import`ing a symbol does not automatically includes the interpretations in its companion lens; they must be explicitly included in the local lens.

Inclusion of interpretations is transitive; meaning, if lens `b` includes lens `a`, and `c` includes `b`, then `c` includes `a` as well:

```scala
lens a {
  typeinstance IntOrdering implements Ordering[Int] { ... }
}

lens b includes a {
  typeinstance DoubleOrdering implements Ordering[Double] { ... }
}

// contains both `IntOrdering` and `DoubleOrdering`
lens c includes b
```

#### Encapsulation

It's possible to utilize access modifiers to encapsulate certain interpretations:

```scala
object a {
  case class Point(x: Int, y: Int) {
    def delta(): Int = math.sqrt(x.sqr() + y.sqr())
  }
}

lens a {
  private extension IntExtensions extends Int {
    def sqr(): Int = this * this
  }

  typeinstance PointOrdering implements Ordering[Point] { ... }
}

object b {
  1.sqr() // doesn't compile
  Seq(Point(4,1), Point(2,3)).sorted() // compiles
}

lens b includes a
```

### Resolving conflicts

Whenever conflicting interpretations are included in the same lens, the will need to be resolved, otherwise the compiler will emit an error.

There are plenty of syntaxes that may work for conflict resolution; [as mentioned earlier](README.md#a-word-about-syntax), this proposal does not concern itself too heavily with the exact syntax that will be chosen. The only concern is that conflict resolution will be declared inside the `lens`:

```scala
lens a {
  typeinstance IntOrderingA implements Ordering[Int] { ... }
  extension IntExtensionsA extends Int {
    def foo() =  ???
  }
  conversion intToStringA(i: Int): String = ???
}

lens b {
  typeinstance IntOrderingB implements Ordering[Int] { ... }
  extension IntExtensionsB extends Int {
    def foo() =  ???
  }
  conversion intToStringB(i: Int): String = ???
}

lens local includes a, b {
  deduce typeclass Ordering[Int] = IntOrderingA
  infer conversion (Int => String) as intToStringB
  resolve Int.foo with extension IntExtensionsA
}
```

It may be a bit cumbersome to finalize the syntax for resolving extension methods, so perhaps it is more appropriate to adopt the "call-site resolution" approach with them:

```scala
object local {
  1.foo() // ambiguous
  IntExtensionsA(1).foo() // resolved
  2.asInstanceOf[IntExtensionsB].foo() // resolved
}

lens local includes a, b
```

#### Shadowing

Much like with name shadowing, it is possible to shadow included interpretations without causing a conflict:

```scala
lens external {
  typeinstance IntOrderingExternal implements Ordering[Int] { ... }
}

lens local includes external {
  // overrides / shadows the external `Ordering[Int]`
  typeinstance IntOrderingLocal implements Ordering[Int] { ... }
}
```

#### Including resolutions

Resolution of conflicting interpretations can be "inherited" by any `lens` that includes the resolving `lens`:

```scala
lens resolutions includes a, b {
  deduce typeclass Ordering[Int] = IntOrderingA
}

// includes both `a` and `b` with the deduction of `Ordering[Int]`
lens local includes resolutions
```

However, included resolutions do not resolve conflicts that are caused by interpretations from other included lenses:

```scala
lens c includes b {
  conversion ...
}

// compiler error - conflict
lens local includes resolutions, c
```

In this example, `Ordering[Int]` is resolved as `IntOrderingA` in `resolutions`, but defined as `IntOrderingB` in `c`.
