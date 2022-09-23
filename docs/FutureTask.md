# 并发编程之Future&FutureTask深入解析

## **前言**

Java线程实现方式主要有四种：

* 继承Thread类
* 实现Runnable接口
* 实现Callable接口通过FutureTask包装器来创建Thread线程
* 使用ExecutorService、Callable、Future实现有返回结果的多线程


```
其中前两种方式线程执行完后都没有返回值，后两种是带返回值的。
```

### Callable 和 Runnable 接口

#### **Runnable**

```
// 实现Runnable接口的类将被Thread执行，表示一个基本的任务
public interface Runnable {
    // run方法就是它所有的内容，就是实际执行的任务
    public abstract void run();
}
```

* 没有返回值

    run 方法没有返回值，虽然有一些别的方法也能实现返回值的效果，比如编写日志文件或者修改共享变量等等，但是不仅容易出错，效率也不高。

* 不能抛出异常

    实现 Runnable接口的类，run方法无法抛出checked Exception，只能在方法内使用try catch进行处理，这样上层就无法得知线程中的异常。


* 设计导致

    其实这两个缺陷主要原因就在于 Runnable 接口设计的 run 方法，这个方法已经规定了 run() 方法的返回类型是 void，而且这个方法没有声明抛出任何异常。所以，当实现并重写这个方法时，我们既不能改返回值类型，也不能更改对于异常抛出的描述，因为在实现方法的时候，语法规定是不允许对这些内容进行修改的。

* Runnable为什么设计成这样？

    假设run()方法可以返回返回值，或者可以抛出异常，也无济于事，因为我们并没有办法在外层捕获并处理，这是因为调用run()方法的类（比如Thread类和线程池）是Java直接提供的，而不是我们编写的。所以就算它能有一个返回值，我们也很难把这个返回值利用到，而Callable接口就是为了解决这两个问题。

#### **Callable**

```
public interface Callable<V> {
    //返回接口，或者抛出异常
    V call() throws Exception;
}
```

可以看到Callable和Runnable接口其实比较相识，都只有一个方法，也就是线程任务执行的方法，区别就是call方法有返回值，而且声明了throws Exception。

##### **Callable 和 Runnable 的不同之处**
 * 方法名：
    Callable规定的执行方法是call()，而Runnable规定的执行方法是run()；
* 返回值：
    Callable的任务执行后有返回值，而Runnable的任务执行后是没有返回值的；
* 抛出异常：
    call()方法可抛出异常，而run()方法是不能抛出受检查异常的；

与Callable配合的有一个Future接口，通过Future可以了解任务执行情况，或者取消任务的执行，还可获取任务执行的结果，这些功能都是Runnable做不到的，Callable的功能要比Runnable强大。

#### **Future接口**

简单来说就是利用线程达到异步的效果，同时还可以获取子线程的返回值。

比如当做一定运算的时候，运算过程可能比较耗时，有时会去查数据库，或是繁重的计算，比如压缩、加密等，在这种情况下，如果我们一直在原地等待方法返回，显然是不明智的，整体程序的运行效率会大大降低。

我们可以把运算的过程放到子线程去执行，再通过 Future 去控制子线程执行的计算过程，最后获取到计算结果。这样一来就可以把整个程序的运行效率提高，是一种异步的思想。

### **Future的方法**

Future 接口一共有5个方法，源代码如下：

