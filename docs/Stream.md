# Stream Pipelines源码分析

## **Stream使用**

Java 8引入函数式编程的原因有二:

1. **代码简洁**: 函数式编程写出的代码简洁且意图明确，使用*stream*接口让你从此告别*for*循环。
2. **多核友好**: Java函数式编程使得编写并行程序从未如此简单，你需要的全部就是调用一下`parallel()`方法。

*stream*并不是某种数据结构，它只是数据源的一种视图。这里的数据源可以是一个数组，Java容器或I/O channel等。正因如此要得到一个*stream*通常不会手动创建，而是调用对应的工具方法，比如:

- 调用`Collection.stream()`或者`Collection.parallelStream()`方法
- 调用`Arrays.stream(T[] array)`方法

常见的*stream*接口继承关系如图:

![java_stream_interfaces](/images/Java_stream_Interfaces.png)

图中4种*stream*接口继承自`BaseStream`, 其中`IntStream, LongStream, DoubleStream`对应三种基本类型(`int, long, double`，注意不是包装类型)，`Stream`对应所有剩余类型的*stream*视图。为不同数据类型设置不同*stream*接口，可以1.提高性能，2.增加特定接口函数。


![WRONG_Java_stream_Interfaces](/images/WRONG_Java_stream_Interfaces.png)


你可能会奇怪为什么不把`IntStream`等设计成`Stream`的子接口？毕竟这接口中的方法名大部分是一样的。答案是这些方法的名字虽然相同，但是返回类型不同，如果设计成父子接口关系，这些方法将不能共存，因为Java不允许只有返回类型不同的方法重载。

虽然大部分情况下*stream*是容器调用`Collection.stream()`方法得到的，但*stream*和*collections*有以下不同：

- **无存储**。*stream*不是一种数据结构，它只是某种数据源的一个视图，数据源可以是一个数组，Java容器或I/O channel等。
- **为函数式编程而生**。对*stream*的任何修改都不会修改背后的数据源，比如对*stream*执行过滤操作并不会删除被过滤的元素，而是会产生一个不包含被过滤元素的新*stream*。
- **惰式执行**。*stream*上的操作并不会立即执行，只有等到用户真正需要结果的时候才会执行。
- **可消费性**。*stream*只能被“消费”一次，一旦遍历过就会失效，就像容器的迭代器那样，想要再次遍历必须重新生成。

对*stream*的操作分为为两类，**中间操作(*intermediate operations*)和结束操作(*terminal operations*)**，二者特点是：

1. __中间操作总是会惰式执行__，调用中间操作只会生成一个标记了该操作的新*stream*，仅此而已。
2. __结束操作会触发实际计算__，计算发生时会把所有中间操作积攒的操作以*pipeline*的方式执行，这样可以减少迭代次数。计算完成之后*stream*就会失效。

如果你熟悉Apache Spark RDD，对*stream*的这个特点应该不陌生。

下表汇总了`Stream`接口的部分常见方法：

|操作类型|接口方法|
|--------|--------|
|中间操作|concat() distinct() filter() flatMap() limit() map() peek() <br> skip() sorted() parallel() sequential() unordered()|
|结束操作|allMatch() anyMatch() collect() count() findAny() findFirst() <br> forEach() forEachOrdered() max() min() noneMatch() reduce() toArray()|

区分中间操作和结束操作最简单的方法，就是看方法的返回值，返回值为*stream*的大都是中间操作，否则是结束操作。

## stream方法使用

*stream*跟函数接口关系非常紧密，没有函数接口*stream*就无法工作。回顾一下：__函数接口是指内部只有一个抽象方法的接口__。通常函数接口出现的地方都可以使用Lambda表达式，所以不必记忆函数接口的名字。

### forEach()

我们对`forEach()`方法并不陌生，在`Collection`中我们已经见过。方法签名为`void forEach(Consumer<? super E> action)`，作用是对容器中的每个元素执行`action`指定的动作，也就是对元素进行遍历。

```Java
// 使用Stream.forEach()迭代
Stream<String> stream = Stream.of("I", "love", "you", "too");
stream.forEach(str -> System.out.println(str));
```
由于`forEach()`是结束方法，上述代码会立即执行，输出所有字符串。

### filter()
![Stream filter](/images/Stream.filter.png)

函数原型为`Stream<T> filter(Predicate<? super T> predicate)`，作用是返回一个只包含满足`predicate`条件元素的`Stream`。

```Java
// 保留长度等于3的字符串
Stream<String> stream= Stream.of("I", "love", "you", "too");
stream.filter(str -> str.length()==3)
    .forEach(str -> System.out.println(str));
```

上述代码将输出为长度等于3的字符串`you`和`too`。注意，由于`filter()`是个中间操作，如果只调用`filter()`不会有实际计算，因此也不会输出任何信息。

### distinct()

![Stream distinct](/images/Stream.distinct.png)

函数原型为`Stream<T> distinct()`，作用是返回一个去除重复元素之后的`Stream`。

```Java
Stream<String> stream= Stream.of("I", "love", "you", "too", "too");
stream.distinct()
    .forEach(str -> System.out.println(str));
```

上述代码会输出去掉一个`too`之后的其余字符串。

<br><br>

### sorted()

排序函数有两个，一个是用自然顺序排序，一个是使用自定义比较器排序，函数原型分别为`Stream<T>　sorted()`和`Stream<T>　sorted(Comparator<? super T> comparator)`。

```Java
Stream<String> stream= Stream.of("I", "love", "you", "too");
stream.sorted((str1, str2) -> str1.length()-str2.length())
    .forEach(str -> System.out.println(str));
```

上述代码将输出按照长度升序排序后的字符串，结果完全在预料之中。

### map()

![Stream map](/images/Stream.map.png)

函数原型为`<R> Stream<R> map(Function<? super T,? extends R> mapper)`，作用是返回一个对当前所有元素执行执行`mapper`之后的结果组成的`Stream`。直观的说，就是对每个元素按照某种操作进行转换，转换前后`Stream`中元素的个数不会改变，但元素的类型取决于转换之后的类型。

```Java
Stream<String> stream　= Stream.of("I", "love", "you", "too");
stream.map(str -> str.toUpperCase())
    .forEach(str -> System.out.println(str));
```
上述代码将输出原字符串的大写形式。

### flatMap()

![Stream flatMap](/images/Stream.flatMap.png)

函数原型为`<R> Stream<R> flatMap(Function<? super T,? extends Stream<? extends R>> mapper)`，作用是对每个元素执行`mapper`指定的操作，并用所有`mapper`返回的`Stream`中的元素组成一个新的`Stream`作为最终返回结果。说起来太拗口，通俗的讲`flatMap()`的作用就相当于把原*stream*中的所有元素都"摊平"之后组成的`Stream`，转换前后元素的个数和类型都可能会改变。

