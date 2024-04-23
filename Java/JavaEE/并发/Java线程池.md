# Java线程池

首先，先来看下Java线程池几个相关类的继承结构:

![image-20220121203547502](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220121203547502.png)

先看看这个继承结构，Executor位于最顶层，也是最简单的一个，就一个execute(Runnable runnable)接口方法定义。

ExecutorService也是接口，在Executor接口的基础上添加了很多的接口方法，所有**一般来说我们会使用这个接口**。

然后再下来一层是AbstractExecutorService，从名字上我们知道，这是抽象类，这些实现了非常有用的一些方法供子类直接使用，之后我们再细说。

然后才到我们的重点部分ThreadPoolExecutor类，这个类提供了关于线程池所需的非常丰富的功能。

另外，还涉及到下图中的这些类:

![image-20220121205215243](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220121205215243.png)

同在并发包中的Executores类，类名中带字母s，我们猜到这个是工具类，里面的方法都是静态方法，如以下我们最常用的用于生成ThreadPoolExecutor的实例的一些方法:

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    } 

public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

```

另外，由于线程池支持**获取线程执行的结果**，所以，引入了Future接口，RunnableFuture继承自此接口，然后我们最需要关心的就是它的实现类FutureTask。到这里，记住这个概念，在线程池的使用过程中，我们是往线程池中提交任务(task)，使用过线程池的都知道，我们提交的每个任务是实现了Runnable接口的，其实就是先将Runnable的任务包装成FutureTask，然后再提交到线程池。这样，读者才能比较容易记住 FutureTask这个类名:它首先是一个任务(task)，然后具有Future接口的语义，即可以在将来(future)得到执行的结果。

线程池中的BlockingQueue也是非常重要的概念，如果线程数达到corePoolSize,我们每个任务会提交到等待队列中，等待线程池中的线程池来取任务并执行。这里的BlockingQueu通常我们使用其实现类LinkedBlockingQueue、ArrayBlockingQueue和SynchronousQueue，每个实现类都有不同的特征，之后会分析。

> 除了上面说的这些类外，还有一个很重要的类，就是定时任务实现类ScheduledThreadPoolExecutor，它继承自ThreadPoolExecutor，用于实现定时执行。

## 1.Executor接口

```java
public interface Executor {
    void execute(Runnable command);
}
```

我们可以看到Executor接口非常简单，就一个`void execute(Runnable command)`方法，代表提交一个任务。

我们经常这样启动一个线程:

```java
new Thread(new Runnable() {
    // do something
}).start();
```

用了线程池Executor后就可以像下面这么使用:

```java
Executor executor = anExecutor;
executor.execute(new RunnableTask1());
executor.execute(new RunnableTask2());
```

如果我们希望线程池同步执行每一个任务，我们可以这么实现这个接口:

```java
class DirectExecutor implements Executor {
    public void execute(Runnable r) {
        r.run(); // 这里不是用的new Thread(r).start(),也就是说没有启动任何一个新的线程
    }
}
```

我们希望每个任务提交进来后，直接启动一个新的线程来执行这个任务，我们可以这么实现:

```java
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start(); // 每个任务都用一个新的线程来执行
    }
}
```

我们再来看怎么组合两个Executor来使用，下面这个实现是将所有的任务都加到一个queue中，然后从queue中取任务，交给真正的执行器执行，这些采用synchronized进行并发控制:

```java
class SerialExecutor implements Executor {
    // 任务队列
    final Queue<Runnable> tasks = new ArrayDeque<>();
    // 这个才是真正的执行器
    final Executor executor;
    // 当前真正执行的任务
    Runnable active;
    
    // 初始化的时候，指定执行器
    SerialExecutor(Executor executor) {
        this.executor = executor;
    }
    
    // 添加任务到线程池:将任务添加到任务队列,scheduleNext 触发执行器去任务队列取任务
    public synchronized void execute(final Runnable r) {
        tasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (active == null) {
            scheduleNext();
        }
    }
    
   	protected synchronized void scheduleNext() {
        if ((active == tasks.poll()) != null) {
            // 具体的执行转给真正的执行器 executor
            executor.execute(active);
        }
    }
}
```

当然了，Executor这个接口只有提交任务的功能，太简单了，我们想要更丰富的功能，比如我们想知道执行结果、我们想知道当前线程池有多少个线程活着、已经完成了多少任务等等，这些都是这个接口的不足的地方。接下来我们介绍的是继承自`Executor`接口的`ExecutorService`接口，这个接口提供了比较丰富的功能，也是我们最长使用到的接口。

## 2.ExecutorService

一般我们定义一个线程池的时候，往往都是使用这个接口:

```java
  ExecutorService executor = Executors.newFixedThreadPool(args...);
  ExecutorService executor = Executors.newCachedThreadPool(args...);
