# Type Conversions

Type conversions provide a way to treat instances of a certain type as if they were another, by declaring that transformation only once.

Type conversions, like any other implicit interpretation, are declared and are "imported" via [lenses](lens.md):

```scala
object MyApp {
  val s: String = 1
  def print(s: String) = ???
  print(2)
  3.toLowerCase()
}

lens MyApp {
  conversion intToString(i: Int): String = i.toString()
}
```

## Chaining conversions

This proposal is undecided whether chaining conversions should be allowed and leaves this open to discussion:

object MyApp {
  val s: String = 1.5
}

lens MyApp {
  conversion doubleToInt(d: Double): Int = d.toInt()
  conversion intToString(i: Int): String = i.toString()
}

## Type classes

Conversions _cannot_ be encoded using [type classes](type-classes.md):

```scala
typeclass Conversion[A, B] {
  def apply(a: A): B
}

object MyApp {
  val s: String = 1 // error
}

lens MyApp {
  typeinstance IntToString implements Conversion[Int, String] {
    override def apply(i: Int): String = i.toString()
  }
}
```

There are two reasons why this doesn't (and shouldn't) work:

1. Conversions work on concrete instances, while type classes work only on bound generic types.
2. Conversions work without invoking any explicit function. Making `Conversion` a special type class with unique semantics is tantamount to implementing mechanics in the language via unique traits, such as `DelayedInit`.
