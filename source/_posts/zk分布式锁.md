---
title: Zookeeper分布式锁
date: 2018-06-01 18:09:31
tags:
  - 分布式
  - Lock
---
基于Zookeeper的分布式锁
<!--more-->

##### 锁流程

![](/img/lock.png)

##### 代码实现

Curator框架已经实现了基于Zookeeper的可重入的分布式锁 InterProcessMutex

###### 构造方法
```java
/**
 * @param client client
 * @param path   the path to lock
 */
public InterProcessMutex(CuratorFramework client, String path)
{
    this(client, path, new StandardLockInternalsDriver());
}

/**
 * @param client client
 * @param path   the path to lock
 * @param driver lock driver
 */
public InterProcessMutex(CuratorFramework client, String path, LockInternalsDriver driver)
{
    this(client, path, LOCK_NAME, 1, driver);
}

InterProcessMutex(CuratorFramework client, String path, String lockName, int maxLeases, LockInternalsDriver driver)
{
    //锁节点路径
    basePath = PathUtils.validatePath(path);
    //主要通过LockInternals类实现锁的功能
    internals = new LockInternals(client, driver, path, lockName, maxLeases);
}
```
###### 获取锁实现 InterProcessMutex.acquire
```java
/**
* Acquire the mutex - blocking until it's available. Note: the same thread
* can call acquire re-entrantly. Each call to acquire must be balanced by a call
* to {@link #release()}
*
* @throws Exception ZK errors, connection interruptions
*/
@Override
public void acquire() throws Exception
{
    //阻塞获取锁 调用internalLock方法
    if ( !internalLock(-1, null) )
    {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}

/**
* Acquire the mutex - blocks until it's available or the given time expires. Note: the same thread
* can call acquire re-entrantly. Each call to acquire that returns true must be balanced by a call
* to {@link #release()}
*
* @param time time to wait
* @param unit time unit
* @return true if the mutex was acquired, false if not
* @throws Exception ZK errors, connection interruptions
*/
@Override
public boolean acquire(long time, TimeUnit unit) throws Exception
{
    //在规定的时间内获取锁 调用internalLock方法
    return internalLock(time, unit);
}

private boolean internalLock(long time, TimeUnit unit) throws Exception
{
    /*
       Note on concurrency: a given lockData instance
       can be only acted on by a single thread so locking isn't necessary
    */

    Thread currentThread = Thread.currentThread();

    //获取当前线程持有的锁
    LockData lockData = threadData.get(currentThread);
    if ( lockData != null )
    {
        // re-entering
        //如果持有锁 再次获取锁 重入
        lockData.lockCount.incrementAndGet();
        return true;
    }

    //通过LockInternals.attemptLock获取锁
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null )
    {
        LockData newLockData = new LockData(currentThread, lockPath);
        //保存当前获取的锁信息
        threadData.put(currentThread, newLockData);
        return true;
    }

    return false;
}

String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
    {
        final long      startMillis = System.currentTimeMillis();
        final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
        final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
        int             retryCount = 0;

        String          ourPath = null;
        boolean         hasTheLock = false;
        boolean         isDone = false;
        while ( !isDone )
        {
            isDone = true;

            try
            {
                //先创建临时顺序节点,任何线程获取锁都会先创建临时的顺序节点
                //{basePath}/lock-{sequenceNodeName}
                //basePath 是我们构造方法传入的
                //这个sequenceNodeName是客户端自动生成加上的 客户端能保证每次请求的逐渐增大的
                ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
                //调用internalLockLoop方法获取锁
                hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
            }
            catch ( KeeperException.NoNodeException e )
            {
                // gets thrown by StandardLockInternalsDriver when it can't find the lock node
                // this can happen when the session expires, etc. So, if the retry allows, just try it all again
                if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
                {
                    isDone = false;
                }
                else
                {
                    throw e;
                }
            }
        }

        if ( hasTheLock )
        {
            return ourPath;
        }

        return null;
    }

private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
    boolean     haveTheLock = false;
    boolean     doDelete = false;
    try
    {
        if ( revocable.get() != null )
        {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }

        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
        {
            //获取锁节点路径下的子节点,并且从小到大排序
            List<String>        children = getSortedChildren();
            //本次需要获取的锁的节点 sequenceNodeName
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash

            //获取锁
            PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if ( predicateResults.getsTheLock() )
            {
                haveTheLock = true;
            }
            else
            {
                //没有获取到锁 监听前一个node. 锁的实现是公平的，按照获取锁的顺序排列。所以只需要监听前一个node
                //变化 看是否被删除
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                synchronized(this)
                {
                    try
                    {
                        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null )
                        {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 )
                            {
                                doDelete = true;    // timed out - delete our node
                                break;
                            }
                            //在没有获取锁的时候，在规定的时间等待、如果监听到node节点变化则会 notify开始获取锁
                            wait(millisToWait);
                        }
                        else
                        {
                            //死等 直到获取到锁
                            wait();
                        }
                    }
                    catch ( KeeperException.NoNodeException e )
                    {
                        // it has been deleted (i.e. lock released). Try to acquire again
                    }
                }
            }
        }
    }
    catch ( Exception e )
    {
        doDelete = true;
        throw e;
    }
    finally
    {
        //如果处理异常了 则删除node放弃获取锁
        if ( doDelete )
        {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}

StandardLockInternalsDriver.class

public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
    {
        int             ourIndex = children.indexOf(sequenceNodeName);
        validateOurIndex(sequenceNodeName, ourIndex);
        //这里的maxLease初始化是1 这个属性可以理解为临界资源默认是1 只允许一个。
        //只有ourIndex=0的时候成立。获取到锁
        boolean         getsTheLock = ourIndex < maxLeases;
        //没有获取到锁的话 则返回前一个node的path
        String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

        return new PredicateResults(pathToWatch, getsTheLock);
    }
```
###### 释放锁 InterProcessMutex.release
```java
/**
 * Perform one release of the mutex if the calling thread is the same thread that acquired it. If the
 * thread had made multiple calls to acquire, the mutex will still be held when this method returns.
 *
 * @throws Exception ZK errors, interruptions, current thread does not own the lock
 */
@Override
public void release() throws Exception
{
    /*
        Note on concurrency: a given lockData instance
        can be only acted on by a single thread so locking isn't necessary
     */

    Thread currentThread = Thread.currentThread();
    //获取当前线程的锁数据
    LockData lockData = threadData.get(currentThread);
    if ( lockData == null )
    {
        throw new IllegalMonitorStateException("You do not own the lock: " + basePath);
    }
    //锁的数量减一
    int newLockCount = lockData.lockCount.decrementAndGet();
    if ( newLockCount > 0 )
    {
       //重入锁 直接返回
        return;
    }
    if ( newLockCount < 0 )
    {
        throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + basePath);
    }
    try
    {
        //调用 LockInternals.releaseLock释放锁
        internals.releaseLock(lockData.lockPath);
    }
    finally
    {
        //删除锁的信息
        threadData.remove(currentThread);
    }
}

void releaseLock(String lockPath) throws Exception
    {
        revocable.set(null);
        //直接删除node节点 释放锁
        deleteOurPath(lockPath);
    }
```

