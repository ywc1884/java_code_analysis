# ConcurrentHashMap源码分析

## Overview

ConcurrentHashMap in JDK8 is a thread-safe version of HashMap, which has a similar data structure
with HashMap. It uses arrays of buckets which could contain either linked list or red black tree.
please checkout the picture below:

![concurrenthashmap_overview](/images/concurrenthashmap_overview.jpg)

## Analysis of Construction Method

```
// Nothing in this constructor
public ConcurrentHashMap() {
}

// Initial capacity of HashMap
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```

This initialization method is somewhat interesting. By providing the initial capacity, the sizeCtl, sizeCtl = 1.5 * initial capacity + 1, is calculated, and then the nearest n power of 2 is taken up. If initial Capacity is 10, then sizeCtl is 16, and if initial Capacity is 11, sizeCtl is 32. Note that the method tableSizeFor is used to get the nearest n power of 2 for
the initial capacity, which is also used in HashMap.

In HashMap, the initial capacity is directly used as tableSizeFor (initial capacity). I don't know why the initial capacity is changed to 1.5 * initial capacity in Concurrent HashMap. For the case of adding 1, the initial capacity = 0 is considered.

The sizeCtl attribute uses many scenarios. Here is the first scenario: Concurrent HashMap initialization. SizeCtl = 0 (that is, the parametric constructor) denotes the default initialization size, otherwise the custom capacity is used.

## put process analysis

```
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 1. Calculate hash value, (h ^ (h > > > > > 16) & HASH_BITS
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 2. Spin ensures that newly added elements will be successfully added to HashMap
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 2.1 If the array is empty, initialize the array. 
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 2.2 The slot corresponding to the hash (also known as bucket bucket) is empty. Just put the new value in it directly.
        //     The array tab is volatile rhetoric, which does not mean that its elements are volatile. U.getObjectVolatile
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // The CAS operation places this new value in the slot, and if it succeeds, it ends.
            // If CAS fails, it is concurrent operation, then take step 3 or 4.
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 2.3 Only if f.hash==MOVED is expanded, the thread adds value after helping with the expansion first.
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 2.4 Lock the corresponding slot, and the rest of the operation is similar to HashMap.
        else {
            V oldVal = null;
            // The following operations are thread-safe
            synchronized (f) {
                // All operations on F after f is locked are thread-safe, but tab itself is not thread-safe.
                // That is to say, tab[i] may change.
                if (tabAt(tab, i) == f) {
                    // 2.4.1 header node hash >= 0, indicating that it is a linked list
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    // 2.4.2 denotes red-black trees
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // The linked list binCount indicates the length of the added element, and the red-black tree binCount=2 cannot perform the treeifyBin method.
            if (binCount != 0) {
                // To determine whether to convert a linked list to a red-black tree, the critical value is 8, just like HashMap.
                if (binCount >= TREEIFY_THRESHOLD)
                    // This method is slightly different from HashMap in that it is not necessarily a red-black tree conversion.
                    // If the length of the current array is less than 64, the array expansion is chosen instead of converting to a red-black tree.
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 3. Adding 1 to the number of elements and judging whether to expand or not, how to ensure thread safety
    addCount(1L, binCount);
    return null;
}
```

```
// All in all, for one purpose, make nodes more evenly distributed in HashMap
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

h is the hashcode corresponding to the key, and the slot of the computing node is the hash% length of the array (hash%length), but if the length of the array is 2^n, the bit operation hash & (length-1) can be used directly. In order to make the hash distributed more randomly, it uses hash's high 16 bit and low 16 bit to carry out XOR operation, so the last 16 bits of hash are not easy to repeat. Note that hashcode may be negative at this time. Negative numbers have a special meaning in Concurrent HashMap. In order to ensure that the calculated hash must be positive, the symbolic bits can be forcibly removed from the calculated hash values to ensure that the results are only in the positive range.

## Array initialization initTable

```
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // sizeCtl=-1 indicates that the array is initializing
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // SizeCtl = 0 (that is, parametric constructor) indicates the default initialization size, otherwise the custom capacity is used
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // n-n/4, or 0.75*n, is the same as the threshold in HashMap, except that bitwise operations are used here.
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

The concurrency problem in the initialization method is controlled by a CAS operation on sizeCtl.The attribute sizeCtl was mentioned earlier when Concurrent HashMap was initialized. Here we introduce another usage scenario of sizeCtl:

1. sizeCtl's first usage scenario: before array initialization. sizeCtl = 0 (that is, the parametric constructor) denotes the default initialization size, otherwise the customized initialization capacity is used.
2. The second use scenario of sizeCtl: in array initialization. sizeCtl=-1 indicates that the array table is being initialized
3. The third use scenario of sizeCtl: after array initialization. SizeCtl > 0 represents the threshold of table expansion
