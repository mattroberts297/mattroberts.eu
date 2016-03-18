---
layout: post
title:  "Adapter pattern in Scala"
date:   2015-04-16 20:00:00
summary: "Use implicit classes to make API interoperability more pleasurable in client code."
tags:
- scala
- pattern
---
 The adapter pattern is a structural design pattern. Its aim is to convert the interface of a class into another interface that client code expects (GoF, 1994). The example given in Design Patterns looks similar to this:

 <img src="//assets.mattro.be/rts/img/gof-object-structural-adapter-1.png" alt="GoF Structural Adapter" class="img-responsive">

In this example the class `TextShape` is the adapter because it adapts `TextView` to the `Shape` interface. In Java, client code would have to wrap TextView instances manually. For example:

```java
Shape shape = new TextShape(new TextView());
```

In Scala, however, `TextShape` may be implemented using an implicit class. Then, provided it is in scope any `TextView` instance can be treated as a `TextShape`. For example:

```scala
object ApiA {
  case class Extent(bottomLeft: (Int, Int), topRight: (Int, Int))

  class TextView {
    def extent: Extent = ???
  }
}

object ApiB {
  case class BoundingBox(bottomLeft: (Int, Int), topRight: (Int, Int))

  trait Shape {
    def boundingBox: BoundingBox
  }

  def invert(shape: Shape): Shape = ???
}

object Adapter {
  import ApiB._
  import ApiA._

  implicit class TextShape(val textView: TextView) extends Shape {
    def boundingBox: BoundingBox = new BoundingBox(
      textView.extent.bottomLeft,
      textView.extent.topRight)
  }
}

object Usage {
  import ApiA._
  import ApiB._
  import Adapter._

  def callsInvert(textView: TextView) = {
    invert(textView) // This will compile.
  }
}
```

The Scala compiler checks for implicits at compile-time, so if you are missing one you will know sooner rather than later. For example:

```bash
[error] Adapter.scala:38: type mismatch;
[error]  found   : ApiA.TextView
[error]  required: ApiB.Shape
[error]     takesShape(textView) // This will compile.
[error]                ^
[error] one error found
[error] (compile:compile) Compilation failed
[error] Total time: 0 s, completed 16-Apr-2015 20:58:06
```

If you are responsible for writing `ApiB` then you should consider using type classes.