```

因为这个接口中定义的一系列方法大部分情况下已经可以满足我们的需要了。

那么我们简单粗略地看一下这个接口中有哪些方法:

```java
public interface ExecutorService extends Executor {	
	// 关闭线程池，已提交的任务继续执行，不接受继续提交新任务
    void shutdown();

  	// 关闭线程池，尝试停止正在执行的所有任务，不接受继续提交新任务
    // 它和前面的方法相比,加了一个单词"now"，区别在于它会停止当前正在进行的任务。
    List<Runnable> shutdownNow();

	// 线程池是否已关闭  
    boolean isShutdown();

  	// 如果调用了 shutdown() 或 shutdownNow() 方法后，所有任务结束了，那么返回true
    // 这个方法必须在调用shutdown或shutdownNow方法之后调用才会返回true
    boolean isTerminated();

	// 等待所有任务完成，并设置超时时间
    // 我们这里理解，实际应用中是，先调用 shutdown或shutdownNow.
    // 然后再调用这个方法等待所有的线程真正地完成，返回值意味着有没有超时
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

   // 提交一个 Callable 任务
    <T> Future<T> submit(Callable<T> task);

  	// 提交一个 Runnable 任务，第二个参数将会放到 Future 中，作为返回值
    // 因为 Runnable 的 run方法本身并不返回任何东西
    <T> Future<T> submit(Runnable task, T result);

   	// 提交一个Runnable 任务
    Future<?> submit(Runnable task);

   	// 执行所有任务，返回Future类型的一个list
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    // 执行所有任务，但是这里设置了超时时间
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

   	// 只要其中一个的任务结束了，就可以返回，返回执行完的那个任务的结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

