AQS 源码主要关注的队列这一部分。

## 同步状态
```java
private volatile int state;

protected final int getState() {
    return state;
}

protected final void setState(int newState) {
    state = newState;
}


protected final boolean compareAndSetState(int expect, int update) {
    return U.compareAndSetInt(this, STATE, expect, update);
}

```

其中，`U`是指 `Unsafe`类:

```java
private static final Unsafe U = Unsafe.getUnsafe();
```

由于 `state`是 volatile 的，所以，在任何时刻，返回的都是都是最新的值。

## 队列节点
```java
abstract static class Node {
    volatile Node prev;       // initially attached via casTail
    volatile Node next;       // visibly nonnull when signallable
    Thread waiter;            // visibly nonnull when enqueued
    volatile int status;      // written by owner, atomic bit ops by others
    
    // methods for atomic operations
    final boolean casPrev(Node c, Node v) {  // for cleanQueue
        return U.weakCompareAndSetReference(this, PREV, c, v);
    }
    final boolean casNext(Node c, Node v) {  // for cleanQueue
        return U.weakCompareAndSetReference(this, NEXT, c, v);
    }
    final int getAndUnsetStatus(int v) {     // for signalling
        return U.getAndBitwiseAndInt(this, STATUS, ~v);
    }
    final void setPrevRelaxed(Node p) {      // for off-queue assignment
        U.putReference(this, PREV, p);
    }
    final void setStatusRelaxed(int s) {     // for off-queue assignment
        U.putInt(this, STATUS, s);
    }
    final void clearStatus() {               // for reducing unneeded signals
        U.putIntOpaque(this, STATUS, 0);
    }
    
    private static final long STATUS
        = U.objectFieldOffset(Node.class, "status");
    private static final long NEXT
        = U.objectFieldOffset(Node.class, "next");
    private static final long PREV
        = U.objectFieldOffset(Node.class, "prev");
}

```

### 节点的类型
注意到，这个 CLH node 是 `abstract`的，这是因为还需要区分这个节点是独占模式还是共享模式下的。并没有在节点中设置 type 这一个属性：

```java
// Concrete classes tagged by type
static final class ExclusiveNode extends Node { }
static final class SharedNode extends Node { }
```

在理论部分说过，条件队列的节点也是 CLH 节点：

```java
static final class ConditionNode extends Node
    implements ForkJoinPool.ManagedBlocker {
    ConditionNode nextWaiter;            // link to next waiting node

    /**
     * Allows Conditions to be used in ForkJoinPools without
     * risking fixed pool exhaustion. This is usable only for
     * untimed Condition waits, not timed versions.
     */
    public final boolean isReleasable() {
        return status <= 1 || Thread.currentThread().isInterrupted();
    }

    public final boolean block() {
        while (!isReleasable()) LockSupport.park();
        return true;
    }
}
```

为什么 `ConditionNode`类还要添加 `nextWaiter`属性呢，而不是直接使用 `Node`的 `next`属性？

+ 当 ConditionNode 被从条件队列转移到等待锁的 CLH 队列中时，加入到 CLH 队列中的节点是 Node 类型，而其中的 next 属性肯定不能指向一个外部的，不在 CLH 队列的 ConditionNode

### 节点的状态
```java
// Node status bits, also used as argument and return values
static final int WAITING   = 1;          // must be 1
static final int CANCELLED = 0x80000000; // must be negative
static final int COND      = 2;          // in a condition wait
```

其中的 `WAITING`字段相当于理论部分的**唤醒位**（signal me）。

除了这 3 个状态以外，我以为还有一个状态，即**空状态（zero），**即 status = 0。理论部分曾说过，一个释放的线程会清空自身状态，这个不是状态的状态也是一种状态。

## CLH 队列的管理
AQS 中，和原本的 CLH 锁一样，具有 `head`，`tail`属性等:

```java
/**
 * Head of the wait queue, lazily initialized.
 */
private transient volatile Node head;

/**
 * Tail of the wait queue. After initialization, modified only via casTail.
 */
private transient volatile Node tail;
// Queuing utilities

private boolean casTail(Node c, Node v) {
    return U.compareAndSetReference(this, TAIL, c, v);
}

/** tries once to CAS a new dummy node for head */
private void tryInitializeHead() {
    Node h = new ExclusiveNode();
    if (U.compareAndSetReference(this, HEAD, null, h))
        tail = h;
}
```

