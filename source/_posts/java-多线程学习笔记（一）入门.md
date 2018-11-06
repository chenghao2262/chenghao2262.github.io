---
title: java 多线程学习笔记（一）入门
date: 2018-11-01 11:18:14
categories:
		- java多线程学习笔记
tags: 
        - java
        - 多线程
---


# Java 多线程学习笔记（一）入门

## Tread 与 Runnable 的区别
众所周知，实现线程有两种模式，其一继承Thread类，覆盖run方法，其二，实现Runnable接口。前一种体现的模板类设计模式，后者体现的是策略模式，二者并无优劣高低之分，各有各的区别，

```java
// SimpleThread的num是各个线程的私有成员，线程对自己num递增不会影响其他线程，若想改成线程相互影响，则需要加入static
class SimpleThread extends Thread {

    private int num = 0;

    public SimpleThread(String name) {
      super(name);
    }

    @Override
    public void run(){
      while (true) {
        System.out.println( this.getName() + "=" + ++num);
        try {
          TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }

  }
  public void threadShow() {
    IntStream.range(0,5).mapToObj(
        (num) -> new SimpleThread("SimpleThread-" + num)
    ).forEach(Thread::start);
  }

// 传入Runnabel接口，几个线程均执行的是该对象的run方法，因此无需要static修饰num，几个线程对num操作会相互影响。
 class SimpleRunnable implements Runnable{
    private int num = 0;
    @Override
    public void run() {
      while (true) {
        System.out.println( Thread.currentThread().getName() + "=" + ++num);
        try {
          TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }
  }

  public void runnableShow() {
    SimpleRunnable simpleRunnable = new SimpleRunnable();
    IntStream.range(0,5).mapToObj(
        (num) -> new Thread(simpleRunnable,"SimpleRunnable-" + num)
    ).forEach(Thread::start);
  }
```

<!--more-->

## Java 内存结构，Thread与虚拟机栈

![JVM内存结构图](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/JVM%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE.png)
* 程序计数器 线程私有
* java虚拟机栈 线程私有 通过-xss配置
* 本地方法栈 jni调用使用 线程私有
* 堆内存，线程共享，几乎存放java运行期间创建的所有对象，是GC重点关注区域，又称“GC堆”
* 方法区 线程共享
* java 8 元空间

粗略认为Java进程内存大小为 堆内存 + 线程数量*栈内存


## 守护线程
又称后台线程，随jvm退出而退出，当jvm存在一个非守护线程时，jvm不会自动退出。

## Thread sleep 与 yield

sleep使当前线程进入睡眠状态。可以使用TimeUnit来代替sleep方法。
yield是启发式方法，提醒调度器自愿放弃当前CPU资源，当CPU资源不紧张时，可能会忽略。

| item  | sleep |  yield |
|:------:|:-------:|:------:|
| 一定执行 | 是     | 否       |
| 捕获中断 | 是     | 否       |


## Thread 优先级
Thread 优先级也是个hint操作，你不能严重依赖优先级来实现业务逻辑。
线程优先级在1-10之间，且不能超过group的优先级。默认情况下，优先级和父线程优先级一致，通常是5，因为main优先级默认是5。

## **Thread interrupt**

### 线程 interrupt 与 可中断方法
以下方法会使线程进入阻塞状态，调用该线程的interrupt方法，可以打断阻塞。

* Object wait 及其重载方法
* Thread sleep 及其重载方法
* Thread join 及其重载方法
* InterruptibleChannel 的 io 操作
* Selector 的 wakeup 方法
* 其他方法

以上方法又称**可中断方法**。记住打断一个线程**并不等于该线程生命周期结束**，仅仅是**打断了当前线程的阻塞状态**。
一旦线程在阻塞情况下被打断，会抛出InterruptException的异常，该异常就像一个signal信号一样，通知当前线程被打断了。

### isInterrupted 和 Interrupted
isInterrupted。该方法是线程成员方法，对线程Interrupt标识的一个判断，并不会影响该标识发生任何改变。
interrupted。是一个静态方法，调用该方法会擦除线程的interrupt标识。如果线程被打断，那么调用该方法，第一次返回true，随后调用只会返回false，除非线程再次被打断。

> 可中断方法（如sleep）在捕获到中断信号，会重置interrupt标识。

``` java
  public static void noSleep() throws InterruptedException {
    Thread thread = new Thread() {
      @Override
      public void run() {
        while (true) {

        }
      }
    };
    thread.setDaemon(true);
    thread.start();
    TimeUnit.MILLISECONDS.sleep(2);
    System.out.printf("Thread is interrupted ? %s\n",thread.isInterrupted());
    thread.interrupt();
    System.out.printf("Thread is interrupted ? %s\n",thread.isInterrupted());
  }
```
![多线程运行结果一](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C%E4%B8%80.png)

``` java
  public static void withSleep() throws InterruptedException {
    Thread thread = new Thread() {
      @Override
      public void run() {
        while (true) {
          try {
            TimeUnit.MINUTES.sleep(1);
          } catch (InterruptedException e) {
            System.out.printf("I am be interrupted ? %s\n", isInterrupted());
          }
        }
      }
    };
    thread.setDaemon(true);
    thread.start();
    TimeUnit.MILLISECONDS.sleep(2);
    System.out.printf("Thread is interrupted ? %s\n",thread.isInterrupted());
    thread.interrupt();
    TimeUnit.MILLISECONDS.sleep(2);
    System.out.printf("Thread is interrupted ? %s\n",thread.isInterrupted());
  }
```
![多线程运行结果二](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C%E4%BA%8C.png)
``` java
  public static void main(String[] args) throws InterruptedException {
    Thread thread = new Thread(){
      @Override
      public void run() {
        while (true) {
          System.out.println(Thread.interrupted());
        }
      }
    };
    thread.setDaemon(true);
    thread.start();

    TimeUnit.MILLISECONDS.sleep(2);
    thread.interrupt();
  }
```
![多线程运行结果三](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C%E4%B8%89.png)

isInterrupted 和 interrupted均调用同一个本地方法：
```java
private native boolean isInterrupted(boolean clearInterrupted);
```
> 当线程在可中断方法前被中断，则后续的可中断方法将立即中断。

## Thread join
当前线程A join 某个线程B，会使当前线程A处于blocked状态进入等待，直到B线程结束或者达到给定的时间。

## Thread退出

### 线程完成后正常结束
### 手动控制线程退出

线程创建与销毁，具有较高的成本。因此一个线程往往循环执行某些工作，比如心跳检查或是其他工作。当决定退出的时候可以使用线程中断等方式使其退出。
``` java
  public void run() {
    while (!isInterrupted()) {
      // working
    }
  }
```
当其他线程调用interrupt后，可使线程退出。
当然，由于interrupt受中断方法影响，故可以采用volatile修饰的开关flag来控制，volatile关键字是使对象修改后强制在各个线程内进行同步。

``` java
  private volatile boolean closed = false;
  @Override
  public void run() {
    while (!closed && !isInterrupt()) {
      // working
    }
  }
```

### 通过unchecked exception使线程异常退出
可以手动checked异常封装程unchecked exception（Runtime Exception）抛出来结束线程。

### 线程假死
需通过其他工具如jconsole，jstack来分析线程假死原因。


