---
layout:     post
title:      "Akka Study"
date:       2018-05-23
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/winter-akka.jpg"
tags:
    - Akka
---

> Akka 用于构建高并发、分布式且具有容错机制的事件驱动型的应用，本文是 Scala Cookbook 一书中 Akka 部分内容的总结。

## Akka Guide
Actor 模型与线程比较，是一种高层次的抽象。说 “高层次的抽象”这句话，就意味着这个东西简单易用，你不需要考虑太多底层的其他东西。所以如果理解了 Actor 模型的原理，就可以专注解决问题，而不需要把注意力放在像线程、锁和共享数据这样低层次的问题上。

**优势**
轻量级的事件驱动模型：对相应的消息事件做出异步回应，并且消息是不可变的，因为需要在不同线程之间共享。
容错性：Akka 可以用于构造 “自我修复型” 系统。
位置透明性：Akka actors 可以跨过多个 JVM 或者 服务器，其被设计为通过纯粹的消息在分布式系统中进行沟通。

在 Actor 模型中，消息是唯一的沟通方式，你要做任何事情都需要发送消息，不能直接更改 Actor 内部的状态或者使用其内部的资源。

**Actor 模型简介**
1、Actor 是 actor 系统中的最小单位，就类似对象是 OOP 系统中的最小单位一样。
2、也正如一个对象，一个 actor 的状态和行为也是被封装起来的。你不能直接去访问或执行一个 actor 的方法或者是成员对象（field），而只能是向这个 actor 发送一个要求访问其内部状态的消息，正如你不可能有读心术去了解别人心里是怎么想的，而只能问对方你的感受如何。
3、每一个 actor 都有一个邮箱用来接收消息，而它们的日常工作就是处理这些消息。
4、你与每个 actor 交互的方式就是向其发送一条不可变的消息，这些消息会进入到这个 actor 的邮箱中。
5、actor 也有自己的层级关系，类似一个公司的组织结构。一个 actor 会有父 actor 或者子 actor 或者兄弟 actor，正如一个高管会有董事长管他，会有其他高管与之一起合作，也会有自己的下属。这些组织结构就是为了实现一个目的：“代理委派”，让所有 actor 合作起来使系统正常运转。
6、最后一点就是 actor 模型的异常处理，当 actor 处理消息时，某些地方可能会出错，可能会抛出一些异常，那么此时怎么处理呢？通常做法是当前 actor 会停止其本身和其子 actor 的所有操作，然后向其父 actor 发送一条通知错误信息的消息，然后父节点根据不同情况做出处理，一般处理方式有以下几种：

1. 恢复子 actor，所有状态保持不变
2. 重启子 actor，所有状态清空，从零开始
3. 终止子 actor （当一个 actor 终止的时候，其邮箱內的消息会进入 dead letter 邮箱）。
4. 把错误抛给上一级 actor 处理

## Actor 之间如何沟通？
一个基于 actor 的系统当然会包含很多 actor，那么它们之间是怎么沟通的呢，让通过一个 ping-pong 的例子来了解一下。

```scala
import akka.actor.{Actor, ActorRef, ActorSystem, Props}

object PingPongTest extends App {
  val system = ActorSystem("PingPongSystem")
  val pong = system.actorOf(Props[Pong], name = "pong")
  val ping = system.actorOf(Props(new Ping(pong)), name = "ping")
  ping ! StartMessage
}

case object PingMessage
case object PongMessage
case object StartMessage
case object StopMessage

class Ping(pong: ActorRef) extends Actor {
  var count = 0
  def incrementAndPrint{ count += 1; println("ping") }
  def receive = {
    case StartMessage =>
      incrementAndPrint
      pong ! PingMessage
    case PongMessage =>
      incrementAndPrint
      if (count > 99) {
        sender ! StopMessage
        println("ping stopped")
        context.stop(self)
      } else {
        sender ! PingMessage
      }
    case _ => println("Ping got something unexpected.")
  }
}

class Pong extends Actor {
  def receive = {
    case PingMessage =>
      println(" pong")
      sender ! PongMessage
    case StopMessage =>
      println("pong stopped")
      context.stop(self)
    case _ => println("Pong got something unexpected.")
  }
}
```