```Java
Stream<List<Integer>> stream = Stream.of(Arrays.asList(1,2), Arrays.asList(3, 4, 5));
stream.flatMap(list -> list.stream())
    .forEach(i -> System.out.println(i));
```

上述代码中，原来的`stream`中有两个元素，分别是两个`List<Integer>`，执行`flatMap()`之后，将每个`List`都“摊平”成了一个个的数字，所以会新产生一个由5个数字组成的`Stream`。所以最终将输出1~5这5个数字。



## **Streams规约操作**

规约操作（*reduction operation*）又被称作折叠操作（*fold*），是通过某个连接动作将所有元素汇总成一个汇总结果的过程。元素求和、求最大值或最小值、求出元素总个数、将所有元素转换成一个列表或集合，都属于规约操作。*Stream*类库有两个通用的规约操作`reduce()`和`collect()`，也有一些为简化书写而设计的专用规约操作，比如`sum()`、`max()`、`min()`、`count()`等。

最大或最小值这类规约操作很好理解（至少方法语义上是这样），我们着重介绍`reduce()`和`collect()`，这是比较有魔法的地方。

## 多面手reduce()

*reduce*操作可以实现从一组元素中生成一个值，`sum()`、`max()`、`min()`、`count()`等都是*reduce*操作，将他们单独设为函数只是因为常用。`reduce()`的方法定义有三种重写形式：

- `Optional<T> reduce(BinaryOperator<T> accumulator)`
- `T reduce(T identity, BinaryOperator<T> accumulator)`
- `<U> U reduce(U identity, BiFunction<U,? super T,U> accumulator, BinaryOperator<U> combiner)`

虽然函数定义越来越长，但语义不曾改变，多的参数只是为了指明初始值（参数*identity*），或者是指定并行执行时多个部分结果的合并方式（参数*combiner*）。`reduce()`最常用的场景就是从一堆值中生成一个值。用这么复杂的函数去求一个最大或最小值，你是不是觉得设计者有病。其实不然，因为“大”和“小”或者“求和"有时会有不同的语义。

需求：*从一组单词中找出最长的单词*。这里“大”的含义就是“长”。
```Java
// 找出最长的单词
Stream<String> stream = Stream.of("I", "love", "you", "too");
Optional<String> longest = stream.reduce((s1, s2) -> s1.length()>=s2.length() ? s1 : s2);
//Optional<String> longest = stream.max((s1, s2) -> s1.length()-s2.length());
System.out.println(longest.get());
```
上述代码会选出最长的单词*love*，其中*Optional*是（一个）值的容器，使用它可以避免*null*值的麻烦。当然可以使用`Stream.max(Comparator<? super T> comparator)`方法来达到同等效果，但`reduce()`自有其存在的理由。

<img src="./Figures/Stream.reduce_parameter.png"  width="400px" align="right" alt="Stream.reduce_parameter"/>

需求：*求出一组单词的长度之和*。这是个“求和”操作，操作对象输入类型是*String*，而结果类型是*Integer*。
```Java
// 求单词长度之和
Stream<String> stream = Stream.of("I", "love", "you", "too");
Integer lengthSum = stream.reduce(0,　// 初始值　// (1)
        (sum, str) -> sum+str.length(), // 累加器 // (2)
        (a, b) -> a+b);　// 部分和拼接器，并行执行时才会用到 // (3)
// int lengthSum = stream.mapToInt(str -> str.length()).sum();
System.out.println(lengthSum);
```
上述代码标号(2)处将i. 字符串映射成长度，ii. 并和当前累加和相加。这显然是两步操作，使用`reduce()`函数将这两步合二为一，更有助于提升性能。如果想要使用`map()`和`sum()`组合来达到上述目的，也是可以的。

`reduce()`擅长的是生成一个值，如果想要从*Stream*生成一个集合或者*Map*等复杂的对象该怎么办呢？终极武器`collect()`横空出世！

## >>> 终极武器collect() <<<

不夸张的讲，如果你发现某个功能在*Stream*接口中没找到，十有八九可以通过`collect()`方法实现。`collect()`是*Stream*接口方法中最灵活的一个，学会它才算真正入门Java函数式编程。先看几个热身的小例子：
```Java
// 将Stream转换成容器或Map
Stream<String> stream = Stream.of("I", "love", "you", "too");
List<String> list = stream.collect(Collectors.toList()); // (1)
// Set<String> set = stream.collect(Collectors.toSet()); // (2)
// Map<String, Integer> map = stream.collect(Collectors.toMap(Function.identity(), String::length)); // (3)
```
上述代码分别列举了如何将*Stream*转换成*List*、*Set*和*Map*。虽然代码语义很明确，可是我们仍然会有几个疑问：

1. `Function.identity()`是干什么的？
2. `String::length`是什么意思？
3. *Collectors*是个什么东西？

## 接口的静态方法和默认方法

*Function*是一个接口，那么`Function.identity()`是什么意思呢？这要从两方面解释：

1. Java 8允许在接口中加入具体方法。接口中的具体方法有两种，*default*方法和*static*方法，`identity()`就是*Function*接口的一个静态方法。
2. `Function.identity()`返回一个输出跟输入一样的Lambda表达式对象，等价于形如`t -> t`形式的Lambda表达式。

上面的解释是不是让你疑问更多？不要问我为什么接口中可以有具体方法，也不要告诉我你觉得`t -> t`比`identity()`方法更直观。我会告诉你接口中的*default*方法是一个无奈之举，在Java 7及之前要想在定义好的接口中加入新的抽象方法是很困难甚至不可能的，因为所有实现了该接口的类都要重新实现。试想在*Collection*接口中加入一个`stream()`抽象方法会怎样？*default*方法就是用来解决这个尴尬问题的，直接在接口中实现新加入的方法。既然已经引入了*default*方法，为何不再加入*static*方法来避免专门的工具类呢！

## 方法引用

诸如`String::length`的语法形式叫做方法引用（*method references*），这种语法用来替代某些特定形式Lambda表达式。如果Lambda表达式的全部内容就是调用一个已有的方法，那么可以用方法引用来替代Lambda表达式。方法引用可以细分为四类：

| 方法引用类别 | 举例 |
|--------|--------|
| 引用静态方法 | `Integer::sum` |
| 引用某个对象的方法 | `list::add` |
| 引用某个类的方法 | `String::length` |
| 引用构造方法 | `HashMap::new` |

我们会在后面的例子中使用方法引用。

