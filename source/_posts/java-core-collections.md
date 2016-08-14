---
title: 集合总结
date: 2016-04-19 15:50:37
toc: true
tags:
- Java
- 集合
categories:
- Java基础

---

在实现方法时，选择不同的数据结构会导致其实现风格以及性能存存着很大差异，JDK帮我们实现了传统的数据结构，我们称之为collection(集合)。
collection（集合）表示一组对象，这些对象也称为collection的元素。一些collection允许有重复的元素，而另一些而不允许。一些collection是有序的，而另一些则是无序的。

# 接口

## Collection接口

Collection接口是Java集合的根接口，JDK不提供此接口的任何直接实现：它提供更具体的子接口（如Set和List）实现。
Collection接口扩展了Iterable接口，因此，对于标准类库中的任何集合都可以使用"for each"循环。

## Iterator接口

Iterator接口包含如下方法：

```
public interface Iterator<E> {
    boolean hasNext();
    E next();
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```
* 通过反复调用next()方法，可以逐个访问集合中的每个元素，但是如果到了集合的末尾，next()方法将抛出一个NoSuchElementException。因为在调用next之前需要调用hasNext()方法。
* 元素被访问的顺序取决于集合类型。如果对ArrayList进行迭代，迭代器将从索引0开始，每迭代一次，索引值加1。然而如果访问HashSet中的元素，每个元素会按照某种随机的次序出现。虽然可以确定在迭代过程中能够遍历到集合中的所有元素，但却无法预知元素被访问的次序。
* remove()方法将删除上次调用next()方法时返回的值。如果调用remove()之前没有调用next()将会抛出IllegalStateException异常。如果想删除相邻的两个元素，不能直接这样调用：

```
it.remove();
it.remove(); // Error!
```
而应该是：

```
it.remove();
it.next();
it.remove();
```

# 具体的集合

Java库中的具体集合如下表所示：

集合类型|描述
-|-
ArrayList|一种可以动态增长和缩减的索引序列
LinkedList|一种可以在任何位置进行高效地插入和删除操作的有序序列
ArrayDeque|一种用循环数组实现的双端队列
HashSet|一种没有重复元素的无序集合
TreeSet|一种有序集
EnumSet|一种包含枚举类型的集合
LinkedHashSet|一种可以记住元素插入次序的集合
PriorityQueue|一种允许高效删除最小元素的集合
HashMap|一种存储键/值关联的数据结构
TreeMap|一种键值有序排队的映射表
EnumMap|一种键值属于枚举类型的映射表
LinkedHashMap|一种可以记住键/值项添加次序的映射表
WeakHashMap|一种其值无用武之地后可以被垃圾回收的映射表
IdentityHashMap|一种用==而不是用equals比较键值的映射表

## 数组列表

ArrayList封装了一个动态再分配的对象数组。主要适用于查询多、修改少的场景。
ArrayList不是线程安全的，只能用在单线程环境下，多线程环境下可以使用如下几种办法：

1. Collections.synchronizedList(List l);
2. 使用concurrent包下的CopyOnWriteArrayList类；
3. 使用Vector类；

ArrayList实现了List接口、底层使用数组保存所有元素，其操作基本上是对数组的操作。
ensureCapacity()方法在容量不够时，默认增加50%+1，代码如下：

```
int newCapacity = (oldCapacity * 3)/2 + 1;
```
与之相比，Vector默认是增加一倍。

## 链表

LinkedList适用于查询少、修改多的场景。
在Java中，所有的链表都是双向链表，即每个结点都保存了它的前驱与后继，从中插入或删除一个元素非常容易，只需要更新附近元素的前驱与后继即可。
链表是一个有序集合，每个对象的位置十分重要。

## 散列集

### HashSet

### HashMap

HashMap可以快速地查找所需要的对象。HashMap为每个对象计算一个整数，即hash code，如果是自定义类，就要负责实现这个类的hashCode方法。

## 树集

## 映射表

### 

## 队列

### 双端队列

### 优先级队列
