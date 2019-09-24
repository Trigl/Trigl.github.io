---
layout:     post
title:      "Java 多线程基础"
date:       2016-04-01
author:     "Ink Bai"
header-style: "text"
catalog:    true
tags:
    - 并发
---

> 接触Java有大半年了，一直听说掌握多线程才是真正的Java程序员，由于项目中没有太多并发的东西，所以一直都没有机会接触。最近趁着项目不太忙决定学习一下，我脑袋很笨所以一上来就学习很难的东西会有点吃不消，本文总结了多线程中非常基础的知识，都是一些很简单的栗子，对于高手来说现在就可以右上角了，后续会继续学习，争取早日能结合项目理解多线程吧。

## 进程和线程的概念
简单地说，进程就是一次程序执行，例如电脑中运行的QQ进程，浏览器进程；线程可以理解成进程中独立运行的子任务，例如QQ运行时可以进行很多子操作，比如同时视频、传输文件、发送表情包消息等。
多线程就是后台异步执行这么多的任务，感觉上仿佛它们是一起执行的，而实际上是多个线程分别争夺CPU资源分步执行的。这样做可以极大提高CPU效率，因为如果其中一个线程阻塞，我们可以继续执行另一个线程，而不用空占CPU资源造成CPU资源浪费。
## 实现多线程的两种方式
实现多线程编程的方式有两种，一种是继承Thread类，另一种是实现Runnable接口。首先看一下Thread类的源码：

> public class Thread implements Runnable

由上可知，Thread实现了Runnable接口，因此这两种方式在本质上都是实现了Runnable接口。而且使用继承Thread类的方法创建新的线程时，最大的局限性就是不支持多继承，因为Java的特点就是单根继承，所以实现多线程最好还是使用实现Runnable接口的方式。
#### 继承Thread类
下面用一个简单实例来演示如何通过继承Thread来实现多线程

**自定义线程类**

```java
package com.trigl.concurrent.simplethread;

/**
 * 证明线程的调用顺序是随机的
 * @author 白鑫
 * @date 2016年3月30日 上午11:02:49
 */
public class RandomThread extends Thread {
	@Override
	public void run() {
		try {
			for (int i = 0; i < 10; i++) {
				int time = (int) (Math.random() * 1000);
				Thread.sleep(time); // 休眠随机秒，使线程挂起
				System.out.println("run=" + Thread.currentThread().getName());
			}
		} catch (InterruptedException e) {
			// TODO: handle exception
			e.printStackTrace();
		}

	}
}
```

**测试类**

```java
package com.trigl.concurrent.simplethread;

import org.junit.Test;

public class SimpleThreadTest {
	@Test
	public void testRandomThread() {
		try {
			RandomThread thread = new RandomThread();
			thread.setName("randomThread");
			thread.start();
			for (int i = 0; i < 10; i++) {
				int time = (int) (Math.random() * 1000);
				Thread.sleep(time); // 休眠随机秒，使线程挂起
				System.out.println("main=" + Thread.currentThread().getName());
			}
		} catch (InterruptedException e) {
			// TODO: handle exception
			e.printStackTrace();
		}
	}
}
```

**输出结果**

```java
run=randomThread
main=main
run=randomThread
main=main
main=main
run=randomThread
main=main
run=randomThread
run=randomThread
main=main
```

从结果可以看到，CPU执行哪个线程是不确定的。Thread类的start()方法的作用是通知“线程管理器”此线程已经准备就绪，等待调用线程对象的run()方法，因此run()方法才是线程实际开始执行的方法。首先调用start()方法是为了让CPU管理当前线程从而实现异步，如果没有调用start()方法而是直接调用run()方法，那就相当于是同步顺序执行了，也就没有实现多线程。
#### 实现Runnable接口
可以直接创建一个实现了Runnable接口的类来实现线程，关键是重写其中的run()方法

**实现Runnable接口**

```java
package com.trigl.concurrent.simplethread;

public class MyRunnable implements Runnable {
	@Override
	public void run() {
		System.out.println("线程运行中！");
	}
}
```

以上写出了实现线程的类，那么怎么调用这个类呢，这就需要用到Thread类，在Thread类里面有两个构造方法：

> public Thread(Runnable target)
> public Thread(Runnable target, String name)

所以，只要将实现了Runnable接口的对象传给Thread的构造器，然后再调用Thread的start()方法，就可以实现多线程了，代码如下：

