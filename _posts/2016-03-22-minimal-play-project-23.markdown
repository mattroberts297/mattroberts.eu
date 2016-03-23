---
layout: post
title:  "Minimal Play 2.3 Project"
date:   2016-03-21 18:32:00 +0000
tags:
- play
- play 2.3
- bootstrap
---

Get started with the Play Framework 2.3 with just 4 files and no Typesafe Activator:

```bash
.
├── app
│   └── controllers
│       └── Application.scala
├── build.sbt
├── conf
│   └── routes
├── project
│   └── plugins.sbt
└── test
```

Start with `plugins.sbt`:

```scala
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.3.9")
```

Then `build.sbt`:

```scala
name := "play-app"

version := "1.0.0-SNAPSHOT"

lazy val root = (project in file(".")).enablePlugins(PlayScala)
```

Then `Application`.scala:

```scala
package controllers

import play.api.mvc._

object Application extends Controller {
  def index = Action {
    Ok("It works!")
  }
}
```

And finally the `routes` file:

```scala
GET  /  controllers.Application.index
```

At version 2.3.x, the `PlayScala` plugin still includes quite a few dependencies that you might not actually want. Including, but not limited to, specs2 (I prefer ScalaTest).

Oh and whilst completely optional, here is my `.gitignore` file:

```
logs
project/project
project/target
target
tmp
dist
.cache
```