其中，`head`是懒初始化的，就是只有用到的时候才会进行初始化。

### 入队
#### 从条件队列入队到 CLH 队列中
```java
/**
 * Enqueues the node unless null. (Currently used only for
 * ConditionNodes; other cases are interleaved with acquires.)
 */
final void enqueue(Node node) {
    if (node != null) {
        for (;;) {
            Node t = tail;
            node.setPrevRelaxed(t);        // avoid unnecessary fence
            if (t == null)                 // initialize
                tryInitializeHead();
            else if (casTail(t, node)) {
                t.next = node;
                if (t.status < 0)          // wake up to clean link
                    LockSupport.unpark(node.waiter);
                break;
            }
        }
    }
}
```

上述这个方法只用于 ConditionNode 从条件队列里转移到 CLH 队列中，其他情况则在 acquire 方法中入队。

上述代码中，为什么入队成功之后（第 12 行），还需要判断 `t.status < 0`？

因为`t.status < 0`说明`tail`节点被取消了，应该被清理出 CLH 队列，唤醒 node 主要目的是能够清理 node 的前一个节点 oldTail，因为 `acquire`方法逻辑中，第一步就是要确保尝试获取锁的节点的前驱有效，故而 oldTail 会很快被清除。

#### acquire 方法中入队
这将在 acquire 方法中说明

## acquire 方法详解
### 方法签名
```java
final int acquire(Node node, 
                  int arg, 
                  boolean shared,
                  boolean interruptible, 
                  boolean timed, 
                  long time) 
```

参数解释：

+ node: 代表当前尝试获取锁的线程。对于 node 是否为 null 有 2 中情况：
    - 直接调用同步器公开的的 `acquire`方法传入的 node 都为 null
    - 调用与同步器关联的 `ConditionObject`对象的 `await` 方法会调用 node 不为 null 的 acquire 方法。在调用此 `acquire`方法之前，`await`方法会确保该 node 已经加入到了 CLH 队列中

后面的参数就很直观了，不做解释。

返回值分 3 种情况：

+ 正数代表获取成功
+ 0 代表超时
+ 负数代表被中断了

### 局部变量
```java
Thread current = Thread.currentThread();
byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
boolean interrupted = false, first = false;
Node pred = null;                // predecessor of node when enqueued
```

解释：

+ `spins` 和`postSpins`：当 node 是当前队列的第一个节点时，可以自旋等待 head 节点（持有锁的那个节点）释放锁，这样的话，就避免了`park`操作所带来的性能消耗：
    - 当 `spins` 大于 0 的时候才会进行自旋，每次自旋 spins 减一
    - `postSpins` 只会增不会减：每次在当前线程 `park` 的时候增加（`spins = postSpins = (byte)((postSpins << 1) | 1)`），这意味着当线程`park`的次数越多，其自旋的次数也会越多
    - `spins`唯一的增加方式就是通过上面增加`postSpins`的语句
    - `spins`和`postSpins`的数据类型是`byte`，这意味着自旋的次数最多只能是 255
+ `interrupted`：判断当前线程是否已经被中断了
+ `first`：当前节点是否是`head`节点（如果是独占模式的话，就是持有锁的那个线程对应的节点）的下一个节点，即第一个将要获得锁的节点。如果是共享锁的话，`head`和`first`节点可能同时持有锁
+ `pred`：传入参数`node`的前驱节点

### 主要逻辑
程序的主要逻辑是一个无限循环，每一次循环执行如下操作：

1. 检查当前节点是否是`first`:
    - 如果是，则确保队列的头部稳定：即头部没有被修改，如果被修改（不稳定）则自旋等待头部稳定
    - 如果不是，则确保前驱节点有效
2. 如果当前节点是第一个节点，或尚未入队，则尝试获取锁
3. 如果当前节点尚未创建，则创建它
4. 如果当前节点尚未入队，尝试将它入队一次
5. 如果当前线程被唤醒，尝试重新获取锁（最多尝试 `postSpins`次）
6. 如果节点的状态尚未设置为`WATTING`，设置该状态并重试
7. 如果以上条件都不满足，则挂起线程并清除`WAITING`状态，检查是否被取消

