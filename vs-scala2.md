# Comparison with Scala 2 implicits

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [Modifications](#modifications)
  - [Declaring implicit interpretations](#declaring-implicit-interpretations)
  - [Importing implicit interpretations](#importing-implicit-interpretations)
- [Added capabilities](#added-capabilities)
  - [Local resolutions](#local-resolutions)
  - [Anonymous implicit parameters](#anonymous-implicit-parameters)
- [Removed capabilities](#removed-capabilities)
  - [Named implicit values](#named-implicit-values)
  - [Importing implicit values](#importing-implicit-values)
  - [Context bounds](#context-bounds)

<!-- /code_chunk_output -->

## Modifications

### Declaring implicit interpretations

Implicit interpretations -- type classes, extensions and conversions -- are modeled in Scala 2 with existing definition constructs prefixed by `implicit`:

```scala
// conversion
implicit def intToString(i: Int): String = i.toString()
// extension
implicit class IntExtension(i: Int) {
  def plusOne(): Int = i + 1
}
// type class
trait Ordering[A] {
  def compare(x: A, y: A): Int
}
implicit object IntOrdering extends Ordering[A] {
  def compare(x: Int, y: Int): Int = ???
}
implicit def optionOrdering[A: Ordering] = new Ordering[Option[A]] {
  def compare(x: Option[A], y: Option[A]): Int = ???
}
```

In this proposal, instead of using the same prefix construct with other existing definition constructs, each form of implicit interpretation is given unique definition constructs:

```scala
// conversion
conversion intToString(i: Int): String = i.toString()
// extension
extension IntExtension extends Int {
  def plusOne(): Int = this + 1
}
// type class
typeclass Ordering[A] {
  def compare(x: A, y: A): Int
}
typeinstance IntOrdering implements Ordering[Int] {
  def compare(x: Int, y: Int): Int = ???
}
typeinstance OptionOrdering[A]<Ordering[A]> implements Ordering[Option[A]] {
  def compare(x: Option[A], y: Option[A]): Int = ???
}
```

Note that this not a mere change of syntax. Implicit interpretations are no longer viewed as objects or instances in the OOP sense and cannot be instantiated or invoked in the way other objects do; instead, they are declared inside `lens` components (and there only) and so they become "implicitly" available in a scope.

### Importing implicit interpretations

`import`ing implicit interpretations in Scala 2 makes them available in the current scope:

```scala
object external {
  implicit object IntOrdering extends Ordering[Int] { ... }
}

object local {
  def sort[A](xs: Seq[A])(implicit ord: Ordering[A]): Seq[A] = ???
  import external._
  sort(Seq(1, 2, 3)) // `IntOrdering` is resolved as the implicit parameter
}
```

In this proposal, `import`ing interpretations is not enough; one has to include them in the local `lens` (see [lens inclusion](lens.md#including-lens)):

```scala
object external {
  typeinstance IntOrdering implements Ordering[Int] { ... }
}

import external // not enough

object local {
  def sort[A]<Ordering[A]>(xs: Seq[A]): Seq[A] = ???
  sort(Seq(1, 2, 3))
}
lens local includes external
```

## Added capabilities

### Local resolutions

Resolution of conflicting implicit interpretations is possible via [lenses](lens.md#resolving-conflicts), while in Scala 2 it is not possible at all.

### Anonymous implicit parameters

It is possible to declare [anonymous implicit parameters](dependency-injection.md#anonymous-parameters) in this proposal.

In Scala 2, this is possible only in a very partial manner via context bounds, which allow for anonymous parameters only for types with a single type-parameter (type class), but not for non-generic types or generic types with more than one type-parameter.

## Removed capabilities

### Named implicit values

In Scala 2, it's possible to declare a named implicit values:

```scala
implicit val ec = ExecutionContext.global
```

In this proposal, this is not possible; instead, every value is implied anonymously:

```scala
imply ExecutionContext.global
```

Implicit interpretations remain named in this proposal.

### Importing implicit values

In Scala 2, it's possible to import implicit values, making them "implied" in the importing scope as well:

```scala
object external {
  implicit val ec = ExecutionContext.global
}

object local {
  import external._
  Future { ... } // `ec` is resolved as the implicit parameter
}
```

In this proposal, however, this is impossible:

```scala
object external {
  imply ExecutionContext.global
}

object local {
  import external._
  Future { ... } // error, no implicit parameter found
}
```

Implicit interpretations can still be "imported" [in this proposal](#importing-implicit-interpretations).

### Context bounds

In Scala 2, context bounds are a short for an anonymous (single) type-parameter implicit parameter:

```scala
def sort[A: Ordering](xs: Seq[A]): Seq[A] = ???
// is equivalent to
def sort[A](xs: Seq[A])(implicit ord: Ordering[A]): Seq[A] = ???
```

There is no such capability in this proposal for implicit (`implied`) parameters.

Instead, this proposal introduces [bounds for type classes](type-classes.md#bounds) -- which are not modeled as implicit parameters -- and can also support type classes with multiple type parameters:

```scala
def sort[A]<Ordering[A]>(xs: Seq[A]): Seq[A] = ???
def contains[A]<Eql[A, A]>(set: Set[A], elem: A): Boolean = ???
```
