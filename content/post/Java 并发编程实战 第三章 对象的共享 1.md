---
title: "Java 并发编程实战 第三章 对象的共享 1"
date: 2023-10-11T21:11:25+08:00
draft: false
author: zgl

math: false
comments: false
---

- 同步的另一个重要方面：**内存可见性**（**Memory Visibility**）

## 3.1 可见性

- **无法确保**的情况：执行**读**操作的线程能适时地看到其他线程**写入**的值
- 为了确保多个线程之间对**内存写入操作**的可见性，必须使用同步机制

```java
// 程序清单 3-1 在没有同步的情况下共享变量（不要这么做）

public class NoVisibility {
    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        @Override
        public void run() {
            while (!ready)
                Thread.yield();
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            new ReaderThread().start();
            number = 42;
            ready = true;
        }
    }
}
```

- 上例程序可能的执行结果：
  - 输出 42
  - **输出 0**
  - **死循环**
- 原因：**重排序**

> 在没有同步的情况下，**编译器**、**处理器**以及**运行时**等都可能对操作的执行顺序进行一些意想不到的调整。在缺乏足够同步的多线程程序中，要想对**内存操作的执行顺序**进行判断，几乎无法得出正确的结论。

- **只要有数据在多个线程之间共享，就使用正确的同步**

### 3.1.1 失效数据

- 当线程在没有同步的情况下**读取**变量时，可能会得到一个失效值
  - 在程序清单 3-1 中，失效数据可能导致输出错误的值或死循环

```java
// 程序清单 3-2 非线程安全的可变整数类

@NotThreadSafe
public class MutableInteger {
    private int value;

    public int getValue() {
        return value;
    }

    public void setValue(int value) {
        this.value = value;
    }
}
```

- 上例中可能出现失效值问题：如果某个线程调用了 set ，那么另一个正在调用 get 的线程可能会看到更新后的 value 值，也可能看不到

```java
// 程序清单 3-3 线程安全的可变整数类

@ThreadSafe
public class SynchronizedInteger {
    @GuardedBy("this")
    private int value;

    public synchronized int getValue() {
        return value;
    }

    public synchronized void setValue(int value) {
        this.value = value;
    }
}
```

- 如果仅对 set 方法进行同步是**不够**的，调用 get 的线程仍然会看见失效值

### 3.1.2 非原子的 64 位操作

- Java 内存模型要求，变量的读取操作和写入操作都必须是原子操作，除了**非 volatile 类型的 long 和 double 变量**
- 对于非 volatile 类型的 long 和 double 变量，JVM 允许**将 64 位的读操作或写操作分解为两个 32 位的操作**
- 当**读取**一个非 volatile 类型的 long 时，如果对该变量的读操作和写操作在不同的线程中执行，那么*很可能会读取到某个值的高 32 位和另一个值的低 32 位*
- 因此，即使不考虑失效数据问题，在多线程程序中使用共享且可变的 long 和 double 等类型的变量也是不安全的

### 3.1.3 加锁与可见性

- 内置锁可以用于确保某个线程以一种**可预测**的方式来**查看**另一个线程的执行结果
- 为什么在访问某个共享且可变的变量时，要求所有线程在**同一个**锁上同步？
  - 就是为了确保某个线程**写入**该变量的值对于其他线程来说都是**可见的**

### 3.1.4 volatile 变量

- volatile 变量用来确保将变量的**更新**操作**通知**到**其他**线程
- 对于 volatile 变量，编译器和运行时**不会**将该**变量上的操作**和其他内存操作一起**重排序**
  - volatile 变量**不会**被缓存在**寄存器**或者**对其他处理器不可见**的地方
- volatile 变量是一种比 sychronized 关键字**更轻量级**的同步机制
  - 在访问 volatile 变量时**不会加锁**
- 从**内存可见性**的角度来看：
  - **写入** volatile 变量等价于**退出**同步代码块
  - **读取** volatile 变量等价于**进入**同步代码块
- volatile 变量的**正确**使用方式包括：
  - 确保它们自身状态的可见性
  - 确保它们所引用对象的状态的可见性
  - 标识一些重要的程序生命周期事件的发生（如**初始化**或**关闭**）
- volatile 变量常用做某个操作完成、发生中断或者状态的标志
  - **典型用法**：**检查某个状态标记以判断是否退出循环**

```java
// 程序清单 3-4 数绵羊

volatile boolean asleep;
...
while(!asleep)
    countSomeSheep();
```

- 加锁机制**既**可以确保可见性**又**可以确保原子性，而 volatile 变量**只**能确保可见性
