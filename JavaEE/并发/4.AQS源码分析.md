# 基于AQS的ReentrantLock

## 1.AQS结构

先来看看AQS有哪些属性。

```java
 
	// 头结点，可以直接把它当做 当前持有锁的线程
    private transient volatile Node head;

    // 阻塞的尾结点，每个新的节点进来，都插入到最后，也就形成了一个链表
    private transient volatile Node tail;

   	// 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于0代表有线程持有当前锁
	// 这个值可以大于1，是因为锁可以重入，每次重入都加上1
    private volatile int state;
	
	// 代表当前持有独占锁的线程，举个最重要的例子，因为锁可以重入
	// reentrantLock.lock()可以嵌套多次，所以每次用这个来判断线程是否已经拥有了锁
	// if (currentThread == getExclusiveOwnerThread()) {state++}
    private transient Thread exclusiveOwnerThread; // 这个属性继承自AbstractOwnableSynchonizer
```

AbstractQueuedSynchronizer的等待队列示意如下所示，注意，之后的分析过程所说的queue，也就是阻塞队列**不包含head**。![aqs-0](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/aqs-0.png)

等待队列中每个线程被包装成Node实例，数据结构是链表，来看看源码。

```java
static final class Node {
        // 标识节点当前在共享模式下
        static final Node SHARED = new Node();
        // 标识节点当前在独占模式下
        static final Node EXCLUSIVE = null;

        // =======下面的几个int常量是给waitStatus用的========
    	// 代表此线程取消了争夺这个锁
        static final int CANCELLED =  1;
        // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
        static final int SIGNAL    = -1;
        
        static final int CONDITION = -2;
        
        static final int PROPAGATE = -3;

    	// 可以这样理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待
    	//  		ps：半天抢不到锁了，不抢了，ReentrantLock是可以指定timeout的
        volatile int waitStatus;
		// 前驱节点的引用
        volatile Node prev;
		// 后继节点的引用
        volatile Node next;
		// 这个就是线程自身
        volatile Thread thread;
}
```

Node的数据结构其实也挺简单的，就是thread + waitStatus+pre+next四个属性而已，需要记在心里。

## 2.ReentrantLock

### 2.1.ReentrantLock的公平锁

下面，说一下ReentrantLock的公平锁，再次强调，这里的阻塞队列不包含head节点。

![aqs-0](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/aqs-0.png)

首先，我们先看下ReenTrantLock的使用方式。

```java
// 我用个web开发中的service概念吧
public class OrderService {
    // 使用static，这样每个线程拿到的是同一把锁，当然，spring mvc中service默认就是单例，别纠结这个
    private static ReentrantLock reentrantLock = new ReentrantLock(true);

    public void createOrder() {
        // 比如我们同一时间，只允许一个线程创建订单
        reentrantLock.lock();
        // 通常，lock 之后紧跟着 try 语句
        try {
            // 这块代码同一时间只能有一个线程进来(获取到锁的线程)，
            // 其他的线程在lock()方法上阻塞，等待获取到锁，再进来
            // 执行代码...
            // 执行代码...
            // 执行代码...
        } finally {
            // 释放锁
            reentrantLock.unlock();
        }
    }
}
```

ReenTrantLock在内部用了Sync来管理锁，所以真正的获取锁和释放锁是由sync的实现类来控制的。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
}
```

Sync有两个实现，分别为NonFairSync(非公平锁）和FairSync(公平锁)，我们先看下FairSync部分。

```java
   public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

#### 2.1.1.线程抢锁

下面跟着代码走。

