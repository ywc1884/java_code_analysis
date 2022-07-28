# ConcurrentHashMap源码分析

## **概述**

本文将主要讲述 JDK1.8 版本 的 ConcurrentHashMap(简称CHM)，其内部结构和很多的哈希优化算法，都是和 JDK1.8 版本的 HashMap是一样的，所以在阅读本文之前，一定要先了解 HashMap.另外 ConcurrentHashMap 中同样有红黑树，这部分可以先不看不影响整体结构把握, CHM的数据结构图和1.8的HashMap是一致的，如下图所示:

![concurrenthashmap_overview](/images/concurrenthashmap_overview.jpg)


CHM 的源码有 6k 多行，包含的内容多，精巧，不容易理解；建议在查看源码的时候，可以首先把握整体结构脉络，对于一些精巧的优化，哈希技巧可以先了解目的就可以了，不用深究；对整体把握比较清楚后，在逐步分析，可以比较快速的看懂。

其主要区别就在 CHM 支持并发：

```
1. 使用 Unsafe 方法操作数组内部元素，保证可见性；（U.getObjectVolatile、U.compareAndSwapObject、U.putObjectVolatile)
2. 在更新和移动节点的时候，直接锁住对应的哈希桶，锁粒度更小，且动态扩展
3. 针对扩容慢操作进行优化:
   * 首先扩容过程的中，节点首先移动到过度表 nextTable ，所有节点移动完毕时替换散列表 table
   * 移动时先将散列表定长等分，然后逆序依次领取任务扩容，设置 sizeCtl 标记正在扩容
   * 移动完成一个哈希桶或者遇到空桶时，将其标记为 ForwardingNode 节点，并指向 nextTable
   * 后有其他线程在操作哈希表时，遇到 ForwardingNode 节点，则先帮助扩容（继续领取分段任务），扩容完成后再继续之前的操作
   * 优化哈希表计数器，采用 LongAdder、Striped64 类似思想
   * 以及大量的哈希算法优化和状态变量优化
```

以上讲的这些不太清楚也没有关系，主要是有一个印象，大致清楚 CHM 的实现方向，具体细节后面还会结合源码详细讲解；


### **类定义和重要成员变量**

```
private static final int MAXIMUM_CAPACITY = 1 << 30;       // 最大容量
private static final int DEFAULT_CAPACITY = 16;            // 默认初始化容量
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;   // 并发级别，为兼容1.7，实际未用
private static final float LOAD_FACTOR = 0.75f;       // 固定负载系数，n - (n >>> 2)
static final int TREEIFY_THRESHOLD = 8;               // 链表长度超过8时，转为红黑树
static final int UNTREEIFY_THRESHOLD = 6;             // 红黑树节点数量低于6时，转为链表
static final int MIN_TREEIFY_CAPACITY = 64;           // 树化最小容量，容量小于64时，先扩容
private static final int MIN_TRANSFER_STRIDE = 16;    // 扩容时拆分散列表，最小线程任务处理步长
private static int RESIZE_STAMP_BITS = 16;            
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;  // 可参与扩容的最大线程

static final int NCPU = Runtime.getRuntime().availableProcessors();  // CPU数
transient volatile Node<K,V>[] table;                // 散列表数组
private transient volatile Node<K,V>[] nextTable;    // 扩容时的过度表
        
private transient volatile int sizeCtl;              // 最重要的状态变量，下面详讲
private transient volatile int transferIndex;        // 扩容进度指示, 使用逆向处理

private transient volatile long baseCount;              // 计数器，基础基数
private transient volatile int cellsBusy;               // 计数器，并发标记
private transient volatile CounterCell[] counterCells;  // 计数器，并发累计

public ConcurrentHashMap() { }

public ConcurrentHashMap(int initialCapacity) {
if (initialCapacity < 0)
    throw new IllegalArgumentException();
int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
        MAXIMUM_CAPACITY :
        tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));  // 注意这里不是0.75，后面介绍
this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0) 
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);   // 注意这里的初始化
    int cap = (size >= (long)MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}

```