   	// 同上一个方法，只有其中的一个任务结束了，就可以返回，返回执行完的那个任务的结果，
    // 不过这个带超时，超过指定的事件，抛出TimeOutException异常
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

这些方法都很好理解，一个简单的线程池主要就是这些功能，能提交任务，能获取结果，能关闭线程池，这也是为什么我们经常用这个接口的原因。

## 3.FutureTask

在继续往下层介绍ExecutorService的实现类之前，我们先来说说相关的类FutureTask。

```
Future      Runnable
   \           /
    \         /
   RunnableFuture
          |
          |
      FutureTask

FutureTask 通过 RunnableFuture 间接实现了 Runnable 接口，
所以每个 Runnable 通常都先包装成 FutureTask，
然后调用 executor.execute(Runnable command) 将其提交给线程池
```

我们知道，Runnable的void run()方法是没有返回值的，所以，如果我们需要的话，会在submit

中指定第二个参数作为返回值:

```java
<T> Future<T> submit(Runnable task, T result);
```

其实到时候会通过这两个参数，将其包装成Callable。它和Runnable的区别在于run()没有返回值，而Callable的call()方法有返回值，同时，如果运行出现异常， call()方法会抛出异常。

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

下面，我们来看看`ExecutorService`的抽象实现`AbstractExecutorService`。

## 4.AbstractExecutorService

AbstractExecutorService抽象类派生自ExecutorService接口，然后在其基础上实现了几个实用的方法，这些方法提供给子类进行调用。

这个抽象类实现了invokeAny和invokeAll方法，这里的两个newTaskFor方法也比较有用，用于将该任务包装成FutureTask。定义于最上层接口Executor中的`void execute(Runnable command)`由于不需要获取结果，不会进行FutureTask的包装。

> 需要获取结果(FutureTask),用submit方法，不需要获取结果，可以用execute方法。

下面，我们来看看源码。

```java
public abstract class AbstractExecutorService implements ExecutorService {

    // RunnableTuture 是用于获取执行结果的，我们常用它的子类FutureTask
    // 下面两个newTaskFor方法用于将我们的任务包装成FutrueTask提交到线程池中执行
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }


    // 提交任务
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        // 1.将任务包装成FutureTask
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 2.交给执行器执行，execute方法由具体的子类来实现
        // 前面也说了，FutureTask间接实现了Runnable接口。
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        // 1.将任务包装成FutureTask
        RunnableFuture<T> ftask = newTaskFor(task, result);
        // 2.交给执行器执行
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        // 1.将任务包装成FutureTask
        RunnableFuture<T> ftask = newTaskFor(task);
        // 2.交给执行器执行
        execute(ftask);
        return ftask;
    }

    // 此方法目的：将tasks集合中的任务提交到线程池执行，任意一个线程执行完后就可以结束了。
    // 第二个参数 timed 代表是否设置超时机制，超时时间为第三个参数
    // 如果timed为true，同时超时还没有一个线程返回结果，那么抛出TimeoutException异常
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        // 任务数
        int ntasks = tasks.size();
        if (ntasks == 0)
            throw new IllegalArgumentException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
        
        // ExecutorCompletionService不是一个真正的执行器，参数this才是真正的执行器
        // 它对执行器进行了包装，每个任务结束后，将结果保存到内部的一个completionQueue队列中
        // 这也是为什么这个类中的名字里有一个Completion的原因吧。
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);

        try {
            // 用于保存异常信息，此方法如果没有得到任何有效的结果，那么我们可以抛出最后得到的一个异常
            ExecutionException ee = null;
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // 首先先提交一个任务，后面的任务到下面的for循环中一个个提交
            futures.add(ecs.submit(it.next()));
			// 提交了一个任务，所以任务数量减1
            --ntasks;
            // 正在执行的任务数(提交的时候+ 1,任务结束的时候 -1)
            int active = 1;

            for (;;) {
                // ecs上面说了，其内部有一个completionQueue用于保存执行完成的结果
                // BlockingQueue的poll方法不阻塞，返回null代表队列为空
                Future<T> f = ecs.poll();
                // 为null，说明刚刚提交的第一个线程还没有执行完成
                // 在前面先提交一个任务，加上这里做一次检查，也是为了提高性能
                if (f == null) {
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    // 这里是else if，不是if。这里说明，没有任务了，同时active 为0 说明
                    // 所有的任务都执行失败，那么这里是for循环出口
                    else if (active == 0)
                        break;
                    else if (timed) {
                        // 先等待的poll方法 
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        // 如果已经超时，抛出TimeoutException异常，这整个方法就结束了
                        if (f == null)
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime();
                    }
                    // 这里是else说明，没有任务需要提交，但是池中的任务没有完成，还没有超时(如果设置了超时)
                   	// take()方法会阻塞，直到有元素返回，说明有任务结束了
                    else
                        f = ecs.take();
                }
                // 上面一段不是很好理解，说理简单说下.
                // 1.首先，这在一个for循环中，我们设想每一个任务都没那么快结束，
                // 		那么，每一次都会进到第一个分支，进行提交任务，直到所有的任务都提交了
                // 2.任务都提交完成后，如果设置了超时，那么for循环其实进入了“一直检测是否超时”
                // 		这件事情上
                // 3.如果没有设置超时机制，那么不必要检测超时，那就会阻塞在ecs.take()方法上，
                // 		等待获取第一个执行结果
                // 4.如果所有的任务都执行失败，也就是说future都返回了，
                // 		但是f.get()抛出异常，那么从active == 0分支出去
                
                // 有任务结束了
                if (f != null) {
                    --active;
                    try {
                        // 返回执行结果，如果有异常，都包装成ExecutionException
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            } // 注意看for循环的范围，一直到这里
            

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            // 方法退出之前，取消其他的任务
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
    }

    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        try {
            return doInvokeAny(tasks, false, 0);
        } catch (TimeoutException cannotHappen) {
            assert false;
            return null;
        }
    }

    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return doInvokeAny(tasks, true, unit.toNanos(timeout));
    }

    // 执行所有的任务，返回任务结果.
    // 先不要看这个方法，我们先想想，其实我们自己提交任务到线程池，也是想要线程池执行所有的任务
    // 只不过，我们每次submit提交一个任务，这里以一个集合作为参数提交
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            // 这个很简单
            for (Callable<T> t : tasks) {
                // 包装成FutureTask
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                // 提交任务
                execute(f);
            }
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    try {
                        // 这是一个阻塞方法，直到获取到值，或抛出了异常
                        // 这里有个小细节，其实get方法签名上是会抛出InterruptedException的
                        // 可是这里没有进行处理，而是抛给外层了。此异常发生于还没执行完的任务被取消了
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            // 这个方法返回，不像其他的场景，返回List<Future>，其实执行结果还没出来
            // 这个方法返回是真正的返回，任务都结束了
            return futures;
        } finally {
            // 为什么要这个？就是上面说的有异常的情况
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }

    // 带超时的 invokeAll，我们找不同的
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));

            final long deadline = System.nanoTime() + nanos;
            final int size = futures.size();

           	// 每提交一个任务，检测一次是否超时
            for (int i = 0; i < size; i++) {
                execute((Runnable)futures.get(i));
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L)
                    return futures;
            }

            for (int i = 0; i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    if (nanos <= 0L)
                        return futures;
                    try {
                        // 调用带超时的get方法，这里的参数nanos是剩余的时间
                        // 因为上面其实已经用掉了一些时间了
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    nanos = deadline - System.nanoTime();
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }

}

```

到这里，我们发现，这个抽象类包装了一些基本的方法，可是像subimit、invoke、invokeAll等方法，它们都没有真正开启线程来执行任务，它们都只是在方法内部调用了execute方法，所以最重要的execute(Runnable runnable)方法还没出现，所以等具体执行器来实现这个最重要的部分，这里我们要说的就是ThreadPoolExecutor类了。

## 5.ThreadPoolExecutor

ThreadPoolExecutor是JDK中的线程池实现，这个类实现了一个线程池需要的各个方法，它实现了任务提交、线程管理、监控等等方法。

我们可以基于它来进行业务上的扩展，以实现我们需要的其他功能，比如实现定时任务的类ScheduledThreadPoolExecutor就继承自ThreadPoolExecutor。

首先，我们先看看线程池实现中的几个概念和处理流程。

我们先回顾下提交任务的几个方法:

```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

一个最基本的概念，submit方法中，参数是Runnable类型(也有Callable类型),这个参数不是用于new Thread(runnable).start()中的，此时的这个参数不是用于启动线程的，这里指的是**任务**，任务要做的事情是run()方法里面定义的或Callable()中的call()方法里面定义的。

可能会经常搞混这个，因为Runnable总在各个地方出现，经常把一个Runnable包到另一个Runnable中。请把它想象成有个Task接口，这个接口里面有一个run()方法。

用一个示意图来描述线程池中的一些主要的构件:

![pool-1](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/pool-1.png)

当然，上图没有考虑队列是否有界，提交任务时队列满了怎么办？什么情况下会创建新的线程？提交任务时线程池满了怎么办？空闲线程怎么关掉？这些问题下面我们会一一解决。

我们经常会使用`Executors`这个工具类来快速构造一个线程池。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

不说有什么区别，它们最终都会导向这个构造方法:

```java
  public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        // 这几个参数都是必须要有的
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();

        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

基本上，上面的构造方法中列出了我们最需要关心的几个属性了，下面逐个介绍下构造方法中出现的这几个属性：

- corePoolSize

> 核心线程数

- maximumPoolSize

> 最大线程数，线程池允许创建的最大线程数

- workQueue

> 任务队列，BlockingQueue接口的某个实现(常使用ArrayBlockingQueue和LinkedBlockingQueue)。

- keepAliveTime

> 空闲线程的保活时间，如果某线程的空闲时间超过这个值都没有任务给它做，那么可以被关闭了。注意这个值并不会对所有线程起作用，如果线程池中的线程数少于等于核心线程数corePoolSize,那么这些线程不会因为空闲太长时间而被关闭，当然，也可以通过调用`allowCoreThreadTimeout(true)`使核心线程数内的线程也可以被回收。

- threadFactory

> 用于生成线程数，一般我们可以用默认的就可以了。通常，我们可以通过它将我们的线程的名字设置得比较可读一些，如Message-Thread-1,Message-Thread-2类似这样

- handler

> 当线程池满了，但是又有新的任务提交的时候，该采取什么策略由这个来指定。有几种方式可供选择，像抛出异常、直接拒绝然后等，也可以实现相应的接口实现自己的逻辑

除了上面几个属性外，我们再看看其他重要的属性。

Doug Lea采用一个32位的整数来存放线程池的状态和当前池中的线程数，其中高3位用于存放线程池状态，低29位表示线程数(即使只有29位，也已经不小了，大概5亿多，现在还没有哪个机器能起这么多线程的吧)。我们知道，java语言在整数编码上是统一的，都是采用补码的形式，下面是简单的移位操作和布尔操作，都是挺简单的。

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
 
