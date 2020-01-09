---
title: ThreadLocal小结
date: 2018-12-21 16:50:08
tags:
		- java
---

# ThreadLocal 小结

在使用ThreadLocal过程中，产生了这样几个疑惑。
* ThreadLocal实例对象和存储值是什么关系？
* ThreadLocalMap中的Entry为什么是弱引用？

<!--more-->

``` java
private void demo() {
	ThreadLocal<String> tr = new ThreadLocal<>();
	tr.set("demo");
}
```

首先，对于每个线程来说，tr对象是均可以被访问到的。
线程内部维持一个ThreadLocalMap对象。
该map对象记录的是ThreadLocal实例对象引用->指。
tr对象是相同的，但是对每个线程来说，其对应的值不一样。

为什么要用弱引用？
我们以main线程调用demo方法为例。
tr对象只在demo内被使用，无任何外部引用，在demo方法调用后，tr对象应当被gc。
但是，如果ThreadLocalMa内部是tr的强引用，那么即时离开demo方法后，线程内仍持有该tr对象强引用，只要线程不被gc，那么该tr对象也不会被gc。