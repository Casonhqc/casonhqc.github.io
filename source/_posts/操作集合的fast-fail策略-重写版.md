---
date: 2025-10-14 00:00:00
modify: 2025-10-14 00:00:00
author: Cason
tags:
  - 并发
  - 集合框架
  - Java
categories:
  - 文章笔记
share: true
title: 为什么遍历集合时不能修改？揭秘Java的fast-fail机制
---

## 一、从一个"诡异"的异常说起

### 1.1 见证一个令人困惑的场景

来看这样一段代码，看起来再正常不过了：

```java
List<String> list = new ArrayList<>();
list.add("Java");
list.add("Python");
list.add("Go");

for (String lang : list) {
    if ("Python".equals(lang)) {
        list.remove(lang);  // 删除Python
    }
}
```

**你觉得会发生什么？** 程序正常运行？还是...

```
Exception in thread "main" java.util.ConcurrentModificationException
    at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
    at java.util.ArrayList$Itr.next(ArrayList.java:859)
```

**卧槽，是不是很神奇？** 我们既没有多线程，也没有并发操作，就是简简单单地遍历一个集合并删除元素，为什么会抛出 `ConcurrentModificationException`？

### 1.2 更诡异的现象

你可能会想：那我换个位置试试。我们把删除的元素改成 "Go"（最后一个元素）：

```java
for (String lang : list) {
    if ("Go".equals(lang)) {
        list.remove(lang);  // 删除最后一个
    }
}
```

**见证奇迹的时刻** —— 这次居然不报错了！程序正常结束！

**这就带来了一个关键问题**：同样是遍历中删除元素，为什么删除中间元素就报错，删除最后一个就没事？这背后到底隐藏着什么机制？

### 1.3 问题的本质

其实，这个异常的名字已经出卖了一切：`ConcurrentModificationException` —— **并发修改异常**。

**你可能会问**：我明明是单线程啊，哪来的并发？

说穿了，这里的"并发"不是指多线程并发，而是指 **遍历和修改这两个操作在"并发"进行**。一边遍历，一边改集合的结构，这就是 Java 集合框架要防止的场景。

**那么问题来了**：为什么 Java 要这么设计？为什么不能一边遍历一边修改？这个限制是保护机制还是设计缺陷？

要回答这个问题，我们得从头说起。

## 二、追本溯源：为什么需要 fast-fail？

### 2.1 并发修改的本质危险

在解释 fast-fail 之前，我们先来看看：**如果允许遍历中修改，会发生什么？**

#### 多线程场景的灾难

想象这样一个场景：

```java
List<String> list = new ArrayList<>();
// 线程A：正在遍历
for (String item : list) {
    // 处理item
}

// 线程B：同时在删除
list.remove(0);
```

当线程 A 遍历到第 3 个元素时，线程 B 突然删除了第 1 个元素。**这意味着什么？**

- 原本下标为 3 的元素，现在变成了下标 2
- 线程 A 的迭代器还以为自己在访问下标 3，但实际上访问的是下标 4（原来的第 5 个元素）
- **结果**：第 4 个元素被跳过了，或者更糟糕的是，访问越界导致崩溃

#### 单线程也不安全

**你可能会说**：那我单线程总行了吧？

其实也不行。来看这个例子：

```java
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("C");

Iterator<String> it = list.iterator();
String first = it.next();  // 获取A
list.remove(0);            // 删除A
String second = it.next(); // 想获取B，实际获取的是C
```

**问题在哪？** 迭代器内部维护了一个 `cursor`（游标），指向下一个要返回的元素的下标。当我们调用 `list.remove(0)` 后：

- 集合的实际结构变了：["B", "C"]
- 但迭代器的 cursor 还是指向下标 1，以为要取第二个元素
- 结果取到的是 "C"，**"B" 被跳过了**

**本质上就是**：迭代器持有的状态和集合的实际状态不一致了。

### 2.2 fast-fail 的设计初衷

分析到这里，你应该明白了：**允许遍历中修改，会导致数据不一致甚至程序崩溃。**

那怎么办呢？Java 设计者给出了一个简单粗暴的方案：

> **与其让错误悄悄发生，不如立即暴露问题！**

这就是 **fail-fast（快速失败）** 的核心思想。

