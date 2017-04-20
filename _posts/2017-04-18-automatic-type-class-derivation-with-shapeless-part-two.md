---
layout: post
title:  "Automatic type class derivation with shapeless - Part Two"
date:   2017-04-18 19:00:00
summary: "Use shapeless' LabelledGeneric to retrieve case class field names at compile time"
tags:
- scala
- patterns
- shapeless
---

This post is part of a series. You might want to read [Part One][part-one] first if you haven't already. And when you're done here you might want to read [Part Three][part-three].

In [Part One][part-one] I explained that I wanted a library that would parse \*nix style options:

```bash
my-app --alpha foo --beta 1 --charlie
```

Into this:

```scala
SimpleArguments(alpha = "foo", beta = 1, charlie = true)
```

I also stated that I wanted to fall-back to case class defaults, but let's ignore that requirement for now. **Don't panic!** I will solve that in [Part Three][part-three]. As before, here's my `build.sbt` file with shapeless and scalatest:

```scala
scalaVersion := "2.12.1"

libraryDependencies ++= Seq(
  "com.chuusai" %% "shapeless" % "2.3.2",
  "org.scalatest" %% "scalatest" % "3.0.1" % "test"
)
```

And here's my second attempt at a parser (I explain how it works below):

```scala
trait LabelledParser[A] {
  def parse(args: List[String]): A
}

object LabelledParser {
  import shapeless.LabelledGeneric
  import shapeless.{HList, HNil, ::}
  import shapeless.Lazy
  import shapeless.Witness
  import shapeless.labelled.FieldType
  import shapeless.labelled.field

  private def create[A](thunk: List[String] => A): LabelledParser[A] = {
    new LabelledParser[A] {
      def parse(args: List[String]): A = thunk(args)
    }
  }

  def apply[A](
    implicit
    st: Lazy[LabelledParser[A]]
  ): LabelledParser[A] = st.value

  implicit def genericParser[A, R <: HList](
    implicit
    generic: LabelledGeneric.Aux[A, R],
    parser: Lazy[LabelledParser[R]]
  ): LabelledParser[A] = {
    create(args => generic.from(parser.value.parse(args)))
  }

  implicit def hlistParser[K <: Symbol, H, T <: HList](
    implicit
    hParser: Lazy[LabelledParser[FieldType[K, H]]],
    tParser: LabelledParser[T]
  ): LabelledParser[FieldType[K, H] :: T] = {
    create { args =>
      val hv = hParser.value.parse(args)
      val tv = tParser.parse(args)
      hv :: tv
    }
  }

  implicit def stringParser[K <: Symbol](
    implicit
    witness: Witness.Aux[K]
  ): LabelledParser[FieldType[K, String]] = {
    val name = witness.value.name
    create { args =>
      val arg = args.dropWhile(a => a != s"--$name").tail.head
      field[K](arg)
    }
  }

  implicit def intParser[K <: Symbol](
    implicit
    witness: Witness.Aux[K]
  ): LabelledParser[FieldType[K, Int]] = {
    val name = witness.value.name
    create { args =>
      val arg = args.dropWhile(a => a != s"--$name").tail.head.toInt
      field[K](arg)
    }
  }

  implicit def booleanParser[K <: Symbol](
    implicit
    witness: Witness.Aux[K]
  ): LabelledParser[FieldType[K, Boolean]] = {
    val name = witness.value.name
    create { args =>
      val arg = args.find(a => a == s"--$name").isDefined
      field[K](arg)
    }
  }

  implicit val hnilParser: LabelledParser[HNil] = {
    create(args => HNil)
  }
}
```

And a test that demonstrates the automatic type class derivation:

```scala
import org.scalatest.{MustMatchers, FlatSpec}

case class SimpleArguments(alpha: String, beta: Int, charlie: Boolean)

class LabelledParserSpec extends FlatSpec with MustMatchers {
  "LabelledParser::apply" must "derive a parser for SimpleArguments" in {
    val args = List("--alpha", "a", "--beta", "1", "--charlie")
    val parsed = LabelledParser[SimpleArguments].parse(args)
    parsed must be (SimpleArguments("a", 1, true))
  }
}

```

