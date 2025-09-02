---
date: 2023-08-07 17:45:52
modify: 2023-08-07 17:45:52
author: Cason
tags:
  - 并发
categories:
  - 文章笔记
share: true
title: 操作集合的fastFail策略
---
[]()### 现象
当我们使用迭代器遍历集合对象时，如果有其他线程在我们遍历时修改元素。或者当前线程在遍历时修改了元素，就会报错`Concurrent Modification Exception`。我理解为这里所谓的`fast`就是提前终止遍历，因为可以预见的是继续遍历下去，线程安全问题依旧存在

### 实现原理
#### 对于集合部分
集合类比如`ArrayList`，会有一个`modCount`记录修改的次数，每次进行增删都会++。
#### 对于迭代器
迭代器有一个`expectedModCount`，每次在调用`next`函数遍历元素时，会先调用`checkForCoModification`检查`ModCount`是否如预期没有被修改，如果被修改直接报错。**感觉就是CAS**

如果我们调用迭代器的`remove`方法，就不会抛异常。因为我们会重写这个方法，在对集合操作后，**又会将迭代器的modCount换成变化后的ModCount，所以在下次的检查中通过**

在调用`next()`之前会先使用`hasNext()`, 判断`cursor`（下一个要返回的变量的下标）是否等于`size()`，如果相等说明已经走完了最后一个元素。
所以遍历集合修改并不一定会报出异常，因为当你删除这个元素后，有可能你的`cursor`和`size`已经相等了，这时候直接退出遍历了。

使用增强`for`遍历集合，**相当于使用迭代器**所以也会报错。

### 使用fast-fail避免抛出异常
#### CopyOnWriteList
当生成一個迭代器時，会有一个`snapshot`快照用来遍历
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202308091656398.png)
**这里是直接记录了所有元素而不是下标，这意味着在遍历过程中，元素个数不会变。而如果进行增删，都会copy一个新的数组**

#### 如果是改呢？
如果是改，会抛出异常
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202308091714450.png)
其实可以理解，因为**我都复制了一个新的集合了，你调用迭代器的修改操作没有意义**
> 这种行为与写时复制的设计哲学一致，该设计着重强调了在并发读取情况下的不变性和安全性，而不是在迭代过程中的灵活修改

#### 增加操作
当我们`add`时需要加锁，这是因为虽然我们是复制新的数组，看起来好像不会覆盖，**但其实并发下会出现多个`index`位置不同元素的副本，而集合只会引用其中一个，所以还是相当于覆盖了。
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202308091718800.png)

#### 删除操作
同样是获取锁，根据`index`将数组分成两部分，分别复制到新的数组中。**如果index等于len-1，那么直接新建一个长度-1的数组**

##### 指定删除某个元素：
先定位下标，如果小于0说明没找到，直接返回。
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202308091748488.png)

1. 如果获得锁之前的副本和现在拿到的副本不一致，说明并发问题，集合已经被修改了。
2. 这时候就再次查找元素o的索引位置。
3. 创建一个新的数组，将o以外的元素复制进去
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202308091750133.png)