类比一下：就像火灾报警器，一旦检测到烟雾，立即发出警报，而不是等火烧起来才发现。虽然可能有误报，但总比火灾蔓延要好。

#### Fail-Fast vs Fail-Safe

在软件设计中，有两种错误处理哲学：

| 策略 | 思想 | 优点 | 缺点 | 典型代表 |
|-----|------|------|------|---------|
| **Fail-Fast** | 一旦发现问题立即抛异常 | 及时暴露错误，避免错误蔓延 | 程序会中断 | ArrayList、HashMap |
| **Fail-Safe** | 容忍问题，想办法绕过 | 程序不会中断 | 可能掩盖问题，数据可能不一致 | CopyOnWriteArrayList |

Java 集合框架在大多数场景下选择了 **Fail-Fast**，因为：

1. **明确的错误提示**：让开发者意识到代码有问题
2. **防止数据不一致**：与其返回错误的结果，不如直接报错
3. **性能考虑**：检测机制简单，开销小

**你可能会问**：那 Fail-Safe 的方案是什么样的？别急，我们后面讲 `CopyOnWriteArrayList` 时会详细分析。

### 2.3 核心问题：如何检测到修改？

**现在让我们来看看**最关键的技术问题：如何检测到集合被修改了？

想一想，我们需要满足这些条件：

1. **检测开销要小**：不能因为检测而严重影响性能
2. **检测要准确**：不能漏掉真正的并发修改
3. **实现要简单**：不能过度复杂化

**说穿了**，就是要找一个简单高效的方式，让迭代器知道"在我遍历期间，集合有没有被改过"。

**那么问题来了**：你会怎么设计这个检测机制？

答案揭晓：**版本号！**

## 三、揭开面纱：fast-fail 的实现机制

### 3.1 核心机制：modCount 计数器

要说 fast-fail 的实现，必须先聊聊 ArrayList 的一个隐藏属性：`modCount`。

**这是什么？** 说穿了，就是一个**修改计数器**，记录集合被修改的次数。

来看 ArrayList 的源码：

```java
public class ArrayList<E> extends AbstractList<E> {
    // 从AbstractList继承的字段
    protected transient int modCount = 0;

    public boolean add(E e) {
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        modCount++;  // 每次添加，modCount++
        return true;
    }

    public E remove(int index) {
        rangeCheck(index);
        modCount++;  // 每次删除，modCount++

        E oldValue = elementData(index);
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        elementData[--size] = null;
        return oldValue;
    }

    public E set(int index, E element) {
        rangeCheck(index);
        E oldValue = elementData(index);
        elementData[index] = element;
        // 注意：set方法不会增加modCount！
        return oldValue;
    }
}
```

**看出规律了吗？**

- 每次 `add()`，modCount++
- 每次 `remove()`，modCount++
- 每次 `clear()`，modCount++
- 但是 `set()` **不会**增加 modCount（因为没有改变结构）

**类比一下**：modCount 就像是集合的"版本号"，每次结构性修改（增删），版本号就升级一次。

### 3.2 迭代器的检查站：expectedModCount

有了 modCount，下一步就是让迭代器记住"我期望的版本号"。

来看迭代器的实现：

```java
private class Itr implements Iterator<E> {
    int cursor;       // 下一个要返回的元素的下标
    int lastRet = -1; // 上一个返回的元素的下标
    int expectedModCount = modCount;  // 记住创建迭代器时的modCount

    public boolean hasNext() {
        return cursor != size;
    }

    public E next() {
        checkForComodification();  // 关键：每次next之前都检查
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

**分析到这里，貌似有了思路**：

1. 创建迭代器时，记录当前的 `modCount` 到 `expectedModCount`
2. 每次调用 `next()` 前，检查 `modCount` 是否等于 `expectedModCount`
3. 如果不相等，说明集合被修改过，立即抛出异常

**其实就是 CAS 的思想**：Compare And Swap，比较当前值和期望值是否一致。

#### 为什么在 next() 而不是 hasNext() 检查？

**你可能会问**：为什么检查放在 `next()` 而不是 `hasNext()` 里？

看 `hasNext()` 的实现：

```java
public boolean hasNext() {
    return cursor != size;
}
```

**说穿了**：`hasNext()` 只是简单比较下标和 size，不涉及真正的元素访问。如果在这里检查，可能会有不必要的检测开销。

而 `next()` 才是真正要返回元素的时候，这时检查最合适：**要么成功返回，要么立即失败。**

### 3.3 为什么迭代器的 remove() 不会报错？

**现在让我们来看看**一个有趣的现象：

```java
// 这样会报错
for (String item : list) {
    list.remove(item);  // ConcurrentModificationException
}

