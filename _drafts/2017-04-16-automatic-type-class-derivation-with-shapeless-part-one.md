---
layout: post
title:  "Automatic type class derivation with shapeless (1/3)"
date:   2017-04-16 10:00:00
summary: "Use the Scala compiler's implicit resolution rules to automatically derive type class instances"
tags:
- scala
- patterns
- shapeless
---

I recently found myself needing to parse command line arguments in Scala. I discovered [scopt](https://github.com/scopt/scopt) and [scallop](https://github.com/scallop/scallop), but felt they required quite a lot of boiler plate. What I wanted was a library that would take a case class and automatically derive a parser for me at compile time. Ideally that library would support *nix style options e.g. `my-app --foo bar` falling-back to any defaults defined by the case class and erroring when that failed. I made [claper](https://github.com/mattroberts297/claper) to do just this and decided to write about how I did it here.

I started by trying to solve for a simpler problem because I thought converting this:

```bash
my-app --charlie --beta 1 --alpha foo
```

Into this:

```
case class SimpleArguments(alpha: String beta: Int, charlie: Boolean)
SimpleArguments(alpha = "foo", beta = 1, charlie = true)
```

Was a little ambitious to start out with. So Instead I tried to convert this:

```bash
my-app foo 1 true
```

Which is much easier because the parameters are in order and I don't need to know the field names of the case class. I also ignored the issue of defaults.

I decided to use [shapeless](https://github.com/milessabin/shapeless) because I find the Scala macros api clumsy and it is still considered experimental which means as a library maintainer your code can get broken easily by new versions. I speak from the first-hand experience of maintaining [slf4s](https://github.com/mattroberts297/slf4s) - where I maintain a branch per Scala release.

Here's what my `build.sbt` file looks like:

```scala
scalaVersion := "2.12.1"

libraryDependencies ++= Seq(
  "com.chuusai" %% "shapeless" % "2.3.2",
  "org.scalatest" %% "scalatest" % "3.0.1" % "test"
)
```

And here's the first attempt at a parser:

```scala
trait Parser[A] {
  def parse(args: List[String]): A
}

object Parser {
  import shapeless.Generic
  import shapeless.Lazy
  import shapeless.{HList, HNil, ::}

  private def create[A](thunk: List[String] => A): Parser[A] = {
    new Parser[A] {
      def parse(args: List[String]): A = thunk(args)
    }
  }

  def apply[A](
    implicit
    st: Lazy[Parser[A]]
  ): Parser[A] = st.value

  implicit def genericParser[A, R <: HList](
    implicit
    generic: Generic.Aux[A, R],
    parser: Lazy[Parser[R]]
  ): Parser[A] = {
    create(args => generic.from(parser.value.parse(args)))
  }

  implicit def hlistParser[H, T <: HList](
    implicit
    hParser: Lazy[Parser[H]],
    tParser: Parser[T]
  ): Parser[H :: T] = {
    create(args => hParser.value.parse(args) :: tParser.parse(args.tail))
  }

  implicit val stringParser: Parser[String] = {
    create(args => args.head)
  }

  implicit val intParser: Parser[Int] = {
    create(args => args.head.toInt)
  }

  implicit val boolParser: Parser[Boolean] = {
    create(args => args.head.toBoolean)
  }

  implicit val hnilParser: Parser[HNil] = {
    create(args => HNil)
  }
}
```

And the test:

```scala
import org.scalatest.{MustMatchers, FlatSpec}

case class SimpleArguments(alpha: String beta: Int, charlie: Boolean)

class ParserSpec extends FlatSpec with MustMatchers {
  "A Parser" must "parse SimpleArguments" in {
    val args = List("a", "1", "true")
    val parsed = Parser[SimpleArguments].parse(args)
    parsed must be (SimpleArguments("a", 1, true))
  }
}
```

Now in my opinion, the real "magic" is here:

```scala
def apply[A](
  implicit
  st: Lazy[Parser[A]]
): Parser[A] = st.value

implicit def genericParser[A, R <: HList](
  implicit
  generic: Generic.Aux[A, R],
  parser: Lazy[Parser[R]]
): Parser[A] = {
  create(args => generic.from(parser.value.parse(args)))
}

implicit def hlistParser[H, T <: HList](
  implicit
  hParser: Lazy[Parser[H]],
  tParser: Parser[T]
): Parser[H :: T] = {
  create(args => hParser.value.parse(args) :: tParser.parse(args.tail))
}
```

The `apply` method causes the Scala compiler to search for an implicit `Parser[A]`. It finds the `genericParser` method and that in turn causes it to look for an implicit `Generic.Aux[A, R]` and `Parser[R]`. Where `R` is a `HList` that `Generic` can create an `A` from. It finds a `Generic.Aux[A, R]` thanks to the `shapeless.Generic` import (and a macro in the shapeless library). The `Parser[R]` requirement, as you may have guessed, is satisfied by the `hlistParser` method. The implicit `Parser[H]` is satisfied by the, rather mundane, implicit values that handle primitive types and the `Parser[T]` is handled recursively until the terminal `HNil` case is reached.

For the `SimpleArguments` example the compiler automatically derives the following for us:

```scala
import org.scalatest.{MustMatchers, FlatSpec}

class ParserSpec extends FlatSpec with MustMatchers {
  "A Parser" must "parse SimpleArguments" in {
    val args = List("a", "1", "true")
    val parsed = Parser[SimpleArguments].parse(args)
    parsed must be (SimpleArguments("a", 1, true))
  }

  it must "derive this automatically" in {
    import Parser._
    import shapeless.Generic
    import shapeless.Lazy
    val args = List("a", "1", "true")
    val parser = Parser[SimpleArguments](
      genericParser(
        Generic[SimpleArguments],
        Lazy(hlistParser(
          Lazy(stringParser),
          hlistParser(
            Lazy(intParser),
            hlistParser(
              Lazy(boolParser),
              hnilParser
            )
          )
        ))
      )
    )
    val parsed = parser.parse(args)
    parsed must be (SimpleArguments("a", 1, true))
  }
}
```

You might be wondering why I use `Lazy` and why I haven't mentioned it. I haven't mentioned it because I think it hinders understanding and readability. It is, however, import to because it stops the compiler giving up it's search when dealing with more complicated derivations.

Part 2/3 will look at `LabelledGeneric`.
Part 3/3 will look at `Default`.