Let's start with the same methods as before, namely `apply`, `genericParser` and `hlistParser`:

```scala
def apply[A](
  implicit
  st: Lazy[LabelledParser[A]]
): LabelledParser[A] = st.value

implicit def genericParser[A, R <: HList](
  implicit
  generic: LabelledGeneric.Aux[A, R],
  parser: Lazy[LabelledParser[R]]
): LabelledParser[A] = {
  create(args => generic.from(parser.value.parse(args)))
}

implicit def hlistParser[K <: Symbol, H, T <: HList](
  implicit
  hParser: Lazy[LabelledParser[FieldType[K, H]]],
  tParser: LabelledParser[T]
): LabelledParser[FieldType[K, H] :: T] = {
  create(args => hParser.value.parse(args) :: tParser.parse(args))
}
```

The `apply` method here looks very similar to the `apply` method defined in [Part One][part-one] except we're asking the compiler to look for a `LabelledParser` for some `A` now. Things start to look a little different when we examine the implicit parameters for `genericParser`. In particular, the compiler is asked to find a `LabelledGeneric.Aux[A, R]` instead of a `Generic.Aux[A, R]`. Other than that, however, the method declaration and definition look reasonably similar. What is the difference between `Generic` and `LabelledGeneric`? From the documentation:

> `LabelledGeneric` is similar to `Generic`, but includes information about field names or class names in addition to the raw structure.

From that we can guess that the `HList` `R` in the `genericParser` method here is not the same as in [Part One][part-one]. This is because it must include information about field names in addition to the raw structure. We can check this intuition by looking at the return type of `hlistParser` which, as expected, states that the `HList` is made up of `FieldType`s.

Let's compare the above definition of `hlistParser` with the one from [Part One][part-one]. First, the type parameters `H` and `T` are the same as in [Part One][part-one] (except the latter will be a `HList` of `FieldType`s). `K` is new and used to state that the field name of the `FieldType` must be a `Symbol`. Second, the Scala compiler now needs to find a head parser for `FieldType[K, H]` and a tail parser for `T`. This tells us that the definitions for primitive types e.g. `String`, `Int`, `Boolean` must also be different form their [Part One][part-one] counterparts (and if you read the full code you will see they are). Third, and finally, the tail parser now receives the same `args` as the head parser (as opposed to just the tail). This makes sense because the arguments could now be in any order.

Lets look at one of the new primitive type definitions. Here is the `String` parser definition:

```scala
implicit def stringParser[K <: Symbol](
  implicit
  witness: Witness.Aux[K]
): LabelledParser[FieldType[K, String]] = {
  val name = witness.value.name
  create { args =>
    val arg = args.dropWhile(a => a != s"--$name").tail.head
    field[K](arg)
  }
}
```

The `Witness` is used to obtain the field name. The documentation for `LabelledGeneric` explains that a `Witness` is required, but I'm still not sure why Miles decided to separate them out (I'm sure there is a good reason):

> Note that the representation does not include the labels! The labels are actually encoded in the generic type representation using `Witness` types.

In comparison, the `field` method is relatively straight forward and lets us return a `FieldType[K, String]` instead of a `String`. This pattern can and is (in the code above) repeated for each primitive type allowing the compiler to complete it's derivation.

If you find yourself struggling with the type definitions then I recommend you read [Part One][part-one] and / or diff the code locally in your favourite text editor. In [Part Three][part-three] I show how to use `Default` to retrieve case class default values at compile time.

The code for Part Two is [available on Github](https://github.com/mattroberts297/automatic-type-class-derivation-part-two).

[part-one]: https://mattroberts.io/posts/2017/04/16/automatic-type-class-derivation-with-shapeless-part-one/

[part-three]: https://mattroberts.io/posts/2017/04/20/automatic-type-class-derivation-with-shapeless-part-three/
