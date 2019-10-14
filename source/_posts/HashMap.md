---
title: HashMap
date: 2017-03-13 16:48:49
tags:
  - Java
---
HashMap 源码解析

## 概述

HashMap 是一个集合对象，存储key-value类型的数据。内部主要使用哈希表的结构去实现。至于哈希表的的结构我们在此不做详细介绍，可自行脑补。下面的源码基于JDK1.8，由于进行了很多优化，代码会有点复杂。

## 成员变量

```java
    /* ---------------- Fields -------------- */

    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     * 保存元素的数组对象
     */
    transient Node<K,V>[] table;

    /**
     * The number of key-value mappings contained in this map.
     * 集合中实际的元素个数
     */
    transient int size;


    /**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    // 集合所能保存元素个数的最大值，当保存元素的个数超过这个值得时候，会进行扩容操作，一般threshold = tables.length * loadFactor 。其中length为tables的长度，默认为16
    int threshold;

    /**
     * The load factor for the hash table.
     * 负载因子 默认是0.75，负载因子大小会影响到threshold的大小，0.75是对空间和时间的平衡选择，建议不要修改。
     * @serial
     */
    final float loadFactor;

```

## put方法

```java
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
实际调用的putVal方法。
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //首先判断table是否为空，如果为空则进行扩容操作。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //根据hash值与table取模得到桶的下标，获取值后赋值给p，并且判断是否为空。
    //如果为空说明当前位置没有值，既没有发现hash碰撞，新建一个Node插入即可。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {   //进入这里说明发生了hash碰撞
        Node<K,V> e; K k;
        //判断hash值和key是否相同，如果相同则直接覆盖就好了。
        if (p.hash == hash &&     
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果key不相同，判断p是否是TreeNode对象（红黑树实现）。
        else if (p instanceof TreeNode)
            //红黑树插入值
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //key不相同，也不是TreeNode对象（红黑树），说明就是普通的链表，然后会遍历链表。
            //并且用binCount 记录链表的长度
            for (int binCount = 0; ; ++binCount) {
                //如果为空说明遍历到链表的末尾了，然后进行链表插入操作，然后退出循环
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //插入完之后判断一下链表的长度是否超过8，如果超过8则会将链表转换成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                         //转换红黑树（红黑树比较复杂，有兴趣的自己研究）
                        treeifyBin(tab, hash);
                    break;
                }
                //在遍历的过程中，发现key值相同的话 直接覆盖。
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果不为空，表示遇到相同的key，则覆盖原来的值,并且直接返回。
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            //onlyIfAbsent这个变量表示是否改变原来的值，如果为true是不改变。false则改变，在HashMap中的put方法内部传入的false。
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            //这个方法在HashMap中是个空方法，主要是给LinkedHashMap复写用来调用的，可以不用关注。
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //判断大小是否到了阀值。
    if (++size > threshold)
        resize();
     //这个方法在HashMap中是个空方法，主要是给LinkedHashMap复写用来调用的，可以不用关注。
    afterNodeInsertion(evict);
    return null;
}
```

## get方法

```java
 public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```
实际调用的getNode方法

```java
    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //判断table是否为空，以及根据hash值取得对应的Node对象，并且赋值给first。
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //如果hash和key都相同，则说明找到了直接返回first
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //如果key不相同，则在红黑树或者链表中查找
            if ((e = first.next) != null) {
                //在红黑树中查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //遍历链表查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```

## hash方法

```java
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
我们put或者get的时候都会通过hash方法获取key对应的哈希值。当key为空的时候直接返回0，这样后面就无需再判读key是否为空，按照正常的hash处理就可以了。当key不为空是首先调用key自己的hashCode方法然后在进行移位运算。至于为什么这样做是处于多方面的考虑，比如速度，功效，质量等。

## resize方法

这个方法是比较重要的方法，主要用来初始化和扩容HashMap

```java
/**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //如果超过最大容量，则不扩容。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //没有则扩充为原来的2倍。
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //计算新的resize的阀门
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
        //把每个Node对象放入到新的tables中去
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                     //没有hash碰撞的元素处理
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                    //红黑树元素处理
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else {
                    // preserve order 链表元素处理 没怎么看懂
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```