## 收集器

相信前面繁琐的内容已彻底打消了你学习Java函数式编程的热情，不过很遗憾，下面的内容更繁琐。但这不能怪Stream类库，因为要实现的功能本身很复杂。

![stream_collect](/images/Stream.collect_parameter.png)

收集器（*Collector*）是为`Stream.collect()`方法量身打造的工具接口（类）。考虑一下将一个*Stream*转换成一个容器（或者*Map*）需要做哪些工作？我们至少需要两样东西：

1. 目标容器是什么？是*ArrayList*还是*HashSet*，或者是个*TreeMap*。
2. 新元素如何添加到容器中？是`List.add()`还是`Map.put()`。

如果并行的进行规约，还需要告诉*collect()* 3. 多个部分结果如何合并成一个。

结合以上分析，*collect()*方法定义为`<R> R collect(Supplier<R> supplier, BiConsumer<R,? super T> accumulator, BiConsumer<R,R> combiner)`，三个参数依次对应上述三条分析。不过每次调用*collect()*都要传入这三个参数太麻烦，收集器*Collector*就是对这三个参数的简单封装,所以*collect()*的另一定义为`<R,A> R collect(Collector<? super T,A,R> collector)`。*Collectors*工具类可通过静态方法生成各种常用的*Collector*。举例来说，如果要将*Stream*规约成*List*可以通过如下两种方式实现：
```Java
//　将Stream规约成List
Stream<String> stream = Stream.of("I", "love", "you", "too");
List<String> list = stream.collect(ArrayList::new, ArrayList::add, ArrayList::addAll);// 方式１
//List<String> list = stream.collect(Collectors.toList());// 方式2
System.out.println(list);
```
通常情况下我们不需要手动指定*collect()*的三个参数，而是调用`collect(Collector<? super T,A,R> collector)`方法，并且参数中的*Collector*对象大都是直接通过*Collectors*工具类获得。实际上传入的**收集器的行为决定了`collect()`的行为**。

## 使用collect()生成Collection

前面已经提到通过`collect()`方法将*Stream*转换成容器的方法，这里再汇总一下。将*Stream*转换成*List*或*Set*是比较常见的操作，所以*Collectors*工具已经为我们提供了对应的收集器，通过如下代码即可完成：
```Java
// 将Stream转换成List或Set
Stream<String> stream = Stream.of("I", "love", "you", "too");
List<String> list = stream.collect(Collectors.toList()); // (1)
Set<String> set = stream.collect(Collectors.toSet()); // (2)
```
上述代码能够满足大部分需求，但由于返回结果是接口类型，我们并不知道类库实际选择的容器类型是什么，有时候我们可能会想要人为指定容器的实际类型，这个需求可通过`Collectors.toCollection(Supplier<C> collectionFactory)`方法完成。
```Java
// 使用toCollection()指定规约容器的类型
ArrayList<String> arrayList = stream.collect(Collectors.toCollection(ArrayList::new));// (3)
HashSet<String> hashSet = stream.collect(Collectors.toCollection(HashSet::new));// (4)
```
上述代码(3)处指定规约结果是*ArrayList*，而(4)处指定规约结果为*HashSet*。一切如你所愿。

## 使用collect()生成Map

前面已经说过*Stream*背后依赖于某种数据源，数据源可以是数组、容器等，但不能是*Map*。反过来从*Stream*生成*Map*是可以的，但我们要想清楚*Map*的*key*和*value*分别代表什么，根本原因是我们要想清楚要干什么。通常在三种情况下`collect()`的结果会是*Map*：

1. 使用`Collectors.toMap()`生成的收集器，用户需要指定如何生成*Map*的*key*和*value*。
2. 使用`Collectors.partitioningBy()`生成的收集器，对元素进行二分区操作时用到。
3. 使用`Collectors.groupingBy()`生成的收集器，对元素做*group*操作时用到。

情况1：使用`toMap()`生成的收集器，这种情况是最直接的，前面例子中已提到，这是和`Collectors.toCollection()`并列的方法。如下代码展示将学生列表转换成由<学生，GPA>组成的*Map*。非常直观，无需多言。
```Java
// 使用toMap()统计学生GPA
Map<Student, Double> studentToGPA =
     students.stream().collect(Collectors.toMap(Function.identity(),// 如何生成key
                                     student -> computeGPA(student)));// 如何生成value
```
情况2：使用`partitioningBy()`生成的收集器，这种情况适用于将`Stream`中的元素依据某个二值逻辑（满足条件，或不满足）分成互补相交的两部分，比如男女性别、成绩及格与否等。下列代码展示将学生分成成绩及格或不及格的两部分。
```Java
// Partition students into passing and failing
Map<Boolean, List<Student>> passingFailing = students.stream()
         .collect(Collectors.partitioningBy(s -> s.getGrade() >= PASS_THRESHOLD));
```
情况3：使用`groupingBy()`生成的收集器，这是比较灵活的一种情况。跟SQL中的*group by*语句类似，这里的*groupingBy()*也是按照某个属性对数据进行分组，属性相同的元素会被对应到*Map*的同一个*key*上。下列代码展示将员工按照部门进行分组：
```Java
// Group employees by department
Map<Department, List<Employee>> byDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::getDepartment));
```
以上只是分组的最基本用法，有些时候仅仅分组是不够的。在SQL中使用*group by*是为了协助其他查询，比如*1. 先将员工按照部门分组，2. 然后统计每个部门员工的人数*。Java类库设计者也考虑到了这种情况，增强版的`groupingBy()`能够满足这种需求。增强版的`groupingBy()`允许我们对元素分组之后再执行某种运算，比如求和、计数、平均值、类型转换等。这种先将元素分组的收集器叫做**上游收集器**，之后执行其他运算的收集器叫做**下游收集器**(*downstream Collector*)。
```Java
// 使用下游收集器统计每个部门的人数
Map<Department, Integer> totalByDept = employees.stream()
                    .collect(Collectors.groupingBy(Employee::getDepartment,
                                                   Collectors.counting()));// 下游收集器
```
上面代码的逻辑是不是越看越像SQL？高度非结构化。还有更狠的，下游收集器还可以包含更下游的收集器，这绝不是为了炫技而增加的把戏，而是实际场景需要。考虑将员工按照部门分组的场景，如果*我们想得到每个员工的名字（字符串），而不是一个个*Employee*对象*，可通过如下方式做到：
```Java
// 按照部门对员工分布组，并只保留员工的名字
Map<Department, List<String>> byDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment,
                        Collectors.mapping(Employee::getName,// 下游收集器
                                Collectors.toList())));// 更下游的收集器
```
如果看到这里你还没有对Java函数式编程失去信心，恭喜你，你已经顺利成为Java函数式编程大师了。

