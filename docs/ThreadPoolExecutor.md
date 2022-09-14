# 线程池ThreadPoolExecutor源码分析

一. 概述
线程池的作用不用太多说了，线程池会按照一定的规则，创建和维护一定数量的线程，这些线程可以被循环利用来处理用户提交的任务。对比不使用线程池的方式，节省了频繁的创建和销毁线程带来的性能开销。

二. 几个比较重要的概念
1. 工作线程（worker）
指的是当前线程池用于处理任务的worker对象，每个worker对象内部都持有一个thread对象实例。

2. 任务
调用方要执行的业务逻辑，一般应该是个Callable或者Runnable的实现。

3. 任务队列
线程池内部维护了一个队列用来存储待处理的任务，每个工作线程都可以从该队列获取任务进行处理。

4. 核心工作线程（worker）数
线程池内部需要维持的一个最小的工作线程数量。工作线程数量不足这个数量的时候，新来的任务都会交给一个新建的工作线程，任务不会被放入队列。工作线程数达到这个数量的时候，新来的任务都会直接放入队列，所有的工作线程会主动去轮询领任务。

5. 最大工作线程（worker）数
线程池内部会限制一个最大的工作线程数量。当工作线程数大于核心线程数值，任务队列也满的情况下，就需要再新建工作线程，但是所有的工作线程数量不能超过最大的工作线程数的设置。

6. 空闲时间
线程池中工作线程数超过核心线程数配置的时候，对于那些空闲超过一定时间的线程会进行回收。

7. 线程池状态
线程池一共有五个状态
```
RUNNING：正常的运行状态，此时可以正常接收和处理任务。
SHUTDOWN：关闭状态，只能通过调用shutdown()方法达到此状态。此时可以处理任务，但不再接受新任务，并且中断空闲的工作线程。
STOP：停止状态，只能通过调用shutdownNow()方法达到此状态。此时清除队列中的未处理的任务，中断所有的（空闲的+正在执行任务的）工作线程。（不建议直接调用shutdownNow，会导致业务逻辑异常终止，会带来很多不可预知的问题）
TIDYING：整理中状态，从SHUTDOWN和STOP自动流转到此状态。此时队列中任务为空，工作线程列表为空。
TERMINATED：终止状态，从TIDYING状态自动流转到此状态。此时队列中任务为空，工作线程列表为空，并且已经执行完terminated回调函数。
```

三. 有点绕的设计

在上一个小部分里提到了工作线程数和线程池状态。由于线程池本省的目的就是为了并行处理任务来提升效率。其对外暴露的方法也自然存在着被并发调用的可能性。而且内部的很多处理逻辑都和这两个变量息息相关。所以：

无论是状态改变、还是数量改变都应该保证原子性。
有时状态改变和数量改变要保证在一个原子范围内。
所以为了简化这两个变量的并发控制，ThreadPoolExecutor在实现的时候，把这两个值融合到了一个AtomicInteger类型的变量中。AtomicInteger能够保证这个值的变更是原子的。同时他内部存储的具体业务值是一个整型数值。
整型一共32个二进制位。

前3位用来存储线程池的状态值
因为状态就5种，3个二进制位刚好足够涵盖5种数值可能（因为2位可表示4种可能，3位可表示8种可能）。
后29位用来存储工作线程数，根据用户的不同的选择和线程池配置，有些创建出来的线程池是可能会产生很多的工作线程的（实际工程中都会严格限制工作线程数，毕竟一般单机服务器的核数都不会达到几十上百个的）
通过以上的一个概述，我们再来看源码中的几个变量的定义就简单明了多了。

