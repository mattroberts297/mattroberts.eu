---
layout: post
title:  "Composite pattern in Scala"
date:   2015-04-14 20:00:00
description: "Use the type system to create sensible inheritance hierarchies and push side effects to the edge of your application."
tags:
- scala
- patterns
---

The composite pattern is a structural design pattern. Its aim is to allow client code to treat individual objects and collections of objects uniformly (GoF, 1994). The example given in Design Patterns looks like so:

<img src="//assets.mattro.be/rts/img/gof-object-structural-composite-1.png" alt="GoF Transparent Structural Composite" class="img-responsive">

Unfortunately, the methods `add(Graphic)`, `remove(Graphic)` and `getChild(int)` defined by the interface `Graphic` make little sense for the classes `Text`, `Rectangle` and `Line`. As a result implementations must either do nothing (potentially hiding an error in client code) or throw an exception resulting in a run-time error (as done by the Java collections library). This can be resolved by moving the definition of these methods down the inheritance hierarchy. For example:

<img src="//assets.mattro.be/rts/img/gof-object-structural-composite-2.png" alt="GoF Safe Structural Composite" class="img-responsive">

Scala pattern matching means that there is no need for type casts if / when client code needs to distinguish between `Text`, `Rectangle`, `Line` and `Picture` instances. For example:

```scala
def render(graphic: Graphic): Unit = graphic match {
  case text: Text => ???
  case rectangle: Rectangle => ???
  case line: Line => ???
  case picture: Picture => ???
}
```

Further, sealing the base trait `Graphic` means that the compiler will warn you should you miss a case. For example:

```scala
sealed trait Graphic {
  def draw(): Unit
}

trait Picture extends Graphic {
  def add(graphic: Graphic): Unit
  def remove(graphic: Graphic): Unit
  def getChild(index: Int): Graphic
}

trait Line extends Graphic

trait Rectangle extends Graphic

trait Text extends Graphic
```

The return type `Unit` for `draw`, `add` and `remove` suggests the methods will have side effects. The first would likely call out to some graphics driver and the last two would probably mutate some private collection. This can be resolved by refactoring the definitions to defer side effects or avoid them all together. For example:

```scala
sealed trait Graphic {
  def draw(canvas: Canvas): Canvas
}

trait Picture extends Graphic {
  val graphics: Vector[Graphic]
  def withChild(graphic: Graphic): Picture
  def withoutChild(graphic: Graphic): Picture
}

trait Line extends Graphic

trait Rectangle extends Graphic

trait Text extends Graphic
```

This could be abstracted and improved upon further, but definitely seems like a step in the right direction. Assuming one is happy stopping here then the end result would look like so:

<img src="//assets.mattro.be/rts/img/gof-object-structural-composite-3.png" alt="GoF Safer Structural Composite" class="img-responsive">