## 使用collect()做字符串join

这个肯定是大家喜闻乐见的功能，字符串拼接时使用`Collectors.joining()`生成的收集器，从此告别*for*循环。`Collectors.joining()`方法有三种重写形式，分别对应三种不同的拼接方式。无需多言，代码过目难忘。
```Java
// 使用Collectors.joining()拼接字符串
Stream<String> stream = Stream.of("I", "love", "you");
//String joined = stream.collect(Collectors.joining());// "Iloveyou"
//String joined = stream.collect(Collectors.joining(","));// "I,love,you"
String joined = stream.collect(Collectors.joining(",", "{", "}"));// "{I,love,you}"
```
## collect()还可以做更多

除了可以使用*Collectors*工具类已经封装好的收集器，我们还可以自定义收集器，或者直接调用`collect(Supplier<R> supplier, BiConsumer<R,? super T> accumulator, BiConsumer<R,R> combiner)`方法，**收集任何形式你想要的信息**。不过*Collectors*工具类应该能满足我们的绝大部分需求，手动实现之间请先看看文档。


## **Stream源码分析**

上面提到的都是如何使用Stream API，但简洁的方法下面似乎隐藏着无尽的秘密，如此强大的API是如何实现的呢？比如Pipeline是怎么执行的，每次方法调用都会导致一次迭代吗？自动并行又是怎么做到的，线程个数是多少？本节我们学习Stream流水线的原理，这是Stream实现的关键所在。

首先回顾一下容器执行Lambda表达式的方式，以`ArrayList.forEach()`方法为例，具体代码如下:

```Java
// ArrayList.forEach()
public void forEach(Consumer<? super E> action) {
    ...
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        action.accept(elementData[i]);// 回调方法
    }
    ...
}
```

我们看到`ArrayList.forEach()`方法的主要逻辑就是一个*for*循环，在该*for*循环里不断调用`action.accept()`回调方法完成对元素的遍历。这完全没有什么新奇之处，回调方法在Java GUI的监听器中广泛使用。Lambda表达式的作用就是相当于一个回调方法，这很好理解。

Stream API中大量使用Lambda表达式作为回调方法，但这并不是关键。理解Stream我们更关心的是另外两个问题：流水线和自动并行。使用Stream或许很容易写入如下形式的代码：

```Java
int longestStringLengthStartingWithA
        = strings.stream()
              .filter(s -> s.startsWith("A"))
              .mapToInt(String::length)
              .max();
```

上述代码求出以字母*A*开头的字符串的最大长度，一种直白的方式是为每一次函数调用都执一次迭代，这样做能够实现功能，但效率上肯定是无法接受的。类库的实现着使用流水线（*Pipeline*）的方式巧妙的避免了多次迭代，其基本思想是在一次迭代中尽可能多的执行用户指定的操作。为讲解方便我们汇总了Stream的所有操作。

<table width="600"><tr><td colspan="3" align="center"  border="0">Stream操作分类</td></tr><tr><td rowspan="2"  border="1">中间操作(Intermediate operations)</td><td>无状态(Stateless)</td><td>unordered() filter() map() mapToInt() mapToLong() mapToDouble() flatMap() flatMapToInt() flatMapToLong() flatMapToDouble() peek()</td></tr><tr><td>有状态(Stateful)</td><td>distinct() sorted() sorted() limit() skip() </td></tr><tr><td rowspan="2"  border="1">结束操作(Terminal operations)</td><td>非短路操作</td><td>forEach() forEachOrdered() toArray() reduce() collect() max() min() count()</td></tr><tr><td>短路操作(short-circuiting)</td><td>anyMatch() allMatch() noneMatch() findFirst() findAny()</td></tr></table>

Stream上的所有操作分为两类：中间操作和结束操作，中间操作只是一种标记，只有结束操作才会触发实际计算。中间操作又可以分为无状态的(*Stateless*)和有状态的(*Stateful*)，无状态中间操作是指元素的处理不受前面元素的影响，而有状态的中间操作必须等到所有元素处理之后才知道最终结果，比如排序是有状态操作，在读取所有元素之前并不能确定排序结果；结束操作又可以分为短路操作和非短路操作，短路操作是指不用处理全部元素就可以返回结果，比如*找到第一个满足条件的元素*。之所以要进行如此精细的划分，是因为底层对每一种情况的处理方式不同。
为了更好的理解流的中间操作和终端操作，可以通过下面的两段代码来看他们的执行过程。
```Java
IntStream.range(1, 10)
   .peek(x -> System.out.print("\nA" + x))
   .limit(3)
   .peek(x -> System.out.print("B" + x))
   .forEach(x -> System.out.print("C" + x));
```
输出为:
A1B1C1
A2B2C2
A3B3C3

中间操作是懒惰的，也就是中间操作不会对数据做任何操作，只是创建pipeline的某个stage以及包装在stage内的Sink, sink内包含数据通过当前stage时进行的操作. 
pipeline中所有stage内包含的sink的操作只有到最后的终结操作才会启动, pipeline会将所有stage内的sink连接起来并找到最开始的sink的操作并开启pipeline的
所有操作, 如下图所示:

![stream_pipeline_stages](/images/stream_pipeline_stages.png)



## 一种直白的实现方式

<img src="./Figures/Stream_pipeline_naive.png"  width="500px" align="right" alt="Stream_pipeline_naive"/>

仍然考虑上述求最长字符串的程序，一种直白的流水线实现方式是为每一次函数调用都执一次迭代，并将处理中间结果放到某种数据结构中（比如数组，容器等）。具体说来，就是调用`filter()`方法后立即执行，选出所有以*A*开头的字符串并放到一个列表list1中，之后让list1传递给`mapToInt()`方法并立即执行，生成的结果放到list2中，最后遍历list2找出最大的数字作为最终结果。程序的执行流程如如所示：

这样做实现起来非常简单直观，但有两个明显的弊端：

1. 迭代次数多。迭代次数跟函数调用的次数相等。
2. 频繁产生中间结果。每次函数调用都产生一次中间结果，存储开销无法接受。

这些弊端使得效率底下，根本无法接受。如果不使用Stream API我们都知道上述代码该如何在一次迭代中完成，大致是如下形式：

```Java
int longest = 0;
for(String str : strings){
    if(str.startsWith("A")){// 1. filter(), 保留以A开头的字符串
        int len = str.length();// 2. mapToInt(), 转换成长度
        longest = Math.max(len, longest);// 3. max(), 保留最长的长度
    }
}
```