```
// Integer.SIZE = 32 ,所以这个 COUNT_BITS = 29，意思是存储工作线程数的二进制为占29位
private static final int COUNT_BITS = Integer.SIZE - 3;

/*
 CAPACITY 实际上就是2的29次方减1，具体的实际数值是多少不重要，重点是关注二进制表示：
    1左移29位实际上就是：0010 0000 0000 0000 0000 0000 0000 0000
    再减个1后得到的就是：0001 1111 1111 1111 1111 1111 1111 1111
 所以这个CAPACITY的有效位就是29个1
*/ 
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

/* --- 状态值的定义 开始 ---*/

// runState is stored in the high-order bits
// 状态值存储在高（3）位
/*
-1的二进制表示：1111 1111 1111 1111 1111 1111 1111 1111 （补码标识法）
-1左移29位之后：111 00000 0000 0000 0000 0000 0000 0000 （左侧切断，右侧补0）
*/
private static final int RUNNING    = -1 << COUNT_BITS;

/*
0的二进制表示：0000 0000 0000 0000 0000 0000 0000 0000
0左移29位之后：000 00000 0000 0000 0000 0000 0000 0000 （还是0）
*/
private static final int SHUTDOWN   =  0 << COUNT_BITS;

/*
1的二进制表示：0000 0000 0000 0000 0000 0000 0000 0001
0左移29位之后：001 00000 0000 0000 0000 0000 0000 0000 
*/
private static final int STOP       =  1 << COUNT_BITS;

/*
2的二进制表示：0000 0000 0000 0000 0000 0000 0000 0010
0左移29位之后：010 00000 0000 0000 0000 0000 0000 0000 
*/
private static final int TIDYING    =  2 << COUNT_BITS;

/*
3的二进制表示：0000 0000 0000 0000 0000 0000 0000 0011
0左移29位之后：011 00000 0000 0000 0000 0000 0000 0000 
*/
private static final int TERMINATED =  3 << COUNT_BITS;

/*
  针对以上5个变量的总结，右侧的29位都是0，
  RUNNING的二进制以1打头，转换为整型为负值，所以这几个状态的整型数值从上到下依次增大。
*/

/* --- 状态值的定义 结束 ---*/



/*
 将状态和线程数整合到一个数值的方法（利用二级制的或运算）
 rs：一定是上面5个状态中的一个
 wc：线程数是个不固定的正数值

 因为rs的右侧29为都是0，而wc在逻辑上限制了其最大值不能超过CAPACITY，所以wc的前3位一定是0，所以整合之后的数值实际上是：rs的前三位 + wc的后29位
*/
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 从整合后数值c中拆分出来状态值
 状态值 = c的前3位 + 29个0
 c 和 CAPACITY取反 做与运算

 CAPACITY：0001 1111 1111 1111 1111 1111 1111 1111
 取反之后：1110 0000 0000 0000 0000 0000 0000 0000
 因为前3位都是1，所以无论c的前3位是什么，与运算后都会保留c的前3位不变
 因为后29位都是0，所以无论c的后面是什么，与运算后都会变为29个0
 这样就还原出状态值。
*/
// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
/*
 从整合后数值c中拆分出来工作线程数
 工作线程数 = 3个0 + c的后29位
 c 和 CAPACITY 做与运算

 CAPACITY：0001 1111 1111 1111 1111 1111 1111 1111
 因为前3位都是0，所以无论c的前3位是什么，与运算后都会变为0
 因为后29位都是1，所以无论c的后面是什么，与运算后都保持c的后29位不变
 这样就还原出了工作线程数的值。
*/
private static int workerCountOf(int c)  { return c & CAPACITY; }

// 把整合后的值包装到一个原子变量中，下文称控制标识
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

## 核心源码分析

* **execute(Runnable command)**

执行逻辑图如下:

![execute](/images/TPE_execute.png)

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
        * Proceed in 3 steps:
        *
        * 1. If fewer than corePoolSize threads are running, try to
        * start a new thread with the given command as its first
        * task.  The call to addWorker atomically checks runState and
        * workerCount, and so prevents false alarms that would add
        * threads when it shouldn't, by returning false.
        *【这一部分注释对应第一个if块的内容】
        * 如果运行的线程数少于核心数量，尝试开启一个新的线程，并将提交的任务
        * 赋给这个新的线程执行，调用addWorker的时候原子化的校验运行状态和工作线程数量，
        * 如果不允许创建工作线程的时候会返回个false。
        * 
        * 2. If a task can be successfully queued, then we still need
        * to double-check whether we should have added a thread
        * (because existing ones died since last checking) or that
        * the pool shut down since entry into this method. So we
        * recheck state and if necessary roll back the enqueuing if
        * stopped, or start a new thread if there are none.
        * 【这一部分注释对应第二个if块的内容】
        * 如果一个任务能够被成功的加入队列，仍然需要再次校验是否还需要添加
        * 一个工作线程（因为在这段间隙中可能原有的工作线程有消亡的）或者任务添加
        * 进来后线程池被关闭。所以需要再次校验一下状态，以便于线程池被关闭的时候
        * 回滚入队操作；或者当没有可用工作线程时再创建一个。
        * 
        * 3. If we cannot queue task, then we try to add a new
        * thread.  If it fails, we know we are shut down or saturated
        * and so reject the task.
        * 【这一部分注释对应 else if块 的内容】
        * 如果任务不能入队，说明任务队列已满（接收不了新任务了），那么需要尝试创建一个新的工作线程。
        * 如果创建线程失败，我们可以推断出线程池已经关闭（不接收新任务了）。
        */
    int c = ctl.get(); // 获取控制标识
    /*
     从控制标识数值中拆分出来（二进制的后29位）工作线程数
     如果工作线程数小于设置的核心线程数，默认设置下（allowCoreThreadTimeOut为false），即使工作线程都闲着、没任务处理，也会继续创建新的工作线程，以达到维持最小核心工作线程数的目的。
     addWorker方法的作用就是创建工作线程，并把任务交给这个新建的工作线程去执行（command不会被放入任务队列）
     addWorker方法下面有单独的解析
    */ 
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) // 如果新的工作线程创建成功，则直接返回
            return;
        /*
         如果走到这一步 ，说明工作线程没有创建成功，
         可能由于并发请求情况下
            其他的请求已经新建了工作线程，本请求再去创建的时候，已经超过核心线程数阈值了，所以创建失败  
            也可能由于其他请求触发了线程池关闭动作，导致不可以再创建新的工作线程
         这个时候再重新获取一下控制标识，便于下面再检查一下线程池的状态和工作线程数
        */
        c = ctl.get();
    }

    // 走到这一步，说明工作线程没有创建成功，任务也没有提交成功 
    /*
     如果线程池处于运行状态，工作线程没有新建成功，那么说明是已经达到核心线程数了，所以直接把任务提交给工作队列，排队等着被执行就可以了
    */
    if (isRunning(c) && workQueue.offer(command)) {
        // 任务被成功添加到队列后，再获取一下控制标识
        int recheck = ctl.get();
        // 再check一下线程池状态，如果不是运行状态了，那么调用remove方法，从队列里删除掉这个任务
        if (! isRunning(recheck) && remove(command))
            reject(command); // 如果成功从任务队列里删除了任务，那么还要调用reject方法来回调一个拒绝任务的处理策略
        /*
         下面这个 else if 如果能够被执行到的话，线程池状态是正常的执行状态 或者 状态不是running但是remove失败
         无论是哪种可能，当前，待处理的任务还存在于队列中。
         那么检查一下工作线程数是否为0，如果为0，说明没人处理任务，那么就需要创建一个工作线程
        */    
        else if (workerCountOf(recheck) == 0) 
            /*
             第一个参数null，意味着没有传递任务，只是新建工作线程而已
             第二个参数false，意味着要创建的不是核心工作线程，那么校验的时候就校验工作线程总数是否超过设置的最大工作线程数
            */
            addWorker(null, false);
    }
    /*
     走到这个逻辑分支的话，说明线程池是关闭（非运行）状态  或者 任务队列满了 
     如果是任务队列满了 ，只需要创建一个工作线程，把任务交给这个线程去执行就可以了（任务不入队列）
     但是如果还是返回失败，就说明可能是：
        1.线程池的状态是关闭（非运行）状态了，不接受新任务了
        2.工作线程数已经达到了最大值，不能再新建了。
        3.工作线程创建或者启动失败。（可能性不大）
     只要任务提交失败，那么就需要调用reject方法来回调一个拒绝任务的处理策略
    */ 
    else if (!addWorker(command, false))
        reject(command);
}
```

