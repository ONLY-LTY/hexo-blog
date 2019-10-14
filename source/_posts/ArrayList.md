---
title: ArrayList
date: 2017-03-13 16:55:46
tags:
  - Java
---
ArrayList 源码解析

## 1 初始化

ArrayList类定义，通过类定义可以分析出ArrayList具备了RandomAccess（支持随机访问）、Cloneable（可以被复制）、Serializable（序列化）

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
}
```

初始化类属性,DEFAULT\_CAPACITY\(默认容量大小\)、EMPTY\_ELEMENTDATA\(用来置空数组的\)、DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA\(初始化时候的数组，根据下面代码看来，初始化的时候elementData默认指向DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA\)、elementData\(存放元素\)、size\(当前数组大小\)

```java
/**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

## 添加元素

ArrayList通过add方法来进行添加元素,`ensureCapacityInternal`这个方法比较复杂，稍后进行讲解，其实ArrayList底部还是数组，添加元素无非是给指定下标指向e到对象而已，然后size＋＋。从这里看来ArrayList是非线程安全的。

```java
 public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

接下来看看`ensureCapacityInternal`这个方法

```java
private void ensureCapacityInternal(int minCapacity) {
      //判断是不是空元素，如果是空就把minCapacity置为DEFAULT_CAPACITY和minCapacity中的最大值
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        //判断是否需要增加集合大小，如果minCapacity值大于elementData.length那么就调用grow方法
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

接下来看看关键性的grow方法

```java
private void grow(int minCapacity) {
        // overflow-conscious code
        //获取到当前集合的大小
        int oldCapacity = elementData.length;
        //确定新集合的大小，是当前集合大小＋oldCapacity左移位运算类似除于2
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果最新集合大小如果比minCapacity小，那么增加的大小就用minCapacity
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //如果newCapacity的值大于MAX_ARRAY_SIZE
        //MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8,那么调用hugeCapacity
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        //原生方法进行数组复制
        elementData = Arrays.copyOf(elementData, newCapacity);
}
//返回最大值
private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
}

```
上述方法主要是当数组长度不够时，自动增长数组的长度，增长因子是当前长度的二分之一。也就是说增长后＝oldLength＋oldLength>>1

#### 添加元素到指定位置
添加元素到指定位置，这个方法效率比较低，需要进行对元素的大量移动操作
```java
public void add(int index, E element) {
//判断是否超过当前数组大小
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //index开始的元素后移一位
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```
#### 删除某个位置的元素
基本跟add(int index, E element)这个方法没有太大区别，一个是数组向后移动，一个是向前移动
```java
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```
#### 删除某个元素
删除某个元素的时候，主要是根据equals方法去判断是否是同一个对象的。
```java
public boolean remove(Object o) {
        if (o == null) {
            //移除null对象
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            //主要是调用equals方法判断是否是同一个对象
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {

                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    /*
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
可以看到 *ArrayList* 中删除和添加都使用到了 *System.arraycopy* 这个方法。这个方法是一
个 *native* 的方法。如下
```Java
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```
@src -- 这是源数组 @srcPos -- 这是源数组中的起始位置 @dest -- 这是目标数组 @destPos -- 这是目标数据中的起始位置  @length -- 这是一个要复制的数组元素的数目

比如：
```java
  int arr1[] = {0,1,2,3,4,5};
  int arr2[] = {0,10,20,30,40,50};
  System.arraycopy(arr1,0,arr2,1,2);

  arr2 = [0,0,1,30,40,50];
```
*System.arraycopy* 方法的实现使用C++实现的，实现原理就是直接操作内存地址操作。因为数组是一段连续的内存地址。
