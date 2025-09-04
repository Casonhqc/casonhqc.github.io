---
date: 2023-07-23 12:35:21
modify: 2023-07-23 12:35:21
author: Cason
tags:
categories:
  - 文章笔记
share: true
title: ThreadLocal内部实现
---

# 各个类之间的关系：
`Thread`里有一个`ThreadLocalMap`，可能一开始是空的，但是当`ThreadLocal`开始`set`一些`value`时，这个`ThreadLocalMap`就会被创建，然后`ThreadLocal`就是`key`，`value`就是传进来的value。

# 调用set的流程
下面这是流程图
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202307231240322.png)

# 调用get的流程
`ThreadLocal`的`get`方法还是去操作`Thread`，会去拿到当前的`Thread`，然后再取出里面的`ThreadLocalMap`，用`this`自己本身去匹配对应的`value`

# 多个线程和一个ThreadLocal（线程隔离下的上下文对象）
## 用保存登录信息的案例来分析：
每个用户登录就是一个独立的线程，但是在代码中，我们都是用一个`ThreadLocal`去存信息的。这是因为`ThreadLocal`只是一个钩子，每个线程有自己的`ThreadLocalMap`存放信息，当不同的线程到来，那么自然钓出的是不同的信息。

**重新再理解一下线程隔离下的上下文对象这个名词**：
这里的上下文比如`UserHolder`对象， 他在不同的线程钓出了不同的`Map`

## ThreadLocal的内存泄漏问题
`ThreadLocalMap`的`entry`键值对的键使用的是弱引用，弱引用可以在强引用断开的情况下，继续连接对象，直到下次的GC回收。
这里使用弱引用就是为了保证`ThreadLocal`的强引用被断开时，那么如果在`entry`这里的`ThreadLocal`设置为强引用，这里是必须得写代码手动跟着断开。但是是弱引用可以保证在下次`gc`时自动回收。**强引用是不受gc主导的**

但是我们没有考虑到`key`对应的`value`在`key`被回收后没有被清除，这里就会造成内存泄漏。所以我们需要用`finally`来调用remove方法，根据key删除value。

