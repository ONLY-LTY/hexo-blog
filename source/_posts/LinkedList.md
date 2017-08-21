---
title: LinkedList
date: 2017-03-13 16:58:41
tags:
  - Java
---
LinkedList 解析
<!--more-->
## 1 初始化
LinkedList也实现了List接口，说明了也具备了插入和删除功能,基于链表，插入和删除比ArrayList快，但是随机访问能力差，这个基础数据结构已经说明了，不再重复。其次LinkedList具备了双端队列的功能，实现了Deque接口
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
}
```
基本字段如下：
```java
   transient int size = 0;

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;

    /**
     * Constructs an empty list.
     */
    public LinkedList() {
    }

```
主要是通过两个节点来操作链表的，first节点和last节点。size用来存储当前节点的大小。

## 2 插入数据
LinkedList 可以往链表前面和链表后面插入数据，这样也具备了双端队列的功能
```java
public void addFirst(E e) {
    linkFirst(e);
}

/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #add}.
 *
 * @param e the element to add
 */
public void addLast(E e) {
    linkLast(e);
}
```
下面两个方法演示了往链表的前和后插入节点，基本都很简单，跟书上数据结构的方式一样
```java
private void linkFirst(E e) {
   //节点f指向first节点
    final Node<E> f = first;
    //新建一个节点
    final Node<E> newNode = new Node<>(null, e, f);
    //first指向新节点
    first = newNode;
    //如果f节点是null，那么就把last节点指向新节点，
    //如果不是null那么f节点的prev节点指向了新节点
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

/**
 * Links e as last element.
 */
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```
## 3 随机访问数据
下面代码是LinkedList随机访问节点的流程，这里做了个优化，查找的时候不是全部遍历节点，而是判断了下index的位置是上半部分还是下半部分。从这里可以看出来，链表的随机访问效率低下的地方，因为需要遍历节点，而不是跟顺序表一样，可以通过下标进行访问
```java
Node<E> node(int index) {
    // assert isElementIndex(index);
    //判断index是否小于size的50%，如果是那么从first节点开始遍历，否则从last开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```
## 4 删除数据
删除节点这里逻辑也非常清晰，跟普通的链表操作一致，从链表头部开始删除。
```java
public E remove() {
    return removeFirst();
}
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    //拿到f节点的对象
    final E element = f.item;
    //拿到f节点的下一个节点
    final Node<E> next = f.next;
    //置为空
    f.item = null;
    f.next = null; // help GC
    把first节点置为next节点
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```