这个例子做了什么呢？首先创建一个名为 “PingPongSystem” 的 ActorSystem，然后用这个 ActorSystem，然后用这个 分别创建 ping 和 pong actor，然后这两个 actor 之间互相发送消息，输出如下：

```scala
ping
 pong
ping
 pong
...
ping stopped
pong stopped
```

每一个基于 actor 模型的应用一般都会有一个 ActorSystem，然后用这个 作为最顶级的 actor，其可以管理所有的子 actor，并可以创造其他 actor。对于普通的 actor，例如 ping 和 pong，它们本身必须可以接受消息并处理，所以每一个 actor 都需要你重写 `receive` 方法，里面是具体的针对各种消息的处理逻辑。

那么如何发送消息呢？上面的例子是这样做的：

```scala
pong ! PingMessage
```

意思就是向 `pong` 这个 actor 发送 `PingMessage`。

上面的例子还用到了 `sender`：

```scala
sender ! PingMessage
```

当一个 actor A 接收到另一个 actor B 的消息时，它同时会接收到这个 actor B 的隐式引用叫 `sender`，你可以使用这个引用再往这个 actor B 回发一条消息。
## Actor 的生命周期
除了构造器以外，下面的方法包含了 Actor 的整个生命周期：

> receive
preStart
postStop
preRestart
postRestart

还是通过一个例子来了解一下：

```scala
import akka.actor.{Actor, ActorSystem, Props}

object LifecycleDemo extends App {
  val system = ActorSystem("LifecycleDemo")
  val kenny = system.actorOf(Props[Kenny], name = "Kenny")

  println("sending kenny a simple String message")
  kenny ! "hello"
  Thread.sleep(1000)

  println("make kenny restart")
  kenny ! ForceRestart
  Thread.sleep(1000)

  println("stopping kenny")
  system.stop(kenny)

  println("shutting down system")
  system.terminate
}

case object ForceRestart

class Kenny extends Actor {
  println("entered the Kenny constructor")
  override def preStart() { println("Kenny: preStart") }
  override def postStop() { println("Kenny: postStop") }

  override def preRestart(reason: Throwable, message: Option[Any]) {
    println("Kenny: preRestart")
    println(s" MESSAGE: ${message.getOrElse("")}")
    println(s" REASON: ${reason.getMessage}")
    super.preRestart(reason, message)
  }

  override def postRestart(reason: Throwable) {
    println("Kenny: postRestart")
    println(s" REASON: ${reason.getMessage}")
    super.postRestart(reason)
  }

  override def receive: Receive = {
    case ForceRestart => throw new Exception("Boom!")
    case _ => println("Kenny received a message")
  }
}
```

输出如下：

```scala
entered the Kenny constructor
sending kenny a simple String message
Kenny: preStart
Kenny received a message
make kenny restart
Kenny: preRestart
 MESSAGE: ForceRestart
 REASON: Boom!
Kenny: postStop
entered the Kenny constructor
Kenny: postRestart
 REASON: Boom!
Kenny: preStart
[ERROR] [05/21/2018 23:21:39.109] [LifecycleDemo-akka.actor.default-dispatcher-4] [akka://LifecycleDemo/user/Kenny] Boom!
java.lang.Exception: Boom!
	at ink.baixin.akka.example.Kenny$$anonfun$receive$1.applyOrElse(LifecycleDemo.scala:45)
	at akka.actor.Actor.aroundReceive(Actor.scala:517)
	at akka.actor.Actor.aroundReceive$(Actor.scala:515)
	at ink.baixin.akka.example.Kenny.aroundReceive(LifecycleDemo.scala:26)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:590)
	at akka.actor.ActorCell.invoke(ActorCell.scala:559)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:257)
	at akka.dispatch.Mailbox.run(Mailbox.scala:224)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:234)
	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

stopping kenny
shutting down system
Kenny: postStop
```