```
public interface Future<V> {

  /**
   * 尝试取消任务，如果任务已经完成、已取消或其他原因无法取消，则失败。
   * 1、如果任务还没开始执行，则该任务不应该运行
   * 2、如果任务已经开始执行，由参数mayInterruptIfRunning来决定执行该任务的线程是否应该被中断，这只是终止任务的一种尝试。若mayInterruptIfRunning为true，则会立即中断执行任务的线程并返回true，若mayInterruptIfRunning为false，则会返回true且不会中断任务执行线程。
   * 3、调用这个方法后，以后对isDone方法调用都返回true。
   * 4、如果这个方法返回true,以后对isCancelled返回true。
   */
    boolean cancel(boolean mayInterruptIfRunning);

   /**
    * 判断任务是否被取消了，如果调用了cance()则返回true
    */
    boolean isCancelled();

   /**
    * 如果任务完成，则返回ture
    * 任务完成包含正常终止、异常、取消任务。在这些情况下都返回true
    */
    boolean isDone();

   /**
    * 线程阻塞，直到任务完成，返回结果
    * 如果任务被取消，则引发CancellationException
    * 如果当前线程被中断，则引发InterruptedException
    * 当任务在执行的过程中出现异常，则抛出ExecutionException
    */
    V get() throws InterruptedException, ExecutionException;

   /**
    * 线程阻塞一定时间等待任务完成，并返回任务执行结果，如果则超时则抛出TimeoutException
    */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

### **get方法(获取结果)**

get 方法最主要的作用就是获取任务执行的结果，该方法在执行时的行为取决于 Callable 任务的状态，可能会发生以下 7 种情况。

* 任务已经执行完

    执行 get 方法可以立刻返回，获取到任务执行的结果。

* 任务还没有开始执行

    比如我们往线程池中放一个任务，线程池中可能积压了很多任务，还没轮到我去执行的时候，就去 get 了，在这种情况下，相当于任务还没开始，我们去调用 get 的时候，会当前的线程阻塞，直到任务完成再把结果返回回来。

* 任务正在执行中

    但是执行过程比较长，所以我去 get 的时候，它依然在执行的过程中。这种情况调用 get 方法也会阻塞当前线程，直到任务执行完返回结果。

* 任务执行过程中抛出异常

    我们再去调用 get 的时候，就会抛出 ExecutionException 异常，不管我们执行 call 方法时里面抛出的异常类型是什么，在执行 get 方法时所获得的异常都是 ExecutionException。

* 任务被取消了
    如果任务被取消，我们用 get 方法去获取结果时则会抛出 CancellationException。

* 任务被中断了

    如果任务被当前线程中断，我们用 get 方法去获取结果时则会抛出InterruptedException。

* 任务超时

    我们知道 get 方法有一个重载方法，那就是带延迟参数的，调用了这个带延迟参数的 get 方法之后，如果 call 方法在规定时间内正常顺利完成了任务，那么 get 会正常返回；但是如果到达了指定时间依然没有完成任务，get 方法则会抛出 TimeoutException，代表超时了。

### **isDone方法（判断是否执行完毕）**

isDone() 方法，该方法是用来判断当前这个任务是否执行完毕了

```
注意：这个方法返回 true 则代表执行完成了，返回 false 则代表还没完成。但返回 true，并不代表这个任务是成功执行的，比如说任务执行到一半抛出了异常。那么在这种情况下，对于这个 isDone 方法而言，它其实也是会返回 true 的，因为对它来说，虽然有异常发生了，但是这个任务在未来也不会再被执行，它确实已经执行完毕了。所以 isDone 方法在返回 true 的时候，不代表这个任务是成功执行的，只代表它执行完毕了。
```

### **cancel方法(取消任务的执行)**

如果不想执行某个任务了，则可以使用 cancel 方法，会有以下三种情况：

* 第一种情况最简单，那就是当任务还没有开始执行时，一旦调用 cancel，这个任务就会被正常取消，未来也不会被执行，那么 cancel 方法返回 true。

* 第二种情况也比较简单。如果任务已经完成，或者之前已经被取消过了，那么执行 cancel 方法则代表取消失败，返回 false。因为任务无论是已完成还是已经被取消过了，都不能再被取消了。

* 第三种情况比较特殊，就是这个任务正在执行，这个时候执行 cancel 方法是不会直接取消这个任务的，而是会根据我们传入的参数做判断。cancel 方法是必须传入一个参数，该参数叫作 「mayInterruptIfRunning」，
它是什么含义呢？
    * 如果传入的参数是 true，执行任务的线程就会收到一个中断的信号，正在执行的任务可能会有一些处理中断的逻辑，进而停止，这个比较好理解。

    * 如果传入的是 false 则就代表不中断正在运行的任务，也就是说，本次 cancel 不会有任何效果，同时 cancel 方法会返回 false。


### **isCancelled方法(判断是否被取消)**

isCancelled 方法，判断是否被取消，它和cancel方法配合使用.

**Callable和Future的关系**

Callable接口相比于Runnable的一大优势是可以有返回结果，返回结果就可以用Future类的get方法来获取 。因此，Future相当于一个存储器，它存储了Callable的call方法的任务结果。

除此之外，我们还可以通过Future的isDone方法来判断任务是否已经执行完毕了，还可以通过cancel方法取消这个任务，或限时获取任务的结果等，总之Future的功能比较丰富。

## **FutureTask**

Future只是一个接口，不能直接用来创建对象，其实现类是FutureTask，JDK1.8修改了FutureTask的实现，JKD1.8不再依赖AQS来实现，
而是通过一个volatile变量state以及CAS操作来实现。FutureTask结构如下所示：

![FutureTask](/images/FutureTask_diagram.png)

可以发现的是FutureTask确实实现了Runnable接口，同时还实现了Future接口，这个Future接口主要提供了后面我们使用FutureTask的一系列函数比如get.

看到这里你应该能够大致想到在FutureTask中的run方法会调用Callable当中实现的call方法，然后将结果保存下来，当调用get方法的时候再将这个结果返回.

**状态表示**

首先我们先了解一下FutureTask的几种状态：

* NEW，刚刚新建一个FutureTask对象。
* COMPLETING，FutureTask正在执行。
* NORMAL，FutureTask正常结束。
* EXCEPTIONAL，如果FutureTask对象在执行Callable实现类对象的call方法的时候出现的异常，那么FutureTask的状态就变成这个状态了。
* CANCELLED，表示FutureTask的执行过程被取消了。
* INTERRUPTING，表示正在终止FutureTask对象的执行过程。
* INTERRUPTED，表示FutureTask对象在执行的过程当中被中断了。

这些状态之间的可能的转移情况如下所示：

* NEW -> COMPLETING -> NORMAL
* NEW -> COMPLETING -> EXCEPTIONAL
* NEW -> CANCELLED
* NEW -> INTERRUPTING -> INTERRUPTED

在FutureTask当中用数字去表示这几个状态：

```
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