采用这种方式我们不但减少了迭代次数，也避免了存储中间结果，显然这就是流水线，因为我们把三个操作放在了一次迭代当中。只要我们事先知道用户意图，总是能够采用上述方式实现跟Stream API等价的功能，但问题是Stream类库的设计者并不知道用户的意图是什么。如何在无法假设用户行为的前提下实现流水线，是类库的设计者要考虑的问题。

## Stream流水线解决方案

我们大致能够想到，应该采用某种方式记录用户每一步的操作，当用户调用结束操作时将之前记录的操作叠加到一起在一次迭代中全部执行掉。沿着这个思路，有几个问题需要解决：

1. 用户的操作如何记录？
2. 操作如何叠加？
3. 叠加之后的操作如何执行？
4. 执行后的结果（如果有）在哪里？

### >> 操作如何记录

![Java_stream_pipeline_classes](/images/Java_stream_pipeline_classes.png)

注意这里使用的是“*操作(operation)*”一词，指的是“Stream中间操作”的操作，很多Stream操作会需要一个回调函数（Lambda表达式），因此一个完整的操作是<*数据来源，操作，回调函数*>构成的三元组。Stream中使用Stage的概念来描述一个完整的操作，并用某种实例化后的*PipelineHelper*来代表Stage，将具有先后顺序的各个Stage连到一起，就构成了整个流水线。跟Stream相关类和接口的继承关系图示。

还有*IntPipeline, LongPipeline, DoublePipeline*没在图中画出，这三个类专门为三种基本类型（不是包装类型）而定制的，跟*ReferencePipeline*是并列关系。图中*Head*用于表示第一个Stage，即调用调用诸如*Collection.stream()*方法产生的Stage，很显然这个Stage里不包含任何操作；*StatelessOp*和*StatefulOp*分别表示无状态和有状态的Stage，对应于无状态和有状态的中间操作。

Stream流水线组织结构示意图如下：

![Stream_pipeline_example](/images/Stream_pipeline_example.png)

图中通过`Collection.stream()`方法得到*Head*也就是stage0，紧接着调用一系列的中间操作，不断产生新的Stream。**这些Stream对象以双向链表的形式组织在一起，构成整个流水线，由于每个Stage都记录了前一个Stage和本次的操作以及回调函数，依靠这种结构就能建立起对数据源的所有操作**。这就是Stream记录操作的方式。

### >> 操作如何叠加

以上只是解决了操作记录的问题，要想让流水线起到应有的作用我们需要一种将所有操作叠加到一起的方案。你可能会觉得这很简单，只需要从流水线的head开始依次执行每一步的操作（包括回调函数）就行了。这听起来似乎是可行的，但是你忽略了前面的Stage并不知道后面Stage到底执行了哪种操作，以及回调函数是哪种形式。换句话说，只有当前Stage本身才知道该如何执行自己包含的动作。这就需要有某种协议来协调相邻Stage之间的调用关系。

这种协议由*Sink*接口完成，*Sink*接口包含的方法如下表所示：

<table width="600px"><tr><td align="center">方法名</td><td align="center">作用</td></tr><tr><td>void begin(long size)</td><td>开始遍历元素之前调用该方法，通知Sink做好准备。</td></tr><tr><td>void end()</td><td>所有元素遍历完成之后调用，通知Sink没有更多的元素了。</td></tr><tr><td>boolean cancellationRequested()</td><td>是否可以结束操作，可以让短路操作尽早结束。</td></tr><tr><td>void accept(T t)</td><td>遍历元素时调用，接受一个待处理元素，并对元素进行处理。Stage把自己包含的操作和回调方法封装到该方法里，前一个Stage只需要调用当前Stage.accept(T t)方法就行了。</td></tr></table>

有了上面的协议，相邻Stage之间调用就很方便了，每个Stage都会将自己的操作封装到一个Sink里，前一个Stage只需调用后一个Stage的`accept()`方法即可，并不需要知道其内部是如何处理的。当然对于有状态的操作，Sink的`begin()`和`end()`方法也是必须实现的。比如Stream.sorted()是一个有状态的中间操作，其对应的Sink.begin()方法可能创建一个盛放结果的容器，而accept()方法负责将元素添加到该容器，最后end()负责对容器进行排序。对于短路操作，`Sink.cancellationRequested()`也是必须实现的，比如Stream.findFirst()是短路操作，只要找到一个元素，cancellationRequested()就应该返回*true*，以便调用者尽快结束查找。Sink的四个接口方法常常相互协作，共同完成计算任务。**实际上Stream API内部实现的的本质，就是如何重写Sink的这四个接口方法**。

有了Sink对操作的包装，Stage之间的调用问题就解决了，执行时只需要从流水线的head开始对数据源依次调用每个Stage对应的Sink.{begin(), accept(), cancellationRequested(), end()}方法就可以了。一种可能的Sink.accept()方法流程是这样的：

```Java
void accept(U u){
    1. 使用当前Sink包装的回调函数处理u
    2. 将处理结果传递给流水线下游的Sink
}
```

Sink接口的其他几个方法也是按照这种[处理->转发]的模型实现。下面我们结合具体例子看看Stream的中间操作是如何将自身的操作包装成Sink以及Sink是如何将处理结果转发给下一个Sink的。先看Stream.map()方法：

```Java
// Stream.map()，调用该方法将产生一个新的Stream
public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
    ...
    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                 StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
        @Override /*opWripSink()方法返回由回调函数包装而成Sink*/
        Sink<P_OUT> opWrapSink(int flags, Sink<R> downstream) {
            return new Sink.ChainedReference<P_OUT, R>(downstream) {
                @Override
                public void accept(P_OUT u) {
                    R r = mapper.apply(u);// 1. 使用当前Sink包装的回调函数mapper处理u
                    downstream.accept(r);// 2. 将处理结果传递给流水线下游的Sink
                }
            };
        }
    };
}
```

上述代码看似复杂，其实逻辑很简单，就是将回调函数*mapper*包装到一个Sink当中。由于Stream.map()是一个无状态的中间操作，所以map()方法返回了一个StatelessOp内部类对象（一个新的Stream），调用这个新Stream的opWripSink()方法将得到一个包装了当前回调函数的Sink。

再来看一个复杂一点的例子。Stream.sorted()方法将对Stream中的元素进行排序，显然这是一个有状态的中间操作，因为读取所有元素之前是没法得到最终顺序的。抛开模板代码直接进入问题本质，sorted()方法是如何将操作封装成Sink的呢？sorted()一种可能封装的Sink代码如下：

