---
title: Java 多线程学习笔记（二）基础
date: 2018-11-01 16:45:26
categories:
		- java多线程学习笔记
tags:
		- java
		- 多线程
		- synchronized
---

# Java 多线程学习笔记（二）基础

## synchronized 关键字

synchronized关键字可修饰方法或代码块，被修饰的部分，对于多线程来说将按照同步方式来执行。

* synchronized 提供一种锁机制，能确保共享变量互斥访问，从而防止数据不一致问题。
* synchronized 关键字包括monitor enter 和 monitor exit 两个jvm指令，保证任何线程执行到monitor enter之前都必须从主存中获取数据，而不是从缓存中，monitor exit之后，共享变量被更新后的值必须刷入到主内存中。
* synchronized 严格遵守java happen-before 原则，一个monitor exit之前必须要有一个monitor exit

### synchronized 修饰同步方法

```java
public synchronized void sync() {
// code 
}

public synchronized static void sync() {
// code
}
```

### synchronized 修饰同步代码块

```java
  private final Object MUTEX = new Object();
  public void sync() {
    synchronized (MUTEX) {
      // code
    }
  }
```

线程需要获得MUTEX对象相关联的monitor锁，才能执行同步代码块。未获得monitor锁的线程，将会处于blocked状态。

<!--more-->

### synchronized注意点

1. 与monitor关联对象不能为空
2. synchronized作用域不应太大
3. 各个线程争取的monitor关联对象应该是同一个
4. 小心死锁

### This Monitor 和 Class Monitor

同步代码块可以手动设定需要关联锁的MUTEX对象。那么，同步方法呢？

同步方法争抢的是This Monitor的关联锁，this是指的该类实例对象。
故，同一对象的不同同步实例方法，以及synchronized(this)都是互斥的。

同步静态方法，争抢的是该类的class实例对象所关联的锁。因而不同同步静态代码也是互斥的。

## wait 和 notify

wait和notify方法，是Object中的方法，意味着所有java对象都有着两个方法。调用某个对象的wait方法，可使执行线程进入等待。而调用某个对象的notify方法，可使因为调用该对象的wait方法进入等待的线程唤醒。

### wait方法有三个重载方法。

```java
public final void wait() throws  InterruptedException
public final void wait(long timeout) throws  InterruptedException
public final void wait(long timeout,int nanos) throws  InterruptedException
```

1. wait这三个方法最终都调用到wait(long)，wait()等价于wait(0)意味着用不超时，后两个会设置超时时间。超时时间指在该时间内未被唤醒，将会触发超时。
2. 调用wait方法，必须先拥有该对象的关联锁，也就是说在该对象非同步代码块中调用wait方法会抛出异常。
3. 当前线程执行了该对象的wait方法后，会**放弃**该对象的monitor锁并进入与该对象相关的wait set中，意味着其他线程有机会继续争抢该monitor的所有权。

### notify方法
```java
public final native void notify()
```
1. 唤醒单个正在执行该对象wait方法的线程。
2. 如果没有这样的线程，忽略该操作。
3. 被唤醒的线程需要重新获取该对象所关联的monitor锁才能继续执行。

关于wait和notify的注意事项
* wait方法是可中断方法，其他线程interrupt是可以将其打断的。
* 线程执行某个对象的wait方法会进入与之对应的wait set中。每个对象的monitor都有一个与之对应的wait set
* wait和notify都必须在该对象的同步代码中使用，否则会报IllegalMonitorStateException，同步代码的monitor和与执行wait notify方法的对象**一致**，简单地说用哪个对象的monitor进行同步，就只能用哪个对象进行wait和notify操作。

### 示例代码
```java
public class EventQueue {
  private final int max;

  static class Event {

  }

  private final LinkedList<Event> eventQueue
      = new LinkedList<>();

  private final static int DEFAULT_MAX_EVENT = 10;

  public EventQueue(int max) {
    this.max = max;
  }

  public EventQueue() {
    this(DEFAULT_MAX_EVENT);
  }

  public void offer(Event event) {
    synchronized (eventQueue) {
      if (eventQueue.size() >= max) {
        try {
          console("this queue is full");
          eventQueue.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }

      console(" the new event is submitted");
      eventQueue.addLast(event);
      eventQueue.notify();
    }
  }

  public Event take() {
    synchronized (eventQueue) {
      if (eventQueue.isEmpty()) {
        try {
          console(" the queue is empty");
          eventQueue.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }

      Event event = eventQueue.removeFirst();
      this.eventQueue.notify();
      console(" the event " + event + " is handled");
      return event;
    }
  }

  private void console(String message) {
    System.out.printf("%s:%s\n",Thread.currentThread().getName(),message);
  }
  
}
```