```java
 static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
		// 争锁
        final void lock() {
            acquire(1);
        }
     
     	// 这个方法来自父类AQS
     	// 我们看到，这个方法，如果tryAcquire(arg)，返回true，也就结束了
     	// 否则，acquireQueued方法会将线程压到队列中
     	public final void acquire(int arg) { // 此时arg == 1
            // 首先调用tryAcquire(1)一下，从名字中可以知道，就只是尝试一下
            // 因为有可能直接就成功了，也就不需要进队列排队了。
            // 对于公平锁的语义就是：本来就没人持有锁，根本没必要进队列等待(又是挂起，又是等待被唤醒的)
            if (!tryAcquire(arg) &&
                // tryAcquire(arg)没有成功,这个时候需要把当前线程挂起，放到阻塞队列中。
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        	}
     
     
     // tryAcquire在AQS是个抽象方法，因此需要实现，ReenTrantLock中的公平锁的实现方法是这样的
     // 尝试直接获取锁，返回值是boolean,代表是否获取到锁
     // 返回true:1.没有线程在等待锁；2.重入锁，线程本来就持有锁，也就可以理所当然可以直接获取
     protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
         	// state == 0 此时此刻没有线程持有锁
            if (c == 0) {
                // 虽然此时此刻是可以用的，但是这是公平锁，既然是公平，就得考虑先来后到。
                // 看看有没有别人在队列中等了半天了
                if (!hasQueuedPredecessors() && // 这个方法是判断队列是否存在前任
                    // 如果没有线程等待，那就用CAS尝试一下，成功了就获取到锁了，
                    // 不成功的话，只能说明一个问题，就在刚刚几乎同一时刻有个线程抢占了
                    // 因为刚刚还没人的，判断过了
                    compareAndSetState(0, acquires)) {
                    
                    // 到这里就是获取锁了，标记一下，告诉大家，现在是我占用了锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
          	// 会进入这个else if分支，说明重入了，需要操作:state = state + 1
         	// 这里不存在并发问题
            else if (current == getExclusiveOwnerThread()) { // 如果当前线程是持有锁的线程，说明可重入了
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
         
         	// 如果到这里，说明前面的if和else if 都没有true，说明没有获取到锁
         	// 回到上面一个外层调用方法继续看.
          	// if (!tryAcquire(arg) &&
            //    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //    selfInterrupt();
     		// }
            return false;
        }
     
     // 假设tryAcquire(arg)返回false，那么代码将执行:
     // 		acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
     // 这个方法，首先需要执行:addWaiter(Node.EXCLUSIVE)
     
     // 此方法的作用是把线程包装成node,同时进入到队列中
     // 参数mode此时是Node.EXCLUSIVE，表示独占模式
     private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // 以下几行代码想把当前node加到链表的最后面去，也就是进到阻塞队列的最后
        Node pred = tail;
         
         // tail != null => 队列不为空(tail == head)的时候，其实队列是空的，不管暂时不管这个
        if (pred != null) {
            // 把当前的队尾节点，设置为自己的前驱
            node.prev = pred;
            // 用CAS把自己设为队尾，如果成功后，tail == node了，这个节点成为阻塞队列新的尾巴
            if (compareAndSetTail(pred, node)) {
                // 进到这里说明设置成功了，当前ndoe == tail，将自己与之前的队尾相连。
                // 上面已经有 node.prev = pred，加上下面这句，也就实现了和之前的尾结点双向连接了
                pred.next = node;
                // 线程入队了，可以返回了
                return node;
            }
        }
         
        // 仔细看看上面的代码，如果会到这里
        // 说明 pred == null(队列是空的) 或者 CAS失败(有线程在竞争入队)
        enq(node);
        return node;
    }
     
     // 采用自旋的方式入队
     // 之前说过，到这个方法只有两种可能:等待队列为空，或者有线程竞争入队
     // 自旋在这边的语义是:CAS设置tail过程中，竞争一次竞争不到，我就多次竞争，总会排到的
      private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 之前说过，队列为空也会进来这里
            if (t == null) { // Must initialize
                // 初始化head节点
                // 原来 head 和 tail 初始化的时候都是 null的
                // 还是一步CAS，现在可能还是很多线程同时进来的
                if (compareAndSetHead(new Node()))
                    // 给后面用:这个时候head节点的waitStatus == 0，看 new Node()的构造方法就知道了
                    // 这个时候有了head，但是tail还是null，设置一下
                    // 把tail指向head，这个时候不要担心，马上就要有线程来了，到时候tail要被抢了
                    // 注意：这里只是设置了tail=head,这里没有return
                    // 所以，设置完了以后，继续for循环，下面就到下面的else分支了
                    tail = head;
            } else {
                // 下面几行，和上一个方法,addWaiter是一样的，
              	// 只是这个套在无限循环里，反正就是将当前线程排到队尾，有线程竞争的话排不上就重复排
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
     
     // 现在，又回到这段代码了
     // if (!tryAcquire(arg) 
    //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
    //     selfInterrupt();
     
     // 下面这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入阻塞队列
     // 注意一下：如果acquireQueued(addWaiter(Node.EXCLUSIVE),arg)返回true的话，
     // 意味着上面这段代码将进入selfInterrupt()，所以正常情况下，下面应该返回false
     // 这个方法非常重要，应该说是真正的线程挂起，然后被唤醒后去获取锁，都在这个方法里了
     final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                // p == head 说明当前节点虽然进到了阻塞队列，但是是阻塞队列的第一个，因为它的前驱是head
                // 注意，阻塞队列不包含head节点，head一般指的是占有锁的线程，head后面的才称为阻塞队列
                // 所以当前节点可以去试抢一下锁
                // 这里我们说一下，为什么可以去试试
                // 首先，它是队头，这个是第一个条件，其次，当前的head有可能是刚刚初始化的node，
                // enq(node)方法里面提到，head是延迟初始化的，而且new Node()的时候没有设置任何线程
                // 也就是说，当前的head不属于任何一个线程，所以作为队头，可以去试一试，
                // tryAcquire已经分析过了，就是简单用CAS是操作一下state
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 到这里，说明上面的if分支没有成功，要么当前node本来就不是队头，
                // 要么就是tryAcquire(arg)没有抢赢别人，继续往下看
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 说明时候 failed 会为 true ?
            // tryAcquire()方法抛异常到的情况
            if (failed)
                cancelAcquire(node);
        }
    }
     
     // 刚刚说过，会到这里就是没有抢到锁，这个方法说的是：“当前线程没有抢到锁，是否需要挂起当前线程？”
     // 第一个参数时前驱节点，第二个参数才是代表当前线程的节点。
     private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
         // 前驱节点的 waitStatus == -1，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
        if (ws == Node.SIGNAL)
            return true;
         
         
         // 前驱节点waitStatus大于0，之前说过，大于0说明前驱节点取消了排队。
         // 这里需要知道这点，进入阻塞队列的线程会被挂起，而唤醒的操作是由前驱节点完成的
         // 所以下面这块代码说的是将当前节点的prev指向waitStatus <= 0 的节点
         // 简单说，就是为了找个好爹，因为你还得依赖它来唤醒呢，如果前驱节点取消了排队,
         // 找前驱节点的前驱节点做爹，往前遍历总能找到一个好爹的。
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            // 仔细想想，如果进入到这个分支意味着什么
            // 前驱节点的waitStatus不等于-1和1，那也就是只可能是0,-2,-3
            // 在我们前面的源码中，都没有看到有设置waitStatus的，所以每个新的node入队时，waitStatus都是0
            // 在正常情况下，前驱节点是当前的tail，那么它的waitStatus应该是0
            // 用CAS将前驱节点的waitStatus设置为Node.SINGAL(也就是-1)
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
         // 然后方法返回false.那么会再走一次for循环，
         // 	然后再次进来此方法，此时就会从第一个分支返回true
        return false;
    }

  	// private static boolean shouldParkAfterFailedAcquire(Node pred, Node node)
    // 这个方法结束根据返回值我们简单分析下:
    // 如果返回true,说明前驱节点的waitStatus == -1,是正常情况下，那么当前线程需要被挂起，等待以后被唤醒
     // 	我们也说过，以后是被前驱节点唤醒，就等着前驱节点拿到锁，然后释放锁的时候叫你好了
     // 如果返回false，说明当前不需要挂起，为什么呢?往后看
     
     // 跳回前面是这个方法
     // if (shouldParkAfterFailedAcquire(p, node) &&
     //               parkAndCheckInterrupt())
     //               interrupted = true;
     //      }
     
     // 1.如果shouldParkAfterFailedAcquire(p,node)返回true。
     // 那么需要执行parkAndCheckInterrupt();
     
     // 这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的
     // 这是用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒=========
      private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
     
     // 2. 接下来说说如果shouldParkAfterFailedAcquire(p, node)返回false的情况
     
     // 仔细看shouldParkAfterFailedAcquire(p, node)，我们可以发现，其实第一次进来的时候，一般都不会返回true，原因很简单，前驱节点的waitStatus=-1,是依赖于后继节点设置的。也就是说，我都还没给前驱设置-1呢，怎么可能是true呢，但是要看到，这个方法是套在循环里的，所以第二次进来的状态就是-1了。
     
     // 解释下为什么shouldParkAfterFailedAcquire(p,node) 返回false的时候不直接挂起线程:
     // => 是为了应对经过这个方法后，node已经是head的直接后继节点了。
 }
```