让我们来分析一下整个过程：
首先创建一个 actor，进入其构造器创建一个 actor 实例；然后启动这个 actor，启动之前会先调用该实例的 `preStart` 方法；之后我们向这个 actor 发送了一条 String 类型的消息，actor 调用其 `receive` 方法进行处理；然后我们向这个 actor 发送了 `ForceRestart` 的消息用于重启，该消息会抛出一个异常，当 actor 检测到异常时，就会启动自动重启机制，调用实例自身的 `preRestart` 方法，此时会终止该 actor 实例及其所以子 actor 然后调用 `postStop` 方法；这个时候之前的 actor 实例的生命周期就彻底结束了，会创建一个新的 actor 实例，创建之后调用这个新实例的 `postRestart` 方法，然后再调用其 `preStart` 方法，之后新的 actor 就启动了，最后我们通过 ActorSystem 停止了这个 actor 实例，调用该实例的 `postStop` 方法，这就是上面 actor 经历的完整生命周期。

可以看到，一个 actor restart 前后实际上是两个不同的实例。
## Start and Stop an Actor
#### 启动 actor
**使用 ActorSystem.actorOf 启动**
可以使用 ActorSystem 创建其他普通 actor

```scala
val system = ActorSystem("HelloSystem")
// the actor is created and started here
val helloActor = system.actorOf(Props[HelloActor], name = "helloactor")
```

**使用 context.actorOf 启动**
每一个普通 actor 都有一个隐含的 context 实例，通过这个实例可以启动其它 actor

```scala
class Parent extends Actor {
  val child = context.actorOf(Props[Child], name = "Child")
  // more code here ...
}
```

#### 停止 actor
**system.stop and context.stop**
在 ActorSystem 级别，可以使用 ActorSystem 实例去停止一个 actor：

```scala
actorSystem.stop(anActor)
```

在一个 actor 内部，可以使用 context 引用停止其子 actor：

```scala
context.stop(childActor)
```

当然一个 actor 也可以停止其自身：

```scala
context.stop(self)
```

**PoisonPill message**
也可以通过向 actor 发送停止指令 `PoisonPill` 来停止它，当这条消息被处理时 actor 就会停止，这条消息也会像普通消息一样出现在邮箱的队列里。
下面是一个 `PoisonPill` 的实例：

```scala
import akka.actor.{Actor, ActorSystem, PoisonPill, Props}

object PoisonPillTest extends App {
  val system = ActorSystem("PoisonPillTest")
  val actor = system.actorOf(Props[TestActor], name = "test")

  // a simple message
  actor ! "before PoisonPill"

  // the PoisonPill
  actor ! PoisonPill

  // these messages will not be processed
  actor ! "after PoisonPill"
  actor ! "hello?!"

  system.terminate
}

class TestActor extends Actor {
  def receive = {
    case s: String => println(s"Message Received: $s")
    case _ => println("TestActor got an unknown message")
  }

  override def postStop(): Unit = {
    println("TestActor::postStop called")
  }
}
```

结果如下：

```scala
Message Received: before PoisonPill
TestActor::postStop called
```

可以看到在 PoisonPill 后面的消息就不会被处理了，对于使用 PoisonPill 来停止 actor 这种方式来说，actor 只会处理其邮箱队列在 PoisonPill 这条消息之前的所有消息，然后就会停止 actor，后面的消息自然不会被处理了。那么后面的消息去了哪里呢？进入到了 ActorSystem 的 deadLetters actor，你可以使用 ActorSystem 的 `deadLetters` 方法进行访问。
与之相对应的，上文中讲的 `stop` 方法却是会把一个 actor 邮箱队列中所有消息都处理完以后才停止这个 actor。

**gracefulStop**
正如其名字表面的，如果你可以等待一小会来优雅地终止一个 actor，那么可以使用 `gracefulStop` 方法，下面是一个例子：

```scala
import akka.actor.{Actor, ActorSystem, Props}
import akka.pattern.gracefulStop

import scala.concurrent.{Await, Future}
import scala.concurrent.duration._

object GracefulStopTest extends App {
  val system = ActorSystem("GracefulStopTest")
  val testActor = system.actorOf(Props[TestActor1], name = "TestActor")

  // try to stop the actor gracefully
  try {
    val stopped: Future[Boolean] = gracefulStop(testActor, 2 seconds)
    Await.result(stopped, 3 seconds)
    println("testActor was stopped")
  } catch {
    case e: Exception => e.printStackTrace
  } finally {
    system.terminate
  }
}

class TestActor1 extends Actor {
  def receive = {
    case _ => println("TestActor got message")
  }

  override def postStop { println("TestActor: postStop") }
}
```