**核心函数和字段**

* FutureTask类当中的核心字段

```
private Callable<V> callable; // 用于保存传入 FutureTask 对象的 Callable 对象

private Object outcome; // 用于保存 Callable 当中 call 函数的返回结果

private volatile Thread runner; // 表示正在执行 call 函数的线程

private volatile WaitNode waiters;// 被 get 函数挂起的线程 是一个单向链表 waiters 表示单向链表的头节点

static final class WaitNode {
  volatile Thread thread; // 表示被挂起来的线程
  volatile WaitNode next; // 表示下一个节点
  WaitNode() { thread = Thread.currentThread(); }
}
```

* 构造函数

```
public FutureTask(Callable<V> callable) {
  if (callable == null)
    throw new NullPointerException();
  this.callable = callable; //保存传入来的 Callable 接口的实现类对象
  this.state = NEW;       // 这个就是用来保存 FutureTask 的状态 初识时 是新建状态
}
```

* run方法，这个函数是实现Runnable接口的方法，也就是传入Thread类之后，Thread启动时执行的方法。

```
public void run() {
  // 如果 futuretask 的状态 state 不是 NEW
  // 或者不能够设置 runner 为当前的线程的话直接返回
  // 不在执行下面的代码 因为 state 已经不是NEW 
  // 说明取消了 futuretask 的执行
  if (state != NEW ||
      !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                   null, Thread.currentThread()))
    return;
  try {
    Callable<V> c = callable;
    if (c != null && state == NEW) {
      V result;
      boolean ran; // 这个值主要用于表示call函数是否正常执行完成如果正常执行完成就为true
      try {
        result = c.call(); // 执行call函数得到我们需要的返回值并且柏村在
        ran = true;
      } catch (Throwable ex) {
        result = null;
        ran = false;
        setException(ex); // call函数异常执行设置state为异常状态并且唤醒由get函数阻塞的线程
      }
      if (ran)
        set(result); // call函数正常执行完成将得到的结果result保存到outcome当中并且唤醒被get函数阻塞的线程
    }
  } finally {
    // runner must be non-null until state is settled to
    // prevent concurrent calls to run()
    runner = null;
    // state must be re-read after nulling runner to prevent
    // leaked interrupts
    int s = state;
    // 如果这个if语句条件满足的话就表示执行过程被中断了
    if (s >= INTERRUPTING)
      // 这个主要是后续处理中断的过程不是很重要
      handlePossibleCancellationInterrupt(s);
  }
}
```

