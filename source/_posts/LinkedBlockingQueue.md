---
title: LinkedBlockingQueue
date: 2017-03-13 16:44:17
tags:
  - Java
---
LinkedBlockingQueue 源码分析

## 概述
LinkedBlockingQueue 是线程安全的阻塞队列。

## 成员变量
```java
    /**
     * Linked list node class
     * 队列结点类型Node
     */
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }

    /** The capacity bound, or Integer.MAX_VALUE if none */
    //队列的的大小，初始化的时候设置，如果没有传入默认为Integer.MAX_VALUE
    private final int capacity;

    /** Current number of elements */
    //队列中元素的个数，由于多线程访问修改，使用了线程安全的AtomicInteger类。
    private final AtomicInteger count = new AtomicInteger();

    /**
     * Head of linked list.
     * Invariant: head.item == null
     * 队列的头结点
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     * 队列的尾节点
     */
    private transient Node<E> last;
```
以下成员变量是锁的实现
```java
    /** Lock held by take, poll, etc */
    //删除元素时候，要加的takeLock锁
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    //获取元素是若队列为空，线程阻塞，直至notEmpty条件满足
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    //插入元素时，要加putLock锁
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    //插入时候，若队列已满，线程阻塞，直至notFull条件满足
    private final Condition notFull = putLock.newCondition();

```
**ReentrantLock** 是一个可重入的互斥锁对象。即就是同一时刻在多线程的环境下，只有一个线程能获得锁。但是多个线程都可以调用lock方法，只有一个会成功，其他的线程将会被阻塞，直达该锁被释放。**可重入** 的意思是说一个线程获取锁以后，在执行方法过程中需要再次获取当前锁，此时lock方法不会阻塞。可再次进入。（如需要对ReentrantLock的实现和深入了解，请看后续文章）。

**Condition** 是竞态条件对象，和ReentrantLock绑定使用。提供了线程间的通信方式（类似信号量），使用基本上和Object的wait，notify相同。（如需要对Condition深入了解，请看后续文章）。

##初始化
```java
    /**
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        //初始化队列大小
        this.capacity = capacity;
        //初始化链表
        last = head = new Node<E>(null);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        //默认为Integer.MAX_VALUE 可认为无限大
        this(Integer.MAX_VALUE);
    }
```

## 入队
LinkedBlocking入队的方法有两种
* put（）方法，为阻塞方法，队列有空余的时候才能加入新的元素，否则一直阻塞
* offer（）方法，为非阻塞方法，如果队列满了，立即返回或者等待一会而在返回，通过返回的ture或者false，标记本次本次入队操作是成功。

```java
/**
     * Inserts the specified element at the tail of this queue, waiting if
     * necessary for space to become available.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        //新建入队结点
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        //获取入队锁，这里调用的lockInterruptibly（）方法，
        //而不是lock（）方法，是为了更友好。lock（）方法在没有成功获取到锁的的时
        //候会一直block，打死不回头。而lockInterruptibly（）方法在阻塞的时候
        //如果被中断，线程会被唤醒并且处理中断异常，选择是继续阻塞还是返回了。
        //（ ReentrantLock对象中还有tryLock()方法，这个方法是，不会阻塞马上返
        //回，拿到锁后就返回true否则返回false，比较潇洒。tryLock（time）方法是
        //拿不到锁的时候，会阻塞time时间，超时就返回false，比较友好。）
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
             //如果容量满了 就一直阻塞
            while (count.get() == capacity) {
                notFull.await();
            }
            //入队操作（后面会讲到）
            enqueue(node);
            //队列的元素个数加一，并且返回队列元素个数（注意getAndIncrement返回的是旧值）
            c = count.getAndIncrement();
            //如果链表没有满，则发出通知 ，唤醒一个等待入队的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            //释放锁
            putLock.unlock();
        }
        //如果入队之前队列是空的，那么现在可以唤醒一个等待出队的线程
        if (c == 0)
            signalNotEmpty();
    }

```
```java
/**
     * Inserts the specified element at the tail of this queue, waiting if
     * necessary up to the specified wait time for space to become available.
     *
     * @return {@code true} if successful, or {@code false} if
     *         the specified waiting time elapses before space is available
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        //将时间转换成纳秒
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                //容量满了 然后判断有没有传入等待时间
                //小于0的（即就是没有配置等待时间）不需要阻塞直接返回，失败
                if (nanos <= 0)
                    return false;
                //等待一定时间，返回0或者小于0的一个值
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }

```
入队的链表操作
```java
    /**
     * Links node at end of queue.
     *
     * @param node the node
     */
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        //其实相当于两步
        // last.next = node 将last的下个结点指向新结点
        // last = last.next 将last重新指向新节点
        last = last.next = node;
    }
```
## 出队
LinkedBlockingQueue的出队列的方式有两种
* take() 方法，阻塞方法，队列有元素的时候才能出队，否则一直阻塞。
* poll() 方法，非阻塞方法，队列没有元素的时候，立即返回或者等待一定的时间。

```java
 /**
   *可以对比着put方法来看。LinkedBlockingQueue的出队和入队是对称的。
   */
 public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        //通过takeLock获取锁，并且支持线程中断
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                //当队列为空时，则让当前线程处于等待
                notEmpty.await();
            }
            x = dequeue();
            //队列元素个数完成原子化操作-1,可以看到count元素会
            //在插入元素的线程和获取元素的线程进行并发修改操作。
            c = count.getAndDecrement();
            //当一个元素出队列之后，队列的大小依旧大于1时
            //当前线程会唤醒其他执行元素出队列的线程,让它们也
            //可以执行元素的获取
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //当c==capaitcy时，即在获取当前元素之前，队列已经满了，
        //而此时获取元素之后，队列就会空出一个位置，故当前线程会
        //唤醒执行插入操作的线程通知其他中的一个可以进行插入操作。
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

```java
//和offer方法几乎是一样的。注释就不写了。大家都能看懂
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

```
```java
//具体的出队操作
//LinkedBlockingQueue队列中的head结点只是一个指向，具体的值是null的，队列中第一个元素是head->next
//出队操作就是将head指向head->next,然后将head->next的值设置为null
private E dequeue() {
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // 帮助GC回收
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```

我们看了LinkedBlockingQueue的出队和入队操作。我们现在来对比一下。

#### 三种入队对比
>* offer(E e)：如果队列没满，立即返回true； 如果队列满了，立即返回false-->不阻塞
* put(E e)：如果队列满了，一直阻塞，直到队列不满了或者线程被中断-->阻塞
* offer(E e, long timeout, TimeUnit unit)：在队尾插入一个元素,，如果队列已满，则进入等待，直到出现以下三种情况：-->阻塞

>  1.被唤醒

>  2.等待时间超时

>  3.当前线程被中断


#### 三种出队对比
>* take()：如果队列空了，一直阻塞，直到队列不为空或者线程被中断-->阻塞
* poll()：如果没有元素，直接返回null；如果有元素，出队-->不阻塞
* poll(long timeout, TimeUnit unit)：如果队列不空，出队；如果队列已空且已经超时，返回null；如果队列已空且时间未超时，则进入等待，直到出现以下三种情况：

 >  1.被唤醒

 >  2.等待时间超时

 >  3.当前线程被中断