**“Killing” an actor**
也可以给 actor 发送一条 `Kill` 的消息来停止它：

```scala
actor ! Kill
```

## actor 的 ask
actor 有一个 `ask` 或者 `?` 方法，这个方法的作用是向 antor 发送消息并等待这个 actor 返回一个结果，返回的结果是一个 `Future`，而 `!` 发送消息 actor 并不会返回结果，下面是一个实例：

```scala
import akka.actor.{Actor, ActorSystem, Props}
import akka.util.Timeout
import akka.pattern.ask

import scala.concurrent.{Await, Future}
import scala.concurrent.duration._

object AskTest extends App {
  // create the system and actor
  val system = ActorSystem("AskTestSystem")
  val myActor = system.actorOf(Props[AskActor], name = "myActor")

  // (1) this is one way to "ask" another actor for information
  implicit val timeout = Timeout(5 seconds)
  val future = myActor ? AskNameMessage
  val result = Await.result(future, timeout.duration).asInstanceOf[String]
  println(result)

  // (2) a slightly different way to ask another actor for information
  val future2: Future[String] = ask(myActor, AskNameMessage).mapTo[String]
  val result2 = Await.result(future2, 1 second)
  println(result2)

  system.terminate
}

case object AskNameMessage

class AskActor extends Actor {
  def receive = {
    case AskNameMessage =>
      // response to the 'ask' request
      sender ! "Fred"
    case _ => println("that was unexpected")
  }
}
```

输出：

```scala
Fred
Fred
```

从上面可以看到，当用 `?` 或者 `ask` 向 actor 发送一条消息时，actor 接收到该消息，然后用 `sender ! "Fred"` 的方法返回一个 `Future`。

## 使用 become 切换 actor 的不同状态
如果一个 actor 在不同的时间需要对应不同的状态，这样可以进行不同的消息处理，此时可以使用 `become` 方法切换 actor 的状态，就可以对应到不同的 `receive` 方法，下面是一个绿巨人变身的例子：

```scala
import akka.actor.{Actor, ActorSystem, Props}

object BecomeHulkExample extends App {
  val system = ActorSystem("BecomeHulkExample")
  val davidBanner = system.actorOf(Props[DavidBanner], name = "DavidBanner")

  davidBanner ! ActNormalMessage // init to normalState
  davidBanner ! Solution
  davidBanner ! BadGuysMakeMeAngry
  davidBanner ! Solution
  davidBanner ! ActNormalMessage
  Thread.sleep(1000)

  system.terminate
}

case object ActNormalMessage
case object Solution
case object BadGuysMakeMeAngry

class DavidBanner extends Actor {
  import context._

  def angryState: Receive = {
    case Solution =>
      println("Fight with you!")
    case ActNormalMessage =>
      println("Phew, I'm back to become David.")
      become(normalState)
  }

  def normalState: Receive = {
    case Solution =>
      println("Looking for solution to my problem ...")
    case BadGuysMakeMeAngry =>
      println("I'm getting angry ...")
      become(angryState)
  }

  def receive = {
    case BadGuysMakeMeAngry => become(angryState)
    case ActNormalMessage => become(normalState)
  }
}
```

输出：

```scala
Looking for solution to my problem ...
I'm getting angry ...
Fight with you!
Phew, I'm back to become David.
```

当使用 `become` 以后，这个 actor 的 `receive` 方法就变成了 `become` 內的 `Receive`，例如这里最开始接收到 `ActNormalMessage` 时，这个 actor 的处理逻辑是 `receive` 方法，但是之后的处理逻辑变成了 `normalState`。

## 总结
本文首先讲了 Actor 模型是什么，Actor 模型的优势，然后详细介绍了 Akka Actor 的一些基本操作，包括建立和启动一个 actor，向 actor 发送消息，切换 actor 的状态，停止 actor 等，相信这些已经足够实现一个简单的基于 actor 模型的系统。
