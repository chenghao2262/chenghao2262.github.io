---
title: Java 多线程学习笔记（三）进阶
date: 2018-11-03 20:48:58
categories:
		- java多线程学习笔记
tags:
---


# Java 多线程学习笔记（三）进阶

## 自定义显示锁BooleanLock

synchronized虽然提供线程同步等功能，但他过于原始，有两个明显缺陷：第一，无法控制阻塞时长，第二，阻塞不可被中断。

定义锁接口
```java
public interface Lock {
  void lock() throws InterruptedException;
  void lock(long mills) throws InterruptedException, TimeoutException;
  void unlock();
  List<Thread> getBlockedThreads();
}
```

<!--more-->

首先给BooleanLock定义如下成员变量，其中bollean变量记录锁状态，currentThread记录持锁线程，blockedList记录阻塞线程列表。

```java
public static class BooleanLock implements Lock {
    private Thread currentThread;
    private boolean locked = false;
    private final List<Thread> blockedList = new ArrayList<>();
    ...
}
```

来具体看lock实现
```java
    @Override
    public void lock() throws InterruptedException {
      synchronized (this) {
        while (locked) {
          final Thread tempThread = currentThread();
          try {
            if (!blockedList.contains(tempThread)) {
              blockedList.add(tempThread);
              this.wait();
            }
          }catch (InterruptedException e) {
            blockedList.remove(tempThread);
            throw e;
          }
        }
        blockedList.remove(currentThread());
        this.locked = true;
        this.currentThread = currentThread();
      }
    }
```
lock是一个同步代码块，while循环当锁被其他线程锁定是会不停循环。实际上当blocked为true时，会添加该线程到阻塞队列里，随后将其进行wait。此时该线程**放弃同步代码块的monitor锁**，其他线程可以进入同步代码块，并且被唤醒后，需要**重新获取锁**，保证该操作肯定是互斥的,只有一个线程能拿到锁。

来看lock(long)实现
```java
    @Override
    public void lock(long mills) throws InterruptedException, TimeoutException {
      synchronized (this) {
        if (mills < 0) {
          this.lock();
        } else {
          long remainingMills = mills;
          long endMills = currentTimeMillis() + remainingMills;
          while (locked) {
            if (remainingMills <= 0) {
              throw new TimeoutException("can not get the lock during" + mills);
            }
            if (!blockedList.contains(currentThread())) {
              blockedList.add(currentThread());
            }
            this.wait(remainingMills);
            remainingMills = endMills - currentTimeMillis();
          }
          blockedList.remove(currentThread());
          this.locked = true;
          this.currentThread = currentThread();
        }
      }
    }
```
巧妙之处在于，调用wait(long)，当然，线程可能会被其他线程唤醒，所以当被唤醒后仍未获得锁，则继续进行循环，只是wait(long)时间缩短，当remainingMills <= 0时，则判断为超时。

unlock代码
```java
    @Override
    public void unlock() {
      synchronized (this) {
        if (currentThread == currentThread()) {
          this.locked = false;
          Optional.of(currentThread().getName() + " release the lock.").ifPresent(System.out::println);
          this.notifyAll();
        }
      }
    }
```
注意下，只有持有锁线程才能进行释放锁操作。重置locked，notifyAll()

## 自定义线程池

从jdk1.5开始，utils包提供了ExecutorService线程池的实现。作为一个线程池，需要管理好线程资源，提高线程利用率和系统效率。

定义一个简单的线程池实现类图
![线程池实现类图](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%AE%9E%E7%8E%B0%E7%B1%BB%E5%9B%BE.png)
简单线程池类满足以下几点要求
* 任务队列：用于换成提交的任务
* 线程数量管理
* 任务拒绝策略
* 线程工厂
* queueSize用于控制任务队列大小
* Keepedalive时间：用了决定线程池自动维护时间

简单线程池采用如下规则管理线程。创建线程池时指定init大小作为初始大小，max大小作为线程自动扩充时最大线程数量，在线程池空闲时需要释放线程但也需要维护一定数量的活跃线程的core，这样线程能控制在一个合理的范围内，三者之间的关系是init<=core<=max