说到这里，也就明白了，多看几遍`final boolean acquireQueued(final Node node, int arg)`这个方法吧。自己推演下各个分支怎么走，那种情况下会发生什么，走到哪里。

#### 2.1.2.解锁操作

介绍下唤醒的动作。我们知道，正常情况下，如果线程没获得到锁，线程会被`LockSupport.park(this)`挂起停止，等待被唤醒；

```java
// 唤醒的代码还是比较简单的，如果上面加锁的看懂了，下面很简单
 public void unlock() {
        sync.release(1);
    }

public final boolean release(int arg) {
    	// 往后看
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

// 回到ReentrantLock看tryRelease方法
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否完全释放锁
    boolean free = false;
    // 其实就是重入的问题，如果c==0.也就是说没有嵌套锁了，可以释放了，否则还不能释放掉
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
   }

// 唤醒后继节点
// 从上面调用处知道，参数node是head头结点
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    // 如果head节点当前waitStatus<0,将其修改为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 下面的代码就是唤醒后节点，但有可能后继节点取消了等待
    // 从队尾往前找，找到waitStatus<=0的所有节点中排在最前面的
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后往前找，仔细看代码，不必担心中间有节点取消(waitStatus==1)的情况
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒线程
        LockSupport.unpark(s.thread);
   }
```

