---
layout: post
title: Java容器类学习笔记
category: 技术
tags: Java Collection
keywords: Java Collection
description: 
---

![collection](https://github.com/bruceFend/brucefend.github.io/blob/master/public/img/Java-collection.JPG)

### 线程不安全
---

#### List接口
- AbstractList [继承`AbstractCollection`]
    - AbstractSequentialList
        - LinkedList [**双向链表**]
    - ArrayList [**动态数组**]

0. `List`表示有顺序或位置的数据集合。`ArrayList`同时还实现了`RandomAccess`接口，该接口无任何代码，用于**声明类的一种属性**
1. `ArrayList`中有个`modCount`字段，记录**修改结构次数**(添加、插入和删除)
2. `ArrayList`非线程安全，`Vector`是Java最早实现(JDK1.0)的容器类之一，内部使用`synchronized`实现线程安全
3. `LinkedList`同时实现了`Deque`队列接口，可以作为**队列**、**栈**、**双端队列**使用。
4. `LinkedList`中会抛出异常的: `add`,`element`,`remove`方法和返回`null`,`false`的: `offer`,`peek`,`poll`，表示在尾部添加数据、查看头部元素和删除头部元素

#### Map接口

- AbstractMap
    - TreeMap [**有序**][**红黑树**]
    - EnumMap [**有序**][基于**数组**]
    - HashMap [**无序**][**数组**+**单向链表**]
        - LinkedHashMap [**有序**][**双向链表**]

0. `HashMap`允许`null`作为`key`和`value`，而`HashTable`不允许
1. `HashMap`不是线程安全，`HashTable`通过`synchronized`实现线程安全，
高并发情况下建议使用`ConcurrentHashMap`，其内部使用的是**分段锁**机制。
2. `ConcurrentHashMap`继承`AbstractMap`抽象类，并实现`ConcurrentMap`接口
3. 用`LinkedHashMap`可以轻易地实现LRU缓存

#### Set接口

- AbstractSet [继承`AbstractCollection`]
    - TreeSet [**有序**][基于TreeMap]
    - EnumSet [**抽象类**[基于**位向量**]
    - HashSet [**无序**][基于HashMap]
        - LinkedHashSet [**有序**][**基于LinkedHashMap**]

0. `HashMap`、`HashSet`的共同实现机制是**哈希表**，共同的特点是**没有顺序**。它们都有一个
能保持顺序的对应类`TreeMap`和`TreeSet`，这两个类的共同实现机制是**排序二叉树**。
(`TreeMap`内部使用的**红黑树**是一种**大致平衡**的排序二叉树)
1. `EnumSet`是抽象类，

#### Deque接口
- LinkedList
- ArrayDeque [双端队列][基于数组]

0. `ArrayDeque`同`LinkedList`一样长度无限制，也有栈的基本方法push/pop/peek
1. `ArrayDeque`没有实现`List`接口，不能按索引位置进行操作

#### Queue接口：堆与优先级队列
堆的应用：
- 优先级队列
- 求前（/第）K个最大（/小）的元素
- 求中位数


### 线程安全
---

#### 线程安全的思路：
- 锁: `synchronized`或`Reentrant-Lock`
- 循环CAS
- 写时复制(CopyOnWrite)

#### 并发容器：

**CopyOnWritArrayList** / **CopyOnWriteArraySet**

0. `CopyOnWriteArraySet`内部通过`CopyOnWriteArrayList`实现，基于内部数组。
1. 内部数组声明为`volatile`关键字，以保证内存可见性。
2. 多个栈不能同时写，写操作要先获得`ReentrantLock`锁对象。
3. 由于每次修改都需要创建新数组，复制原数组的所有内容，因此不适合数组很大且频繁修改的场景。

**ConcurrentHashMap**

0. 是`HashMap`的并发版本，读操作支持完全并行，写操作支持一定程度的并行，并且支持一些**原子复合操作**。
1. 与同步容器`Collections.synchronizedMap`相比，迭代时不用加锁，不会抛出`ConcurrentModificationException`，但可能反映最新修改也可能不反映(取决于修改和迭代的时间)
2. 跟`HashTable`一样，不允许`null`作为key/value

**ConcurrentSkipListMap** / **ConcurrentSkipListSet**

0. 基于跳表实现(**链表**+**多层索引**)，有序，无锁非阻塞(循环CAS)，完全并行(包括写)。

**无锁非阻塞并发队列**
- `ConcurrentLinkedQueue`
- `ConcurrentLinkedDeque`

**普通阻塞队列**

> 常用于**生产者/消费者模式**

- 基于数组的`ArrayBlockingQueue`
- 基于链表的`LinkedBlockingQueue`
- 基于链表的`LinkedBlockingDeque`

**优先级阻塞队列**
- `PriorityBlockingQueue`

### 其他
---

#### Collections类

以静态方法的方式提供：
1) 对容器接口对象进行操作
    - 查找和替换
    - 排序和调整顺序
    - 添加和修改
2) 返回一个容器接口对象
    - 适配器：将其他类型的数据转换为容器接口对象
        - 空容器方法
        - 单一对象方法
        - 其他适配方法
    - 装饰器：修饰一个给定容器接口对象，增加某种性质
        - 写安全
        - 类型安全
        - 线程安全