ThreadPool接口定义如下。execute(Runnable)用来提交任务。
```java
public interface ThreadPool {
  void execute(Runnable runnable);
  void shutdown();
  int getInitSize();
  int getMaxSize();
  int getCoreSize();
  int getQueueSize();
  int getActiveCount();
  boolean isShutdown();
}
```

RunnableQueue接口定义如下。offer用于向队列提交任务，take用来获取任务。
```java
public interface RunnableQueue {
  void offer(Runnable runnable);
  Runnable take() throws InterruptedException;
  int size();
}
```

DenyPolicy接口定义如下。注意下RunnerDenyPolicy，直接在当前线程执行任务，这种设计在很多地方都有。
```java
public interface DenyPolicy {
  void reject(Runnable runnable, ThreadPool threadPool);

  class DiscardDenyPolicy implements DenyPolicy {

    @Override
    public void reject(Runnable runnable, ThreadPool threadPool) {

    }
  }

  class AbortDenyPolicy implements DenyPolicy {

    @Override
    public void reject(Runnable runnable, ThreadPool threadPool) {
      throw new RunnableDenyException("The runnable " + runnable + " will be abort.");
    }
  }

  class RunnerDenyPolicy implements DenyPolicy {

    public void reject(Runnable runnable, ThreadPool threadPool) {
      if (!threadPool.isShutdown()) {
        runnable.run();
      }
    }
  }
}
```
RunnableDenyException定义如下。
```java
public class RunnableDenyException extends RuntimeException{
  public RunnableDenyException(String message) {
    super(message);
  }
}
```
InternalTask用来定义线程池内部线程逻辑，只要线程仍然处于running状态且不被打断，就会不停从RunnableQueue中获取task来执行。
```java
public class InternalTask implements Runnable {

  private final RunnableQueue runnableQueue;
  private volatile boolean running = true;

  public InternalTask(RunnableQueue runnableQueue) {
    this.runnableQueue = runnableQueue;
  }

  @Override
  public void run() {
    while (running && !Thread.currentThread().isInterrupted()) {
      try {
        Runnable task = runnableQueue.take();
        task.run();
      } catch (InterruptedException e) {
        running = false;
        break;
      }
    }
  }

  public void stop() {
    this.running = false;
  }
}
```
LinkedRunnableQueue，用LinkedList实现的RunnableQueue，当线任务超过最大值，则reject，若是take不够，阻塞线程，当然是因为进行offer的是用户线程，而take的是线程池内部线程。
```java
public class LinkedRunnableQueue implements RunnableQueue {

  private final int limit;

  private final DenyPolicy denyPolicy;

  private final LinkedList<Runnable> runnableList = new LinkedList<>();

  private final ThreadPool threadPool;

  public LinkedRunnableQueue(int limit, DenyPolicy denyPolicy, ThreadPool threadPool) {
    this.limit = limit;
    this.denyPolicy = denyPolicy;
    this.threadPool = threadPool;
  }

  @Override
  public void offer(Runnable runnable) {
    synchronized (runnableList) {
      if (runnableList.size() >= limit) {
        denyPolicy.reject(runnable,threadPool);
      } else {
        runnableList.addLast(runnable);
        runnableList.notifyAll();
      }
    }
  }

  @Override
  public Runnable take() throws InterruptedException {
    synchronized (runnableList) {
      while (runnableList.isEmpty()) {
        try {
          runnableList.wait();
        } catch (InterruptedException e) {
          throw e;
        }
      }

      return runnableList.removeFirst();
    }
  }

  @Override
  public int size() {
    synchronized (runnableList) {
      return runnableList.size();
    }
  }
}
```

最后也是最关键的是BasicThreadPool，作为ThreadPool实现类。BasicThreadPool继承Thread并实现了ThreadPool接口。在init方法内部默认启动自身线程。BasicThreadPool默认创建initSize大小的线程，并通过keepAliveTime参数控制线程执行周期，用来增加或者减少线程池线程数量。其内部使用threadQueue成员，来管理线程池线程。

