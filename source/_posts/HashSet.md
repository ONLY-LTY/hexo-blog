---
title: HashSet
date: 2017-03-13 17:01:15
tags:
  - Java
---
HashSet 源码解析

## 1 初始化

HashSet是一个实现非常简单的类，为什么这么说呢？因为它的实现基本都依赖于HashMap，可以先来看一下初始化的内容
```java
public class HashSet<E>
  extends AbstractSet<E>
  implements Set<E>, Cloneable, java.io.Serializable
{
  static final long serialVersionUID = -5024744406713321676L;

  private transient HashMap<E,Object> map;

  // Dummy value to associate with an Object in the backing Map
  private static final Object PRESENT = new Object();

  /**
   * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
   * default initial capacity (16) and load factor (0.75).
   */
  public HashSet() {
      map = new HashMap<>();
  }

```
从上面代码就可以知道，HashSet依赖于HashMap的存储，HashSet是一个不允许重复数据放入集合，他的实现依赖于HashMap得key，把对象放入到key里面，按照HashMap的特性，Key是不允许重复的，那么依照这种特性，完全可以使用key去存储。

## 2 操作

按照上面说的思路来说，其实添加和删除，无非是对HashMap的key进行一个操作而已
```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

/**
 * Removes the specified element from this set if it is present.
 * More formally, removes an element <tt>e</tt> such that
 * <tt>(o==null&nbsp;?&nbsp;e==null&nbsp;:&nbsp;o.equals(e))</tt>,
 * if this set contains such an element.  Returns <tt>true</tt> if
 * this set contained the element (or equivalently, if this set
 * changed as a result of the call).  (This set will not contain the
 * element once the call returns.)
 *
 * @param o object to be removed from this set, if present
 * @return <tt>true</tt> if the set contained the specified element
 */
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

```