其中，第`1`和第`2`步是每次循环都会被执行，其它步骤只当条件满足的时候才会被执行。

#### 一、检查当前节点是否是 first
```java
if (!first && (pred = (node == null) ? null : node.prev) != null &&
    !(first = (head == pred))) {
    if (pred.status < 0) {
        cleanQueue();           // predecessor cancelled
        continue;
    } else if (pred.prev == null) {
        Thread.onSpinWait();    // ensure serialization
        continue;
    }
}
```

这段代码不仅判断了当前节点是否是`first`，而且当不是的时候，确保了其前驱节点是有效的。

确保前驱节点有效关注的是什么有效呢？

+ `status`属性：如果前驱节点被取消了，则需要将其从队列中清除开，这就是`cleanQueue`方法完成的事情
+ `prev`属性：按道理来说，当前节点不是`head`，那么其前驱自然不是 null，所以，此时的 pred 是无效的

没有关注的是：

+ `waiter`属性：表示的是节点所代表的线程。前驱节点所代表的线程不会影响到本节点，故而没有检查
+ `next`属性：next 属性当然就是本节点，故而不用检查

#### 二、尝试获取锁
```java
if (first || pred == null) {
    boolean acquired;
    try {
        if (shared)
            acquired = (tryAcquireShared(arg) >= 0);
        else
            acquired = tryAcquire(arg);
    } catch (Throwable ex) {
        cancelAcquire(node, interrupted, false);
        throw ex;
    }
    if (acquired) {
        if (first) {
            node.prev = null;
            head = node;
            pred.next = null;
            node.waiter = null;
            if (shared)
                signalNextIfShared(node);
            if (interrupted)
                current.interrupt();
        }
        return 1;
    }
}
```

> 第一步里面提到过，如果当前节点是 first，则需要确保 head 稳定，从上述代码中可以看到，这是在获取到锁之后进行稳定的（13~21 行）
>
> 此操作保证了 head 是当前获取到锁的节点。
>
> 注意第 20 行代码，如果线程被中断了，则会主动调用 interrupt 方法设置线程的中断状态，然后返回，让子类自己决定如何处理中断。
>

能够尝试获取锁的条件（任一条件满足）是：

+ 当前节点是`first`
+ 当前节点尚未入队（pred = null）

持有锁的是`head`节点，为什么`first`节点也能获取锁？

+ 为了提高框架的可用性，以适应读写锁这种可以多个线程同时读的情况

为什么尚未入队的节点也能尝试获取锁？

+ 为了提高框架的性能，让来得早不如来得巧的线程可以尝试获得锁

节点获取到锁的条件是：

+ `tryAcquire`方法返回`true`

---

由于只有`tryAcquire`方法返回`true`才能获取锁，所以 AQS 的具体实现类可以借此来实现公平锁和读写锁：

+ 公平锁：让只有在队列中的节点才能拿到锁，让那些来得早不如来得巧的线程不能获得锁，只有`first`节点才能尝试获得锁

#### 三、如果节点没有创建，则新建节点
```java
if (node == null) {                 // allocate; retry before enqueue
    if (shared)
        node = new SharedNode();
    else
        node = new ExclusiveNode();
```

可以看到，在`acquire`方法中创建的节点只能是共享模式或独占模式下的节点，而不可能是条件节点，这也侧面说明了`acquire`方法的`node`参数只有在条件节点调用`await`方法的时候才不为 null

#### 四、尝试入队
```java
} else if (pred == null) {          // try to enqueue
    node.waiter = current;
    Node t = tail;
    node.setPrevRelaxed(t);         // avoid unnecessary fence
    if (t == null)
        tryInitializeHead();
    else if (!casTail(t, node))
        node.setPrevRelaxed(null);  // back out
    else
        t.next = node;
```

入队首先是将节点的`waiter`属性设置好，确保自己可以被其他的线程唤醒。

然后就尝试入队，如果没有入队成功，则第`8`行就回退。没有入队成功的原因是有其它的线程也正在入队，只有等待其它线程入队成功之后，再进行尝试入队。