上面有几个重要的地方这里单独讲：

```
LOAD_FACTOR:

这里的负载系数，同 HashMap 等其他 Map 的系数有明显区别：

通常的系数默认 0.75，可以由构造函数传入，当节点数 size 超过 loadFactor * capacity 时扩容

而 CMH 的系数则固定 0.75（使用 n - (n >>> 2) 表示），构造函数传入的系数只影响初始化容量，见第5个构造函数

上面第二个构造函数中，initialCapacity + (initialCapacity >>> 1) + 1)，这里居然不是使用的默认0.75，可以看作bug，也可视作优化，见

https://bugs.openjdk.java.net/browse/JDK-8202422

https://stackoverflow.com/questions/50083966/bug-parameter-initialcapacity-of-concurrenthashmaps-construct-method


sizeCtl:

sizeCtl 是 CHM 中最重要的状态变量，其中包括很多中状态，这里先整体介绍帮助后面源码理解；

sizeCtl = 0 ：初始值，还未指定初始容量

sizeCtl > 0 ：

table 未初始化，表示初始化容量
table 已初始化，表示扩容阈值（0.75n)
sizeCtl = -1 ：表示正在初始化

sizeCtl < -1 ：表示正在扩容，具体结构如图所示：

```
![sizectl](/images/CHM_sizectl.png)

计算代码如下：

```
/*
 * n=64
 * Integer.numberOfLeadingZeros(n)＝26
 * resizeStamp(64) = 0001 1010 | 1000 0000 0000 0000 = 1000 0000 0001 1010
 */
static final int resizeStamp(int n) {
  return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}

```

所以 resizeStamp(64) << RESIZE_STAMP_SHIFT) + 2 ，表示扩容目标为 64，有一个线程正在扩容.

***

### **Node节点**

```
static class Node<K,V> implements Map.Entry<K,V> {  // 哈希表普通节点
  final int hash;
  final K key;
  volatile V val;
  volatile Node<K,V> next;
  
  Node<K,V> find(int h, Object k) {}   // 主要在扩容时，利用多态查询已转移节点
}

static final class ForwardingNode<K,V> extends Node<K,V> {  // 标识扩容节点
  final Node<K,V>[] nextTable;  // 指向成员变量 ConcurrentHashMap.nextTable
  
  ForwardingNode(Node<K,V>[] tab) {
    super(MOVED, null, null, null);  // hash = -1，快速确定 ForwardingNode 节点
    this.nextTable = tab;
  }
  
  Node<K,V> find(int h, Object k) {}
}

static final class TreeBin<K,V> extends Node<K,V> { // 红黑树根节点
  TreeBin(TreeNode<K,V> b) {
    super(TREEBIN, null, null, null);  // hash = -2，快速确定红黑树
    ...
  }
}  
static final class TreeNode<K,V> extends Node<K,V> { } // 红黑树普通节点，其 hash 同 Node 普通节点 > 0

```

### **哈希计算**

```
static final int MOVED     = -1;          // forwarding nodes hash值
static final int TREEBIN   = -2;          // 红黑树根节点Treebin
static final int RESERVED  = -3;          // hash for transient reservations
static final int HASH_BITS = 0x7fffffff;  // usable bits of normal node hash

// 让高位16位，参与哈希桶定位运算的同时，保证hash为正, CHM内hash为整数表示正常节点，否则为负数表示有其他含义
// 通过和HASH_BITS与能确保结果是正数
static final int spread(int h) {
  return (h ^ (h >>> 16)) & HASH_BITS;
}

```

除此之外还有，

* tableSizeFor : 将容量转为大于n，且最小的2的幂
* 除留余数法 ：hash % length = hash & (length-1)
* 扩容后哈希桶定位：(e.hash & oldCap)，0 - 位置不变，1 - 原来的位置 + oldCap.

以上这些哈希优化的具体原理和Hashmap一样就不在重复了，可以参考HashMap相关的文档说明.


