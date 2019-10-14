---
title: LinkedHashMap
date: 2017-03-13 16:51:21
tags:
  - Java
---
LinkedHashMap 源码解析

## 概述

众所周知HashMap里面元素是无序的，而LinkedHashMap说直接点就是元素有序的HashMap。

## 初始化

```java
/**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }

```

LinkedHashMap的构造方法调用的父类HashMap的父类方法。然后新增了一个accessOrder这个参数。根据英文注释。当accessOrder为true时，元素按照访问顺序排列，为false的时候,元素按照插入顺序排列，默认值为false。我们后面会介绍这个参数的具体在哪里用到。

## 如何做到有序

```java
 /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```
我们看到集合中具体的元素对象Entry中多了两个成员变量before ， after。由名称就可以猜出来，每个元素具有指向前一个元素和后一个元素的引用，所以LinkedHashMap元素是用双向链表实现的，并且还有指向头元素和尾元素的引用。我们可以从LinkedHashMap的成员变量中看到，如下：
```java
 /**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> tail;
```
## 添加元素
LinkedHashMap的没有复写父类的put方法（具体的put方法请看HashMap解析），而是直接调用父类的方法。但是复写了父类put方法中的几个子方法。
```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p); //将添加的元素插入到末尾
        return p;
    }

    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p); //将添加的元素插入到末尾
        return p;
    }

```
JDK1.8中对对解决hash冲突做的很全面，最开始遇到冲突是用链表的形式也就是Entry对象，当链表具有一定的长度后会转换成红黑树也就是TreeNode对象。上面代码中可以看到当我们添加完一个元素后会调用linkNodeList（）把元素插入到末尾。具体代码如下
```java
// link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }ava

```
由此我们可以知道，每当我们put一个元素的时候，会讲该元素插入到双向链表的末尾。遍历的时候会按照插入的顺序查找。

## 获取元素
LinkedHashMap复写了父类的get方法。
```java
 public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)  //访问顺序控制
            afterNodeAccess(e);
        return e.value;
    }
```
方法中我们可以看到具体的获取元素仍然是调用父类的getNode方法（具体解析请看HashMap）。当我们获取到元素后，判断accessOrder的值，也就是我们初始化的传入的值，如果为true。会调用afterNodeAccess方法。下面我们看这个方法干了什么。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
代码仔细看一下还是能看懂,把访问到的元素移动到链表的末尾。这样就可以做到根据访问的顺序排列。

## 应用
根据LinkedHashMap可以根据访问顺序排序这一特性，我们可以基于它实现LRU缓存。LRU（Last Recently Used）最近最少使用的缓存。每当我们获取元素的时候会将该元素移动到链表的末尾。当我们内存已满的时候，链表最前面的元素即就是长时间没有被访问的，然后就可以释放了。以此来做到LRU。