##### 分析总结

1.  Curator的InterProcessMutex提供了多种锁机制，互斥锁，读写锁，以及可定时数的互斥锁。

2.  所有申请锁都会创建临时顺序节点，保证了都能够有机会去获取锁。

3.  内部用了线程的wait()和notifyAll()这种等待机制，可以及时的唤醒最渴望得到锁的线程。避免常规利用Thread.sleep()这种无用的间隔等待机制。

4.  使用Zookeeper可以有效的解决锁无法释放的问题，因为在创建锁的时候，客户端会在ZK中创建一个临时节点，一旦客户端获取到锁之后突然挂掉（Session连接断开），那么这个临时节点就会自动删除掉。其他客户端就可以再次获得锁。

5.  使用Zookeeper可以实现阻塞的锁，客户端可以通过在ZK中创建顺序节点，并且在节点上绑定监听器，一旦节点有变化，Zookeeper会通知客户端，客户端可以检查自己创建的节点是不是当前所有节点中序号最小的，如果是，那么自己就获取到锁，便可以执行业务逻辑了。

6.  使用Zookeeper也可以有效的解决不可重入的问题，客户端在创建节点的时候，把当前客户端的主机信息和线程信息直接写入到节点中，下次想要获取锁的时候和当前最小的节点中的数据比对一下就可以了。如果和自己的信息一样，那么自己直接获取到锁，如果不一样就再创建一个临时的顺序节点，参与排队。

7.  使用Zookeeper可以有效的解决单点问题，ZK是集群部署的，只要集群中有半数以上的机器存活，就可以对外提供服务。
