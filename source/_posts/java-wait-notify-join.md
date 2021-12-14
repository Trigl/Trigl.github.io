---
layout:     post
title:      "最熟悉的陌生人：Java 的 wait、notify/notifyAll、join、sleep、interrupt、yield 方法"
date:       2021-12-09
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/java-wait-notify.jpg"
tags:
    - Java 基础
---

> 对于 Javaer 来说，wait、notify、sleep 等方法应该是最熟悉不过了，但是你真的知道它们的区别和内在原理吗，你知道如何正确使用它们吗，今天来做一个总结。

首先明确一点，以上这些方法，有的是 Object 类下面的方法，有的是 Thread 类下面的方法，但是它们起作用的都是线程，而非某个对象。就是为了让线程之间可以更好地协作，协作使用锁或者 CPU 这些共享资源。

## wait
wait 是 Object 上的方法，它的直观作用是使当前正在运行的线程休眠等待。

调用 wait 的时候必须获取到对象锁，也就是进入 synchronized 代码块，拿到对象的 monitor 锁。如果没有拿到 monitor 锁就调用 wait，即使可以编译通过，但是运行时仍然会报 IllegalMonitorStateException。

wait 方法会使当前线程释放掉 monitor 对象锁，同时释放 CPU 使用权。

wait 调用之后不会自己苏醒，需要通过以下四种方式才能苏醒：

- 其他线程调用了 notify 方法，并且正好 notify 了当前正在休眠的线程，正好二字说明具有一定的偶然性，具体下文讲 notify 的时候再说
- 其他线程调用了 notifyAll 方法，将当前线程唤醒
- 其他线程中断了正在 wait 的当前线程
- 调用 wait 的时候设置了一个 wait 的时限，如调用了 `wait(1000)`，那么时间到达后线程会自动苏醒

## notify/notifyAll
notify/notifyAll 也是 Object 上的方法，直接作用就是唤醒其他正在等待当前对象锁、处于休眠状态的线程。

notify 只能唤醒等待线程中的任意一个，而 notifyAll 会唤醒所有。由于 notify 只能唤醒一个，某些 wait 的线程就有可能永远不被唤醒，因此实际编程还是推荐使用 notifyAll

notify/notifyAll 和 wait 使用限制一样，必须先获取到对象锁才能调用，并且它唤醒的对象也是其他正在等这个对象锁的线程。同样地，没有拿到 monitor 锁时调用会抛出 IllegalMonitorStateException。

notify/notifyAll 不会使当前线程释放掉 monitor 对象锁，也不会释放 CPU 使用权，调用之后当前线程继续正常执行，它其实只是顺口告诉了其他正在等待这把锁的休眠线程，别睡了，准备抢红包了！其他线程还得靠自己的努力去竞争锁。

## join
接下来讲 join，因为 join 的源码恰好为我们展示了 wait 和 notify 是如何结合起来使用的。

join 是 Thread 对象上的方法，它的作用用文字来描述会让人感觉很绕，我们看代码来说明：

```java
public static void main(String[] args) throws InterruptedException {
    Thread t = new Thread(() -> {
        System.out.println("first run");
    });
    t.start();

    Thread.sleep(100);

    System.out.println("last run");
}
```

这里为了两个线程执行的先后顺序，我们就在执行 main 线程的 `System.out.println("last run");` 之前 sleep 了一会儿。

而 join 方法也可以起到类似的作用，上面的代码完全可以写成这样：

```java
public static void main(String[] args) throws InterruptedException {
    Thread t = new Thread(() -> {
        System.out.println("first run");
    });
    t.start();

    t.join();

    System.out.println("last run");
}
```

join 方法的作用是：在当前线程 A 中调用另外一个线程 B 的 join 方法后，会让当前线程 A 等待，直到线程 B 执行结束了，A 线程才会苏醒，然后继续执行自己的业务逻辑。

我们看下 join 方法的源码：

```java
public final void join() throws InterruptedException {
    join(0);
}

public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

基本逻辑很简单，有一个 while 循环一直判断线程是否是 alive 的，如果是的话就 wait。

看到 wait，上文中我们说了调用某个对象的 wait 方法必须持有这个对象的 monitor 锁，join 是被 synchronized 修饰的，因此每次调用的时候会先获取到对象锁。

那么问题来了，既然被 wait 了，那么什么时候被唤醒呢？猜测是线程结束时会获取到自身的对象锁，然后调用 notifyAll，事实上也的确就是这样实现的，线程结束退出的逻辑是在 JVM cpp 源码中实现的，这里就不深究了。

> 小结：通过看 join 的实现，我们了解到了 wait 和 notify 一定是成对出现的，还说明了 wait 和 notify 的应用场景：
当条件不满足时，调用 wait 进入等待状态；当条件满足时，调用 notify 唤醒线程，继续执行任务。
```java
while (条件判断) {
    wait();
}
```

## sleep
上面的例子中也提到了 sleep，sleep 是 Thread 的方法，虽然它的直接作用也是使当前线程“暂停”一段时间，但是与 wait 方法不同的是，sleep 方法仅释放一段时间的 CPU 资源，过了这段时间它自己就又可以抢 CPU 了，不需要被动等别人叫醒。

所以为什么我们说 wait 和 notify 对线程之间的协作功不可没，因为可以让多个线程一起玩耍，你需要我，我需要你。而像 sleep 这种方法，自己玩就够了，不需要别人的帮助。

## interrupt
interrupt 首要作用是让之前调用了 wait/join/sleep 之后阻塞的线程立刻抛出 InterruptedException 错误并且唤醒，它作用的对象是处于阻塞状态的线程。对于 running 状态的线程，interrupt 只会打一个 interrupted 的标，调用 isInterrupted 方法会返回 true，除此之外没有其他影响，歌照唱舞照跳马照跑。

此时我们有一个问题，我想打断一个 running 中的线程该怎么办呢，最佳实践就是打一个明确的标志，然后在线程跑的过程中不停检查这个标志来判断是否停止，写法也比较简单：

```java
public class StopThreadDemo implements Runnable {
    private boolean isRunning;

    @Override
    public void run() {
        while (isRunning) {
            // 业务逻辑
        }
    }

    public void start() {
        isRunning = true;
    }

    public void stop() {
        isRunning = false;
    }
}
```

## yield
yield 作用是使当前正在 running 的线程变到可执行状态，就是暂时放弃 CPU 使用权，并且释放了 CPU 之后只有同级别或更高优先级的线程有执行的机会，如果没有优先级更高的线程，就会发生刚释放了 CPU 马上又获取到了 CPU 接着使用。
