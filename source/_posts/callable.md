---
title: Callable Future
date: 2017-01-05 17:31:16
tags:
  - Thread
---
Java Callable Future使用详解。
<!--more-->
```Java
public interface Future<V> {
    //取消任务
    boolean cancel(boolean mayInterruptIfRunning);
    //任务是否取消
    boolean isCancelled();
    //任务是否完成
    boolean isDone();
    //获得任务结果
    V get() throws InterruptedException, ExecutionException;
   //获得任务结果，并指定一定时间内，如果unit时间没有返回，抛出异常
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
```Java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}

```
&emsp;&emsp;Java线程池中执行的任务的最小单位可以是实现Runnable接口的，也可以是Callable接口，二者唯一的不同就是Callable能返回任务执行的结果。而Runnable则不可以。Runnable接口执行的run方法，Callable接口执行的call方法。我们使用Callable的时候同时，需要配合Future使用，Future可以理解为一个任务的执行周期，比如是否执行完成，执行完成后的结果是什么，执行过程中的异常是什么，取消任务等等。下面我们看代码来理解这二者的配合使用。

```Java
private static void test3() throws InterruptedException, ExecutionException, TimeoutException {
        //工厂方法创建一个线程池
       ExecutorService executor = Executors.newFixedThreadPool(1);
       //这里通过java8 表达式创建了一个Callable任务，并返回任务的Future对象
       Future<Integer> future = executor.submit(() -> {
           try {
               TimeUnit.SECONDS.sleep(1);
               return 123;
           }
           catch (InterruptedException e) {
               throw new IllegalStateException("task interrupted", e);
           }
       });
       //获取任务的值 这里返回123
       future.get();
   }
```
&emsp;&emsp;上面的代码就是很简单的Callable和Future的配合使用。我们下面来看看源码走一下流程，加深印象。首先是submit方法。

```Java
/**
  * @throws RejectedExecutionException {@inheritDoc}
  * @throws NullPointerException       {@inheritDoc}
  */
 public <T> Future<T> submit(Callable<T> task) {
     if (task == null) throw new NullPointerException();
     //这里将我们我们传入的Callable对象封装成RunnableFuture对象去执行
     RunnableFuture<T> ftask = newTaskFor(task);
     execute(ftask);
     return ftask;
 }
```
```Java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        //这里返回的FutureTask对象
        return new FutureTask<T>(callable);
    }
```
```Java
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        //将Callable复制给成员变量
        this.callable = callable;
        //设置任务的状态为新建
        this.state = NEW;       // ensure visibility of callable
    }
```
&emsp;&emsp;上面几步我们知道了，线程池在submit一个Callable的时候首先将Callable封装成了RunnableFuture对象，这个是一个接口，代码中我们可以看到使用的具体实现的FutureTask这个实现类。初始化的的时候设置任务需要具体执行的Callable以及设置了任务的状态为新建。下面的代码是任务的所有状态。
```Java
private volatile int state;
private static final int NEW          = 0; //新建任务
private static final int COMPLETING   = 1; //任务完成
private static final int NORMAL       = 2; //任务正常
private static final int EXCEPTIONAL  = 3; //任务异常
private static final int CANCELLED    = 4; //任务取消
private static final int INTERRUPTING = 5; //任务中断中
private static final int INTERRUPTED  = 6; //任务已中断
```
&emsp;&emsp;构建RunnableFuture对象后就去执行了，看对象的名字就知道该接口实现了Runnable接口，所以执行任务肯定是指行run方法，这里我们去看FutureTask的run方法。

```Java
public void run() {
        //首先判断任务的状态是不是新建
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            //这里的callable是上面初始化的赋值过的，这里又赋值给v
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    //去执行具体的任务，并将结果放在临时result里面
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    //如果抛异常了，这里会将异常封装 具体看后面
                    result = null;
                    ran = false;
                    setException(ex);
                }
                //如果正常执行完任务，设置结果 具体看后面
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
```Java
//任务正常执行情况
protected void set(V v) {
  //跟新任务状态为完成
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
      //将结果赋值给成员变量outcome
      outcome = v;
      //更新状态为正常
      UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
      finishCompletion();
    }
}
```
```Java
//任务异常
protected void setException(Throwable t) {
   //更新任务状态为完成
   if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
       //将异常对象赋值给成员变量outcome
       outcome = t;
       //更新任务状态为异常
       UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
       finishCompletion();
   }
}
```
&emsp;&emsp;上面就是执行任务的过程，流程很清晰，下面我们看FutureTask的get方法，获取任务的执行结果。

```Java
public V get() throws InterruptedException, ExecutionException {
  int s = state;
  //如果任务没有完成，则一直阻塞等待，直到任务完成
  if (s <= COMPLETING)
      s = awaitDone(false, 0L);
  //返回结果
  return report(s);
}
```
```Java
private V report(int s) throws ExecutionException {
    //将结果赋值给x 这里可能是正常返回值，也可能是异常信息
    Object x = outcome;
    //这里状态如果是正常的话直接将结果返回
    if (s == NORMAL)
        return (V)x;
    //如果取消了任务就抛出取消异常
    if (s >= CANCELLED)
        throw new CancellationException();
    //这里就是执行任务的时候有异常，线程池将异常封装成ExecutionException抛出。
    throw new ExecutionException((Throwable)x);
}
```
&emsp;&emsp;这样我们看了这个流程，应该会加深我们对Callable和Future的认识。
