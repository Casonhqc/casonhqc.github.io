---
author: Cason
tags:
  - Java基础
share: true
categories:
  - 源码分析
abbrlink: '0'
date: 2023-12-05 21:13:28
modify: 2023-12-05 21:13:28
aliases:
---
### String s = new String("xyz")创建几个实例和变量
#### 一般情况
“xyz”对应一个String实例，new String（）会创建一个String实例，`s`变量指向创建后的实例。两个实例和一个变量。

这里可以看到只执行了一次new方法，**那么另一个String实例在哪创建的？**
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312052125819.png)

还有一个实例在类加载的时候创建。
类加载的解析阶段是把常量池的符号引用替换为直接引用的过程，对于字面量“xyz”会创建一个常量的String实例作为对应。

所以应该理解为：在类加载的解析阶段已经创建了一个String实例，执行代码的时候，又new了一个String实例。

#### JVM优化的情况
按道理这段代码会循环，不断创建String实例，造成的频繁的GC。但是JVM默认开启[[JVM的逃逸分析|逃逸分析]]，newString不再创建新的String实例。
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312052136700.png)

#### 考虑Klass和oop
JVM是基于C++实现的，C++也面向对象，最简单地做法就是每个JAVA类生成一个对应的C++类，但是JVM实际是设计了Klass-oop模型。

klass，一个java类被加载后，他的元信息就是以klass的形式存在。
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312052237534.png)
opp，也就是对象在JVM的存在形式，每次创建一个对象，在JVM内部就会相应的创建一个对应的OOP对象，这其实就是对象头。
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312052238354.png)

这里`Class对象`与元空间的`klass`对应，实例对象创建后，创建的`instanceOopDesc`就是他的头部
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061056734.png)

字符串常量池底层就是Hashtable，往其中添加字符串引用就是通过String的内容和长度生成一个hash值，然后定位到下标，将String实例对应的`instanceOopDesc`封装成`Entry`存储起来

![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061058895.png)

### String s = new String("xyz") 和 s = "xyz"的区别
`String s = new String("xyz")`首先会去字符串常量池找有没有`xyz`对应的对象引用
- **如果没有找到**：
	- 首先创建一个新的String对象和内部的`char`数组对象
	- 将创建的String对象封装为`Entry`存到字符串常量池中
	- 再创建一个String对象，使用相同的char数组，放到堆区
- **如果找到：**
	- 那么省略前两步，直接在堆区创建一个String对象，char数组直接指向已经存在的char数组
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061106095.png)

String s = "xyz" 的执行逻辑，也是先去字符串常量池找：
- **如果找到了**：那么就会直接返回对应的String对象
- **没找到**：
	- 会创建一个String对象和char数组
	- 将创建的String对象封装成`HashtableEnrty`，放到字符串常量池这个hashtable中
	- 返回创建的这个String
这里只创建了一个String对象
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061903096.png)

### 字符串拼接
`String s = s1 + s2`其实底层是`StringBuilder.append`然后在`toString`方法创建一个新的String对象。
这里可以看到调用的是String的构造方法，复制了一份传入的`char[]`，然后创建一个新的String，`value`指向新的字符数组。**但是这里并没有封装成Entry放到字符串常量池中**。
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061906497.png)
所以拼接后的字符串是新建的，而不是字符串常量池的
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061909665.png)

但是如果拼接的字符串加上了`final`修饰，那么就会直接优化。比如`aa + bb`变成`aabb`
### Intern方法
调用`StringBuilder#toString`方法不会驻留字符串，那么如果想要将其驻留，就需要调用`intern`方法，他会创建一个Entry，Entry的value就是指向调用方法的String。
**但是在1.6之前，字符串常量池是在永久代，intern会把字符串实例复制到永久代，所以返回的引用是复制体的引用，而不是原先的引用**
![image.png](https://obsidian-1317277327.cos.ap-chengdu.myqcloud.com/attachment/202312061911725.png)
