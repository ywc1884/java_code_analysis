# Java Collection源码分析

Java集合类的继承关系如下图:

![collection_hiarachy](/images/collections_class_hiarachy.png)

**Iterable 接口**

Iterable : 可迭代接口.任何类的迭代器都需要通过实现该接口的Iterator iterator()方法返回.

* iterator()：用于获取一个迭代器。

* forEach() ：JDK8 新增。一个基于函数式接口实现的新迭代方法:
```
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

List相关的类继承关系如下图:

![list_hiarachy](/images/list_hiarachy.png)



## ArrayList

1. 数据结构

* ArrayList 底层是 Object[]数组，被 RandomAccess 接口标记，具有根据下标高速随机访问的功能


2. 扩容缩容

* ArrayList 扩容是扩大1.5倍，只有构造方法指定初始容量为0时，才会在第一次扩容出现小于10的容量，否则第一次扩容后的容量必然大于等于10；

* ArrayList 的添加可能会因为扩容导致数组“膨胀”，同理，不是所有的删除都会引起数组“缩水”：当删除的元素是队尾元素，或者clear()方法都只会把下标对应的地方设置为null，而不会真正的删除数组这个位置；

3. 迭代中的操作

ArrayList 在循环中删除——准确的讲，是任何会引起 modCount变化的结构性操作——可能会引起意外:

* 在forEach()删除元素会抛ConcurrentModificationException异常，因为 forEach()在循环开始前就获取了 modCount，每次循环都会比较旧 modCount和最新的 modCount，如果循环进行了会使modCount变动的操作，就会在下一次循环开始前抛异常

* 在for循环里删除实际上是以步长为2对节点进行删除，因为删除时数组“缩水”导致原本要删除的下一下标对应的节点，却落到了当前被删除的节点对应的下标位置，导致被跳过。

* 如果从队尾反向删除，就不会引起数组“缩水”，因此是正常的。



还一点建议是如果提前大概知道list的容量，可以先通过指定初始容量的constructor创建ArrayList, 避免添加元素后反复扩容.


## LinkedList

1. 数据结构

* LinkedList底层实现基于双向的双端链表，他无法像ArrayList那样直接随机通过索引去访问元素，每一次获取迭代器或者节点，都需要通过node()方法遍历整个链表

* LinkedList实现了Deque接口，因此可以把它当成队列或者栈使用。实际上, 他也提供了对应的同名方法

2. 迭代优化

* 尽管LinkedList针对这个问题做了优化，在遍历之前，通过位运算获取当前链表长度的一半，借此判断要获取的节点在链表前半截还是后半节，以选择从头还是尾开始遍历，保证了最坏情况下也只需要遍历一半的节点就能找到目标节点

因此，如非必要，最好不要通过下标去删除LinkedList中的元素，或者尽量在一次循环中完成所有操作

迭代操作

* LinkedList的迭代器创建的时候就会获取modCount，所以在迭代器里调用非迭代器自带的结构性操作方法，都会导致在下一次调用迭代器的结构性操作方法的时候抛出ConcurrentModificationException异常.

* LinkedList在for循环中删除元素，同ArrayList一样，会因为索引“偏移”导致漏删，解决方式也是倒序删除或者其他不会导致索引“偏移”的方法——当然，考虑到性能问题，最好不要在for循环去操作 LinkedList.


## LinkedHashMap

1. 数据结构

* LinkedHashMap是HashMap的子类，它在HashMap提供的哈希表的基础上又实现了双向链表，链表默认按插入顺序排序，这是他有序性的结构基础。 双向链表通过重写父类HashMap的NewNode方法
实现.

2. 有序性

* LinkedHashMap默认按插入顺序排序，可以通过在构造方法中指定accessOrder=true，可以开启按访问顺序排序，此时当访问节点后，被访问的节点会被移动到链表的尾部。 accessOrder也是通过
实现重写父类HashMap的AfterNodeAccess调整访问的节点到链表尾部.

3. 迭代

* LinkedHashMap也重写了forEach()，因此它与 HashMap一样，可以通过forEach()或者视图集合进行迭代。此外，它同样实现了fast-fail机制。

* LinkedHashMap重写了entrySet()，values()和keySet()方法，并且视图集合的迭代器都依赖于LinkedHashIterator，而该迭代器总是从链表头迭代到链表位，因此通过视图进行迭代也是有序的。

4. LRU 容器的实现

* 自定义一个类继承LinkedHashMap，重写removeEldestEntry()，并在父类构造方法中指定accessOrder=true开启按访问顺序排序即可。另外，需要在父类构造器中指定容量需要大于指定容量 / 负载系数，避免扩容。


## TreeMap

TreeMap是基于红黑树实现的Map, 遍历时支持按照key的大小排序遍历, 如果传入自定义Comparator则按指定规则顺序遍历. 大概总结点如下:

1. 基于红黑树的数据结构实现
2. 不允许插入为Null的key，HashMap可以为空，注意区别
3. 若Key重复，则后面插入的直接覆盖原来的Value
4. 非线程安全：底层没有synchronized这类的关键字
5. 可传入自己的比较器：从构造方法就可以看出

关于红黑树的介绍可以查看这篇文章: https://www.cnblogs.com/sanzao/p/10509626.html


## **Set**

Set接口是Collection接口下三大子接口之一。其下实现类都为元素不可重复的，不保证线程安全的集合。他有两个主要实现，即无序的HashSet与有序的TreeSet。

Set相对List集合与Queue集合不同之处在于，他的实现类需要依赖与Map集合的实现类密切相关。这体现在以下两点:

* HashSet实际依赖于HashMap，他使用HashMap的key作为存储容器。TreeSet同理，依赖于TreeMap实现。
* Map集合中的keySet与EntrySet视图集合往往以实现了Set接口的内部类出现在Map的实现类中。

![set_hiarachy](/images/set_class_hiarachy.png)

Set接口总结如下:

* Set接口是Collection下一个不可重复，线程不安全的集合，主要有HashSet与TreeSet两大实现类，分别依赖于Map接口下HashMap与TreeMap实现。

* Set继承了Collection接口，因而Set集合可以通过forEach()或者迭代器迭代。

* Set接口存在子接口SortedSet与孙子接口NavigableSet，规定了实现该接口的类可以根据自然顺序或者比较器排序保证顺序。

AbstractSet抽象类

* AbstractSet抽象类继承了Collection抽象类，实现了Set接口。由于Set接口没有除Collction接口提供的方法以外的新抽象方法，故Set接口的大部分实现有其父类AbstractCollection提供，AbstractSet只实现了removeAll()，equlas()与 hashCode()方法。