	// 这里COUNT_BITS设置为29(32-3)，意味着前三位用于存放线程状态，后29位用于存放线程数
	private static final int COUNT_BITS = Integer.SIZE - 3;
	
	// 000 11111111111111111111111111111
	// 这里得到的是29个1，也就是说线程池的最大线程数是2 ^ 29 - 1 = 536870911
	// 以我们现在的计算机的实际情况，这个数量还是够用的
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // 我们说了，线程池的状态存放在高 3 位
	// 运算结果为111跟29个0:111 00000000000000000000000000000
    private static final int RUNNING    = -1 << COUNT_BITS;
	// 000 00000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
	// 001 00000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS;
	// 010 00000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;
	// 011 00000000000000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 将整数c的低29位修改为0，就得到了线程池的状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
	// 将整数c的高3位修改为0，就得到了线程池中的线程数
    private static int workerCountOf(int c)  { return c & CAPACITY; }

    private static int ctlOf(int rs, int wc) { return rs | wc; }

    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
```

上面就是对一个整数的简单的位操作。

在这里，介绍下线程池中的各个状态和状态变化的转换过程:

- RUNNING:这是最正常的状态：接受新的任务，处理等待队列中的任务
- SHUTDOWN:不接受新的任务提交，但是会继续处理等待队列中的任务
- STOP:不接受新的任务提交，不再处理等待队列中的任务，中断正在执行任务的线程
- TIDYING:所有的任务都销毁了，workCount为0。线程池的状态在转换为TIDYING状态时，会执行钩子方法terminated()
- TERIMINATED：terminated()方法结束后，线程池的状态就会变成这个

> RUNNING定义为-1，SHUTDOWN定义为0，其他的都比0大，所以等于0的时候不能提交任务，大于0 的话，连正在执行的任务也需要中断。

各个状态的转换过程有以下几种:

- RUNNING->SHUTDOWN：当调用了shutdown()后，会发生这个状态转换，这也是最重要的。
- (RUNNING or SHUTDOWN) -> STOP:当调用shutdownNow()后，会发生这个状态转换这下要清楚shutDown()和shutDownNow()的区别了
-  SHUTDOWN->TIDYING:当任务队列和线程池都清空后，会由SHUTDOWN转换为TIDYING
- STOP->TIDYING:当任务队列清空后，发生这个转换
- TIDYING->TERMINATED:当terminated()方法结束后

上面的几个记住核心的就可以了，尤其是第一个和第二个。

另外，我们还要看看一个内部类Worker，因为Doug Lea把线程池中的线程包装成了一个个Worker，翻译成工人，就是线程池中做任务的线程，所以到这里，我们知道**任务是Runnable(内部变量名叫task或command)，线程是Worker**.

Worker这里又用到了抽象类AbstractQueuedSynchronizer.

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    private static final long serialVersionUID = 6138294804551838833L;

    // 这个是真正的线程，执行任务
    final Thread thread;
   	
    // 前面说了，这里的Runnable是任务。为什么叫firstTask?因为在创建线程的时候，如果同时指定了
   	// 这个线程起来以后需要执行的第一个任务，那么第一个任务就是存放在这里的(线程可不止执行这一个任务)
    // 当然了，也可以为null，这样线程起来了，自己到任务队列(BlockingQueue)中取任务(getTask方法)就行了
    Runnable firstTask;
   
    // 用于存放此线程完成的任务数，注意了，这里用了volatile，保证可见性
    volatile long completedTasks;

    // Worker 只有这一个构造方法，传入firstTask，也可以传null
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 调用ThreadFactory 来创建一个新的线程
        this.thread = getThreadFactory().newThread(this);
    }

    // 这里调用外部类的runWorker方法
    public void run() {
        runWorker(this);
    }
    
    ...// 其他几个方法没什么好看的，就是用AQS操作，来获取这个线程的执行权，用来独占锁
}
```

我们看看ThreadPoolExecutor的execute方法，前面各种方法最终依赖于execute方法:

```java
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
     	
		// 前面说的那个表示"线程池状态"和"线程数"的整数
        int c = ctl.get();
     
     	// 如果当前线程数少于核心线程数，那么直接添加一个worker来执行任务
     	// 创建一个新的线程，并把当前任务 command作为这个线程的第一个任务(firstTask)
        if (workerCountOf(c) < corePoolSize) {
            // 添加任务成功，那么就结束了。提交任务，线程池已经接收了这个任务，这个方法也就可以返回了
            // 至于执行的结果，到时候会包装到FutureTask中
            // 返回false代表线程池不允许提交任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
     	// 到这里说明，要么当前线程数大于等于核心线程数，要么刚刚addWorker失败了
     
     	// 如果线程处于RUNNING状态，把这个任务添加到任务队列workQueue中
        if (isRunning(c) && workQueue.offer(command)) {
            // 这里面说的是，如果任务进入了workQueue，我们是否需要开启新的线程
            // 因为线程数在[0, corePoolSize)是无条件开启新的线程
            // 如果线程数已经大于等于corePoolSize，那么将任务添加到队列中，然后进到这里
            int recheck = ctl.get();
            // 如果线程池已不处于RUNNING状态，那么移除已经入队的这个任务，并且执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池还是RUNNING的，并且线程数为0，那么开启新的线程
            // 到这里，这块代码的真正意图是：担心任务提交到队列中，但是线程都关闭了
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
     	// 如果workQueue队列满了，那么进入到这个分支
     	// 以 maximumPoolSize为界创建新的worker，
     	// 如果失败，说明当前线程数已经达到maximumPoolSize,执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

> 对创建线程的错误理解：如果线程数少于corePoolSize，创建一个线程，如果线程数在[corePoolSize, maximumPoolSize]之间那么可以创建线程或复用空闲线程，keepAliveTime对这个区间的线程有效。
>
> 从上面几个分支，我们就可以看出，上面的这段话是错误的

下来的这个方法非常重要,addWorker(Runnable firstTask,boolean core)方法，我们看看它是怎么创建线程的:

```java
// 第一个参数是准备提交给这个线程执行的任务，之前说了，可以为null
// 第二个参数为true代表使用核心线程数corePoolSize作为创建线程的界限，也就说创建这个线程的时候
// 		如果线程池中的线程总数已经达到corePoolSize,那么不能响应这次创建线程的请求
// 		如果是false，代表使用最大线程数maximumPoolSize作为界限
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 下面这个有点不好理解
            // 如果线程池已关闭，并满足以下条件之一，那么不创建新的worker:
            // 1.线程池状态大于SHUTDOWN，其实也就是STOP，TIDYING，或TERMINATED
            // 2.firstTask != null
            // 3.workQueue.isEmpty()
            // 简单分析下：
            // 还是状态控制的问题，当线程池处于SHUTDOWN的时候，不允许提交任务，但是已有的任务继续执行
            // 当状态大于SHUTDOWN时，不允许提交任务，且中断正在执行的任务
            // 多说一句：如果线程池处于SHUTDOWN，但是firstTask为null，且workQueue非空，那么是允许创建worker的
            // 这是因为 SHUTDOWN的语义:不允许提交新的任务，但是要把已经进入到workQueue的任务执行完，所以在满足条件的基础上，是允许创建新的worker的
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 如果成功，那么就是所有创建线程前的条件校验都满足了，准备创建线程执行任务了
                // 这里失败的话，说明有其他线程也在尝试往线程池中创建线程
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 由于有并发，重新再读取一下ctl
                c = ctl.get(); 
                // 正常如果是CAS失败的话，进到下一个里层的for循环就可以了
                // 可是如果是因为其他线程的操作，导致线程池的状态发生了变更，如有其他线程关闭了这个线程池
                // 那么需要回到外层的for循环
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }
    
    	/*
    	 * 到这里，我们认为在这个时刻，可以开始创建线程来执行任务了
    	 * 因为该校验的都校验完了，至于以后会发生什么，那是以后的事，至少当前是满足条件的
    	 */

    	// worker是否已经启动
        boolean workerStarted = false;
   		// 是否已将这个worker添加到workers这个HashSet中
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 把firstTask传给worker的构造方法
            w = new Worker(firstTask);
            // 取worker中的线程对象，之前说了，worker的构造方法会调用ThreadFactory来创建一个新的线程
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                // 这个是整个线程池的全局锁，持有这个锁才能让下面的操作“顺理成章”
                // 因为关闭一个线程池需要这个锁，至少我持有锁的期间，线程池不会被关闭
                mainLock.lock();
                try {
                    int rs = runStateOf(ctl.get());
					
                    // 小于SHUTDOWN那就是RUNNING，这个自不必说，是最正常的情况
                    // 如果等于SHUTDOWN，前面说了，不接受新的任务，但是会继续执行等待队列中的任务
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // worker里面的thread不能是启动的
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();
                        // 加到workers这个HashSet中
                        workers.add(w);
                        int s = workers.size();
                        // largestPoolSize用于记录workers中的个数的最大值
                        // 因为workers是不断增加减少的，通过这个值可以知道线程池的大小曾经达到的最大值
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 添加成功的话，启动这个线程
                if (workerAdded) {
                    // 启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
    	// 返回线程是否启动成功
        return workerStarted;
    }

```

简单看下addWorkFailed的处理:

```java
	// workers 中删除掉相应的worker
	// workCount减1
	private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```

回过头来，继续往下走。我们知道，worker中的线程start后，其run方法会调用runWorker方法:

```java
// Worker类的run()方法
public void run() {
    runWorker(this);
}
```

继续往下看runWorker方法：

```java
// 此方法由worker线程启动后调用，这里用一个while循环来不断地从等待队列中获取任务并执行
// 前面说了，worker在初始化的时候，可以指定firstTask，那么第一个任务也就可以不需要从队列中获取
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 该线程第一个任务(如果有的话)
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); 
    boolean completedAbruptly = true;
    try {
        // 循环调用getTask获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
           	// 如果线程池状态大于等于STOP,那么意味着该线程也要中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 这是一个钩子方法，留给需要的子类实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 到这里终于可以执行任务了
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    // 这里不允许抛出Throwable，所以转换为Error
                    thrown = x; throw new Error(x);
                } finally {
                    // 也是一个钩子方法，将task和异常作为参数，留给需要的子类实现
                    afterExecute(task, thrown);
                }
            } finally {
                // 置空task，准备getTask获取下一个任务
                task = null;
                // 累加完成的任务数
                w.completedTasks++;
                // 释放掉worker的独占锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 如果到这里，需要执行线程关闭:
        // 1.说明getTask返回null，也就是说，队列中已经没有任务需要执行了，执行关闭
        // 2.任务执行过程中发生了异常
        // 第一种情况，已经在代码处理了将workCount减1，这个在getTask方法分析中会说
        // 第二种情况，workCount没有处理，所以需要在processWorkerExit中处理
        processWorkerExit(w, completedAbruptly);
    }
}
```

我们看看getTask()是怎么获取任务的，这个方法写的很好，每一行都很简单，组合起来却所有的情况都考虑到了。

```java
	// 此方法由三种可能
	// 1.阻塞直至获取到任务返回。我们知道，默认corePoolSize之内的线程是不被回收的
	// 		它们会一直等待任务
	// 2.超时退出。KeepAliveTime起作用的时候，也就是说如果这么多时间内都没有任务，那么应该执行关闭
	// 3.如果发生了以下条件，此方法必须返回null:
	// 		- 池中有大于maximumPoolSize个workers存在(通过调用setMaxPoolSize进行设置)
	// 		- 线程池处于 SHUTDOWN，而且workQueue是空的，前面说了，这种不再接收新的任务
	// 		- 线程池处于 STOP，不仅不接受新的线程，连workQueue中的线程也不再执行
	private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
			
            // 两种可能
            // 1. rs == SHUTDOWN && workQueue.isEmpty()
            // 2. rs >= STOP
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
               	// cAS 操作，减少工作线程数
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 允许核心线程数内的线程回收，或当前线程数超过了核心线程数，那么有可能发生超时关闭
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // 两个判断一起看：如果当前线程数wc > maximumPoolSize，或者超时，都返回null
            // 那这里的问题来了，wc > maximumPoolSize的情况，为什么要返回null?
            // 		换句话说，返回null意味着关闭线程.
            // 	那是因为有可能开发者调用了setMaximumPoolSize()将线程池的maximumPoolSize调小了，那么多余的Worker就需要被关闭
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            // wc <= maximumPoolSize同时没有超时
            try {
                // 到workQueue中获取任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                // 如果此worker发生发生了中断，采取的方案是重试
                // 解释下为什么会发生中断，这个读者要去看setMaximumPoolSize方法.
                
                // 如果开发者将maximumPoolSize调小了，导致其小于当前的workers数量
                // 那么意味着超出的部分线程要被关闭。重新进入for循环，自然会有部分线程返回null
                timedOut = false;
            }
        }
    }