## **源码解析**

1. initTable 方法

```
private final Node<K,V>[] initTable() {
  Node<K,V>[] tab; int sc;
  while ((tab = table) == null || tab.length == 0) {
    if ((sc = sizeCtl) < 0) Thread.yield();  //有其他线程已经在初始化
    else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  // 通过CAS设置sizectl为-1表示当前线程准备初始化table
      try {
        if ((tab = table) == null || tab.length == 0) {
          // 注意此时的 sizeCtl 表示初始容量，完毕后表示扩容阈值
          // 如果通过有容量参数构造器创建的CHM, 则使用对应容量对应的值, 否则使用默认值16
          int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  
          @SuppressWarnings("unchecked")
          Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
          table = tab = nt;
          sc = n - (n >>> 2);  // 同 0.75n
        }
      } finally {
        sizeCtl = sc;  // 注意这里没有CAS更新，这就是状态变量的高明了，因为前面设置了-1，此时这里没有竞争
      }
      break;
    }
  }
  return tab;
}

```

2. get 方法


```
public V get(Object key) {
  Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
  int h = spread(key.hashCode()); // 计算 hash
  if ((tab = table) != null && (n = tab.length) > 0 &&  // 确保 table 已经初始化
    
    // 确保对应的哈希桶不为空，注意这里是 Volatile 语义获取；因为扩容的时候，是完全拷贝，所以只要不为空，则链表必然完整
    (e = tabAt(tab, (n - 1) & h)) != null) {
    if ((eh = e.hash) == h) {
      if ((ek = e.key) == key || (ek != null && key.equals(ek)))
        return e.val;
    }
    
    // hash < 0有2种情况
    // hash = -1，表示当前bucket节点是forwarding node表示在扩容，原来位置的节点可能全部移动到 i + oldCap 位置，所以利用多态到 nextTable 中查找
    // hash = -2, 表示该节点为红黑树根节点Treebin, 同样是通过多态的find方法在红黑树中查找节点
    else if (eh < 0) return (p = e.find(h, key)) != null ? p.val : null;
    
    while ((e = e.next) != null) { // 否则该bucket内的就是链表, 遍历链表查找
      if (e.hash == h &&
        ((ek = e.key) == key || (ek != null && key.equals(ek))))
        return e.val;
    }
  }
  return null;
}

```

3. putVal 方法

注意 CHM 的 key 和 value 都不能为空. value不能为空据说是因为无法区分get返回结果为null时到底是由于value是null还是因为
CHM内没有对应的key造成的. 在HashMap中可以通过containsKey方法判断, 但是CHM在get方法和containsKey方法之间调用时可能存在
由于其他线程的操作导致结果不一致的情况:
```
if(map.containsKey(key)) {
    Object v = map.get(key);
}
```

