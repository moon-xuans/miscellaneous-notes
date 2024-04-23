# ReentrantLock源码剖析 

## 1.1.FairSync

### 1.1.1.lock操作

```java
final void lock() {
    // 这里会调用父类也就是AQS的获取操作
    acquire(1);
}

public final void acquire(int arg) {
    	// 这里首先会尝试获取一下,在AQS中是个抽象方法，需要子类实现，即调用公平锁的实现方法
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread(); // 获取当前线程
            int c = getState(); // 这里调用AQS的获取状态方法，0代表锁没有被占用，大于0则说明重入的次数			
    		// 说明此时锁没有被占用
            if (c == 0) {
                // 由于是公平锁，看看队列中有没有等待时间过长的线程，判断是否存在前驱元素
                if (!hasQueuedPredecessors() &&
                    // 没有的话，则试着去通过CAS去占用，这里失败的原因是可能有这一瞬间有线程在进行争抢
                    compareAndSetState(0, acquires)) { 
                    setExclusiveOwnerThread(current); // 占用成功的话，则将此线程设置为独占
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) { // 此时状态大于0，可以获取独占的线程，判断是否与当前线程相同，相同的话，则说明可重入
                int nextc = c + acquires; // 重入次数++
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

           
// 如果说占用锁失败，则会将此线程放入阻塞队列中            
// if (!tryAcquire(arg) &&
//  acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
//		selfInterrupt();


private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode); // 将此线程包装成一个独占的线程
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail; // 获取尾结点
        if (pred != null) { // 尾结点不为空的话.则进行指向
            node.prev = pred; 
            if (compareAndSetTail(pred, node)) { // 并将这个节点通过CAS设置成尾结点
                pred.next = node; // 进行指向
                return node;
            }
        }
        enq(node); // 到这里了，说明尾结点为空，则进行自旋入队
        return node;
    }


private Node enq(final Node node) {
        for (;;) {
            Node t = tail; // 获取尾结点
            if (t == null) { // 若尾结点为空
                if (compareAndSetHead(new Node())) // 先创建一个节点，并通过CAS设置成头结点
                    tail = head; // 给尾结点赋值同样的节点
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) { // 通过CAS将这个节点设置成尾结点，并进行指向
                    t.next = node;
                    return t;
                }
            }
        }
    }

// 添加到阻塞队列之后,调用acquireQueued(final Node node, int arg)方法
 final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor(); 
                if (p == head && tryAcquire(arg)) { // 先判断当前节点的前驱节点是否是阻塞队列的第一个节点，如果是的话，可以尝试去抢夺锁。因为它是队头，并且head节点可能是之前创建的node，并没有进行赋值
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 到这里，是因为node不是队头，或者没有抢过别人，因此去将获取失败锁的进行挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }


private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus; // 获取前驱节点的状态
        if (ws == Node.SIGNAL) // 若前驱节点状态为SINGAL，说明当前节点状态正确，可以去挂起了
            return true;
        if (ws > 0) { // 如果是前驱节点的状态大于0，则说明已经取消等待了
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do { // 则往前找，找到状态为-1的
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else { // 这个条件则说明前驱节点的状态为0，则说明是刚刚创建的，并没有状态
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL); // 因此，需要到这里将前驱节点设置为SINGAL
        }
        return false;
    }


 //if (shouldParkAfterFailedAcquire(p, node) &&
 //                   parkAndCheckInterrupt())
 //                   interrupted = true;
 // shouldParkAfterFailedAcquire(p, node)返回true后，则说明该节点可以挂起了，调用parkAndCheckInterrupt()

  private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

### 1.1.2.unlock操作

```java
public void unlock() {
    sync.release(1); // 这里调用的是AQS的release方法
}

public final boolean release(int arg) {
   		// 先去尝试释放锁
        if (tryRelease(arg)) { // tryRelease()方法是个抽象方法，因此这里调用的是子类公平锁的实现方法
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

 protected final boolean tryRelease(int releases) { 
            int c = getState() - releases; // 计算减去后的重入次数
            if (Thread.currentThread() != getExclusiveOwnerThread()) // 如果说解锁线程不是加锁线程，则抛出非法监控状态异常
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) { // 如果说次数为0，则说明锁不被占用了
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

// 		if (tryRelease(arg)) {  // 释放成功的话
//			 Node h = head; // 获取头节点 
//            if (h != null && h.waitStatus != 0) // 如果说头结点不为空，并且状态不为0，则说明要唤醒后面的线程
//                unparkSuccessor(h);
//            return true;


private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0) // 如果head节点当前waitStatus<0,将其修改为0
            compareAndSetWaitStatus(node, ws, 0);  // 这里状态置为0，是为了后来节点将它会置为-1
		// 接着就要唤醒后面的节点了，但是后继节点可能取消了等待
        Node s = node.next; 
        if (s == null || s.waitStatus > 0) { // 如果后继节点为空，或者此时它取消等待了，则重新找一个
            s = null; // 先释放空间
            for (Node t = tail; t != null && t != node; t = t.prev) // 从后往前找，找到那个waitStatus<=0最前面的，这个节点说明后继节点需要去唤醒
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null) // 如果找到了这个节点，则去唤醒
            LockSupport.unpark(s.thread);
    }

 public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
// 又回到这个方法了:acquireQueued(final Node node, int arg),这个时候，node的前驱是head了
```