```

到这里，基本上也说完了整个流程，读者这个时候应该回到execute(Runnable command)方法，看看各个分支。

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
     	
		// 前面说的那个表示"线程池状态"和"线程数"的整数
        int c = ctl.get();
     
     	// 如果当前线程数少于核心线程数，那么直接添加一个worker来执行任务
     	// 创建一个新的线程，并把当前任务 command作为这个线程的第一个任务(firstTask)
        if (workerCountOf(c) < corePoolSize) {
            // 添加任务成功，那么就结束了。提交任务，线程池已经接收了这个任务，这个方法也就可以返回了
            // 至于执行的结果，到时候会包装到FutureTask中
            // 返回false代表线程池不允许提交任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
     	// 到这里说明，要么当前线程数大于等于核心线程数，要么刚刚addWorker失败了
     
     	// 如果线程处于RUNNING状态，把这个任务添加到任务队列workQueue中
        if (isRunning(c) && workQueue.offer(command)) {
            // 这里面说的是，如果任务进入了workQueue，我们是否需要开启新的线程
            // 因为线程数在[0, corePoolSize)是无条件开启新的线程
            // 如果线程数已经大于等于corePoolSize，那么将任务添加到队列中，然后进到这里
            int recheck = ctl.get();
            // 如果线程池已不处于RUNNING状态，那么移除已经入队的这个任务，并且执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池还是RUNNING的，并且线程数为0，那么开启新的线程
            // 到这里，这块代码的真正意图是：担心任务提交到队列中，但是线程都关闭了
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
     	// 如果workQueue队列满了，那么进入到这个分支
     	// 以 maximumPoolSize为界创建新的worker，
     	// 如果失败，说明当前线程数已经达到maximumPoolSize,执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

上面各个分支中，有两种情况会调用reject(command)来处理任务，因为按照正常的流程，线程池此时不能接受这个任务，所以需要执行我们的拒绝策略。接下来，我们说一下ThreadPoolExecutor中的拒绝策略。

```java
	final void reject(Runnable command) {
        // 执行拒绝策略
        handler.rejectedExecution(command, this);
    }