**Runnable对象传参到Thread构造器**

```java
    public void testMyRunnable() {
    	Runnable runnable = new MyRunnable();
    	Thread thread = new Thread(runnable);
    	thread.start();
    	System.out.println("运行结束！");
    }
```

## 线程安全问题
自定义线程类中的实例变量针对其他线程可以有共享和不共享之分，这在多线程交互时可能产生线程安全问题。
#### 不共享数据的情况
Java的封装性默认可以实现多个线程不共享数据，下面通过一个栗子来了解：

**自定义线程类**

```java
package com.trigl.concurrent.simplethread;

/**
 * 不共享数据的线程
 * @author 白鑫
 * @date 2016年3月30日 下午1:10:09
 */
public class NonSharedThread extends Thread {
	private int count = 3;
	public NonSharedThread(String name) {
		super();
		this.setName(name); // 设置线程名称
	}

	@Override
	public void run() {
		super.run();
		while (count > 0) {
			count--;
			System.out.println("由 " + this.currentThread().getName() + " 计算，count=" + count);
		}
	}
}
```

**测试方法**

```java
	@Test
	public void testNonSharedThread() {
		NonSharedThread a = new NonSharedThread("A");
		NonSharedThread b = new NonSharedThread("B");
		NonSharedThread c = new NonSharedThread("C");
		a.start();
		b.start();
		c.start();
	}
```

**输出结果**

```java
由 A 计算，count=2
由 B 计算，count=2
由 C 计算，count=2
由 B 计算，count=1
由 A 计算，count=1
由 B 计算，count=0
由 A 计算，count=0
由 C 计算，count=1
由 C 计算，count=0
```

由于在测试方法中我们创建了三个不同的线程对象，因此各个线程都减少各自的count值，互不影响，这种情况下实例变量就没有共享。
#### 共享数据的情况
共享数据一般发生在多个线程访问同一个变量时，例如多个人抢火车票。一般来说在单实例中会出现数据共享的情况，这时可能存在线程安全问题。仍然通过一个实例来说明：

**自定义线程类**

```java
package com.trigl.concurrent.simplethread;

/**
 * 共享数据的线程
 * @author 白鑫
 * @date 2016年3月30日 下午1:27:59
 */
public class SharedThread extends Thread {
	private int count = 3;

	@Override
	public void run() {
		super.run();
		count--;
		System.out.println("由 " + this.currentThread().getName() + " 计算，count=" + count);
	}
}
```

**测试方法**

```java
	@Test
	public void testSharedThread() {
		SharedThread thread = new SharedThread();
		Thread a = new Thread(thread, "A");
		Thread b = new Thread(thread, "B");
		Thread c = new Thread(thread, "C");
		a.start();
		b.start();
		c.start();
	}
```

**输出结果**

```java
由 A 计算，count=1
由 B 计算，count=1
由 C 计算，count=0
```

从结果可以看出来，线程A和B打印的count的值都是1，而我们希望看到的是不重复的递减的值，这说明A和B对count同时进行了处理，这就是由共享数据所引起的 `非线程安全` 问题。非线程安全主要是指多个线程对同一对象的同一个实例变量进行操作时会出现值被更改、值不同步的情况，进而影响程序的执行流程。
在某些JVM中，`i--` 的操作要分成3步：

1. 取得原有i值
2. 计算i-1
3. 对i进行赋值

在这3个步骤中，如果有多个线程同时访问，那么一定会出现非线程安全问题。
#### 如何实现线程安全
为了让上面的例子的count值可以顺序减1，我们需要对线程实现同步操作，即各个线程之间排队执行 `i--` 操作，Java是通过synchronized关键字来实现的，代码如下：

**实现了同步的线程**

```java
package com.trigl.concurrent.simplethread;

/**
 * 同步的共享数据资源的线程
 * @author 白鑫
 * @date 2016年3月30日 下午2:19:22
 */
public class SynSharedThread extends Thread {
	private int count = 5;

	@Override
	synchronized public void run() {
		super.run();
		count--;
		System.out.println("由 " + this.currentThread().getName() + " 计算，count=" + count);
	}
}
```

**测试方法不变，结果如下**

```java
由 A 计算，count=2
由 B 计算，count=1
由 C 计算，count=0
```

