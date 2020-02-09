# Extensions

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [Rationale](#rationale)
- [Syntax and semantics](#syntax-and-semantics)
  - [Basic extensions](#basic-extensions)
  - [Generic extensions](#generic-extensions)
  - [With type classes](#with-type-classes)

<!-- /code_chunk_output -->

## Rationale

Despite not being in the early days of Scala, I understand that implicit parameters were introduced to the language as an abstraction of extension methods (and conversions). However, extension methods really have nothing to do with implicit parameters, and should be completely separated from that concept.

Extension methods are an alternative to extending types via inheritance. They are defined inside a component -- an extension -- which is similar to a class but has its own unique semantics; the extension is never explicitly instantiated, but is rather associated with an instance of a type when instructed (it's not an inherent part of that instance).

Extensions can be mistaken for type classes -- which may also play as an alternative to inheritance -- of a specific form where the type class has only one type parameter and only infix methods; however, extensions are different in that they do not need an interface, while type classes do.

With type classes, the interface component is essential for declaring where the type class behavior is _required_. Extensions, on the other hand, add behavior that is not intended to be required, but rather to assist when possible; they can be seen as syntactic sugar for infix "static" utility operations.

This proposal doesn't change much in the way extensions behave; it simply attempts at differentiating it from implicits via a new distinct syntax.

## Syntax and semantics

### Basic extensions

Extensions, like any other implicit interpretation, are declared and are "imported" via [lenses](lens.md):

```scala
object MyApp {
  1.prettyToString()
}

lens MyApp {
  extension Pretty extends Any {
    def prettyToString() = s"Pretty: ${this.toString()}"
  }
}
```

As seen in the example, `this` in an `extension` refers to the extended instance.

### Generic extensions

Extensions, just like `implicit class`es before them, can be generic:

```scala
object MyApp {
}

lens MyApp {
  extension OptionsOps[A] extends A {
    def some: Option[A] = Some(this)
  }
}
```

And they can also extend `trait`s:

```scala
trait RandomSelection[A] {
  def random(): A
}

object MyApp {
  val pokemons = Seq("Charmander", "Balbazor", "Squirtle")
  val startingPokemon = pokemons.random()
}

lens MyApp {
  extension SeqOps[A] extends Seq[A] with RandomSelection[A] {
    override def random(): A = ???
  }
}
```

### With type classes

Being able to be generic means that extensions may also require type classes via [bounds](type-classes.md#bounds):

```scala
case class Foo(bar: Int)

object MyApp {
  assert(Foo(2) > Foo(1))
  assert(Foo(1) < Foo(2))
  assert(Foo(1) == Foo(1))
}

lens MyApp {
  extension OrderingOps[A]<Ordering[A]> extends A {
    def >(other: A): Boolean = Ordering.compare(this, other) > 0
    def <(other: A): Boolean = Ordering.compare(this, other) < 0
    def ==(other: A): Boolean = Ordering.compare(this, other) == 0
  }

  typeinstance FooOrdering implements Ordering[Foo] { ... }
}
```
