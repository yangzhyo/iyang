---
layout: post
title: "你的单例模式实现正确吗？"
date: 2018-11-21 12:24:00
categories: Java
tags: [Java]
---

> 单例模式是大家在日常开发过程中最常用的设计模式之一，其最常见的一种实现如下：

```java
// Multithreaded version
// "Double-Checked Locking" idiom
class Foo { 
  private Helper helper = null;
  public Helper getHelper() {
    if (helper == null) 
      synchronized(this) {
        if (helper == null) 
          helper = new Helper();
      }    
    return helper;
  }
  // other functions and members...
}
```

[双重检查锁定模式（Double-Checked Locking）](https://zh.wikipedia.org/wiki/%E5%8F%8C%E9%87%8D%E6%A3%80%E6%9F%A5%E9%94%81%E5%AE%9A%E6%A8%A1%E5%BC%8F)，懒加载，支持多线程并发，性能优异。看起来很完美，但很不幸，这里暗藏巨坑。

<!-- more -->

# 问题一

先来看一段 JIT 编译出来的代码。

```java
singletons[i].reference = new Singleton();
```

编译成：

```asm
0206106A   mov         eax,0F97E78h
0206106F   call        01F6B210                  ; allocate space for
                                                 ; Singleton, return result in eax
02061074   mov         dword ptr [ebp],eax       ; EBP is &singletons[i].reference 
                                                ; store the unconstructed object here.
02061077   mov         ecx,dword ptr [eax]       ; dereference the handle to
                                                 ; get the raw pointer
02061079   mov         dword ptr [ecx],100h      ; Next 4 lines are
0206107F   mov         dword ptr [ecx+4],200h    ; Singleton's inlined constructor
02061086   mov         dword ptr [ecx+8],400h
0206108D   mov         dword ptr [ecx+0Ch],0F84030h
```

可以看到，对 singletons[i].reference 引用的赋值要早于 Singleton 的构造器调用。这在 Java 现有的内存模型下是完全合法的。C,C++ 亦是如此。

**导致的后果是什么？**

后进来的线程检查到 Helper 引用不为 NULL，直接返回了并未完成初始化的实例对象，其在后续使用过程中，极有可能发生崩溃。

**为什么会这样？**

原因是为了优化性能，编译器可以自由地重排变量的初始化和访问顺序。

更多细节可以参考
* [more detailed description of compiler-based reorderings](http://gee.cs.oswego.edu/dl/cpj/jmm.html)
* [The Java Memory Model](http://www.cs.umd.edu/~pugh/java/memoryModel/)

# 问题二

在多核架构上，如果两个线程运行在不同的处理器上，每个线程针对共享变量，可以在寄存器中拥有自己的 Local Cache, 线程对变量值的更新不一定会实时反映到主存中，导致其他线程对变量的访问出现不一致。

如下图所示：
![https://cdncontribute.geeksforgeeks.org/wp-content/uploads/volatile-keyword-in-java.png](https://cdncontribute.geeksforgeeks.org/wp-content/uploads/volatile-keyword-in-java.png)

# 解决方案

## 静态 Singleton

类的加载机制可以确保 singleton 的正确初始化。

```java
class HelperSingleton {
  static Helper singleton = new Helper();
}
```

## 32 位基元类型 Singleton

32 位基元类型（如 int, float 等）的读写是原子的，特别注意的是 64 位不是原子的（如 double, long 等），详见 [https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html)

```java
// Correct Double-Checked Locking for 32-bit primitives
class Foo { 
  private int cachedHashCode = 0;
  public int hashCode() {
    int h = cachedHashCode;
    if (h == 0) 
    synchronized(this) { // 如果 computeHashCode() 没有副作用，不需要同步块
      if (cachedHashCode != 0) return cachedHashCode;
      h = computeHashCode();
      cachedHashCode = h;
      }
    return h;
  }
  // other functions and members...
}
```

## 显式使用内存屏障

[内存屏障（Memory barrier）](https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C)，也称内存栅栏，内存栅障，屏障指令等，是一类同步屏障指令，是 CPU 或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作。

```c++
// C++ implementation with explicit memory barriers
// Should work on any platform, including DEC Alphas
// From "Patterns for Concurrent and Distributed Objects",
// by Doug Schmidt
template <class TYPE, class LOCK> TYPE *
Singleton<TYPE, LOCK>::instance (void) {
    // First check
    TYPE* tmp = instance_;
    // Insert the CPU-specific memory barrier instruction
    // to synchronize the cache lines on multi-processor.
    asm ("memoryBarrier");
    if (tmp == 0) {
        // Ensure serialization (guard
        // constructor acquires lock_).
        Guard<LOCK> guard (lock_);
        // Double check.
        tmp = instance_;
        if (tmp == 0) {
                tmp = new TYPE;
                // Insert the CPU-specific memory barrier instruction
                // to synchronize the cache lines on multi-processor.
                asm ("memoryBarrier");
                instance_ = tmp;
        }
    return tmp;
}
```

## 使用 Thread Local Storage

每个线程通过维护一个线程本地标记来判断同步是否已经完成。

```java
class Foo {
    /** If perThreadInstance.get() returns a non-null value, this thread
    has done synchronization needed to see initialization
    of helper */
    private final ThreadLocal perThreadInstance = new ThreadLocal();
    private Helper helper = null;
    public Helper getHelper() {
        if (perThreadInstance.get() == null) createHelper();
            return helper;
    }
    private final void createHelper() {
        synchronized(this) {
            if (helper == null)
                helper = new Helper();
        }
        // Any non-null value would do as the argument here
        perThreadInstance.set(perThreadInstance);
    }
}
```

## 使用 Volatile 变量

JDK 1.5 之后，Java 增加了 Volatile 语义，它可以
* 变量的值永远不会被线程本地缓存，所有读写都将直接进入主存;
* 禁止编译器作重排优化，volatile 的读和写建立了一个 happens-before 关系，类似于申请和释放一个互斥锁。

Volatile 的详细介绍：
* [https://www.javamex.com/tutorials/synchronization_volatile.shtml](https://www.javamex.com/tutorials/synchronization_volatile.shtml)
* [https://zh.wikipedia.org/wiki/Volatile变量](https://zh.wikipedia.org/wiki/Volatile变量)
* [https://www.geeksforgeeks.org/volatile-keyword-in-java/](https://www.geeksforgeeks.org/volatile-keyword-in-java/)

```java
// Works with acquire/release semantics for volatile
// Broken under current semantics for volatile
  class Foo {
        private volatile Helper helper = null;
        public Helper getHelper() {
            if (helper == null) {
                synchronized(this) {
                    if (helper == null)
                        helper = new Helper();
                }
            }
            return helper;
        }
    }
```

# 参考
* [http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