```java
package org.chenghao.concurrent;

import java.util.concurrent.TimeUnit;

public class EventClient {
  public static void main(String[] args) {
    final EventQueue eventQueue = new EventQueue();
    new Thread(
        () -> {
          while (true) {
            eventQueue.offer(new EventQueue.Event());
          }
        }, "Producer"
    ).start();

    new Thread(
        () -> {
          while (true) {
            eventQueue.take();
            try {
              TimeUnit.MILLISECONDS.sleep(10);
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
        }, "Consumer"
    ).start();
  }
}
```

## 多线程通信

上述示例代码，在多线程会出现问题。原因在于consumer和producer都是由同一个对象阻塞，producer并不能指定唤醒consumer，相反很可能唤醒一个producer。同样的情况对consumer也是存在的。

![consumer](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89producer%20%E9%97%AE%E9%A2%98.png)
![producer](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89consumer%20%E9%97%AE%E9%A2%98.png)

修改方法,将if->while,notify->notifyAll

```java
public void offer(Event event) {
    synchronized (eventQueue) {
      while (eventQueue.size() >= max) {
        try {
          console("this queue is full");
          eventQueue.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }

      console(" the new event is submitted");
      eventQueue.addLast(event);
      eventQueue.notifyAll();
    }
  }

  public Event take() {
    synchronized (eventQueue) {
      while (eventQueue.isEmpty()) {
        try {
          console(" the queue is empty");
          eventQueue.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }

      Event event = eventQueue.removeFirst();
      this.eventQueue.notifyAll();
      console(" the event " + event + " is handled");
      return event;
    }
  }
```

## ThreadGroup

java程序中，任何一个线程，都会属于一个ThreadGroup中，默认情况下，新创建的线程会被加入到main线程所在的ThreadGroup中。
创建ThreadsGroup语法如下：
```java
public ThreadGroup(String name)
public ThreadGroup(ThreadGroup parent, String name)
```
若是不指定其父group，则默认是创建线程所在的ThreadGroup。
ThreadGroup提供复制Threads功能
```java
public int enumerate(Thread[] list)
public int enumerate(Thread[] list, boolean recurse)
```
enumerate(list)默认调用enumerate(list,true),会迭代复制。
类似的也有复制整个ThreadGroup数组
```java
public int enumerate(ThreadGroup[] list)
public int enumerate(Thread[] list, boolean recurse)
```
效果类似。
守护ThreadGroup，将ThreadGroup设置为daemon，并不会影响其内部的线程daemon属性。当ThreadGroup的daemon设置为true后，那么当该group中没有任何active线程的时候，group将自动destroy。

## 线程运行时异常

### UncaughtExceptionHandler
线程的执行单元是**不能**抛出checked异常的，可以使用UncaughtExceptionHandler接口来获取线程运行时异常信息，当线程运行出现异常是，将会回调该方法。
```java
@FunctionInterface
public interface UncaughtExceptionHandler {
  void uncaughtException(Thread t, Throwable e);
}
```
该方法会被Thread中的dispatchUncaughtException方法调用。
```java
public class CaptureThreadException {
  public static void main(String[] args) {
    Thread.setDefaultUncaughtExceptionHandler(
        (t, e) -> {
          System.out.println(t.getName() + " occur exception");
          e.printStackTrace();
        }
    );
    final Thread thread = new Thread(
        () -> {
          try {
            TimeUnit.SECONDS.sleep(2);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }

          System.out.println(1/0);
        }, "Test-Thread"
    );
    thread.start();
  }
}
```
![UncaughtExceptionHandler演示](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/UncaughtExceptionHandler%E6%BC%94%E7%A4%BA.png)
如果Thread未注入UncaughtExceptionHandler，将会返回group。group也是一个该接口的实现类，默认情况下，会寻找父group的Handler，如果没有父group，则寻找全局默认Handler，都没有，则输出到err中。

### 钩子线程

可通过Runtime.getRuntime().addShutdownHook(Thread t)来注入钩子线程，当jvm退出时，线程被执行。可以注入多个线程，均会被启动。钩子线程可以用来在jvm退出时，清除锁等操作。

注意钩子线程只有在收到退出信号时才会执行，比如ctrl+c或者kill。
但是kill -9不行。