#### 五、线程被唤醒、尝试获得锁
```java
} else if (first && spins != 0) {
    --spins;                        // reduce unfairness on rewaits
    Thread.onSpinWait();
```

为什么说执行这一段逻辑的线程是被唤醒的？

+ 因为`spins`不等于 0，因为只有被`park`之后，spins 才会增加，故而其一定是被唤醒的

#### 六 、设置唤醒位
```java
} else if (node.status == 0) {
    node.status = WAITING;          // enable signal and recheck
```

为设么要设置唤醒位呢？

+ 这是在为第七步在做准备。`park`操作挂起的线程其状态一定设置了`WAITTING`位（信号位），但这不意味着在队列中的节点的状态都设置了`WAITTING`位，这是因为加入队列是在设置信号位之前（第`4`步）。
+ **设置唤醒位一方面为线程提供了一次重新获取锁的机会，另一方面，也给**`**release**`**操作方便，只需要唤醒**`**status != 0**`**的节点就行了，对于**`**status = 0**`**的节点来说，**`**release**`**操作不会**`**unpark**`**该节点，这就避免了由于**`**unpark**`**操作提前调用，导致接下来的**`**park**`**操作无效，从而加剧锁的竞争**。

`release`操作也操作该信号位，先来看一下`release`操作：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        signalNext(head);
        return true;
    }
    return false;
}
```

如果`tryRelease`方法返回 true，则调用`signalNext`方法：

```java
/**
 * Wakes up the successor of given node, if one exists, and unsets its
 * WAITING status to avoid park race. This may fail to wake up an
 * eligible thread when one or more have been cancelled, but
 * cancelAcquire ensures liveness.
 */
private static void signalNext(Node h) {
    Node s;
    if (h != null && (s = h.next) != null && s.status != 0) {
        s.getAndUnsetStatus(WAITING);
        LockSupport.unpark(s.waiter);
    }
}
```

从这段注释中，我们知道，在唤醒的时候不设置`WAITTING`可以避免`park race`。

#### 七、挂起线程，并处理唤醒后的状态检查
```java
} else {
    long nanos;
    spins = postSpins = (byte)((postSpins << 1) | 1);
    if (!timed)
        LockSupport.park(this);
    else if ((nanos = time - System.nanoTime()) > 0L)
        LockSupport.parkNanos(this, nanos);
    else
        break;
    
    node.clearStatus();
    if ((interrupted |= Thread.interrupted()) && interruptible)
        break;
}
```

每次`park`前都要增加`spins`和`postspins`，让其被唤醒后，可以自旋尝试获得锁。

每次唤醒之后都要清除`status`属性。

第`12`行检查线程是否被中断并且是否允许被中断，如果满足时，会 break 出去调用`cancelAcquire`方法并返回该方法的返回值(调用`park`方法时，如果线程被中断了，会导致线程被唤醒)：

```java
/**
 * Cancels an ongoing attempt to acquire.
 *
 * @param node the node (may be null if cancelled before enqueuing)
 * @param interrupted true if thread interrupted
 * @param interruptible if should report interruption vs reset
 */