```java
public class BasicThreadPool extends Thread implements ThreadPool {

  private final int initSize;
  private final int maxSize;
  private final int coreSize;
  private int activeCount;
  private final ThreadFactory threadFactory;
  private final RunnableQueue runnableQueue;
  private volatile boolean isShutdown = false;
  private final Queue<ThreadTask> threadQueue = new ArrayDeque<>();

  private final static DenyPolicy DEFAULT_DENY_POLICY = new DenyPolicy.DiscardDenyPolicy();
  private final static ThreadFactory  DEFAULT_THREAD_FACTORY = new DefaultThreadFactory();

  private final long keepAliveTime;
  private final TimeUnit timeUnit;

  public BasicThreadPool(int initSize, int maxSize, int coreSize,
                         int queueSize) {
    this(initSize,maxSize,coreSize,DEFAULT_THREAD_FACTORY,
        queueSize,DEFAULT_DENY_POLICY,10,TimeUnit.SECONDS);
  }

  public BasicThreadPool(int initSize,int maxSize,int coreSize,
                         ThreadFactory threadFactory, int queueSize,
                         DenyPolicy denyPolicy, long keepAliveTime,
                         TimeUnit timeUnit) {
    this.initSize = initSize;
    this.maxSize = maxSize;
    this.coreSize = coreSize;
    this.threadFactory = threadFactory;
    this.runnableQueue = new LinkedRunnableQueue(queueSize,denyPolicy,this);
    this.keepAliveTime = keepAliveTime;
    this.timeUnit = timeUnit;
    this.init();
  }

  private void init() {
    start();
    for (int i = 0; i < initSize; i++) {
      newThread();
    }
  }

  @Override
  public void execute(Runnable runnable) {
    if (this.isShutdown) {
      throw new IllegalArgumentException("The thread pool is destroy");
    }
    this.runnableQueue.offer(runnable);
  }

  private void newThread() {
    InternalTask internalTask = new InternalTask(runnableQueue);
    Thread thread = this.threadFactory.createThread(internalTask);
    ThreadTask threadTask = new ThreadTask(thread,internalTask);
    threadQueue.offer(threadTask);
    this.activeCount++;
    thread.start();
  }

  private void removeThread() {
    ThreadTask threadTask = threadQueue.remove();
    threadTask.internalTask.stop();
    this.activeCount--;
  }

  @Override
  public void run() {
    while (!isShutdown && !isInterrupted()) {
      try {
        timeUnit.sleep(keepAliveTime);
      } catch (InterruptedException e) {
        isShutdown = true;
        break;
      }

      synchronized (this) {
        if (isShutdown) {
          break;
        }

        if (runnableQueue.size() > 0 && activeCount < coreSize) {
          for (int i = initSize; i < coreSize; i++) {
            newThread();
          }
          continue;
        }

        if (runnableQueue.size() > 0 && activeCount < maxSize) {
          for (int i = coreSize; i < maxSize; i++) {
            newThread();
          }
        }

        if (runnableQueue.size() == 0 && activeCount > coreSize) {
          for (int i = coreSize; i < activeCount; i++) {
            removeThread();
          }
        }
      }
    }
  }

  @Override
  public void shutdown() {
    synchronized (this) {
      if (isShutdown) {
        return;
      }
      isShutdown = true;
      threadQueue.forEach(threadTask -> {
        threadTask.internalTask.stop();
        threadTask.thread.interrupt();
      });
      this.interrupt();
    }
  }

  @Override
  public int getInitSize() {
    if (isShutdown) {
      throw new IllegalStateException("The thread pool is destory");
    }
    return this.initSize;
  }

  @Override
  public int getMaxSize() {
    if (isShutdown) {
      throw new IllegalStateException("The thread pool is destory");
    }
    return this.maxSize;
  }

  @Override
  public int getCoreSize() {
    if (isShutdown) {
      throw new IllegalStateException("The thread pool is destory");
    }
    return this.coreSize;
  }

  @Override
  public int getQueueSize() {
    if (isShutdown) {
      throw new IllegalStateException("The thread pool is destory");
    }
    return runnableQueue.size();
  }

  @Override
  public int getActiveCount() {
    synchronized (this) {
      return this.activeCount;
    }
  }

  @Override
  public boolean isShutdown() {
    return this.isShutdown;
  }

  private static class ThreadTask {
    Thread thread;
    InternalTask internalTask;
    public ThreadTask(Thread thread, InternalTask internalTask ){
      this.thread = thread;
      this.internalTask = internalTask;
    }
  }

  private static class DefaultThreadFactory implements ThreadFactory {

    private static final AtomicInteger GROUP_COUNTER = new AtomicInteger(1);

    private static final ThreadGroup group = new ThreadGroup("MyThreadPool-" + GROUP_COUNTER.getAndDecrement());

    private static final AtomicInteger COUNTER = new AtomicInteger(0);

    @Override
    public Thread createThread(Runnable runnable) {
      return new Thread(group,runnable,"thread-pool-" + COUNTER.getAndDecrement());
    }
  }
}
```