```

此处的handler我们需要在构造线程池的时候就传入这个参数，它是RejectedExecutionHandler的实例。

RejectedExecutionHandler在ThreadPoolExecutor中有四个已经定义好的实现类可供我们直接使用，当然，我们也可以实现自己的策略，不过一般也没必要问题。

```java
// 只要线程池没有被关闭，那么由提交任务的线程自己来执行这个任务。
public static class CallerRunsPolicy implements RejectedExecutionHandler {
        public CallerRunsPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

// 不管怎样，直接抛出RejectedExecutionException异常
// 这个是默认的策略，如果我们构造线程池的时候不传入相应的handler的话，那就回指定使用这个
public static class AbortPolicy implements RejectedExecutionHandler {
        public AbortPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

// 不做任何处理，直接忽略掉这个任务
 public static class DiscardPolicy implements RejectedExecutionHandler {
        public DiscardPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

// 这个相对霸道一点，如果线程池没有被关闭的话
// 把队列队头的任务（也就是等待了最长时间的)直接扔掉，然后提交这个任务到等待队列中
 public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public DiscardOldestPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

到这里,ThreadPoolExecutor的源码分析结束了。

## 6.Executors

Executors它仅仅是工具类，它的所有方法都是static的。

- 生成一个固定大小的线程池:

```java
  public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

最大线程数设置为与核心线程数相等，此时keepAliveTime设置为0(因为这里它是没用的，即使不为0，线程池默认也不会回收corePoolSize内的线程)，任务队列采用LinkedBlockingQueue，无界队列。

过程分析：刚开始，每提交一个任务，都创建一个worker，当worker的数量达到nThreads后，不再创建新的线程，而是把任务提交到LinkedBlockingQueue中，而且之后线程数始终为nThreads.

- 生成只有**一个线程**的固定线程池，这个更简单，和上面的一样，只要设置线程数为1就可以了:

```java
   public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

- 生成一个需要的时候就创建新的线程，同时可以复用之前创建的线程(如果这个线程当前没有任务)的线程池:

```java
 public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

核心线程数为0，最大线程数为Integer.MAX_VALUE,keepAliveTime为60秒，任务队列采用SynchronousQueue.

这种线程池对于任务可以比较快速地完成的情况有比较好性能。如果线程空闲了60秒都没有任务，那么将关闭此线程并从线程池中移除。所以如果线程池空闲了很长时间也不会有问题，因为随着所有的线程都会被关闭，整个线程池不会占用任何的系统资源。

过程分析：我把execute方法的主体粘贴过来，让大家看明白点。鉴于corePoolSize是0，那么提交任务的时候，直接将任务提交到队列中，由于采用了SynchronousQueue，所以如果是第一个任务提交的时候，offer方法肯定会返回false，因为此时没有任何worker对这个任务进行接收，那么将进入到最后一个分支来创建第一个worker。之后再提交任务的话，取决于是否有空闲下来的线程对任务进行接收，如果有，会进入到第二个if语句块中，否则就是和第一个任务一样，进到最后的else if分支创建新线程。

```java
int c = ctl.get();
// corePoolSize 为 0，所以不会进到这个 if 分支
if (workerCountOf(c) < corePoolSize) {
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
// offer 如果有空闲线程刚好可以接收此任务，那么返回 true，否则返回 false
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
else if (!addWorker(command, false))
    reject(command);
```

> SynchronousQueue是一个比较特殊的BlockingQueue，其本身不存储任何元素，它有一个虚拟队列(或虚拟栈),不管读操作还是写操作，如果当前队列中存储的是与当前操作相同模式的线程，那么当前操作也进入队列中等待；如果是相反模式，则配对成功，从当前队列中取头节点。

## 7.总结

一些重点内容。

- java 线程池有哪些关键属性？

> corePoolSize，maximumPoolSize，workQueue，keepAliveTime，rejectedExecutionHandler
>
> corePoolSize 到 maximumPoolSize 之间的线程会被回收，当然 corePoolSize 的线程也可以通过设置而得到回收（allowCoreThreadTimeOut(true)）。
>
> workQueue 用于存放任务，添加任务的时候，如果当前线程数超过了 corePoolSize，那么往该队列中插入任务，线程池中的线程会负责到队列中拉取任务。
>
> keepAliveTime 用于设置空闲时间，如果线程数超出了 corePoolSize，并且有些线程的空闲时间超过了这个值，会执行关闭这些线程的操作
>
> rejectedExecutionHandler 用于处理当线程池不能执行此任务时的情况，默认有**抛出 RejectedExecutionException 异常**、**忽略任务**、**使用提交任务的线程来执行此任务**和**将队列中等待最久的任务删除，然后提交此任务**这四种策略，默认为抛出异常。

- 说说线程池中的线程创建时机？

> - 如果当前线程数少于 corePoolSize，那么提交任务的时候创建一个新的线程，并由这个线程执行这个任务；
> - 如果当前线程数已经达到 corePoolSize，那么将提交的任务添加到队列中，等待线程池中的线程去队列中取任务；
> - 如果队列已满，那么创建新的线程来执行任务，需要保证池中的线程数不会超过 maximumPoolSize，如果此时线程数超过了 maximumPoolSize，那么执行拒绝策略。

**注意**：如果将队列设置为无界队列，那么线程数达到 corePoolSize 后，其实线程数就不会再增长了。因为后面的任务直接往队列塞就行了，此时 maximumPoolSize 参数就没有什么意义。

- 任务执行过程中发生异常怎么处理？

> 如果某个任务执行出现异常，那么执行任务的线程会被关闭，而不是继续接收其他任务。然后会启动一个新的线程来代替它。

- 什么时候会执行拒绝策略？

> 1. workers 的数量达到了 corePoolSize（任务此时需要进入任务队列），任务入队成功，与此同时线程池被关闭了，而且关闭线程池并没有将这个任务出队，那么执行拒绝策略。这里说的是非常边界的问题，入队和关闭线程池并发执行，读者仔细看看 execute 方法是怎么进到第一个 reject(command) 里面的。
> 2. workers 的数量大于等于 corePoolSize，将任务加入到任务队列，可是队列满了，任务入队失败，那么准备开启新的线程，可是线程数已经达到 maximumPoolSize，那么执行拒绝策略。