private int cancelAcquire(Node node, boolean interrupted,
                          boolean interruptible) {
    if (node != null) {
        node.waiter = null;
        node.status = CANCELLED;
        // 不是 head 节点
        if (node.prev != null)
            cleanQueue();
    }
    if (interrupted) {
        if (interruptible)
            return CANCELLED;
        else
            Thread.currentThread().interrupt();
    }
    return 0;
}
```

需要注意的是，Thread.interrupted 方法调用之后，会清除线程的中断状态，所以使用变量来判断是否被中断了。

### 完整代码
```java
    final int acquire(Node node, int arg, boolean shared,
                      boolean interruptible, boolean timed, long time) {
        Thread current = Thread.currentThread();
        byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
        boolean interrupted = false, first = false;
        Node pred = null;                // predecessor of node when enqueued
        
        for (;;) {
            if (!first && (pred = (node == null) ? null : node.prev) != null &&
                !(first = (head == pred))) {
                if (pred.status < 0) {
                    cleanQueue();           // predecessor cancelled
                    continue;
                } else if (pred.prev == null) {
                    Thread.onSpinWait();    // ensure serialization
                    continue;
                }
            }
            if (first || pred == null) {
                boolean acquired;
                try {
                    if (shared)
                        acquired = (tryAcquireShared(arg) >= 0);
                    else
                        acquired = tryAcquire(arg);
                } catch (Throwable ex) {
                    cancelAcquire(node, interrupted, false);
                    throw ex;
                }
                if (acquired) {
                    if (first) {
                        node.prev = null;
                        head = node;
                        pred.next = null;
                        node.waiter = null;
                        if (shared)
                            signalNextIfShared(node);
                        if (interrupted)
                            current.interrupt();
                    }
                    return 1;
                }
            }
            if (node == null) {                 // allocate; retry before enqueue
                if (shared)
                    node = new SharedNode();
                else
                    node = new ExclusiveNode();
            } else if (pred == null) {          // try to enqueue
                node.waiter = current;
                Node t = tail;
                node.setPrevRelaxed(t);         // avoid unnecessary fence
                if (t == null)
                    tryInitializeHead();
                else if (!casTail(t, node))
                    node.setPrevRelaxed(null);  // back out
                else
                    t.next = node;
            } else if (first && spins != 0) {
                --spins;                        // reduce unfairness on rewaits
                Thread.onSpinWait();
            } else if (node.status == 0) {
                node.status = WAITING;          // enable signal and recheck
            } else {
                long nanos;
                spins = postSpins = (byte)((postSpins << 1) | 1);
                if (!timed)
                    LockSupport.park(this);
                else if ((nanos = time - System.nanoTime()) > 0L)
                    LockSupport.parkNanos(this, nanos);
                else
                    break;
                node.clearStatus();
                if ((interrupted |= Thread.interrupted()) && interruptible)
                    break;
            }
        }
        return cancelAcquire(node, interrupted, interruptible);
    }

```

## cleanQueue 方法详解
此方法的作用：

+ 清理 CLH 队列中被取消的节点

此方法可能会反复地从队尾开始遍历，直到没有遇到需要清理的节点。同时会将遇到的所有被取消的节点清除，并维护队列的完整性。

在清理的过程中，如果发现清理后为队列的第一个节点的节点时，会唤醒该节点。

为什么要从队尾开始遍历呢？

+ 虽然节点有维护下一个节点的`next`指针，但是其在`acquire`方法中的赋值不是通过原子操作来完成的，而只是简单的赋值操作，如果在清理的同时还需要等待`next`属性赋值完毕，那么耗时将是不确定的：

```java
} else if (pred == null) {          // try to enqueue
    node.waiter = current;
    Node t = tail;
    node.setPrevRelaxed(t);         // avoid unnecessary fence
    if (t == null)
        tryInitializeHead();
    else if (!casTail(t, node))
        node.setPrevRelaxed(null);  // back out
    else
        t.next = node;

```

#### 完整代码
```java
private void cleanQueue() {
    for (;;) {                               // restart point
        // 从队尾开始遍历
        for (Node q = tail, s = null, p, n;;) { // (p, q, s) triples，形成一个 (p,q,s) 三元祖
            if (q == null || (p = q.prev) == null)
                return;                      // end of list
            // 这里 s.status < 0 也要 break 的目的是让 s 先被清除
            if (s == null ? tail != q : (s.prev != q || s.status < 0))
                break; // inconsistent
            
            // 节点 q 被取消了
            if (q.status < 0) {              // cancelled
                if ((s == null ? casTail(q, p) : s.casPrev(q, p)) &&
                    q.prev == p) {
                    // 这里失败是可以的是因为下面有专门处理这种情况的分支
                    p.casNext(q, s);         // OK if fails
                    // 即使上面的 CAS 操作出现问题，没有成功，signalNext 也是可以的
                    if (p.prev == null)
                        signalNext(p);
                }
                break;
            }
            
            if ((n = p.next) != q) {         // help finish
                if (n != null && q.prev == p) {
                    p.casNext(n, q);
                    if (p.prev == null)
                        signalNext(p);
                }
                break;
            }
            s = q;
            q = q.prev;
        }
    }
}
```

从上述代码中，可以看到**无锁并发**主要是由 CAS 操作来完成的，如果 CAS 操作失败了，就通过循环的方式来重试，直到更新成功。当然，CAS 失败之后，要更新其 expect 值，让下一次更新能够成功。



