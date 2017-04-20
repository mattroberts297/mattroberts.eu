---
layout: post
title:  "Automatic type class derivation with shapeless - Part Three"
date:   2017-04-20 07:00:00
summary: "Use shapeless' Default to retrieve case class defaults at compile time"
tags:
- scala
- patterns
- shapeless
---

This post is part of a series. You might want to read [Part One][part-one] and [Part Two][part-two] first if you haven't already.

In [Part One][part-one] I explained that I wanted to fall back to case class defaults where possible. For example, given the following case class:

```scala
case class SimpleArguments(alpha: String = "alpha", beta: Int, charlie: Boolean)
```

And run-time arguments:

```bash
my-app --beta 1 --charlie
```

I would like the parser to return:

```scala
SimpleArguments(alpha = "alpha", beta = 1, charlie = true)
```

If you read [Part Two][part-two] then you might be thinking: maybe there is a `LabelledGenericWithDefaults` that we can use. Unfortunately, there isn't. That makes sense to me because they feel like separate concerns. There is however a `Default`. The 'trick' we're going to use here is to have an underlying parser that accepts two arguments (one being a `HList` of defaults represented as `Option`s) and then convert it to a parser that requires one argument at the last moment.

Let's start with the `UnderlyingParser` (I explain how it works below):

```scala
trait UnderlyingParser[A, B] {
  def parse(args: List[String], defaults: B): A
}

object UnderlyingParser {
  import shapeless.LabelledGeneric
  import shapeless.{HList, HNil, ::}
  import shapeless.Lazy
  import shapeless.Witness
  import shapeless.labelled.FieldType
  import shapeless.labelled.field
  import shapeless.Default

  def create[A, B](thunk: (List[String], B) => A): UnderlyingParser[A, B] = {
    new UnderlyingParser[A, B] {
      def parse(args: List[String], defaults: B): A = thunk(args, defaults)
    }
  }

  implicit def hlistParser[K <: Symbol, H, T <: HList, TD <: HList](
    implicit
    witness: Witness.Aux[K],
    hParser: Lazy[UnderlyingParser[FieldType[K, H], Option[H]]],
    tParser: UnderlyingParser[T, TD]
  ): UnderlyingParser[FieldType[K, H] :: T, Option[H] :: TD] = {
    create { (args, defaults) =>
      val hv = hParser.value.parse(args, defaults.head)
      val tv = tParser.parse(args, defaults.tail)
      hv :: tv
    }
  }

  implicit def stringParser[K <: Symbol](
    implicit
    witness: Witness.Aux[K]
  ): UnderlyingParser[FieldType[K, String], Option[String]] = {
    val name = witness.value.name
    create { (args, defaultArg) =>
      val providedArg = getArgFor(args, name)
      val arg = providedArg.getOrElse(
        defaultArg.getOrElse(
          // TODO: Use Either instead of throwing an exception.
          throw new IllegalArgumentException(s"Missing argument $name")
        )
      )
      field[K](arg)
    }
  }

  implicit def intParser[K <: Symbol](
    implicit
    witness: Witness.Aux[K]
  ): UnderlyingParser[FieldType[K, Int], Option[Int]] = {
    val name = witness.value.name
    create { (args, defaultArg) =>
      val providedArg = getArgFor(args, name).map(_.toInt)
      val arg = providedArg.getOrElse(
        defaultArg.getOrElse(
          // TODO: Use Either instead of throwing an exception.
          throw new IllegalArgumentException(s"Missing argument $name")
        )
      )
      field[K](arg)
    }
  }

  implicit def booleanParser[K <: Symbol](
    implicit
    witness: Witness.Aux[K]
  ): UnderlyingParser[FieldType[K, Boolean], Option[Boolean]] = {
    val name = witness.value.name
    create { (args, default) =>
      val arg = args.find(a => a == s"--$name").isDefined
      field[K](arg)
    }
  }

  implicit val hnilParser: UnderlyingParser[HNil, HNil] = {
    create { (_, _) => HNil }
  }

  private def getArgFor(args: List[String], name: String): Option[String] = {
    val indexOfName = args.indexOf(s"--$name")
    val indexAfterName = indexOfName + 1
    if (indexOfName > -1 && args.isDefinedAt(indexAfterName)) {
      Some(args(indexAfterName))
    } else {
      None
    }
  }
}
```

You'll notice there is no `apply` or `genericParser` method. This is because I decided that client code should not be able to instantiate an `UnderlyingParser`. It also means that there is only one particularly difficult method to wrap our heads around:

```scala
implicit def hlistParser[K <: Symbol, H, T <: HList, TD <: HList](
  implicit
  witness: Witness.Aux[K],
  hParser: Lazy[UnderlyingParser[FieldType[K, H], Option[H]]],
  tParser: UnderlyingParser[T, TD]
): UnderlyingParser[FieldType[K, H] :: T, Option[H] :: TD] = {
  create { (args, defaults) =>
    val hv = hParser.value.parse(args, defaults.head)
    val tv = tParser.parse(args, defaults.tail)
    hv :: tv
  }
}
```