// 这样不会报错
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String item = it.next();
    it.remove();  // 正常工作
}
```

**你可能会问**：同样是删除，为什么迭代器的 `remove()` 就可以？

来看源码：

```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();  // 先检查一次

    try {
        ArrayList.this.remove(lastRet);  // 调用ArrayList的remove
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;  // 关键：同步expectedModCount
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

**本质上就是**：迭代器知道自己改了集合，所以主动更新 `expectedModCount`。

类比一下：就像你修改了文档，Word 会自动保存新版本号。下次检查时，版本号就对上了。

#### 为什么增强 for 循环会报错？

**你可能会好奇**：我用的是增强 for 循环啊，为什么也报错？

```java
for (String item : list) {
    list.remove(item);  // 报错
}
```

**其实特别简单**：增强 for 循环的本质就是迭代器的语法糖。上面的代码等价于：

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String item = it.next();
    list.remove(item);  // 注意：这里调用的是list的remove，不是迭代器的
}
```

**看出问题了吗？** 我们调用的是 `list.remove()`，不是 `it.remove()`，所以 `expectedModCount` 没有同步更新。

### 3.4 一个神奇的边界情况

**卧槽，是不是很神奇？** 我们开头说过，删除最后一个元素不报错：

```java
List<String> list = new ArrayList<>();
list.add("Java");
list.add("Python");
list.add("Go");

for (String lang : list) {
    if ("Go".equals(lang)) {
        list.remove(lang);  // 不报错！
    }
}
```

**为什么？** 难道 fast-fail 失效了？

**其实可以理解**：来看执行流程。

假设遍历到 "Go" 时：
- `cursor = 2`（指向下标2，也就是 "Go"）
- `size = 3`

当我们调用 `list.remove("Go")` 后：
- `modCount++`（被修改了）
- `size = 2`（元素个数减少）

下一次循环时，先调用 `hasNext()`：

```java
public boolean hasNext() {
    return cursor != size;  // 2 != 2，返回false
}
```

**返回 false，循环退出！** 根本没有机会调用 `next()`，所以也就没有机会执行 `checkForComodification()`。

**说穿了就是**：还没来得及检查就退出了。

#### 对比：删除倒数第二个元素

如果我们删除 "Python"（倒数第二个）呢？

假设遍历到 "Python" 时：
- `cursor = 1`
- `size = 3`

删除后：
- `size = 2`

下次循环：
- `hasNext()` 返回 `true`（1 != 2）
- 调用 `next()`
- 执行 `checkForComodification()`
- **发现 modCount 被改了，抛异常！**

**分析到这里**，你应该完全理解 fast-fail 的工作机制了。

## 四、避坑指南：如何安全地遍历修改集合

### 4.1 方案一：使用迭代器的修改方法

既然问题出在 `modCount` 和 `expectedModCount` 不同步，那最简单的方案就是：**用迭代器自己的修改方法。**

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String lang = it.next();
    if ("Python".equals(lang)) {
        it.remove();  // 用迭代器的remove
    }
}
```

**优点**：
- 简单直接，不会抛异常
- 性能好，没有额外开销

**局限性**：
- 只支持删除，不支持添加
- 只能删除刚刚 `next()` 返回的元素
- 不支持在一次 `next()` 后多次调用 `remove()`

```java
it.remove();  // OK
it.remove();  // IllegalStateException：必须先调用next()
```

### 4.2 方案二：CopyOnWriteArrayList 的巧妙设计

**现在让我们来看看**另一个思路：如果禁止遍历中修改太不方便，能不能有个更优雅的方案？

**这就带来了一个关键问题**：有没有一种集合，既支持并发读写，又不会抛 `ConcurrentModificationException`？

答案是：**CopyOnWriteArrayList（写时复制列表）**