至于key为啥不能为空个人就不太清楚了, 但是作者本人的说法是在map中引入支持key, value为null都是容易导致bug产生的，因此不建议
这么做.

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
  if (key == null || value == null) throw new NullPointerException();
  int hash = spread(key.hashCode());  // hash计算
  int binCount = 0;                   // 状态变量，主要表示对应bucket的链表节点数，最后判断是否转为红黑树
  for (Node<K,V>[] tab = table;;) { //自旋
    Node<K,V> f; int n, i, fh;
    if (tab == null || (n = tab.length) == 0) tab = initTable();  // 初始化
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {      // cas 获取哈希桶
      if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) // 如果当前bucket是null表示未被占用, 直接使用CAS更新，失败时继续循环更新
        break;  // no lock when adding to empty bin
    }
    else if ((fh = f.hash) == MOVED) tab = helpTransfer(tab, f);  // 正在扩容的时候，先帮助扩容, 帮助完扩容之后再重试操作
    else {
      V oldVal = null;
      synchronized (f) {  // 注意这里只锁定了一个哈希桶，所以比 1.7 中的 Segment 分段锁粒度更低
        if (tabAt(tab, i) == f) {
          if (fh >= 0) {   // hash >=0 则必然是普通链表节点，直接遍历链表即可
            binCount = 1;
            for (Node<K,V> e = f;; ++binCount) {
              K ek;
              if(e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) { // 查找成功
                oldVal = e.val;
                if (!onlyIfAbsent) e.val = value;
                break;
              }
              Node<K,V> pred = e;
              if ((e = e.next) == null) {  // 查找失败时，直接在末尾添加新节点
                pred.next = new Node<K,V>(hash, key, value, null);
                break;
              }
            }
          }
          else if (f instanceof TreeBin) {  // 红黑树树根节点
            Node<K,V> p;
            binCount = 2;
            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {  // 红黑树查找
              oldVal = p.val;
              if (!onlyIfAbsent) p.val = value;
            }
          }
        }
      }
      if (binCount != 0) {
        if (binCount >= TREEIFY_THRESHOLD) treeifyBin(tab, i); // 如果链表长度大于8，检查是否可以转为红黑树
        if (oldVal != null)
          return oldVal;
        break;
      }
    }
  }
  addCount(1L, binCount); // 计数加一，注意这里使用的是计数器，普通的Atomic变量仍然可能称为性能瓶颈
  return null;
}