* **addWorker**

```
/**
* Checks if a new worker can be added with respect to current
* pool state and the given bound (either core or maximum). If so,
* the worker count is adjusted accordingly, and, if possible, a
* new worker is created and started, running firstTask as its
* first task. This method returns false if the pool is stopped or
* eligible to shut down. It also returns false if the thread
* factory fails to create a thread when asked.  If the thread
* creation fails, either due to the thread factory returning
* null, or due to an exception (typically OutOfMemoryError in
* Thread.start()), we roll back cleanly.
* 
* 在当前的线程池状态和给定的边界控制逻辑的情况下，如果允许新建一个工作线程创建，
* 那么工作线程数会随之调整，随之创建一个新的工作线程并且启动它，并且把需要运行的任务交给它去运行。
* 如果线程池已经被停止了或者被合法的关闭，那么这个方法会直接返回false，也就不会创建新的线程。
* 如果创建新的工作线程是失败了（可能因为线程工厂返回了null或者抛出了异常），那么这个方法同样也会返回false。
* 
* @param firstTask the task the new thread should run first (or
* null if none). Workers are created with an initial first task
* (in method execute()) to bypass queuing when there are fewer
* than corePoolSize threads (in which case we always start one),
* or when the queue is full (in which case we must bypass queue).
* Initially idle threads are usually created via
* prestartCoreThread or to replace other dying workers.
*
* -- firstTask 指的是要交给新建的工作线程运行的第一个任务（并不一定是线程池的第一个任务）
* 当工作线程数小于核心线程数的时候或者当工作队列满的时候，会创建一个新的工作线程，并运行该任务
* 
* @param core if true use corePoolSize as bound, else
* maximumPoolSize. (A boolean indicator is used here rather than a
* value to ensure reads of fresh values after checking other pool
* state).
*
* core如果为true，则工作线程数的校验边界就是 设置的核心线程数，否则就是设置的最大线程数。
* 
* 
* @return true if successful
*/
private boolean addWorker(Runnable firstTask, boolean core) {
    retry: // goto作用的标号，下面一定要接一个循环
    for (;;) { // 这是个死循环，只能依赖内部的逻辑跳出，这种用法在concurrent的源码里很常见，主要都是为了循环检测状态
        int c = ctl.get(); // 获取控制标识
        int rs = runStateOf(c); // 获取当前线程池的状态

        /*
         如果 rs >= SHUTDOWN说明，线程池状态可能为SHUTDOWN、STOP、TIDYING、TERMINATED，
         无论哪个状态，线程池都是拒绝再接收新任务的。在这样的前提下：
            rs == SHUTDOWN ： 说明正在关闭 ，还没有到终止状态
            firstTask == null：说明并没有提交任务，只是为了增加工作线程
            ! workQueue.isEmpty()：队列不为空说明还有任务未消费，可以通过增加工作线程协助消费

            同时满足以上三个条件，也是可以继续往下执行新增工作线程的逻辑的，但是如果没有同时满足以上三个条件则 直接返回false
        */
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        // 走到这一步起码说明，线程池还是可以继续处理任务的
        for (;;) { // 又一个循环检测
            int wc = workerCountOf(c); // 获取worker（工作线程）数目
            /*
             如果 工作线程数 已经大于CAPACITY（2的29次方减1） ，这个值很大了，很少有计算机能够支持这么多的线程数，如果大于这个值就直接返回false
             如果core为true，说明调用方的目的是想创建核心工作线程，此时就要检测当前的工作线程数是否小于设置的核心线程数，如果大于这个值就直接返回false
             如果core为false，说明调用方的目的就是想单纯的新增工作线程，此时就要检测当前工作线程数是否小于设置的最大线程数，如果大于这个值就直接返回false
            */
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;

            // 走到这一步，说明可以创建工作线程了，那么先把工作线程数递增（加1）  
            if (compareAndIncrementWorkerCount(c))
                break retry; // 如果工作线程数递增成功，则通过break retry 可以跳出最外层的for循环

            // 能够走到这一步，说明工作线程数递增失败，可能是由于其他线程的并发调用更改了工作线程数 或者 线程池状态发生了变更    
            c = ctl.get();  // Re-read ctl  重新获取一下控制标识 （在以上逻辑执行过程中，其他并发调用，可能会引起线程池状态变更）
            // 重新获取线程池状态 ，如果和之前获取的不一致，那么需要跳转到retry标号，重新执行一次外层循环的逻辑，这样就可以重新获取一次线程池状态
            if (runStateOf(c) != rs)  
                continue retry;

            // 如果不是因为线程池状态问题导致的工作线程数变更失败，那么就继续执行内部循环逻辑（自旋重试）  
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false; // 工作线程是否启动
    boolean workerAdded = false; // 工作线程是否添加
    Worker w = null; // 声明一个工作线程对象
    try {
        w = new Worker(firstTask); // 创建一个工作线程对象
        final Thread t = w.thread; // 获取工作线程对象内部用于执行任务的thread对象
        /*
         t什么时候为空？
         在执行Worker的构造函数时，就会实例化其内部的thread属性，实例化的方式，就是调用线程工厂的newThread方法。
         而线程工厂是个接口，是可以用户自定义实现类的（也有默认的实现），用户实现的newThread方法是有可能由于编码失误返回null的。
        */ 
        if (t != null) { 
            final ReentrantLock mainLock = this.mainLock; // 获取当前线程池的显示锁
            mainLock.lock(); // 上锁
            try {
                // Recheck while holding lock. 
                // Back out on ThreadFactory failure or if 
                // shut down before lock acquired.
                // 在持有锁的情况下重新check线程池状态，防止在创建worker阶段或者获取锁之前的逻辑执行时，状态发生变化。
                int rs = runStateOf(ctl.get());
                /*
                 rs < SHUTDOWN：说明是RUNNING状态，那么线程池是可以正常接收新提交的任务的
                 rs == SHUTDOWN && firstTask == null ： 说明处于关闭状态，但是没有提交任务，这种情况线程池还是可以正常处理队列中现存的任务的。
                */
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 检查一下worker对象中的线程对象是否已经执行了start方法
                    // 如果已经执行了，说明状态不对，抛出线程状态异常。   
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w); // 将worker对象w添加到工作线程集合
                    int s = workers.size(); // 获取工作线程集合大小
                    // 这个largestPoolSize是统计线程的生命周期内曾经达到过的工作线程数的最大值
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true; // 工作线程是否添加的标识置为true
                }
            } finally {
                mainLock.unlock(); // 释放锁
            }
            if (workerAdded) {  // 如果成功的添加了工作线程
                /*
                 启动工作线程
                 这个方法最终会执行Worker对象的run方法，内部又会调用runWorker方法。达到的效果就是，先处理firstTask，处理完之后再去队列中获取其他任务。
                */ 
                t.start(); 
                workerStarted = true;  // 工作线程是否启动的标识置为true
            }
        }
    } finally { 
        /*
         在finally里块中检测工作线程是否启动标识
         如果workerStarted为false （工作线程创建失败 或者 工作线程启动失败）
         需要调用addWorkerFailed方法进行一些回滚操作
            工作线程列表移除掉启动失败的工作线程
            工作线程计数递减
        */ 
        if (! workerStarted) 
            addWorkerFailed(w);
    }
    // 最后返回工作线程是否启动的标识
    return workerStarted; 
}
```