#### 4.2.1 写时复制的核心思想

**说穿了**，CopyOnWriteArrayList 的设计哲学就是：

> **读写分离：读操作不加锁，写操作复制新数组。**

类比一下：就像拍照留念，照片不会因为现场变化而改变。你遍历的是"照片"（快照），我修改的是"现场"（实际集合），互不干扰。

来看源码：

```java
public class CopyOnWriteArrayList<E> {
    private transient volatile Object[] array;  // 注意：volatile保证可见性

    final Object[] getArray() {
        return array;
    }

    final void setArray(Object[] a) {
        array = a;
    }
}
```

**本质上就是**：用一个 `volatile` 数组持有所有元素。

#### 4.2.2 迭代器的快照机制

来看迭代器的实现：

```java
static class COWIterator<E> implements ListIterator<E> {
    private final Object[] snapshot;  // 快照
    private int cursor;

    COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;  // 直接保存数组引用
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public E next() {
        if (!hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];  // 从快照读取
    }
}
```

**看到了吗？** 迭代器直接保存了整个数组的引用！

**这意味着什么？**

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("A");
list.add("B");
list.add("C");

Iterator<String> it = list.iterator();  // 此时快照：["A", "B", "C"]

list.add("D");  // 写操作，会复制新数组

while (it.hasNext()) {
    System.out.println(it.next());  // 还是遍历快照["A", "B", "C"]
}
// 输出：A、B、C，不包括D
```

- 创建迭代器时，获取当前数组的引用
- 后续的增删操作，都会创建新数组
- 迭代器依然遍历旧数组（快照）
- **遍历过程中，元素个数不会变**

#### 4.2.3 为什么迭代器不支持修改？

**你可能会问**：既然这么安全，为什么迭代器不提供 `remove()`、`add()` 方法？

```java
public void remove() {
    throw new UnsupportedOperationException();  // 直接抛异常
}

public void add(E e) {
    throw new UnsupportedOperationException();
}
```

**其实可以理解**：快照是只读的，修改没有意义。

类比一下：你拿着一张照片，能修改照片里的人吗？不能。照片是静态的历史记录，修改要在现场进行。

**本质上就是**：与写时复制的设计哲学一致。该设计着重强调了在并发读取情况下的不变性和安全性，而不是在迭代过程中的灵活修改。

如果你要修改，应该直接操作 list 本身：

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String item = it.next();
    if (condition) {
        list.remove(item);  // 直接操作list，不会影响迭代
    }
}
```

#### 4.2.4 写操作的加锁机制

**现在让我们来看看**写操作是如何实现的。

**你可能会问**：既然都复制新数组了，为什么还要加锁？

来看 `add()` 的源码：

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();  // 加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);  // 复制新数组
        newElements[len] = e;  // 添加元素
        setArray(newElements);  // 替换数组引用
        return true;
    } finally {
        lock.unlock();
    }
}
```

**为什么要加锁？** 想象这样的场景：

```java
// 线程A：添加元素X
Object[] arr1 = getArray();  // 获取当前数组
Object[] new1 = Arrays.copyOf(arr1, arr1.length + 1);
new1[arr1.length] = X;
setArray(new1);  // 设置新数组

// 线程B：同时添加元素Y
Object[] arr2 = getArray();  // 获取当前数组（和arr1一样）
Object[] new2 = Arrays.copyOf(arr2, arr2.length + 1);
new2[arr2.length] = Y;
setArray(new2);  // 设置新数组，覆盖了new1！
```

**看出问题了吗？**

- 两个线程都基于同一个旧数组创建副本
- 都添加了一个元素
- 但最终只有一个数组被保留（后执行的覆盖先执行的）
- **元素 X 或 Y 其中一个丢失了！**

**本质上就是**：避免"丢失更新"问题。虽然我们复制了新数组，但多个线程可能基于同一个旧数组创建多个新副本，最终只有一个会生效。

**说穿了**：加锁是为了保证写操作的原子性，而不是为了保护数组本身。

#### 4.2.5 删除操作的优化细节

**分析到这里**，我们再来看看删除操作的实现细节。

##### 按索引删除

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;

        if (numMoved == 0)  // 删除最后一个元素
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 分段复制：index前 + index后
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index, numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

**看出优化了吗？**

- 如果删除最后一个元素（`numMoved == 0`），直接复制前 n-1 个元素
- 否则，分两段复制：index 前的部分 + index 后的部分

**这种设计**避免了不必要的数组操作，提升了性能。

##### 按元素删除

```java
public boolean remove(Object o) {
    Object[] snapshot = getArray();  // 先获取快照
    int index = indexOf(o, snapshot, 0, snapshot.length);

    if (index < 0) {
        return false;  // 没找到，直接返回
    } else {
        return remove(o, snapshot, index);  // 找到了，执行删除
    }
}