通过在需要同步的方法前面加上关键字synchronized，就相当于给这个方法加了一把锁，synchronized可以在任意方法和对象上加锁，而加锁的这段代码称为“互斥区”或“临界区”。每当一个线程想要执行这个方法的时候，首先会查看是否锁住，如果没有锁住就可以访问了，如果已经锁住了说明其他线程正在访问这个方法，那么该线程会继续尝试拿到锁权，当有多个线程同时想要执行该方法时，就会同时去抢锁。
## Thread类的常用方法

* currentThread()：返回代码段正在被哪个线程调用
* isAlive()：判断当前的线程是否处于活动状态
* sleep()：在指定的毫秒数内让“当前正在执行的线程”休眠（暂停执行）
* getId()：取得线程的唯一标识

#### currentThread()
需要注意的是Thread.currentThread().getName()和this.getName()，前者是当前实际执行的线程名，后者是this对应的线程对象的名称，也许不是实际的线程名，而是间接调用的线程名。通过一个实例更好理解：

**自定义线程**

```java
package com.trigl.concurrent.simplethread;

/**
 * 辨别线程名
 * @author 白鑫
 * @date 2016年3月30日 下午5:53:45
 */
public class IdentifyThread extends Thread {
	public IdentifyThread() {
		System.out.println("IdentifyThread---begin");
		System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());
		System.out.println("this.getName()=" + this.getName());
		System.out.println("IdentifyThread---end");
	}

	@Override
	public void run() {
		System.out.println("run---begin");
		System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());
		System.out.println("this.getName()=" + this.getName());
		System.out.println("run---end");
	}
}
```

**测试方法**

```java
	@Test
	public void testIdentifyThread() {
		IdentifyThread it = new IdentifyThread();
		it.setName("it");
		Thread t = new Thread(it);
		t.setName("t");
		t.start();
	}
```

**输出结果**

```java
IdentifyThread---begin
Thread.currentThread().getName()=main
this.getName()=Thread-0
IdentifyThread---end
run---begin
Thread.currentThread().getName()=t
this.getName()=it
run---end
```

从上面结果可以看出，当线程t执行时，线程t间接实现是it的run()方法，所以Thread.currentThread().getName()得到的是当前实际执行的线程，即t；而this.getName()得到的是当前线程对象it对应的线程名。
#### isAlive()
通过一个实例来学习：

**自定义线程**

```java
package com.trigl.concurrent.simplethread;

/**
 * 验证线程是否处于活动状态
 * @author 白鑫
 * @date 2016年3月30日 下午6:22:08
 */
public class CheckAlive extends Thread {
	@Override
	public void run() {
		System.out.println("run=" + this.isAlive());
	}
}
```

**测试方法**

```java
	@Test
	public void testCheckAlive() {
		CheckAlive thread = new CheckAlive();
		System.out.println("test=" + Thread.currentThread().isAlive() + " thread=" + thread.isAlive());
		thread.start();
	}
```

**输出结果**

```java
test=true thread=false
run=true
```

#### sleep()
这是Thread的静态方法，直接通过Thread.sleep()来调用，还是通过一个实例来学习：

**自定义线程类**

```java
package com.trigl.concurrent.simplethread;

/**
 * 使线程休眠
 * @author 白鑫
 * @date 2016年3月30日 下午6:34:32
 */
public class SleepThread extends Thread {
	@Override
	public void run() {
		try {
			System.out.println("run threadName=" + Thread.currentThread().getName() + " begin=" + System.currentTimeMillis());
			Thread.sleep(2000);
			System.out.println("run threadName=" + Thread.currentThread().getName() + " end=" + System.currentTimeMillis());
		} catch (InterruptedException e) {
			// TODO: handle exception
			e.printStackTrace();
		}
	}
}
```

**测试方法**

```java
	@Test
	public void testSleepThread() throws InterruptedException {
		SleepThread thread = new SleepThread();
		System.out.println("begin=" + System.currentTimeMillis());
		thread.start();
		System.out.println("end=" + System.currentTimeMillis());
		Thread.sleep(2000);
	}
```

**输出结果**

```java
begin=1459334153351
end=1459334153351
run threadName=Thread-0 begin=1459334153351
run threadName=Thread-0 end=1459334155351
```

可以看到起始时间相差两秒。
#### getId()
仍然通过一个栗子：

**测试方法**

```java
	@Test
	public void testGetId() {
		Thread thread = Thread.currentThread();
		System.out.println("thread=" + thread.getName() + " id=" + thread.getId());
	}
```

**输出结果**

```java
thread=main id=1
```