* **Worker.run线程运行逻辑**

```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
           //死循环获取任务，然后执行任务。这里getTask()方法会有阻塞情况的，我们这里知道一下就行，下面马上讲。
            while (task != null || (task = getTask()) != null) {
               //获取w锁。前面说过了，Worker对象继承AbstractQueuedSynchronizer,所以本身就内置了一把锁
                w.lock();
                // 判断同一个时刻当前线程和线程池的状态是否合法，不合法结束呗
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                   //任务执行前的处理逻辑
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                    //任务执行后的处理逻辑
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    //当前Worker完成的任务数量
                    w.completedTasks++;
                    //释放w锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //处理Worker退出的逻辑
            processWorkerExit(w, completedAbruptly);
        }
    }
```

整个方法的逻辑其实也不算复杂，就是当前Worker不断死循环获取队列里面是否有任务。有，就加锁然后执行任务。无，就阻塞等待获取任务。那什么情况下才会跳出整个死循环，执行processWorkerExit呢？这里就需要看下getTask() 方法逻辑了.

```
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 判断线程池状态和任务队列的情况，不满足条件直接返回 null，结束。
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);
            
            // 超时时间的标识，[是否设置了核心线程数的超时时间 或者 当前线程数量是否大于核心线程数 ]，
    //因为我们知道线程池运行的线程数量如果大于核心线程数，多出来的那部分线程是需要被回收的。
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
            // 如果timed为false，则一直阻塞等待，直到获取到元素，然后返回
            // 如果timed为true，则一直阻塞等待keepAliveTime超时后返回，
            //到这里其实就知道如何结束runWorker方法的那个死循环了，也就意味着Worker它的线程生命周期结束了。
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

* **processWorkerExit**

```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        //获取mainLock锁
        mainLock.lock();
        try {
        //添加任务数量，然后移除worker
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
        // 释放mainLock锁
            mainLock.unlock();
        }
        //尝试将线程池状态设置为 terminate
        tryTerminate();
        
        //主要判断当前线程池的线程数是否小于corePoolSize，如果小于继续添加Worker对象
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