private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;

        if (snapshot != current) {  // 并发问题：数组已经被修改
            // 重新查找元素位置
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i;
                    break;
                }
            }
            if (index >= len) return false;  // 元素已被删除
        }

        // 创建新数组，排除要删除的元素
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1, newElements, index, len - index - 1);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

**这里有个巧妙的设计**：

1. **先定位元素位置**（不加锁，用快照查找）
2. **获取锁后，检查快照是否过期**（`snapshot != current`）
3. 如果过期，说明有并发修改，**重新查找元素位置**
4. 创建新数组，排除要删除的元素

**为什么要这样设计？**

- 定位元素是费时操作（可能要遍历整个数组）
- 先不加锁定位，可以减少锁的持有时间
- 但获取锁后要再次验证，防止并发问题

**分析到这里**，你应该能感受到：CopyOnWriteArrayList 的每个细节都体现了对性能的极致追求。

#### 4.2.6 适用场景与权衡

**你可能会问**：CopyOnWriteArrayList 这么好，为什么不都用它？

**其实特别简单**：因为有代价。

| 方面 | ArrayList | CopyOnWriteArrayList |
|-----|-----------|---------------------|
| **读性能** | O(1)，快速 | O(1)，快速，且无锁 |
| **写性能** | O(1) 或 O(n) | **O(n)，每次都要复制整个数组** |
| **内存开销** | 只有一个数组 | **写时需要双倍内存** |
| **迭代安全** | fast-fail | **完全安全，不会抛异常** |
| **适用场景** | 读写均衡 | **读多写少** |

**典型场景**：

✅ **适合用 CopyOnWriteArrayList**：
- 监听器列表（添加删除很少，遍历通知很频繁）
- 黑白名单（配置很少改，检查很频繁）
- 缓存数据（数据更新不频繁，读取很频繁）

❌ **不适合用 CopyOnWriteArrayList**：
- 频繁增删的列表
- 数据量很大的集合（复制开销大）
- 实时性要求高的场景（可能读到旧数据）

**说穿了**：CopyOnWriteArrayList 是用空间换时间，用写的代价换读的安全。

### 4.3 方案三：其他选择

除了上面两种方案，还有其他选择：

#### 使用普通 for 循环（倒序删除）

```java
for (int i = list.size() - 1; i >= 0; i--) {
    if (condition) {
        list.remove(i);  // 不会影响后续元素的索引
    }
}
```

**为什么倒序删除可以？** 因为删除后面的元素不会影响前面元素的索引。

#### 使用 removeIf（Java 8+）

```java
list.removeIf(item -> "Python".equals(item));
```

**本质上**：内部使用了正确的迭代器删除逻辑。

#### 收集后批量删除

```java
List<String> toRemove = new ArrayList<>();
for (String item : list) {
    if (condition) {
        toRemove.add(item);
    }
}
list.removeAll(toRemove);
```

**代价**：需要额外的集合和两次遍历。

## 五、架构升华：从 fast-fail 看 Java 集合设计哲学

### 5.1 设计模式的体现

**分析到这里**，让我们跳出具体实现，看看 Java 集合框架体现了哪些设计智慧。

#### 迭代器模式：职责分离

```
集合（ArrayList）：负责存储和管理元素
迭代器（Iterator）：负责遍历和访问元素
```

**这种分离的好处**：

1. **解耦**：遍历逻辑和存储逻辑互不干扰
2. **多样性**：可以有多种迭代方式（正向、反向、过滤等）
3. **安全性**：迭代器可以封装状态检查逻辑

#### 快照模式：数据隔离

