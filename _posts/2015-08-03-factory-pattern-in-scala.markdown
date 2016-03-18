---
layout: post
title:  "Factory pattern in Scala"
date:   2015-08-03 20:30:00
summary: "Decouple your client code from constructors, unit test actors and materialize type classes."
tags:
- scala
- pattern
---

The factory pattern provides an interface to create objects without directly depending on concrete classes:

```scala
trait Statement

case class SelectStatement(id: Int) extends Statement

trait Result

trait Database {
  def execute(statement: Statement): Result
}

class Postgres extends Database {
  def execute(statement: Statement): Result = ???
}

class H2 extends Database {
  def execute(statement: Statement): Result = ???
}

trait DatabaseFactory {
  def create: Database
}

object PostgresFactory extends DatabaseFactory {
  def create: Database = new Postgres
}

object H2Factory extends DatabaseFactory {
  def create: Database = new H2
}
```

With the above it is possible to write the following client code:

```scala
class UserRepository(databaseFactory: DatabaseFactory) {
  val database = databaseFactory.create
  def read(id: Int): Result = database.execute(SelectStatement(id))
}
```

The client code depends on interfaces instead of concrete classes:

<img src="//assets.mattro.be/rts/img/scala-object-creational-factory.png" alt="Factory class diagram" class="img-responsive">

This keeps library code and client code loosely coupled meaning the library maintain can rename Postgres or H2 without breaking client code.

In addition, because client code no longer has to invoke the class constructor with the new keyword it is easier to unit test:

```scala
class UserRepositorySpec extends WordSpec with Matchers {
  "A UserRepository" when {
    "constructed" should {
      "invoke DatabaseFactory::create" in new MockContext {
        when(mockFactory.create).thenReturn(mockDatabase)
        val repo = new UserRepository(mockFactory)
        verify(mockFactory).create
      }
    }

    "read is invoked" should {
      "invoke Database::execute" in new MockContext {
        when(mockFactory.create).thenReturn(mockDatabase)
        val repo = new UserRepository(mockFactory)
        repo.read(1)
        verify(mockDatabase).execute(SelectStatement(1))
      }
    }
  }

  trait MockContext extends MockitoSugar {
    val mockDatabase = mock[Database]
    val mockFactory = mock[DatabaseFactory]
  }
}
```

This is especially true when attempting to unit test Akka Actors whilst maintaining a sensible supervision hierarchy:

```scala
trait ActorFactory {
  def createBar(factory: ActorRefFactory): ActorRef
  def createBaz(factory: ActorRefFactory): ActorRef
}

class FooActor(factory: ActorFactory) extends Actor {
  val bar = factory.createBar(context)
  val baz = factory.createBaz(context)

  def receive: Receive = {
    case string: String => bar ! string
    case int: Int => baz ! int
  }
}
```

The above code can be tested by passing a mock ActorFactory to FooActor that returns test probes:

```scala
class FooActorSpec extends AkkaSpec {
  "The FooActor" when {
    "sent a message of type String" should {
      "send it to bar" in new MockContext with Behaviour {
        val fooActor = TestActorRef[FooActor](
          Props(classOf[FooActor], mockActorFactory))
        fooActor ! "Hello, world!"
        barTestProbe.expectMsg(timeout, "Hello, world!")
      }

      "not send it to baz" in new MockContext with Behaviour {
        val fooActor = TestActorRef[FooActor](
          Props(classOf[FooActor], mockActorFactory))
        fooActor ! "Hello, world!"
        bazTestProbe.expectNoMsg(timeout)
      }
    }
  }

  trait MockContext extends MockitoSugar {
    val mockActorFactory = mock[ActorFactory]
    val barTestProbe = TestProbe()
    val bazTestProbe = TestProbe()
  }

  trait Behaviour { self: MockContext =>
    when(mockActorFactory.createBar(any[ActorRefFactory]))
      .thenReturn(barTestProbe.ref)
    when(mockActorFactory.createBaz(any[ActorRefFactory]))
      .thenReturn(bazTestProbe.ref)
  }
}

abstract class AkkaSpec
  extends TestKit(ActorSystem(UUID.randomUUID.toString))
  with ImplicitSender
  with WordSpecLike
  with Matchers
  with BeforeAndAfterAll {

  implicit val timeout = 100.milliseconds

  override def afterAll {
    TestKit.shutdownActorSystem(system)
  }
}
```

Factories are also useful when you want to materialize one or more type classes into existent at an input / output boundary:

```scala
trait NumericFactory[U] {
  def apply(typeRepresentation: String): U = {
    typeRepresentation match {
      case "Byte" => apply[Byte]
      case "Short" => apply[Short]
      case "Int" => apply[Int]
      case "Long" => apply[Long]
      case "Float" => apply[Float]
      case "Double" => apply[Double]
      case _ => throw new IllegalArgumentException
    }
  }

  def apply[T : Numeric]: U
}

object OneAsStringNumericFactory extends NumericFactory[String] {
  def apply[T : Numeric]: String = {
    implicitly[Numeric[T]].one.toString
  }
}

OneAsStringNumericFactory("Byte")
```

Finally, if you can use them then dependency injection frameworks such as Guice and Spring let you remove the extra level of abstraction that factories introduce.
