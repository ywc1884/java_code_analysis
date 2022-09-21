# SynchronousQueue源码分析

## **简介**

　　本篇是在分析Executors源码时，发现JUC集合框架中的一个重要类没有分析，SynchronousQueue，该类在线程池中的作用是非常明显的，所以很有必要单独拿出来分析一番，这对于之后理解线程池有很有好处，SynchronousQueue是一种阻塞队列，其中每个插入操作必须等待另一个线程的对应移除操作 ，反之亦然。同步队列没有任何内部容量，甚至连一个队列的容量都没有。

### **SynchronousQueue数据结构**

由于SynchronousQueue的支持公平策略和非公平策略，所以底层可能两种数据结构：队列（实现公平策略）和栈（实现非公平策略），队列与栈都是通过链表来实现的。具体的数据结构如下:

![data_structure](/images/SyncQueue_structure.png)

说明：数据结构有两种类型，栈和队列；栈有一个头结点，队列有一个头结点和尾结点；栈用于实现非公平策略，队列用于实现公平策略。 其中栈的实现中入栈出栈都是操作Head节点; 而队列
中Head节点是个Dummy虚拟节点, 每次匹配出队列都是针对Head的下一个节点.

## **SynchronousQueue源码分析**

### SynchronousQueue类

类图如下:
![class diagram](/images/SyncQueue_class_diagram.png)

#### 内部类

SynchronousQueue的内部类框架图如下:

![internal_classes](/images/SyncQueue_internal_classes.png)

说明：其中比较重要的类是左侧的三个类，Transferer是TransferStack栈和TransferQueue队列的公共类，定义了转移数据的公共操作，由TransferStack和TransferQueue具体实现，WaitQueue、LifoWaitQueue、FifoWaitQueue表示为了兼容JDK1.5版本中的SynchronousQueue的序列化策略所遗留的，这里不做具体的讲解。下面着重看左侧的三个类。

##### **结构细节**

SynchronousQueue 底层结构和其它队列完全不同，有着独特的两种数据结构：队列和堆栈，我们一起来看下数据结构：