唤醒线程以后，被唤醒的线程将从以下代码中继续往前走:

```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);  // 刚刚线程被挂起在这里了
        return Thread.interrupted();
 }
// 又回到这个方法了:acquireQueued(final Node node, int arg),这个时候，node的前驱是head了
```

#### 2.1.3.总结

在并发情况下，加锁和解锁需要以下三个部件的协调:

- 锁状态。我们要知道锁是不是被别的线程占有了，这个就是state的作用，它为0的时候代表没有线程占有锁，可以去争抢这个锁，用CAS将state设为1，如果CAS成功，说明抢到了锁，这样其他线程就抢不到了，如果锁重入的话，state进行+1就可以，解锁就是减1，直到state又变为0，代表释放锁，所以lock()和unlock()必须要配置啊。然后唤醒等待队列中的第一个线程，让其来占有锁。
- 线程的阻塞和解除阻塞。AQS采用了LockSupport.park(thread)来挂起线程，用unpark来唤醒锁。
- 阻塞队列。因为争抢锁的线程可能很多，但是只能有一个线程拿到锁，其他的线程都必须等待，这个时候就需要一个queue来管理这些线程，AQS用的是一个FIFO的队列，就是一个链表，每个node都持有后继节点的引用。

#### 2.1.4.示例图解析

简单的实例。

首先，第一个线程调用reentrantLock.lock(),翻到最上面可以发现，tryAcquire(1)直接就返回true了，结束。只是设置了state=1，连head都没有初始化，更谈不上什么阻塞队列了。要是线程1调用unlock()了，才有线程2来，那就不需要AQS了。

如果线程1没有调用unlock()之前，线程2调用了lock(),想想会发生什么?

线程2会初始化head,同时线程2也会插入到阻塞队列并挂起(注意看这里是一个for循环，并且设置head和tail的部分是不return的，只有入队成功才会跳出循环)

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

首先，是线程2初始化head节点，此时head == tail，waitStatus == 0

![aqs-1](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/aqs-1.png)

然后线程2入队:

![aqs-2](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/aqs-2.png)

同时我们也要看到此时节点的waitStatus，我们知道head节点是线程2初始化的，此时的waitStatus没有设置，java默认会设置为0，但是到shouldParkFailedAcquire这个方法的时候，线程2会把前驱节点，也就是head的waitStatus设置为-1。

那线程2节点此时的waitStatus是多少呢，由于没有设置，所以是0；

如果线程3此时再进来，直到插到线程2的后面就可以了，此时线程3的waitStatus是0，到shouldParkAfterFailedAcquire方法的时候把前驱节点线程2的waitStatus设置为-1.

![aqs-3](C:\Users\DELL\Desktop\aqs-3.png)

这里可以简单说下waitStatus中SINGAL(-1)状态的意思，源码中注释的是:代表后继节点需要被唤醒。也就是说这个waitStatus其实代表的不是自己的状态，而是后继节点的状态，我们知道，每个node在入队的时候，都会把前驱节点的状态改为SINGAL，然后阻塞，等待被前驱唤醒。

### 2.2.公平锁与非公平锁的不同

ReentrantLock默认采用非公平锁，除非在默认方法中初入参数true.

```java
 	public ReentrantLock() {
        // 默认非公平锁
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

#### 2.2.1.lock操作

公平锁的lock方法

```java
static final class FairSync extends Sync {

    final void lock() {
        acquire(1);
    }
    
    
     public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
     protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 1.和非公平锁相比，这里多了一个判断：是否有线程在等待
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
}
```

非公平锁的lock方法

```java
static final class NonfairSync extends Sync {
    
    final void lock() {
        // 2. 和非公平锁相比，这里会直接先进行一次CAS，成功就返回了
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    
      public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
      protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    
    final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 这里没有对阻塞队列进行判断
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

#### 2.2.2.总结

公平锁和非公平锁只有两处不同:

- 非公平锁在调用lock后，首先就会调用CAS进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
- 非公平锁在CAS失败后，和公平锁一样都会进入到tryAcquire方法，在tryAcquire方法中，如果发现锁这个时候释放了(state == 0)，非公平锁会直接CAS抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。

公平锁和非公平锁就这两点区别，如果这两次CAS都不成功，那么后面非公平锁和公平锁是一样的，都要进入阻塞队列等待唤醒。

相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致阻塞队列中的线程长期处于饥饿状态。