```

putVal流程图大致如下:

![putval](/images/putval_flow.png)


4. 扩容

扩容操作一直都是比较慢的操作，而 CHM 中巧妙的利用任务划分，使得多个线程可能同时参与扩容；另外扩容条件也有两个：

* 有链表长度超过 8，但是容量小于 64 的时候，发生扩容
* 节点数超过阈值的时候，发生扩容

其扩容的过程可描述为：

* 首先扩容过程的中，节点首先移动到过度表nextTable ，所有节点移动完毕时替换散列表table
* 移动时先将散列表定长等分，然后逆序依次领取任务扩容，设置 sizeCtl 标记正在扩容
* 移动完成一个哈希桶或者遇到空桶时，将其标记为ForwardingNode节点，并指向nextTable
* 后有其他线程在操作哈希表时，遇到 ForwardingNode 节点，则先帮助扩容(继续领取分段任务),扩容完成后再继续之前的操作

图形化表示如下：

![CHM_expand](/images/CHM_expand.png)


**transfer方法**:
```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
  int n = tab.length, stride;
  if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
    stride = MIN_TRANSFER_STRIDE; // 根据CPU数量计算任务步长
  if (nextTab == null) {          // 初始化nextTab, 第一个扩容线程传入参数为null, 其余帮助扩容的会传入nextTable
    try {
      @SuppressWarnings("unchecked")
      Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];  // 扩容一倍
      nextTab = nt;
    } catch (Throwable ex) {
      sizeCtl = Integer.MAX_VALUE; // 发生OOM时，不再扩容
      return;
    }
    nextTable = nextTab;
    transferIndex = n;
  }
  int nextn = nextTab.length;
  ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);  // 标记空桶，或已经转移完毕的桶
  boolean advance = true; // advance的作用是逆向获取当前线程可以领取的任务区段, 当已经获取到时改为false, 如果任务做完了会继续改成true继续领取任务
  boolean finishing = false; // to ensure sweep before committing nextTab, fnishing表示扩容马上完成
  for (int i = 0, bound = 0;;) {  // 逆向遍历扩容
    Node<K,V> f; int fh;
    while (advance) {  // 向前获取哈希桶
      int nextIndex, nextBound;
      if (--i >= bound || finishing)               // 已经取到哈希桶，或已完成时退出
        advance = false;
      else if ((nextIndex = transferIndex) <= 0) { // 遍历到达头节点，已经没有待迁移的桶，线程准备退出
        i = -1;
        advance = false;
      }
      else if (U.compareAndSwapInt
           (this, TRANSFERINDEX, nextIndex,
            nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {  // 当前任务完成，领取下一批哈希桶
        bound = nextBound;  // bound表示领取到的任务范围的起始index, i表示任务范围结束的index
        i = nextIndex - 1;  // 索引指向下一批哈希桶
        advance = false;
      }
    }
    
    // i < 0  ：表示扩容结束，已经没有待移动的哈希桶
    // i >= n ：扩容结束，再次检查确认, 不清楚为啥还要再次确认
    // i + n >= nextn: 不清楚什么情况会满足这个条件
    if (i < 0 || i >= n || i + n >= nextn) { // 完成扩容准备退出
      int sc;
      if (finishing) {  // 两次检查，只有最后一个扩容线程退出时，才更新变量
        nextTable = null;
        table = nextTab;
        sizeCtl = (n << 1) - (n >>> 1); // 更新阈值为0.75*2*n
        return;
      }
      if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {  // 扩容线程减一
        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT) return;  // 不是最后一个线程，直接退出
        finishing = advance = true;   // 最后一个线程，再次检查
        i = n;                        // recheck before commit
      }
    }
    else if ((f = tabAt(tab, i)) == null)  // 当前节点为空，直接标记为ForwardingNode，然后继续获取下一个桶
      advance = casTabAt(tab, i, null, fwd);
    
    // 之前的线程已经完成该桶的移动，直接跳过，正常情况下自己的任务区间，不会出现ForwardingNode节点，
    else if ((fh = f.hash) == MOVED)  // 此处为极端条件下的健壮性检查
      advance = true; // already processed
    
    // 开始处理链表
    else {
      // 注意在 get 的时候，可以无锁获取，是因为扩容是全拷贝节点，完成后最后在更新哈希桶
      // 而在 put 的时候，是直接将节点加入尾部，获取修改其中的值，此时如果允许 put 操作，最后就会发生脏读，
      // 所以 put 和 transfer，需要竞争同一把锁，也就是对应的哈希桶，以保证内存一致性效果
      synchronized (f) { 
        if (tabAt(tab, i) == f) {  // 确认锁定的是同一个桶
          Node<K,V> ln, hn;
          if (fh >= 0) {  // 正常节点
            int runBit = fh & n;  // hash & n，判断扩容后的索引
            Node<K,V> lastRun = f;
            
            // 此处找到链表最后扩容后处于同一位置的连续节点，这样最后一节就不用再一次复制了
            for (Node<K,V> p = f.next; p != null; p = p.next) {
              int b = p.hash & n;
              if (b != runBit) {
                runBit = b;
                lastRun = p;
              }
            }
            if (runBit == 0) {
              ln = lastRun;
              hn = null;
            }
            else {
              hn = lastRun;
              ln = null;
            }
            
            // 依次将链表拆分成，lo、hi 两条链表，即位置不变的链表，和位置 + oldCap 的链表
            // 注意最后一节链表没有new，而是直接使用原来的节点
            for (Node<K,V> p = f; p != lastRun; p = p.next) {
              int ph = p.hash; K pk = p.key; V pv = p.val;
              if ((ph & n) == 0)
                ln = new Node<K,V>(ph, pk, pv, ln);
              else
                hn = new Node<K,V>(ph, pk, pv, hn);
            }
            setTabAt(nextTab, i, ln);      // 插入 lo 链表
            setTabAt(nextTab, i + n, hn);  // 插入 hi 链表
            setTabAt(tab, i, fwd);         // 哈希桶移动完成，标记为ForwardingNode节点
            advance = true;                // 继续获取下一个任务范围
          }
          else if (f instanceof TreeBin) { // 拆分红黑树
            TreeBin<K,V> t = (TreeBin<K,V>)f;
            TreeNode<K,V> lo = null, loTail = null; // 为避免最后在反向遍历，先留头结点的引用，
            TreeNode<K,V> hi = null, hiTail = null; // 因为顺序的链表，可以加速红黑树构造
            int lc = 0, hc = 0;  // 同样记录 lo，hi 链表的长度
            for (Node<K,V> e = t.first; e != null; e = e.next) {  // 中序遍历红黑树
              int h = e.hash;
              TreeNode<K,V> p = new TreeNode<K,V>(h, e.key, e.val, null, null);  // 构造红黑树节点
              if ((h & n) == 0) {
                if ((p.prev = loTail) == null)
                  lo = p;
                else
                  loTail.next = p;
                loTail = p;
                ++lc;
              }
              else {
                if ((p.prev = hiTail) == null)
                  hi = p;
                else
                  hiTail.next = p;
                hiTail = p;
                ++hc;
              }
            }
            
            // 判断是否需要将其转化为红黑树，同时如果只有一条链，那么就可以不用在构造
            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) : (hc != 0) ? new TreeBin<K,V>(lo) : t;
            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) : (lc != 0) ? new TreeBin<K,V>(hi) : t;
            setTabAt(nextTab, i, ln);
            setTabAt(nextTab, i + n, hn);
            setTabAt(tab, i, fwd);
            advance = true;
          }
        }
      }
    }
  }
}