```
// 堆栈和队列共同的接口
// 负责执行 put or take
abstract static class Transferer<E> {
    // e 为空的，会直接返回特殊值，不为空会传递给消费者
    // timed 为 true，说明会有超时时间
    abstract E transfer(E e, boolean timed, long nanos);
}

// 堆栈 后入先出 非公平
// Scherer-Scott 算法
static final class TransferStack<E> extends Transferer<E> {
}

// 队列 先入先出 公平
static final class TransferQueue<E> extends Transferer<E> {
}

private transient volatile Transferer<E> transferer;

// 无参构造器默认为非公平的
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

从源码中我们可以得到几点：

堆栈和队列都有一个共同的接口，叫做 Transferer，该接口有个方法：transfer，该方法很神奇，会承担 take 和 put 的双重功能；

在我们初始化的时候，是可以选择是使用堆栈还是队列的，如果你不选择，默认的就是堆栈，类注释中也说明了这一点，堆栈的效率比队列更高。
接下来我们来看下堆栈和队列的具体实现。

#### **非公平的堆栈**

![TranferStack](/images/SyncQueue_TransferStack.png)

从上图中我们可以看到，我们有一个大的堆栈池，池的开口叫做堆栈头，put 的时候，就往堆栈池中放数据。take 的时候，就从堆栈池中拿数据，两者操作都是在堆栈头上操作数据，从图中可以看到，越靠近堆栈头，数据越新，所以每次 take 的时候，都会拿到堆栈头的最新数据，这就是我们说的后入先出，也就是非公平的。

图中 SNode 就是源码中栈元素的表示，我们看下源码：

![SNode](/images/SyncQueue_Snode.png)

* volatile SNode next

   栈的下一个，就是被当前栈压在下面的栈元素

* volatile SNode match

    节点匹配，用来判断阻塞栈元素能被唤醒的时机
    比如我们先执行 take，此时队列中没有数据，take 被阻塞了，栈元素为 SNode1
    当有 put 操作时，会把当前 put 的栈元素赋值给 SNode1 的 match 属性，并唤醒 take 操作
    当 take 被唤醒，发现 SNode1 的 match 属性有值时，就能拿到 put 进来的数据，从而返回

* volatile Thread waiter

    栈元素的阻塞是通过线程阻塞来实现的，waiter 为阻塞的线程

* Object item

    未投递的消息，或者未消费的消息, 方法传入参数时生产者传入需要传递给消费者的消息，如果是消费者则为null

##### **transfer方法**

transfer 方法思路比较复杂，因为 take 和 put 两个方法都揉在了一起A

```
@SuppressWarnings("unchecked")
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    
    // e 为空: take 方法，非空: put 方法
    int mode = (e == null) ? REQUEST : DATA;
    
    // 自旋
    for (;;) {
        // 头节点情况分类
        // 1：为空，说明队列中还没有数据
        // 2：非空，并且是 take 类型的，说明头节点线程正等着拿数据
        // 3：非空，并且是 put 类型的，说明头节点线程正等着放数据
        SNode h = head;
        
        // 栈头为空，说明队列中还没有数据。
        // 栈头非空且栈头的类型和本次操作一致
        //	比如都是 put，那么就把本次 put 操作放到该栈头的前面即可，让本次 put 能够先执行
        if (h == null || h.mode == mode) {  // empty or same-mode
            // 设置了超时时间，并且 e 进栈或者出栈要超时了，
            // 就会丢弃本次操作，返回 null 值。
            // 如果栈头此时被取消了，丢弃栈头，取下一个节点继续消费
            if (timed && nanos <= 0) {
                // 无法等待, 对于Executors.newCachedThreadPool, 如果是生产者调用offer(E), 
                // 则传入timed=true, nanos=0, 如果栈为空或者栈内部都是生产者，则会直接返回null
                // 导致workQueue.offer失败从而创建新的Worker线程执行任务
                // 栈头操作被取消
                if (h != null && h.isCancelled())
                    // 丢弃栈头，把栈头的后一个元素作为栈头
                    casHead(h, h.next);     // 将取消的节点弹栈
                // 栈头为空，直接返回 null
                else
                    return null;
            // 没有超时，直接把 e 作为新的栈头
            } else if (casHead(h, s = snode(s, e, h, mode))) {
                // e 等待出栈，一种是空队列 take，一种是 put
                SNode m = awaitFulfill(s, timed, nanos);
                if (m == s) {               
                    // wait was cancelled， 
                    // 如果awaitFulfill返回值和当前SNode相等表示当前Node已取消, 取消时会把match字段置为当前Node并返回该match字段
                    clean(s);
                    return null;
                }
                // 本来s是栈头的，现在s不是栈头了，s后面又来了一个数，把新的数据作为栈头
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        // 栈头正在等待其他线程 put 或 take
        // 比如栈头正在阻塞，并且是 put 类型，而此次操作正好是 take 类型，走此处
        } else if (!isFulfilling(h.mode)) { // 当前头节点没有正在匹配
            // 栈头已经被取消，把下一个元素作为栈头
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);        // pop and retry
            // snode 方法第三个参数 h 代表栈头，赋值给 s 的 next 属性
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) { //将s节点压入栈顶并添加匹配FULFILLING的模式尝试跟head节点匹配
                for (;;) { // loop until matched or waiters disappear
                    // m就是栈头，通过上面snode方法刚刚赋值
                    SNode m = s.next;       // m is s's match
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                     // tryMatch 非常重要的方法，两个作用：
                     // 1 唤醒被阻塞的栈头m，2 把当前节点s赋值给m的match属性
                     // 这样栈头m被唤醒时，就能从m.match中得到本次操作s
                     // 其中s.item记录着本次的操作节点，也就是记录本次操作的数据
                    if (m.tryMatch(s)) {
                        casHead(s, mn);     // pop both s and m, 将s和之前的head节点出栈
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        s.casNext(m, mn);   // help unlink
                }
            }
        } else {                            // help a fulfiller, head节点已经在匹配了，帮忙处理匹配并出栈匹配成功Node
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}
```

总结一下操作思路：

* 判断是put方法还是take方法
* 判断栈头数据是否为空，如果为空或者栈头的操作和本次操作一致，是的话走 3，否则走 5
* 判断操作有无设置超时时间，如果设置了超时时间并且已经超时，返回 null，否则走 4
* 如果栈头为空，把当前操作设置成栈头，或者栈头不为空，但栈头的操作和本次操作相同，也把当前操作设置成栈头，并看看其它线程能否满足自己，不能满足则阻塞自己。比如当前操作是 take，但队列中没有数据，则阻塞自己
* 如果栈头已经是阻塞住的，需要别人唤醒的，判断当前操作能否唤醒栈头，可以唤醒走 6，否则走 4
* 把自己当作一个节点，赋值到栈头的 match 属性上，并唤醒栈头节点
* 栈头被唤醒后，拿到 match 属性，就是把自己唤醒的节点的信息，返回。

在整个过程中，有一个节点阻塞的方法，源码如下：

当一个节点/线程将要阻塞时，它会设置其waiter字段，然后在真正park之前至少再检查一次状态，从而涵盖了竞争与实现者的关系，并注意到waiter非空，因此应将其唤醒。

当由出现在调用点位于堆栈顶部的节点调用时，对停放的调用之前会进行旋转，以避免在生产者和消费者及时到达时阻塞。 这可能只足以在多处理器上发生。

从主循环返回的检查顺序反映了这样一个事实，即优先级: 中断 > 正常的返回 > 超时。 （因此，在超时时，在放弃之前要进行最后一次匹配检查。）除了来自非定时SynchronousQueue的调用。{poll / offer}不会检查中断，根本不等待，因此陷入了转移方法中 而不是调用awaitFulfill。

```
/**
 * 旋转/阻止，直到节点s通过执行操作匹配。
 * @param s 等待的节点
 * @param timed true if timed wait
 * @param nanos 超时时间
 * @return 匹配的节点, 或者是 s 如果被取消
 */
SNode awaitFulfill(SNode s, boolean timed, long nanos) {
    // deadline 死亡时间，如果设置了超时时间的话，死亡时间等于当前时间 + 超时时间，否则就是 0
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    // 自旋的次数，如果设置了超时时间，会自旋 32 次，否则自旋 512 次。
    // 比如本次操作是 take 操作，自旋次数后，仍无其他线程 put 数据
    // 就会阻塞，有超时时间的，会阻塞固定的时间，否则一致阻塞下去
    int spins = (shouldSpin(s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        // 当前线程有无被打断，如果过了超时时间，当前线程就会被打断
        if (w.isInterrupted())
            s.tryCancel();

        SNode m = s.match;
        if (m != null)
        //这里返回退出有2种可能
        //1.超时或者中断取消设置m == s
        //2.被匹配成功且unpark, s.match会被设置为匹配的Node
            return m;
        if (timed) {
            nanos = deadline - System.nanoTime();
            // 超时了，取消当前线程的等待操作
            if (nanos <= 0L) {
                s.tryCancel();
                continue;
            }
        }
        // 自选次数减1
        if (spins > 0)
            spins = shouldSpin(s) ? (spins-1) : 0;
        // 把当前线程设置成 waiter，主要是通过线程来完成阻塞和唤醒
        else if (s.waiter == null)
            s.waiter = w; // establish waiter so can park next iter
        else if (!timed)
            // 通过 park 进行阻塞，这个我们在锁章节中会说明
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```

可以发现其阻塞策略，并不是一上来就阻塞住，而是在自旋一定次数后，仍然没有其它线程来满足自己的要求时，才会真正的阻塞。 其中涉及到一个方法tryCancel的源码如下，

```
/**
    * Tries to cancel a wait by matching node to itself.
    */
void tryCancel() {
    //设置Node的match值为Node自己, 表示已取消
    SMATCH.compareAndSet(this, null, this);
}
```

队列的实现策略通常分为公平模式和非公平模式，本文我们重点介绍公平模式。

#### **公平队列**

**元素组成**

* volatile QNode head

    队列头, 虚拟Dummy头结点，不参与匹配

* volatile QNode tail

    队列尾

* volatile QNode next

    当前元素的下一个元素

* volatile Object item // CAS’ed to or from null

    当前元素的值，如果当前元素被阻塞住了，等其他线程来唤醒自己时，其他线程会把自己 set 到 item 里面

* volatile Thread waiter // to control park/unpark

    可以阻塞住的当前线程

* final boolean isData

    true 是 put，false 是 take


公平队列主要使用的是 TransferQueue 内部类的 transfer 方法，看源码：

```
E transfer(E e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    // true ： put false ： get
    boolean isData = (e != null);

    for (;;) {
        // 队列头和尾的临时变量,队列是空的时候，t=h
        QNode t = tail;
        QNode h = head;
        // tail 和 head 没有初始化时，无限循环
        // 虽然这种 continue 非常耗cpu，但感觉不会碰到这种情况
        // 因为 tail 和 head 在 TransferQueue 初始化的时候，就已经被赋值空节点了
        if (t == null || h == null)
            continue;
        // 首尾节点相同，说明是空队列
        // 或者尾节点的操作和当前节点操作一致
        if (h == t || t.isData == isData) {
            QNode tn = t.next;
            // 当 t 不是 tail 时，说明 tail 已经被修改过了
            // 因为 tail 没有被修改的情况下，t 和 tail 必然相等
            // 因为前面刚刚执行赋值操作： t = tail
            if (t != tail)
                continue;
            // 队尾后面的值还不为空，t 还不是队尾，直接把 tn 赋值给 t，这是一步加强校验。
            if (tn != null) {
                advanceTail(t, tn);
                continue;
            }
            //超时直接返回 null
            if (timed && nanos <= 0)        // can't wait
                return null;
            //构造node节点
            if (s == null)
                s = new QNode(e, isData);
            //如果把 e 放到队尾失败，继续递归放进去
            if (!t.casNext(null, s))        // failed to link in
                continue;

            advanceTail(t, s);              // swing tail and wait
            // 阻塞住自己
            Object x = awaitFulfill(s, e, timed, nanos);
            if (x == s) {                // wait was cancelled
                clean(t, s);
                return null;
            }

            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;
        // 队列不为空，并且当前操作和队尾不一致
        // 也就是说当前操作是队尾是对应的操作
        // 比如说队尾是因为 take 被阻塞的，那么当前操作必然是put
        } else {                            // complementary-mode
            // 如果是第一次执行，此处的 m 代表就是 tail
            // 也就是这行代码体现出队列的公平，每次操作时，从头开始按照顺序进行操作
            // 每次匹配都是从head头结点的下一个节点开始
            QNode m = h.next;               // node to fulfill
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;
            if (isData == (x != null) ||    // m already fulfilled, mode类型相同无法匹配
                x == m ||                   // m cancelled, m被取消, 队列内的QNode取消时会把item项设置为Node自身
                // m 代表栈头
                // 这里把当前的操作值赋值给阻塞住的 m 的 item 属性
                // 这样 m 被释放时，就可得到此次操作的值
                !m.casItem(x, e)) {         // lost CAS, 将m的item项设置为e，这样下面unpark m的线程时m节点继续执行awaitFulfill方法时由于item值和之前的不同可以直接返回, 匹配成功且离开队列
                advanceHead(h, m);          // dequeue and retry
                continue;
            }
            // 当前操作放到队头
            advanceHead(h, m);              // successfully fulfilled
            // 释放队头阻塞节点
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}
```

其中的awaitFulfill方法和非公平的栈类似这里就不分析了，但是该方法中的tryCancel实现变了，源码如下:

```
/**
    * Tries to cancel by CAS'ing ref to this as item.
    */
void tryCancel(Object cmp) {
    //cancel时将item项设置为节点自身
    QITEM.compareAndSet(this, cmp, this);
}
```

最后需要注意的一点是对于Executors.newCachedThreadPool使用到的SynchronousQueue, 往Queue内放入task时是作为生产者调用offer(E), 而Worker线程在调用getTask方法
获取task时则调用的是poll(long timeout, TimeUnit unit), SynchronousQueue内的方法实现如下:

```
    /**
     * Inserts the specified element into this queue, if another thread is
     * waiting to receive it.
     *
     * @param e the element to add
     * @return {@code true} if the element was added to this queue, else
     *         {@code false}
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        //这里调用transfer方法传入的timed = true, nanos = 0, 如果stack/queue是空的或者也是同类型的Node, 则根据transfer的逻辑会直接返回null
        //从而导致workQueue.offer执行返回false, 进而会创建新线程直接执行task
        //同时导致stack/queue内能够排队等待的Node只能是作为消费者的线程(poll), 因此每次匹配成功后双方返回的内容都将是生产者提供的task
        //这样的结果就是往queue内放置任务的workQueue.offer会成功，同时线程池内匹配成功的线程会拿到这个task开始执行
        return transferer.transfer(e, true, 0) != null;
    }

    /**
     * Retrieves and removes the head of this queue, waiting
     * if necessary up to the specified wait time, for another thread
     * to insert it.
     *
     * @return the head of this queue, or {@code null} if the
     *         specified waiting time elapses before an element is present
     * @throws InterruptedException {@inheritDoc}
     */
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E e = transferer.transfer(null, true, unit.toNanos(timeout));
        if (e != null || !Thread.interrupted())
            return e;
        throw new InterruptedException();
    }
```

### **总结**

如下是TransferStack和TransferQueue的逻辑总结图:

TransferQueue:

![queue logic](/images/TransferQueue_summary.jpeg)

TransferStack:

![stack logic](/images/TransferStack_summary.jpeg)