* set方法，主要是用于设置state的状态，并且唤醒由get函数阻塞的线程

```
protected void set(V v) { // call 方法正常执行完成执行下面的方法 v 是 call 方法返回的结果
  // 这个是原子交换 state 从 NEW 状态变成 COMPLETING 状态
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    outcome = v; // 将 call 函数的返回结果保存到 outcome 当中 然后会在 get 函数当中使用 outcome 
                                // 因为 get 函数需要得到 call 函数的结果 因此我们需要在 call 函数当中返回 outcome
    // 下面代码是将 state 的状态变成 NORMAL 表示程序执行完成
    UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
    // 因为其他线程可能在调用 get 函数的时候 call 函数还没有执行完成 因此这些线程会被阻塞 下面的这个方法主要是将这些线程唤醒
    finishCompletion();
  }
}

protected void setException(Throwable t) {
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    outcome = t; // 将异常作为结果返回
    // 将最后的状态设置为 EXCEPTIONAL
    UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
    finishCompletion();
  }
}
```

* get方法，这个方法主要是从FutureTask当中取出数据，但是这个时候可能call函数还没有执行完成，因此这个方法可能会阻塞调用这个方法的线程

```
public V get() throws InterruptedException, ExecutionException {
  int s = state;
  // 如果当前的线程还没有执行完成就需要将当前线程挂起
  if (s <= COMPLETING)
    // 调用 awaitDone 函数将当前的线程挂起
    s = awaitDone(false, 0L);
  // 如果 state 大于 COMPLETING 也就说是完成状态 可以直接调用这个函数返回 
  // 当然也可以是从 awaitDone 函数当中恢复执行才返回
  return report(s);// report 方法的主要作用是将结果 call 函数的返回结果 返回出去 也就是将 outcome 返回
  
}

private V report(int s) throws ExecutionException {
  Object x = outcome;
  if (s == NORMAL) // 如果程序正常执行完成则直接返回结果
    return (V)x;
  if (s >= CANCELLED) // 如果 s 大于 CANCELLED 说明程序要么是被取消要么是被中断了 抛出异常
    throw new CancellationException();
  // 如果上面两种转台都不是那么说明在执行 call 函数的时候程序发生异常
  // 还记得我们在 setException 函数当中将异常赋值给了 outcome 吗？
  // 在这里将那个异常抛出了
  throw new ExecutionException((Throwable)x);
}
```

* awaitDone方法，这个方法主要是将当前线程挂起

