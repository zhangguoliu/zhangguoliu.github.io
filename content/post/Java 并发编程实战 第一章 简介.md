---
title: "Java 并发编程实战 第一章 简介"
date: 2023-10-05T20:09:26+08:00
draft: false
author: zgl

math: false
comments: false
---

## 线程的优势

- 发挥多处理器的强大能力
- 建模的简单性
- 异步事件的简化处理
- 响应更灵敏的用户界面

## 线程带来的风险

- 安全性问题

```java
// 程序清单 1-1

@NotThreadSafe
public class UnsafeSequence {
    private int value;

    public int getNext() {
        return value++;
    }
}
```

```java
// 程序清单 1-2

@ThreadSafe
public class SafeSequence {
    private int value;

    public synchronized int getNext() {
        return value++;
    }
}
```

- 活跃性问题
  - 单线程：死循环
  - 多线程：**饥饿**、**死锁**、**活锁**
- 性能问题