## 单例模式

单例模式是最常见的设计模式之一，在多线程情况下，我们需要保证单例模式满足这样的要求：线程安全、高性能、懒加载。

### 饿汉式

实现简单，实例对象在使用Singleton1类时，便会进行加载创建。线程安全，但是不满足懒加载特性。

```java
  public static class Singleton1 {
    private byte[] data = new byte[1024];
    private static Singleton1 instance = new Singleton1();
    private Singleton1() {
      
    }
    public  static  Singleton1 getInstance() {
      return instance;
    }
  }
```

### 懒汉式

实现简单，instance实例化是在实际调用到getInstance之后才触发的，符合懒加载的特点，但显然，不能保证线程安全。

```java
  public static class Singleton2 {
    private byte[] data = new byte[1024];
    private static Singleton2 instance = null;
    private Singleton2() {

    }
    public  static  Singleton2 getInstance() {
      if (instance == null) {
        instance = new Singleton2();
      }
      return instance;
    }
  }
```

### 同步懒汉式

给getInstance方法加上同步约束，可以保证懒加载和线程安全，但是每次getInstance均会进行同步，影响性能。

```java
  public static class Singleton3 {
    private byte[] data = new byte[1024];
    private static Singleton3 instance = null;
    private Singleton3() {

    }
    public static synchronized Singleton3 getInstance() {
      if (instance == null) {
        instance = new Singleton3();
      }
      return instance;
    }
  }
```

### 两次验证懒汉式

这种写法，基本能回避前面三种遇到的问题，事实上，大学期间大作业，也是多用这种方式实现单例模式。
但仔细思考，这种方式仍存在问题，比如，当实例化需要较长的时间，比如内部需要操作socket或者connect等耗时操作，instance!=null虽然满足，但是内部操作未完成，其他线程使用instance实例，仍然会出错。
（这是*JAVA高并发编程详解*给出不好理由，并未实际进行验证）

```java
  public static class Singleton4 {
    private byte[] data = new byte[1024];
    private static Singleton4 instance = null;
    private Singleton4() {

    }
    public static Singleton4 getInstance() {
      if (instance == null) {
        synchronized (Singleton4.class) {
          if (instance == null) {
            instance = new Singleton4();
          }
        }
      }
      return instance;
    }
  }
```

### volatile两次验证懒汉式

volatile可以防止指令重排。这种模式符合三大特性。

```java
private volatile static Singleton5 instance = null;
```

### 内部类

这种设计较为巧妙，利用java类加载机制，直到调用 getInstance后才会对内部类Holder进行加载，并对instance实例化，而java类加载机制，能保证只被加载一次。符合三大特性。
这种方式是较为常见单例模式设计之一。

```java
  public static class Singleton6 {
    private byte[] data = new byte[1024];
    private Singleton6() {

    }
    private static class Holder {
      private static Singleton6 instance = new Singleton6();
    }
    public static Singleton6 getInstance() {
      return Holder.instance;
    }
  }
```

### 枚举方式

这是一种更巧妙的方法，用java枚举类特性保证单例化与线程安全，但是并不能完全保证懒加载，当访问到静态方法时，也可能会实例化。
```java
  public static enum  Singleton7 {
    INSTANCE;
    private byte[] data = new byte[1024];
    private static void method() {
      //...
    }
    public static Singleton7 getInstance() {
      return INSTANCE;
    }
  }
```
对其使用内部类修缮
```java
  public static class Singleton8 {
    private byte[] data = new byte[1024];
    private Singleton8() {

    }
    private enum Holder {
      INSTANCE;
      private Singleton8 instance;
      Holder() {
        this.instance = new Singleton8();
      }
      private Singleton8 getSingleton() {
        return instance;
      }

    }
    public static Singleton8 getInstance() {
      return Holder.INSTANCE.getSingleton();
    }
  }
```



