---
layout: post
title:  "Automatic type class derivation with shapeless - Part One"
date:   2017-04-16 10:00:00
summary: "Use the Scala compiler's implicit resolution rules to automatically derive type class instances"
tags:
- scala
- patterns
- shapeless
---

This post is part of a series. You might want to read [Part Two][part-two] and Part Three when you're done here.

I recently found myself needing to parse command line arguments in Scala. I discovered [scopt](https://github.com/scopt/scopt) and [scallop](https://github.com/scallop/scallop), but felt they required quite a lot of boiler plate. What I wanted was a library that would take a case class and automatically derive a parser for me at compile time. Ideally that library would parse \*nix style options e.g. `my-app --foo bar`, fall-back to defaults defined by the case class for missing options and return (not throw) an error when that failed. I made [claper](https://github.com/mattroberts297/claper) to do just this and decided to write about how I did it here.

I started by trying to solve for a simpler problem because I thought solving all of the above with my first try was a little ambitious to start out with. So Instead I tried to convert this:

```bash
my-app foo 1 true
```

Into this:

```scala
SimpleArguments(alpha = "foo", beta = 1, charlie = true)
```

Which is much easier because the parameters are in order and I don't need to know the field names of the case class. I also ignored the issue of defaults. **Don't panic!** I did solve for my actual use case and I'll cover how in parts two and three.

I decided to use [shapeless](https://github.com/milessabin/shapeless) because I find the Scala macros api clumsy and it is still considered experimental. The former is really a matter of personal taste, but the latter means as a library maintainer your code can be broken by new versions of Scala. This makes supporting old, current and future versions of Scala difficult! I speak from the first-hand experience of maintaining [slf4s](https://github.com/mattroberts297/slf4s) - where I maintain a branch per Scala release.

Here's what my `build.sbt` file looks like with shapeless and scalatest:

```scala
scalaVersion := "2.12.1"

libraryDependencies ++= Seq(
  "com.chuusai" %% "shapeless" % "2.3.2",
  "org.scalatest" %% "scalatest" % "3.0.1" % "test"
)
```

And here's my first attempt at a parser (I explain how it works below):

```scala
trait Parser[A] {
  def parse(args: List[String]): A
}

object Parser {
  import shapeless.Generic
  import shapeless.{HList, HNil, ::}
  import shapeless.Lazy

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

And a test that demonstrates the automatic type class derivation:

```scala
import org.scalatest.{MustMatchers, FlatSpec}

case class SimpleArguments(alpha: String, beta: Int, charlie: Boolean)

class ParserSpec extends FlatSpec with MustMatchers {
  "Parser::apply" must "derive a parser for SimpleArguments" in {
    val args = List("a", "1", "true")
    val parsed = Parser[SimpleArguments].parse(args) // Magic.
    parsed must be (SimpleArguments("a", 1, true))
  }
}
```

Now, in my opinion, the real "magic" is here (I explain the trick below):

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
  "Parser::apply" must "let us derive a parser for SimpleArguments" in {
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

You might be wondering why I haven't mentioned `Lazy` or why I use it. First, the easy answer: I haven't mentioned it because I think it hinders understanding and readability. As for why I use it, in short: it stops the compiler giving up it's search for implicits too early. In more detail: the Scala compiler uses heuristics to make sure that it doesn't get stuck in an infinite recursion when resolving implicits. For more complex data types, these heuristics tend to be too aggressive. The use of `Lazy` lets us workaround this issue.

The code for part one is [available on Github]( https://github.com/mattroberts297/automatic-type-class-derivation-part-one).

[Part Two][part-two]looks at `LabelledGeneric`.

Part three will look at `Default`.

[part-two]: https://mattroberts.io/posts/2017/04/18/automatic-type-class-derivation-with-shapeless-part-two/