```

**tryPresize方法**:

下面的treeifyBin方法是在putVal方法最后判断链表节点binCount大于8点时候会调用的方法:
```
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                ...
            }
        }
    }

```

如上面方法所示, 链表长度大于8时是否转成红黑树最终还需要判断散列表数组的长度是否小于64, 如果小于64
则改为扩容1倍. 如下是tryPresize的源码分析:

```
    private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1); //c表示table容量
        int sc;
        while ((sc = sizeCtl) >= 0) { // siztCtl为正数表示没有在扩容
            Node<K,V>[] tab = table; int n;
            // 这里又进行了一次和initTable一样的table初始化, 原因是使用ConcurrentHashMap(Map<? extends K, ? extends V> m)构造函数时会直接调用tryPresize, 因此需要考虑这种情况进行可能的初始化
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY) //table已初始化，容量没达到扩容阈值或者容量已超过最大值时不扩容
                break;
            else if (tab == table) {
                int rs = resizeStamp(n); //准备开始扩容, 计算本次扩容的标识值
                //将扩容标示值左移16位, 低16位表示当前扩容线程数+1, 这里rs左移16位+2表示是第一个扩容的线程
                if (U.compareAndSetInt(this, SIZECTL, sc,
                                        (rs << RESIZE_STAMP_SHIFT) + 2)) 
                    transfer(tab, null); //开始扩容
            }
        }
    }

```


**helpTransfer方法**

在putVal方法中有如下一段代码表示当前正在扩容, 需要帮忙扩容:
```
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
```

helpTransfer源码分析如下:

```
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // 确认当前bucket节点的确是ForwardingNode, 且该ForwardingNode对应在扩容的临时nextTable不为null
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length); //获取本次扩容标识值
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) { //sizeCtl < 0表示当前的确在扩容中
                //此处条件用于判断在哪些情况下不需要帮忙扩容, 此处代码有bug
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) { //可以参与帮助扩容，将sizeCtl值+1, 表示多了一个线程参与本次扩容
                    transfer(tab, nextTab); //参与本次扩容
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }

```
这段代码中至于一点需要注意的，在判断在哪些情况下不需要参与扩容时当前1.8的代码是存在bug的.
正确的代码应该如下:

```
int rs = resizeStamp(tab.length) << RESIZE_STAMP_SHIFT;
while (nextTab == nextTable && table == tab &&
        (sc = sizeCtl) < 0) {
    if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
        transferIndex <= 0)
        break;
    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
        transfer(tab, nextTab);
        break;
    }
}
```

相关bug信息可以看这个网页(java官方在JDK12修复了这个问题, 修复代码即为上面的代码):
https://stackoverflow.com/questions/53493706/how-the-conditions-sc-rs-1-sc-rs-max-resizers-can-be-achieved-in

如上述正确代码所示, 在如下几个条件下不需要参与扩容:

* sc == rs + MAX_RESIZERS: 目前参与扩容线程数达到上限
* sc == rs + 1: 由于sc的低16位表示参与扩容线程数+1, 因此这个条件表示扩容已完成
* transferIndex <= 0 : transferIndex是记录扩容进度的index, 该值<=0也表示扩容已完成



**addCount方法**

CHM中采取了和LongAdder类似的方案, 使用了一个baseCount作为基础计数器, 同时也使用了counterCells数组让多个线程
在数组中随机挑选位置将计数值更新到该位置, 这样做的好处是多个线程把计数值写到数组的不同slot中, 提高了并发度和效率.
其中每个线程在counterCells数组的位置采用ThreadLocalRandom.getProbe()方法生成一个线程独有的随机数, 然后针对
counterCells数组长度-1取模获取slot位置.

```
final long sumCount() {
    // basecount + counterCells数组内所有计数值即为最终计数值
    CounterCell[] cs = counterCells;
    long sum = baseCount;
    if (cs != null) {
        for (CounterCell c : cs)
            if (c != null)
                sum += c.value;
    }
    return sum;
}
private final void addCount(long x, int check) {
        CounterCell[] cs; long b, s;
        // 如果counterCells数组不为空则优先使用counterCells, 这样效率高一些
        // 如果counterCells数组还没初始化, 则直接试着使用CAS的方式更新计数值到baseCount
        if ((cs = counterCells) != null ||
            !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell c; long v; int m;
            boolean uncontended = true;
            // 如果counterCells数组没有初始化, 或者当前线程随机到的slot为null(未被占用)
            // 或者counterCells已初始化且随机到的slot也不为null, 但是直接CAS更新计数值失败
            // 这些情况下调用fullAddCount进行更复杂的处理
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            // 获取总节点计数值
            s = sumCount();
        }
        //进行扩容, 此处check有可能等于负数, 在remove方法中会调用addCount同时check传值-1
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // 如果节点总数大于等于阈值则进行扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) { //sizeCtl <0表示已经在扩容了
                    //此处bug和前面的tryPresize一样, 不满足条件则不参与扩容
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //否则将sizeCtl+1并参与扩容
                    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                //当前线程是第一个开始扩容的，将sizeCtl值设为rs标识左移16位+2
                else if (U.compareAndSetInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null); //开始扩容
                s = sumCount();
            }
        }
    }

private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        if ((h = ThreadLocalRandom.getProbe()) == 0) { //如果当前线程随机值为0, 则重新初始化并获取新随机数
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            CounterCell[] cs; CounterCell c; int n; long v;
            if ((cs = counterCells) != null && (n = cs.length) > 0) {
                if ((c = cs[(n - 1) & h]) == null) { //随机到的slot未被占用
                    if (cellsBusy == 0) {            // Try to attach new Cell, cellsBusy表示是否能操作这个countercell数组, 1表示被占用，0表示无人占用
                        CounterCell r = new CounterCell(x); // Optimistic create, 创建一个节点并设置到当前随机到的位置上
                        if (cellsBusy == 0 &&
                            U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash, 进来时wasUncontended就是false, 表示外面的CAS失败了, 需要经过下面advanceProbe重新计算随机位置后重试
                else if (U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x)) //尝试直接使用CAS在当前位置更新计数值
                    break;
                else if (counterCells != cs || n >= NCPU)  // counterCells扩容有上限
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 &&
                         U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == cs) // Expand table unless stale
                            counterCells = Arrays.copyOf(cs, n << 1); //counterCells中碰撞过多，扩容1倍
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = ThreadLocalRandom.advanceProbe(h);
            }
            else if (cellsBusy == 0 && counterCells == cs &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == cs) { //初始化counterCells数组
                        CounterCell[] rs = new CounterCell[2];
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (U.compareAndSetLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base, 实在不行再试试CAS设置baseCount
        }
    }
```


需要注意一点的是当获取Map.size的时候，如果使用Atomic变量，很容易导致过度竞争(多个线程CAS很容易产生冲突)，产生性能瓶颈，所以 CHM 中使用了计数器的方式.