```Java
// Stream.sort()方法用到的Sink实现
class RefSortingSink<T> extends AbstractRefSortingSink<T> {
    private ArrayList<T> list;// 存放用于排序的元素
    RefSortingSink(Sink<? super T> downstream, Comparator<? super T> comparator) {
        super(downstream, comparator);
    }
    @Override
    public void begin(long size) {
        ...
        // 创建一个存放排序元素的列表
        list = (size >= 0) ? new ArrayList<T>((int) size) : new ArrayList<T>();
    }
    @Override
    public void end() {
        list.sort(comparator);// 只有元素全部接收之后才能开始排序
        downstream.begin(list.size());
        if (!cancellationWasRequested) {// 下游Sink不包含短路操作
            list.forEach(downstream::accept);// 2. 将处理结果传递给流水线下游的Sink
        }
        else {// 下游Sink包含短路操作
            for (T t : list) {// 每次都调用cancellationRequested()询问是否可以结束处理。
                if (downstream.cancellationRequested()) break;
                downstream.accept(t);// 2. 将处理结果传递给流水线下游的Sink
            }
        }
        downstream.end();
        list = null;
    }
    @Override
    public void accept(T t) {
        list.add(t);// 1. 使用当前Sink包装动作处理t，只是简单的将元素添加到中间列表当中
    }
}
```

上述代码完美的展现了Sink的四个接口方法是如何协同工作的：
1. 首先begin()方法告诉Sink参与排序的元素个数，方便确定中间结果容器的的大小；
2. 之后通过accept()方法将元素添加到中间结果当中，最终执行时调用者会不断调用该方法，直到遍历所有元素；
3. 最后end()方法告诉Sink所有元素遍历完毕，启动排序步骤，排序完成后将结果传递给下游的Sink；
4. 如果下游的Sink是短路操作，将结果传递给下游时不断询问下游cancellationRequested()是否可以结束处理。

### >> 叠加之后的操作如何执行

![Stream_pipeline_Sink](/images/Stream_pipeline_Sink.png)

Sink完美封装了Stream每一步操作，并给出了[处理->转发]的模式来叠加操作。这一连串的齿轮已经咬合，就差最后一步拨动齿轮启动执行。是什么启动这一连串的操作呢？也许你已经想到了启动的原始动力就是结束操作(Terminal Operation)，一旦调用某个结束操作，就会触发整个流水线的执行。

结束操作之后不能再有别的操作，所以结束操作不会创建新的流水线阶段(Stage)，直观的说就是流水线的链表不会在往后延伸了。结束操作会创建一个包装了自己操作的Sink，这也是流水线中最后一个Sink，这个Sink只需要处理数据而不需要将结果传递给下游的Sink（因为没有下游）。对于Sink的[处理->转发]模型，结束操作的Sink就是调用链的出口。

我们再来考察一下上游的Sink是如何找到下游Sink的。一种可选的方案是在*PipelineHelper*中设置一个Sink字段，在流水线中找到下游Stage并访问Sink字段即可。但Stream类库的设计者没有这么做，而是设置了一个`Sink AbstractPipeline.opWrapSink(int flags, Sink downstream)`方法来得到Sink，该方法的作用是返回一个新的包含了当前Stage代表的操作以及能够将结果传递给downstream的Sink对象。为什么要产生一个新对象而不是返回一个Sink字段？这是因为使用opWrapSink()可以将当前操作与下游Sink（上文中的downstream参数）结合成新Sink。试想只要从流水线的最后一个Stage开始，不断调用上一个Stage的opWrapSink()方法直到最开始（不包括stage0，因为stage0代表数据源，不包含操作），就可以得到一个代表了流水线上所有操作的Sink，用代码表示就是这样:

```Java
// AbstractPipeline.wrapSink()
// 从下游向上游不断包装Sink。如果最初传入的sink代表结束操作，
// 函数返回时就可以得到一个代表了流水线上所有操作的Sink。
final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
    ...
    for (AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
        sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
    }
    return (Sink<P_IN>) sink;
}
```

现在流水线上从开始到结束的所有的操作都被包装到了一个Sink里，执行这个Sink就相当于执行整个流水线，执行Sink的代码如下：

```Java
// AbstractPipeline.copyInto(), 对spliterator代表的数据执行wrappedSink代表的操作。
final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
    ...
    if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
        wrappedSink.begin(spliterator.getExactSizeIfKnown());// 通知开始遍历
        spliterator.forEachRemaining(wrappedSink);// 迭代
        wrappedSink.end();// 通知遍历结束
    }
    ...
}
```

