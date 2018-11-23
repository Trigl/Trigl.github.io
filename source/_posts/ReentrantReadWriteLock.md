---
layout:     post
title:      "ReentrantReadWriteLock 使用"
date:       2018-10-18
author:     "Ink Bai"
header-img: "/img/post/read-write-lock.jpg"
tags:
    - 并发
---
ReentrantReadWriteLock 提供了读写锁机制可以方便我们更好实现并发场景，首先明确读写锁地特征：读锁与读锁不互斥，写锁与读锁互斥，写锁与写锁互斥。

依据此，我们就可以应用在这样地场景下：如果多个线程可以并行跑，那么我们就可以给它们都分配读锁；如果某个线程必须阻塞其他所有线程开跑，那我们就给它设置写锁。

用一个例子来讲解一下，现在共有四个线程，其中三个可以并行跑，一个必须阻塞跑（实际场景如对元数据进行备份，这种情况必须阻塞跑），实现代码如下：

```scala
object LockExample extends App {
  val t1 = new Thread {
    override def run(): Unit = {
      runWithLock(this.getName ,3000, false)
    }
  }

  val t2 = new Thread {
    override def run(): Unit = {
      runWithLock(this.getName ,5000, false)
    }
  }

  val t3 = new Thread {
    override def run(): Unit = {
      runWithLock(this.getName ,7000, false)
    }
  }

  val t4 = new Thread {
    override def run(): Unit = {
      runWithLock(this.getName ,9000, true)
    }
  }

  private def runWithLock(threadName: String, time: Int, blocking: Boolean) = {
    if (!GlobalLockService.acquire(blocking)) {
      println("can't get lock!")
    }
    try {
      println(s"enter $threadName")
      val start = System.currentTimeMillis()
      Thread.sleep(time)
      val end = System.currentTimeMillis()
      println(s"leave $threadName, spend time: ${(end - start) / 1000}s")
    } finally {
      GlobalLockService.release(blocking)
    }
  }

  t1.start()
  t2.start()
  t3.start()
  t4.start()
}

object GlobalLockService {
  val lock = new ReentrantReadWriteLock()

  def acquire(blocking: Boolean) =
    if (blocking)
      lock.writeLock.tryLock(60, TimeUnit.MINUTES)
    else
      lock.readLock.tryLock(90, TimeUnit.MINUTES)

  def release(blocking: Boolean) =
    if (blocking)
      lock.writeLock.unlock()
    else
      lock.readLock.unlock()
}
```

可能会有多种结果输出：

```
enter Thread-0
enter Thread-2
enter Thread-1
leave Thread-0, spend time: 3s
leave Thread-1, spend time: 5s
leave Thread-2, spend time: 7s
enter Thread-3
leave Thread-3, spend time: 9s
```

或者：

```
enter Thread-3
leave Thread-3, spend time: 9s
enter Thread-2
enter Thread-1
enter Thread-0
leave Thread-0, spend time: 3s
leave Thread-1, spend time: 5s
leave Thread-2, spend time: 7s
```

可以看到线程 0，1，2 都是并发执行，而线程 3 是单独阻塞执行。