CopyOnWriteArrayList 体现了快照模式：

```
写操作：修改最新版本
读操作：访问历史快照
```

**这种模式的优势**：

1. **无锁读取**：读操作完全不需要加锁
2. **数据一致性**：每个迭代器看到的都是完整一致的快照
3. **并发友好**：读写互不阻塞

#### 防御式编程：及早失败

fast-fail 是防御式编程的典型案例：

```
与其返回错误的结果，不如立即抛出异常
```

**这体现了**：宁可程序崩溃，也不要静默地产生错误数据。

### 5.2 权衡的艺术

**本质上**，软件设计就是不断权衡的过程。

#### ArrayList vs CopyOnWriteArrayList

| 权衡点 | ArrayList | CopyOnWriteArrayList |
|-------|----------|---------------------|
| **性能 vs 安全** | 性能优先，检测开销小 | 安全优先，写时复制开销大 |
| **便利 vs 正确** | 禁止遍历中修改 | 允许遍历中修改 |
| **内存 vs 并发** | 内存占用小 | 需要双倍内存 |

**没有银弹**，只有最适合的方案：

- **读写均衡、单线程为主**：ArrayList
- **读多写少、并发场景**：CopyOnWriteArrayList
- **高并发读写**：ConcurrentHashMap（如果是 Map 结构）

### 5.3 核心要点回顾

让我们回顾一下这篇文章的核心知识点：

#### fast-fail 的本质

1. **目的**：防止遍历和修改并发进行，避免数据不一致
2. **实现**：通过 `modCount` 和 `expectedModCount` 实现版本校验
3. **时机**：在迭代器的 `next()` 方法中检查
4. **特例**：迭代器自己的 `remove()` 会同步更新 `expectedModCount`

#### 安全遍历修改的方法

| 方法 | 适用场景 | 优点 | 缺点 |
|-----|---------|------|------|
| **迭代器 remove()** | 简单删除 | 性能好，无额外开销 | 只支持删除 |
| **CopyOnWriteArrayList** | 读多写少 | 完全无锁读取 | 写操作开销大 |
| **倒序删除** | 简单场景 | 实现简单 | 不够优雅 |
| **removeIf** | Java 8+ | 代码简洁 | 需要 Java 8+ |

#### CopyOnWriteArrayList 的精髓

1. **写时复制**：每次写操作都创建新数组
2. **快照隔离**：迭代器遍历历史快照，不受后续修改影响
3. **读写分离**：读操作无锁，写操作加锁
4. **适用场景**：监听器列表、黑白名单、读多写少的缓存

### 5.4 思考题

最后，给你留几个思考题：

1. **HashMap 的迭代器也是 fast-fail 吗？实现原理是否相同？**

   提示：HashMap 也有 `modCount`，但并发场景下的问题更复杂。

2. **CopyOnWriteArrayList 真的适合所有读多写少的场景吗？**

   提示：考虑数据量、实时性要求、内存限制。

3. **如果让你设计一个支持遍历中修改的集合，你会怎么做？**

   提示：可以参考数据库的 MVCC（多版本并发控制）机制。

4. **为什么 ArrayList 的 set() 方法不增加 modCount？**

   提示：想想"结构性修改"和"元素修改"的区别。

---

## 小结

从一个看似简单的 `ConcurrentModificationException` 出发，我们深入剖析了 Java 集合框架的 fast-fail 机制：

- **Why**：为什么需要 fast-fail？—— 防止并发修改导致数据不一致
- **How**：如何实现 fast-fail？—— 通过 modCount 版本号校验
- **What**：如何安全地遍历修改？—— 迭代器 remove()、CopyOnWriteArrayList、removeIf 等方案

**说穿了**，fast-fail 不是设计缺陷，而是一种防御机制。理解它的原理，才能写出更安全、更高效的代码。

而 CopyOnWriteArrayList 则为我们展示了另一种设计哲学：**用空间换时间，用写的代价换读的安全。** 它的巧妙之处不在于多么复杂的算法，而在于对业务场景的深刻理解和对性能与安全的精准权衡。

**本质上**，这些设计都在回答一个核心问题：**如何在性能、安全和便利之间找到最佳平衡点？** 这也是我们在实际开发中需要不断思考的问题。
