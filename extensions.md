# Extensions

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [Rationale](#rationale)
- [Syntax and semantics](#syntax-and-semantics)
  - [Basic extensions](#basic-extensions)
  - [Generic extensions](#generic-extensions)

<!-- /code_chunk_output -->

## Rationale

Despite not being in the early days of Scala, I understand that implicit parameters were introduced to the language as an abstraction of extension methods (and conversions). However, extension methods really have nothing to do with implicit parameters, and should be completely separated from that concept.

Extension methods are an alternative to extending types via inheritance. They are defined inside a component -- an extension -- which is similar to a class but has its own unique semantics; the extension is never explicitly instantiated, but is rather associated with an instance of a type when instructed (it's not an inherent part of that instance).

Extensions can be mistaken for type classes -- which may also play as an alternative to inheritance -- of a specific form where the type class has only one type parameter and only infix methods; however, extensions and type classes differ in two characteristics:

1. Extensions directly extend types, while type classes need both an interface and an implementation (type instance).
2. Extensions work on concrete instances, while type classes work only on bound generic types.

## Syntax and semantics

### Basic extensions

Extensions, like any other implicit interpretation, are declared and are "imported" via [lenses](lens.md):

```scala
object MyExtensions {
  1.prettyToString()
}

lens MyExtensions {
  extension Pretty extends Any {
    def prettyToString() = s"Pretty: ${this.toString()}"
  }
}
```

As seen in the example, `this` in an `extension` refers to the extended instance.

### Generic extensions

Extensions, just like `implicit class`es before them, can be generic and / or extend `trait`s:

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

Being able to be generic means that extensions may also require type classes via [bounds](type-classes.md#bounds):

```scala
class Painter {
  def paint[A]<Palette[A]>(xs: Seq[A]): Painting = {
    val darks = xs.darkOnly()
    ???
  }
}

object Painter {
  trait Color {
    def isDark(): Boolean
  }

  typeclass Palette[A] {
    def colorOf(obj: A): Color
  }
}

lens Painter {
  extension SeqOps[A]<Palette[A]> extends Seq[A] {
    def darkOnly(): Seq[A] = this.filter(x => Palette.colorOf(x).isDark())
  }
}
```