这个方法主要就是移除Worker对象，然后尝试将线程池的状态更改为terminate。这里需要讲一下tryTerminate方法逻辑，因为它和线程池awaitTermination()方法有一定的关联，来看看它的代码。

```
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            //判断线程池状态，还在运行或者已经是 terminate的状态直接结束了
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
             // 就是中断空闲的Worker，后面讲shutDown方法的时候聊
            if (workerCountOf(c) != 0) { 
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            

            final ReentrantLock mainLock = this.mainLock;
            //获取mainLock锁
            mainLock.lock();
            try {
            //线程池设置成TIDYING状态
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                    //钩子方法，线程池终止时执行的逻辑
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                    // termination为mainLock锁的condition实例，这个是来实现线程之间的通信。
                    //其实这里是来唤醒awaitTermination()方法，后面分析awaitTermination源码会看到。
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
            // 释放锁
                mainLock.unlock();
            }
           
        }
    }
```

线程池execute方法大致的逻辑如下面的时序图:
![execute_sequence](/images/execute_sequence.png)


* **shutdown**

中断线程池的线程，会等待正在执行的线程结束执行

```
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        //获取mainLock锁，防止其他线程执行
        mainLock.lock();
        try {
        //检查权限，确保用户线程有关闭线程池的权限
            checkShutdownAccess();
        //通过CAS将线程池状态设置成 SHUTDOWN
            advanceRunState(SHUTDOWN);
          //中断所有空闲的Workers , 下面分析这个方法
            interruptIdleWorkers();
          //钩子方法，让子类进行收尾的逻辑
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
        // 释放mainLock锁
            mainLock.unlock();
        }
        //execute方法，我们分析过了，主要就是尝试将线程池的状态设置为terminate
        tryTerminate();
    }
```