上述代码首先调用wrappedSink.begin()方法告诉Sink数据即将到来，然后调用spliterator.forEachRemaining()方法对数据进行迭代（Spliterator是容器的一种迭代器，[参阅](https://github.com/CarpenterLee/JavaLambdaInternals/blob/master/3-Lambda%20and%20Collections.md#spliterator)），最后调用wrappedSink.end()方法通知Sink数据处理结束。逻辑如此清晰。

### >> 执行后的结果在哪里

最后一个问题是流水线上所有操作都执行后，用户所需要的结果（如果有）在哪里？首先要说明的是不是所有的Stream结束操作都需要返回结果，有些操作只是为了使用其副作用(*Side-effects*)，比如使用`Stream.forEach()`方法将结果打印出来就是常见的使用副作用的场景（事实上，除了打印之外其他场景都应避免使用副作用），对于真正需要返回结果的结束操作结果存在哪里呢？

> 特别说明：副作用不应该被滥用，也许你会觉得在Stream.forEach()里进行元素收集是个不错的选择，就像下面代码中那样，但遗憾的是这样使用的正确性和效率都无法保证，因为Stream可能会并行执行。大多数使用副作用的地方都可以使用[归约操作](./5-Streams%20API(II).md)更安全和有效的完成。

```Java
// 错误的收集方式
ArrayList<String> results = new ArrayList<>();
stream.filter(s -> pattern.matcher(s).matches())
      .forEach(s -> results.add(s));  // Unnecessary use of side-effects!
// 正确的收集方式
List<String>results =
     stream.filter(s -> pattern.matcher(s).matches())
             .collect(Collectors.toList());  // No side-effects!
```

回到流水线执行结果的问题上来，需要返回结果的流水线结果存在哪里呢？这要分不同的情况讨论，下表给出了各种有返回结果的Stream结束操作。

<table width="350px"><tr><td align="center">返回类型</td><td align="center">对应的结束操作</td></tr><tr><td>boolean</td><td>anyMatch() allMatch() noneMatch()</td></tr><tr><td>Optional</td><td>findFirst() findAny()</td></tr><tr><td>归约结果</td><td>reduce() collect()</td></tr><tr><td>数组</td><td>toArray()</td></tr></table>

1. 对于表中返回boolean或者Optional的操作（Optional是存放 一个 值的容器）的操作，由于值返回一个值，只需要在对应的Sink中记录这个值，等到执行结束时返回就可以了。
2. 对于归约操作，最终结果放在用户调用时指定的容器中。collect(), reduce(), max(), min()都是归约操作，虽然max()和min()也是返回一个Optional，但事实上底层是通过调用reduce()方法实现的。
3. 对于返回是数组的情况，毫无疑问的结果会放在数组当中。这么说当然是对的，但在最终返回数组之前，结果其实是存储在一种叫做*Node*的数据结构中的。Node是一种多叉树结构，元素存储在树的叶子当中，并且一个叶子节点可以存放多个元素。这样做是为了并行执行方便。关于Node的具体结构，我们会在下一节探究Stream如何并行执行时给出详细说明。


## 并行流

大家应该已经对Stream有过很多的了解，对其原理及常见使用方法已经也有了一定的认识。流在处理数据进行一些迭代操作的时候确认很方便，但是在执行一些耗时或是占用资源很高的任务时候，串行化的流无法带来速度/性能上的提升，并不能满足我们的需要，通常我们会使用多线程来并行或是分片分解执行任务，而在Stream中也提供了这样的并行方法，那就是使用parallelStream()方法或者是使用stream().parallel()来转化为并行流。开箱即用的并行流的使用看起来如此简单，然后我们就可能会忍不住思考，并行流的实现原理是怎样的？它的使用会给我们带来多大的性能提升？我们可以在什么场景下使用以及使用时应该注意些什么？

首先我们看一下Java 的并行 API 演变历程基本如下:
- 1.0-1.4 中的 java.lang.Thread
- 5.0 中的 java.util.concurrent
- 6.0 中的 Phasers 等
- 7.0 中的 Fork/Join 框架
- 8.0 中的 Lambda

## parallelStream是什么？
先看一下`Collection`接口提供的并行流方法
```java
/**
 * Returns a possibly parallel {@code Stream} with this collection as its
 * source.  It is allowable for this method to return a sequential stream.
 *
 * <p>This method should be overridden when the {@link #spliterator()}
 * method cannot return a spliterator that is {@code IMMUTABLE},
 * {@code CONCURRENT}, or <em>late-binding</em>. (See {@link #spliterator()}
 * for details.)
 *
 * @implSpec
 * The default implementation creates a parallel {@code Stream} from the
 * collection's {@code Spliterator}.
 *
 * @return a possibly parallel {@code Stream} over the elements in this
 * collection
 * @since 1.8
 */
default Stream<E> parallelStream() {
    return StreamSupport.stream(spliterator(), true);
}
```
注意其中的代码注释的返回值 `@return a possibly parallel` 一句说明调用了这个方法，只是可能会返回一个并行的流，流是否能并行执行还受到其他一些条件的约束。
parallelStream其实就是一个并行执行的流，它通过默认的`ForkJoinPool`，**可能**提高你的多线程任务的速度。
引用[Custom thread pool in Java 8 parallel stream](https://stackoverflow.com/questions/21163108/custom-thread-pool-in-java-8-parallel-stream)上面的两段话：
> The parallel streams use the default `ForkJoinPool.commonPool` which [by default has one less threads as you have processors](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html), as returned by `Runtime.getRuntime().availableProcessors()` (This means that parallel streams use all your processors because they also use the main thread)。

做个实验来证明上面这句话的真实性：
```java
public static void main(String[] args) {
    IntStream list = IntStream.range(0, 10);
    Set<Thread> threadSet = new HashSet<>();
    //开始并行执行
    list.parallel().forEach(i -> {
        Thread thread = Thread.currentThread();
        System.err.println("integer：" + i + "，" + "currentThread:" + thread.getName());
        threadSet.add(thread);
    });
    System.out.println("all threads：" + Joiner.on("，").join(threadSet.stream().map(Thread::getName).collect(Collectors.toList())));
}
```
![thread_result_1](/images/thread_result_1.png)

从运行结果里面我们可以很清楚的看到parallelStream同时使用了主线程和`ForkJoinPool.commonPool`创建的线程。
值得说明的是这个运行结果并不是唯一的，实际运行的时候可能会得到多个结果，比如：

![thread_result_1](/images/thread_result_2.png)

甚至你的运行结果里面只有主线程。

来源于java 8 实战的书籍的一段话：
> 并行流内部使用了默认的`ForkJoinPool`（7.2节会进一步讲到分支/合并框架），它默认的线程数量就是你的处理器数量，这个值是由`Runtime.getRuntime().available- Processors()`得到的。 但是你可以通过系统属性`java.util.concurrent.ForkJoinPool.common. parallelism`来改变线程池大小，如下所示： `System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism","12");` 这是一个全局设置，因此它将影响代码中所有的并行流。反过来说，目前还无法专为某个 并行流指定这个值。一般而言，让`ForkJoinPool`的大小等于处理器数量是个不错的默认值， 除非你有很好的理由，否则我们强烈建议你不要修改它。

```java
// 设置全局并行流并发线程数
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "12");
System.out.println(ForkJoinPool.getCommonPoolParallelism());// 输出 12
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "20");
System.out.println(ForkJoinPool.getCommonPoolParallelism());// 输出 12
```
为什么两次的运行结果是一样的呢？上面刚刚说过了这是一个全局设置，`java.util.concurrent.ForkJoinPool.common.parallelism`是final类型的，整个JVM中只允许设置一次。既然默认的并发线程数不能反复修改，那怎么进行不同线程数量的并发测试呢？答案是：`引入ForkJoinPool`
```java
IntStream range = IntStream.range(1, 100000);
// 传入parallelism
new ForkJoinPool(parallelism).submit(() -> range.parallel().forEach(System.out::println)).get();
```
因此，使用parallelStream时需要注意的一点是，**多个parallelStream之间默认使用的是同一个线程池**，所以IO操作尽量不要放进parallelStream中，否则会阻塞其他parallelStream。
> Using a ForkJoinPool and submit for a parallel stream does not reliably use all threads. If you look at this ( [Parallel stream from a HashSet doesn't run in parallel](https://stackoverflow.com/questions/28985704/parallel-stream-from-a-hashset-doesnt-run-in-parallel) ) and this ( [Why does the parallel stream not use all the threads of the ForkJoinPool?](https://stackoverflow.com/questions/36947336/why-does-the-parallel-stream-not-use-all-the-threads-of-the-forkjoinpool) ), you'll see the reasoning.

```java
// 获取当前机器CPU处理器的数量
System.out.println(Runtime.getRuntime().availableProcessors());// 输出 4
// parallelStream默认的并发线程数
System.out.println(ForkJoinPool.getCommonPoolParallelism());// 输出 3
```
为什么parallelStream默认的并发线程数要比CPU处理器的数量少1个？文章的开始已经提过了。因为最优的策略是每个CPU处理器分配一个线程，然而主线程也算一个线程，所以要占一个名额。
这一点可以从源码中看出来：
```java
static final int MAX_CAP      = 0x7fff;        // max #workers - 1
// 无参构造函数
public ForkJoinPool() {
        this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
             defaultForkJoinWorkerThreadFactory, null, false);
}bs-channel
```

## 从parallelStream认识[Fork/Join 框架](https://www.infoq.cn/article/fork-join-introduction/)
Fork/Join 框架的核心是采用分治法的思想，将一个大任务拆分为若干互不依赖的子任务，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务。同时，为了最大限度地提高并行处理能力，采用了工作窃取算法来运行任务，也就是说当某个线程处理完自己工作队列中的任务后，尝试当其他线程的工作队列中窃取一个任务来执行，直到所有任务处理完毕。所以为了减少线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。
- Fork/Join 的运行流程图
![forkjoin_process](/images/forkjoin_process.png)

简单地说就是大任务拆分成小任务，分别用不同线程去完成，然后把结果合并后返回。所以第一步是拆分，第二步是分开运算，第三步是合并。这三个步骤分别对应的就是Collector的*supplier*,*accumulator*和*combiner*。
- 工作窃取算法
Fork/Join最核心的地方就是利用了现代硬件设备多核,在一个操作时候会有空闲的CPU,那么如何利用好这个空闲的cpu就成了提高性能的关键,而这里我们要提到的工作窃取（work-stealing）算法就是整个Fork/Join框架的核心理念,工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。  
![work_steal](/images/work_steal.png)

## 使用parallelStream的利弊
使用parallelStream的几个好处：
1) 代码优雅，可以使用lambda表达式，原本几句代码现在一句可以搞定；
2) 运用多核特性(forkAndJoin)并行处理，大幅提高效率。
关于并行流和多线程的性能测试可以看一下下面的几篇博客：  
[并行流适用场景-CPU密集型](https://blog.csdn.net/larva_s/article/details/90403578)    
[提交订单性能优化系列之006-普通的Thread多线程改为Java8的parallelStream并发流](https://blog.csdn.net/blueskybluesoul/article/details/82817007)

然而，任何事物都不是完美的，并行流也不例外，其中最明显的就是使用(parallel)Stream极其不便于代码的跟踪调试，此外并行流带来的不确定性也使得我们对它的使用变得格外谨慎。我们得去了解更多的并行流的相关知识来保证自己能够正确的使用这把双刃剑。

parallelStream使用时需要注意的点：
1) **parallelStream是线程不安全的；**
```java
List<Integer> values = new ArrayList<>();
IntStream.range(1, 10000).parallel().forEach(values::add);
System.out.println(values.size());
```
values集合大小可能不是10000。集合里面可能会存在null元素或者抛出下标越界的异常信息。  
原因：List不是线程安全的集合，add方法在多线程环境下会存在并发问题。
当执行add方法时，会先将此容器的大小增加。。即size++，然后将传进的元素赋值给新增的`elementData[size++]`，即新的内存空间。但是此时如果在size++后直接来取这个List,而没有让add完成赋值操作，则会导致此List的长度加一，，但是最后一个元素是空（null），所以在获取它进行计算的时候报了空指针异常。而下标越界还不能仅仅依靠这个来解释，如果你观察发生越界时的数组下标，分别为10、15、22、33、49和73。结合前面讲的数组自动机制，数组初始长度为10，第一次扩容为15=10+10/2，第二次扩容22=15+15/2，第三次扩容33=22+22/2...以此类推，我们不难发现，越界异常都发生在数组扩容之时。
`grow()`方法解释了基于数组的ArrayList是如何扩容的。数组进行扩容时，会将老数组中的元素重新拷贝一份到新的数组中，通过`oldCapacity + (oldCapacity >> 1)`运算，每次数组容量的增长大约是其原容量的1.5倍。
 ```java
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);// 1.5倍扩容
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);// 拷贝旧的数组到新的数组中
    }


    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!! 检查array容量
        elementData[size++] = e;// 赋值，增大Size的值
        return true;
    }
```
解决方法：
加锁、使用线程安全的集合或者采用`collect()`或者`reduce()`操作就是满足线程安全的了。
```java
List<Integer> values = new ArrayList<>();
for (int i = 0; i < 10000; i++) {
    values.add(i);
}
List<Integer> collect = values.stream().parallel().collect(Collectors.toList());
System.out.println(collect.size());
``` 
2) parallelStream 适用的场景是CPU密集型的，只是做到别浪费CPU，假如本身电脑CPU的负载很大，那还到处用并行流，那并不能起到作用；
- I/O密集型 磁盘I/O、网络I/O都属于I/O操作，这部分操作是较少消耗CPU资源，一般并行流中不适用于I/O密集型的操作，就比如使用并流行进行大批量的消息推送，涉及到了大量I/O，使用并行流反而慢了很多
- CPU密集型 计算类型就属于CPU密集型了，这种操作并行流就能提高运行效率。

3) 不要在多线程中使用parallelStream，原因同上类似，大家都抢着CPU是没有提升效果，反而还会加大线程切换开销；
4) 会带来不确定性，请确保每条处理无状态且没有关联；
5) 考虑NQ模型：N可用的数据量，Q针对每个数据元素执行的计算量，乘积 N * Q 越大，就越有可能获得并行提速。N * Q>10000（大概是集合大小超过1000） 就会获得有效提升；
6) parallelStream是创建一个并行的Stream,而且它的并行操作是*不具备线程传播性*的,所以是无法获取ThreadLocal创建的线程变量的值；
7) **在使用并行流的时候是无法保证元素的顺序的，也就是即使你用了同步集合也只能保证元素都正确但无法保证其中的顺序**；
8) lambda的执行并不是瞬间完成的，所有使用parallel stream的程序都有可能成为阻塞程序的源头，并且在执行过程中程序中的其他部分将无法访问这些workers，这意味着任何依赖parallel streams的程序在什么别的东西占用着common ForkJoinPool时将会变得不可预知并且暗藏危机。


Note: parallel stream的源码分析可以参考这篇文章: https://juejin.cn/post/7005899099186659335
## 结语

本文属于另外一个github文章的copy的文章, 用于方便自己学习, 原文链接: https://github.com/CarpenterLee/JavaLambdaInternals
