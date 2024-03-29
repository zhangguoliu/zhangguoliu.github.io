---
title: "Java 并发编程实战 第二章 线程安全性"
date: 2023-10-05T20:43:45+08:00
draft: false
author: zgl

math: false
comments: false
---

- 在编写并发代码时，应该始终遵循的**原则**
  - **首先使代码正确运行，然后再提高代码的速度**
  - 这是因为**并发错误是非常难以重现和调试的**

## 什么是线程安全性

- 当多个线程访问某个类时，这个类始终都能表现出**正确的行为**，那么就称这个类是线程安全的
- 在线程安全性的定义中，最核心的概念就是**正确性**
  - 正确性的含义是，某个类的行为与**其规范**完全一致
  - 在良好的规范中，通常会定义各种**不变性条件**（**Invariant**）来约束对象的状态，以及定义各种**后验条件**（**Postcondition**）来描述对象操作的结果
- 在线程安全类中封装了必要的同步机制，因此客户端无须进一步采取同步措施

### 无状态对象一定是线程安全的

```java
// 程序清单 2-1 一个无状态的 Servlet

@ThreadSafe
public class StatelessFactorizer implements Servlet {
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        encodeIntoResponse(resp, factors);
    }
}
```

## 原子性

```java
// 程序清单 2-2 在没有同步的情况下统计已处理请求数量的 Servlet（不要这么做）

@NotThreadSafe
public class UnsafeCountingFactorizer implements Servlet {
    private long count = 0;

    public long getCount() {
        return count;
    }

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        ++count;
        encodeIntoResponse(resp, factors);
    }
}
```

- 非原子操作：包含 **读取-修改-写入** 三个操作
- 在并发编程中，这种由于**不恰当的执行时序**而出现不正确的结果是一种非常重要的情况，它有一个正式的名字：**竞态条件**（**Race Condition**）

### 竞态条件

- 当某个计算的**正确性**取决于**多个线程的交替执行时序**时，就会发生竞态条件
- 现实情况中的竞态条件：星巴克案例

- 最常见的竞态条件类型：**先检查后执行（Check-Then-Act）**
  - 本质：通过一个**可能失效**的观测结果来决定下一步的动作
  - 补充理解：观测结果失效的时间一般为观测结束到执行下一步动作之间的空隙
- **读取-修改-写入** 是另一种竞态条件
  - 要递增一个计数器，必须知道它之前的值，并确保在执行更新的过程中，没有其他线程会修改或使用这个值

- **延迟初始化**中的竞态条件：类似单例模式中的**懒汉式**

```java
// 程序清单 2-3 延迟初始化中的竞态条件（不要这么做）

@NotThreadSafe
public class LazyInitRace {
    private ExpensiveObject instance = null;

    public ExpensiveObject getInstance() {
        if (instance == null)
            instance = new ExpensiveObject();
        return instance;
    }
}
```

### 复合操作

- 复合操作必须是原子的
  - 递增运算：读取-修改-写入
  - 延迟初始化：先检查后执行

### 原子变量类

- 在 java.util.concurrent.atomic 包下

```java
// 程序清单 2-4 使用 AtomicLong 类型的变量来统计已处理请求的数量

@ThreadSafe
public class CountingFactorizer implements Servlet {
    private final AtomicLong count = new AtomicLong(0);

    public long getCount() {
        return count.get();
    }

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        count.incrementAndGet();
        encodeIntoResponse(resp, factors);
    }
}
```

- count 是 final 修饰的
  - 需要在构造方法中进行初始化
  - 变更操作前需要先复制它的一份拷贝

## 加锁机制

### 不变性条件

```java
// 程序清单 2-5 该 Servlet 在没有足够原子性保证的情况下对其最近计算结果进行缓存（不要这么做）

@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
    private final AtomicReference<BigInteger> lastNumber = new AtomicReference<>();
    private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<>();

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        if (i.equals(lastNumber.get()))
            encodeIntoResponse(resp, lastFactors.get());
        else {
            BigInteger[] factors = factor(i);
            lastNumber.set(i);
            lastFactors.set(factors);
            encodeIntoResponse(resp, factors);
        }
    }
}
```