Whilst it looks a little daunting the above `hlistParser` has only one extra type parameter when compared to the method in [Part Two][part-two]: `TD`. `TD` is used to represent the tail of the `HList` that contains default parameters. The `UnderlyingParser[FieldType[K, H], Option[H]]` type is also more complicated, but again there is just one extra parameter when compared to [Part Two][part-two]: `Option[H]`. This is the head of the `HList` that contains default parameters. You can see this again in the method return type: `UnderlyingParser[FieldType[K, H] :: T, Option[H] :: TD]`. If I were to read this allowed I would say:

> The `hlistParser` method returns a generic `UnderlyingParser` where both type parameters are `HList`s. The head of the first `HList` is a generic `FieldType` and the tail is some `T` that is also a `HList`. The head of the second `HList` is a generic `Option` and the tail is some `TD` that is also a `HList`.

We, as humans, implicitly know that the tail of each `HList` will have to contain field types and options. If we wanted to we could make use of shapeless' `HList` constraints to codify this in the method:

```scala
import shapeless.UnaryTCConstraint._
import shapeless.LUBConstraint._

implicit def hlistParser[
  K <: Symbol,
  H,
  T <: HList : <<:[FieldType[_, _]]#λ,
  TD <: HList : *->*[Option]#λ
](
  implicit
  witness: Witness.Aux[K],
  hParser: Lazy[UnderlyingParser[FieldType[K, H], Option[H]]],
  tParser: UnderlyingParser[T, TD]
): UnderlyingParser[FieldType[K, H] :: T, Option[H] :: TD] = {
  create { (args, defaults) =>
    val hv = hParser.value.parse(args, defaults.head)
    val tv = tParser.parse(args, defaults.tail)
    hv :: tv
  }
}
```

The above explicitly states that `T` must be a `HList` containing `FieldType`s and `TD` must be a `HList` of `Option`. This is good because it explicitly states what we know to be implicitly the case. The downside is we have to write more code and a lot of people find it hard to read. When I wrote Claper I didn't know about `HList` constraints and didn't use them, but hopefully they'll be useful to someone.

On an aside, shapeless also has a `KeyConstraint` that, from the documentation, sounds like it might have been useful:

> Type class witnessing that every element of `L` is of the form `FieldType[K, V]` where `K` is an element of `M`.

I'm afraid you'll have to have a dig around the [shapeless](https://github.com/milessabin/shapeless) code (there are tests!) if you want to find out more abut that though.

Back to parsing, so we have our `UnderlyingParser` and we're reasonably confident that the Scala compiler will be able to complete it's derivation, so let's make use of it:

```scala
trait DefaultedParser[A] {
  def parse(args: List[String]): A
}

object DefaultedParser {
  import shapeless.LabelledGeneric
  import shapeless.{HList, HNil, ::}
  import shapeless.Lazy
  import shapeless.Witness
  import shapeless.labelled.FieldType
  import shapeless.labelled.field
  import shapeless.Default

  def apply[A](
    implicit
    st: Lazy[DefaultedParser[A]]
  ): DefaultedParser[A] = st.value

  def create[A](thunk: List[String] => A): DefaultedParser[A] = {
    new DefaultedParser[A] {
      def parse(args: List[String]): A = thunk(args)
    }
  }

  implicit def genericParser[A, R <: HList, D <: HList](
    implicit
    defaults: Default.AsOptions.Aux[A, D],
    generic: LabelledGeneric.Aux[A, R],
    parser: Lazy[UnderlyingParser[R, D]]
  ): DefaultedParser[A] = {
    create { args => generic.from(parser.value.parse(args, defaults())) }
  }
}
```

The interesting method here is `genericParser` it states. `Default.AsOptions.Aux[A, D]` is provided by the `shapeless.Default` import (and a macro in shapeless). `LabelledGeneric.Aux[A, R]` is provided by `shapeless.LabelledGeneric` import (and another macro). `Lazy[UnderlyingParser[R, D]]` is provided `hlistParser` in the `UnderlyingParser` object (and some hard work by the compiler). The `args` and the applied `defaults` are then passed to the `UnderlyingParser`'s `parse` method which returns a `HList` from which we can construct an `A` using generic.

Here's a test showing that it all works:

```scala
import org.scalatest.{MustMatchers, FlatSpec}

case class SimpleArguments(alpha: String = "alpha", beta: Int, charlie: Boolean)

class DefaultedParserSpec extends FlatSpec with MustMatchers {
  "DefaultedParser::apply" must "derive a parser for SimpleArguments" in {
    val args = List("--beta", "1", "--charlie")
    val parsed = DefaultedParser[SimpleArguments].parse(args)
    parsed must be (SimpleArguments("alpha", 1, true))
  }
}
```

If you find yourself suffering from type blindness (I sometimes do) then I recommend you read [Part One][part-one] and [Part Two][part-two] and / or diff the code locally in your favourite text editor.

The code for Part Three is [available on Github](https://github.com/mattroberts297/automatic-type-class-derivation-part-three).

[part-one]: https://mattroberts.io/posts/2017/04/16/automatic-type-class-derivation-with-shapeless-part-one/

[part-two]: https://mattroberts.io/posts/2017/04/18/automatic-type-class-derivation-with-shapeless-part-two/