该方法我们比较关注的点是interruptIdleWorkers方法，是怎样中断空闲Worker，然后是如何保证Worker执行完毕的? 分析下interruptIdleWorkers方法:
```
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        //获取mainLock锁
        mainLock.lock();
        try {
        //轮询workers逐一中断
            for (Worker w : workers) {
                Thread t = w.thread;
                //判断 如果当前线程未中断且能够获取w锁，则执行中断
                // 如果当前线程未中断但不能获取w锁，那么就会阻塞，直到获取锁为止。
                //这里的w锁，就是前面在分析execute时，有个死循环不断取任务，取到任务就会获取w锁。
                //所以这边如果获取不到w锁，就证明还有任务没有执行完。
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                    //中断线程
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
到这里，核心逻辑就是通过w这个锁来完成的, 非空闲的worker线程内都是先获取w锁然后开始执行task的.

* **shutdownNow**

```
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
    }
```

源码和shutdown差不多，只不过将线程池状态设置为stop，然后调用interruptWorkers方法，看看interruptWorkers方法:

```
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
```

代码中并没有获取w锁的逻辑，所以这个方法会直接中断所有线程，并不会等待那些正在执行任务的worker把任务执行完。

* **awaitTermination**

调用awaitTermination方法会一直阻塞等待线程池状态变为 terminated才返回或者等待超时返回.

```
    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
            //如果已经是terminated状态直接返回
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                // （1）等待mainLock锁的condition实例来唤醒，不然持续阻塞。
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
```

（1）处的代码已经告诉了该方法什么时候返回，就是mainLock锁的termination条件变量被唤醒返回。在上面分析中termination条件变量被唤醒是在执行tryTerminate()时完成的，因为内部调用termination.signalAll()。而tryTerminate() 方法被shutDown() 和shutDownNow()调用过，所以如果要让awaitTermination返回，调用这2个方法就行.