```
private int awaitDone(boolean timed, long nanos) // timed 表示是否超时阻塞  nanos 表示如果是超时阻塞的话 超时时间是多少
  throws InterruptedException {
  final long deadline = timed ? System.nanoTime() + nanos : 0L;
  WaitNode q = null;
  boolean queued = false;
  for (;;) { // 注意这是个死循环
    
    // 如果线程被中断了 那么需要从 “等待队列” 当中移出去
    if (Thread.interrupted()) {
      removeWaiter(q);
      throw new InterruptedException();
    }

    int s = state;
    // 如果 call 函数执行完成 注意执行完成可能是正常执行完成 也可能是异常 取消 中断的任何一个状态
    if (s > COMPLETING) {
      if (q != null)
        q.thread = null;
      return s;
    }
    // 如果是正在执行的话 说明马上就执行完成了 只差将 call 函数的执行结果赋值给 outcome 了
    // 因此可以不进行阻塞先让出 CPU 让其它线程执行 可能下次调度到这个线程 state 的状态很可能就
    // 变成 完成了
    else if (s == COMPLETING) // cannot time out yet
      Thread.yield();
    else if (q == null)
      q = new WaitNode();
    else if (!queued) // 如果节点 q 还没有入队
      // 下面这行代码稍微有点复杂 其中 waiter 表示等待队列的头节点
      // 这行代码的作用是将 q 节点的 next 指向 waiters 然后将 q 节点
      // 赋值给 waiters 也就是说 q 节点变成等待队列的头节点 整个过程可以用
      // 下面代码标识
      // q.next = waiters;
      // waiters = q;
      // 但是将 q 节点赋值给 waiter这个操作是原子的 可能成功也可能不成功
      // 如果不成功 因为 for 循环是死循环下次喊 还会进行 if 判断
      // 如果 call 函数已经执行完成得到其返回结果那么就可以直接返回
      // 如果还没有结束 那么就会调用下方的两个 if 分支将线程挂起
      queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                           q.next = waiters, q);
    else if (timed) {
      // 如果是使用超时挂起 deadline 表示如果时间超过这个值的话就可以将线程启动了
      nanos = deadline - System.nanoTime();
      if (nanos <= 0L) {
        // 如果线程等待时间到了就需要从等待队列当中将当前线程对应的节点移除队列
        removeWaiter(q);
        return state;
      }
      LockSupport.parkNanos(this, nanos);
    }
    else
      // 如果不是超时阻塞的话 直接将这个线程挂起即可
      // 这个函数前面已经提到了就是将当前线程唤醒
      // 就是将调用park方法的线程唤醒
      LockSupport.park(this);
  }
}
```

* finishCompletion方法，这个方法是用于将所有被get函数阻塞的线程唤醒

```
private void finishCompletion() {
  // assert state > COMPLETING;
  for (WaitNode q; (q = waiters) != null;) {
    // 如果可以将 waiter 设置为 null 则进入 for 循环 在 for 循环内部将所有线程唤醒
    // 这个操作也是原子操作
    if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
      for (;;) {
        Thread t = q.thread;
        // 如果线程不等于空 则需要将这个线程唤醒
        if (t != null) {
          q.thread = null;
          // 这个函数已经提到了 就是将线程 t 唤醒
          LockSupport.unpark(t);
        }
        // 得到下一个节点
        WaitNode next = q.next;
        // 如果节点为空 说明所有的线程都已经被唤醒了 可以返回了
        if (next == null)
          break;
        q.next = null; // unlink to help gc
        q = next; // 唤醒下一个节点
      }
      break;
    }
  }

  done();// 这个函数是空函数没有实现

  callable = null;        // to reduce footprint
}
```

* cancel方法，这个方法主要是取消FutureTask的执行过程

```
public boolean cancel(boolean mayInterruptIfRunning) {
  // 参数 mayInterruptIfRunning 表示可能在线程执行的时候中断
  
  // 只有 state == NEW 并且能够将 state 的状态从 NEW 变成 中断或者取消才能够执行下面的 try 代码块
  // 否则直接返回 false
  if (!(state == NEW &&
        UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                                 mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
    return false;
  try {    // in case call to interrupt throws exception
    // 如果在线程执行的时候中断代码就执行下面的逻辑
    if (mayInterruptIfRunning) {
      try {
        Thread t = runner; // 得到正在执行 call 函数的线程
        if (t != null)
          t.interrupt();
      } finally { // final state
        // 将 state 设置为 INTERRUPTED 状态
        UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
      }
    }
  } finally {
    // 唤醒被 get 函数阻塞的线程
    finishCompletion();
  }
  return true;
}
```

**`注`**: 本文章为转载，原文链接如下: 

[FutureTask源码深度剖析](https://segmentfault.com/a/1190000042279932)