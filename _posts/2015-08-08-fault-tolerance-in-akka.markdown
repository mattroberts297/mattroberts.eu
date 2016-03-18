---
layout: post
title:  "Fault Tolerance in Akka"
date:   2015-08-08 12:00:00
summary: "Supervisor strategies, deciders, guardians and the command failed pattern in Akka."
tags:
- scala
- akka
- distributed systems
---

The official documentation on [Fault Tolerance](http://doc.akka.io/docs/akka/2.3.12/scala/fault-tolerance.html) gives a partial introduction to supervisor strategies and a worked example of supervision. If you want the complete lowdown on supervision then you need to check out the official documentation on [Supervision and Monitoring](http://doc.akka.io/docs/akka/2.3.12/general/supervision.html) and [the source code](https://github.com/akka/akka/tree/v2.3.12).

I struggled to extract what I consider to be the salient facts about supervision from the official documentation. That probably reflects more poorly on me than on the authors of the documentation. Nevertheless, I wanted something written down for my own reference.

## Key Points

- Each actor supervises its children using a supervision strategy.
- System level actors are supervised by a guardian actor: `/user`.
- The message that caused an actor to crash is not reprocessed.

**That last one is a real gotcha, more on that later.**

## Supervisor Strategies

When creating an actor using `context.actorOf` the resulting actor is supervised by the invoking actor. By default, every actor uses the `supervisorStrategy` defined in [Actor.scala](https://github.com/akka/akka/blob/v2.3.12/akka-actor/src/main/scala/akka/actor/Actor.scala#L493):

```scala
def supervisorStrategy: SupervisorStrategy =
  SupervisorStrategy.defaultStrategy
```

The `SupervisorStrategy.defaultStrategy` is defined in [FaultHandling.scala](https://github.com/akka/akka/blob/v2.3.12/akka-actor/src/main/scala/akka/actor/FaultHandling.scala#L166):

```scala
final val defaultStrategy: SupervisorStrategy = {
  OneForOneStrategy()(defaultDecider)
}
```

The `OneForOneStrategy` only applies the `Decider` to the faulty child actor. For reference, the only alternative is the `AllForOneStrategy` that applies the `Decider` to all child actors regardless of who is at fault. For the most part it is the `Decider` that you will want to change.

The `defaultDecider` is also defined in [FaultHandling.scala](https://github.com/akka/akka/blob/v2.3.12/akka-actor/src/main/scala/akka/actor/FaultHandling.scala#L154):

```scala
final val defaultDecider: Decider = {
  case _: ActorInitializationException ⇒ Stop
  case _: ActorKilledException         ⇒ Stop
  case _: DeathPactException           ⇒ Stop
  case _: Exception                    ⇒ Restart
}
```

So basically, unless you override it, any `Exception` other than those first three will result in the child actor being restarted. Restarting an actor causes the state of an actor to be reset without notifying the supervisor.

Generally speaking, restarting an actor only makes sense if you believe its state has been corrupted. As a result, there are times when you really do not want that behaviour. Imagine, for example, a child actor that is responsible for running some stateful processing over several windows of data. You probably would not want to loose the accumulated state because of one bad window of data.

Fortunately defining a custom `Decider` is straight forward and the directives `Resume` and `Escalate` allow you to either Resume processing whilst maintaining the child actors state or propagate the exception upwards:

```scala
class Supervisor extends Actor {
  override def receive: Receive = {
    case _ => ???
  }

  override def supervisorStrategy = OneForOneStrategy()(decider)

  private def decider: Decider = {
    case _: ActorInitializationException ⇒ Stop
    case _: ActorKilledException         ⇒ Stop
    case _: DeathPactException           ⇒ Stop
    case _: Exception                    ⇒ Resume
  }
}
```

You can find out more about each directive in: [FaultHandling.scala](https://github.com/akka/akka/blob/v2.3.12/akka-actor/src/main/scala/akka/actor/FaultHandling.scala#L103).

## Guardian Actor

When creating an actor using `system.actorOf` the resulting actor is supervised by the guardian actor. The guardian actor supervisor strategy is stored in   [reference.conf](https://github.com/akka/akka/blob/v2.3.12/akka-actor/src/main/resources/reference.conf#L71):

```scala
akka.actor {
  guardian-supervisor-strategy = "akka.actor.DefaultSupervisorStrategy"
}
```

That class is an implementation of `SupervisorStrategyConfigurator` defined in [FaultHandling.scala](https://github.com/akka/akka/blob/v2.3.12/akka-actor/src/main/scala/akka/actor/FaultHandling.scala#L80):

```scala
final class DefaultSupervisorStrategy extends SupervisorStrategyConfigurator {
  override def create(): SupervisorStrategy =
    SupervisorStrategy.defaultStrategy
}
```

You can override it by providing an application.conf:

```scala
akka.actor {
  guardian-supervisor-strategy = "akka.actor.StoppingSupervisorStrategy"
}
```

## Reprocessing Messages

There are a lot of ways you can reprocess failed messages. Each with there own advantages and disadvantages. Once you decide on an approach you should really try to be consistent throughout the project. You can take just about any actor:

```scala
class UnreliableNetwork extends Actor {
  import Protocol._

  override def receive: Receive = {
    case packet: Packet => if(shouldDrop()) {
      sender() ! PacketSent(packet)
    } else {
      throw new PacketException(packet, sender())
    }
  }

  private def shouldDrop() = {
    import scala.util.Random
    Random.nextInt(2) == 0
  }
}

object Protocol {
  sealed trait Command
  case class Packet(id: Int) extends Command
  sealed trait Event
  case class PacketSent(packet: Packet) extends Event
  case class PacketException(packet: Packet, sender: ActorRef)
    extends Exception
}
```

And make it reprocess messages using the `postRestart` hook:

```scala
class ReliableNetwork extends Actor {
  import Protocol._

  override def postRestart(throwable: Throwable) = {
    throwable match {
      case PacketException(packet, sender) => self.!(packet)(sender)
      case _ =>
    }
    super.postRestart(throwable)
  }

  override def receive: Receive = {
    case packet: Packet => if(shouldDrop()) {
      sender() ! PacketSent(packet)
    } else {
      throw new PacketException(packet, sender())
    }
  }

  private def shouldDrop() = {
    import scala.util.Random
    Random.nextInt(2) == 0
  }
}
```

There are a couple of issues with this approach however. First, there is an implicit assumption about the parent actors `SupervisorStrategy`. Fortunately, the default strategy would work. However, there is no way to enforce this assumption. Second, assuming the sender doesn't wait for a `PacketSent` then the packets are no longer sent in order. Third, and this is really a philosophical point, you have changed the responsibility and intent of the actor. It no longer models an unreliable network.

Instead of throwing an exception you could reply to the sender with a `CommandFailed` message.

```scala
class UnreliableNetwork extends Actor {
  import Protocol._
  import scala.util.Random

  def shouldDrop() = {
    Random.nextInt(2) == 0
  }

  override def receive: Receive = {
    case packet: Packet => if(shouldDrop()) {
      sender() ! PacketSent(packet)
    } else {
      sender() ! CommandFailed(packet)
    }
  }
}

object Protocol {
  sealed trait Command
  case class Packet(id: Int) extends Command
  sealed trait Event
  case class PacketSent(packet: Packet) extends Event
  case class CommandFailed(command: Command) extends Event
}
```

This is the pattern used by most of Akka's IO extensions and it works really well if the sender is in the same JVM as you i.e. `deploy = local`. Providing reliability then becomes the responsibility of another actor:

```scala
class ReliableNetwork(unreliableNetworkFactory: ActorRefFactory => ActorRef) extends Actor {
  import scala.collection.mutable
  import Protocol._

  val queue = mutable.Queue.empty[(Packet, ActorRef)]
  val unreliableNetwork = unreliableNetworkFactory(context)

  override def receive: Receive = {
    case packet: Packet =>
      queue.enqueue(packet -> sender())
      if (queue.size == 1) unreliableNetwork ! packet
    case sent: PacketSent =>
      val (_, commander) = queue.dequeue()
      commander ! sent
      if (!queue.isEmpty) {
        val (packet, _) = queue.front
        unreliableNetwork ! packet
      }
    case failed@CommandFailed(packet) =>
      unreliableNetwork ! packet
  }
}
```

This implementation ensures that packets are sent in order and that the `PacketSent` events are sent back to the original sender. The mutable state means that, again, you make an implicit assumption about the parent actors `SupervisorStrategy` however. Perhaps the biggest problem with the strategy is that it does not scale across JVM boundaries. This is because the `CommandFailed` event may be lost between JVMs. The only solution to this is to use timeouts. A more detailed discussion can be found in the official documentation on [Message Delivery Reliability](http://doc.akka.io/docs/akka/2.3.12/general/message-delivery-reliability.html).