- 当在不变性条件中涉及**多个**变量时，各个变量之间**并不是**彼此独立的，而是**某个变量的值会对其他变量的值产生约束**
- 上例中的不变性条件是：在 lastFactors 中缓存的因数之积应该等于在 lastNumber 中缓存的数值

### 内置锁

- 支持原子性的内置锁机制：**同步代码块**
- **每个** Java 对象都可以用做一个实现同步的锁，称为**内置锁**或**监视器锁**
  - 静态方法则是 **Class 对象**

```java
// 程序清单 2-6 这个 Servlet 能正确地缓存最新的计算结果，但并发性却非常糟糕（不要这么做）

@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
    private final AtomicReference<BigInteger> lastNumber = new AtomicReference<>();
    private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<>();

    public synchronized void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        if (i.equals(lastNumber.get()))
            encodeIntoResponse(resp, lastFactors.get());
        else {
            BigInteger[] factors = factor(i);
            lastNumber.set(i);
            lastFactors.set(factors);
            encodeIntoResponse(resp, factors);
        }
    }
}
```

### 重入

- 内置锁是可重入的
- “重入” 意味着**获取锁的操作的粒度是“线程”**，而不是“调用”
- 重入的一种实现方法：为**每个锁**关联一个**所有者线程**和一个**获取计数值**

```java
// 程序清单 2-7 如果内置锁不是可重入的，那么这段代码将发生死锁

public class Widget {
    public synchronized void doSomething() {
        ...
    }
}

public class LoggingWidget extends Widget {
    public synchronized void doSomething() {
        System.out.println(toString() + ": calling doSomething");
        super.doSomething();
    }
}
```

### 用锁来保护状态

- 如果用同步来协调对某个变量的访问，那么在访问这个变量的**所有**位置上都需要使用同步
- 而且，当使用锁来协调对某个变量的访问时，在访问变量的**所有**位置上都要使用**同一个**锁
- 对于每个包含**多个**变量的**不变性条件**，其中涉及的**所有**变量都需要由**同一个**锁来保护
- 如果只是将每个方法都作为同步方法就行了吗？不行

```java
if (!vector.contains(element))
    vector.add(element);
```

### 活跃性与性能

- 不良并发应用程序：可同时调用的数量，不仅受到可用处理资源的限制，还受到应用程序本身结构的限制
- 通过**缩小同步代码块的作用范围**，兼顾并发性和线程安全性
  - 同步代码块不能过小，即不能破坏原子操作
  - 应该尽量将**不影响**共享状态且执行时间**较长**的操作从同步代码块中**分离**出去
- 当执行时间较长的计算或者可能无法快速完成的操作时（例如，网络 I/O 或控制台 I/O），一定不要持有锁

```java
// 程序清单 2-8 缓存最近执行因数分解的数值及其计算结果的 Servlet

@ThreadSafe
public class CachedFactorizer implements Servlet {
    @GuardedBy("this")
    private BigInteger lastNumber;
    @GuardedBy("this")
    private BigInteger[] lastFactors;
    @GuardedBy("this")
    private long hits;
    @GuardedBy("this")
    private long cacheHits;

    public synchronized long getHits() {
        return hits;
    }

    public synchronized double getCacheHitRatio() {
        return (double) cacheHits / (double) hits;
    }

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = null;
        synchronized (this) {
            ++hits;
            if (i.equals(lastNumber) {
                ++cacheHits;
                factors = lastFactors.clone();
            }
        }
        if (factors == null) {
            factors = factor(i);
            synchronized (this) {
                lastNumber = i;
                lastFactors = factors.clone();
            }
        }
        encodeIntoResponse(resp, factors);
    }
}
```
