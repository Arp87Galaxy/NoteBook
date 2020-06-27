















# 第一章 并发





## 1.1 并发基础

![image-20200610161303791](C:\Users\Arpgalaxy\AppData\Roaming\Typora\typora-user-images\image-20200610161303791.png)



![img](C:\Users\Arpgalaxy\Desktop\md文件\images\java\jmm)	

![img](C:\Users\Arpgalaxy\Desktop\md文件\images\java\cpum)

## 1.2 volatile关键字

### 1.2.1 原子操作

read --从主内存读取共享数据到线程内存中，供load使用

load --从线程内存中将数据加载到工作内存中

use  --将工作内存中的数据传递给执行引擎，需要使用变量值的字节码指令就会执行这个操作

assign --从执行引擎接收到的值赋给工作内存变量，虚拟机遇到赋值字节码指令会执行这个操作

store --将工作内存数据存入主内存，供write使用

write -- 将store的数据赋给主内存变量

lock --将主内存变量标识为一条线程独占状态

unlock --将lock的主内存变量释放出来，才可以被其他线程锁定

### 1.2.2  三大性质

可见性，有序性，原子性，

**volatile 实现可见性**：通过mesi缓存一致性，在assgin操作时汇编代码中加lock操作，使得use操作缓存失效，线程需要重新执行load和assgin操作，再use操作，使得可见性得到保证。

**有序性实现**：禁止指令重排序，指令重排序是机器级别的优化，没有volatile时，在A线程中，赋值操作

``` 
先

dosth（）函数（执行一些配置加载），

然后

A=true，

两个操作没有关联，那么机器优化后，这两个可能顺序变了，如果只有A线程就没有什么影响，

但是B线程中有if（A==true）{

用到dosth（）里的加载信息

}

如果A线程 A=true先执行了，然后切换B线程，if成立，就会发现没有加载信息，

这就时指令重排导致的错误。
```



**没有实现原子性**：虽然修改了volatile修饰的变量会可见，但是指令在从工作内存写回主内存中间会有线程非原子性发生。

## 1.3 JUC

### 1.3.1集合

 1 ConcurrentHashMap
  基于散列链表+红黑树实现，类似于 HashMap，JDK 8 进行了优化，利用 volatile + CAS 实现无锁化操作，保证线程安全的同时，提高性能。默认容量16，默认加载因子0.75；
  散列链表和红黑树的内部类定义如下：

// 基本结构
static class Node<K,V> implements Map.Entry<K,V> {
	final int hash;
	final K key;
	volatile V val;
	volatile Node<K,V> next;
}
// 红黑树结构，链表长度大于8时转换
static final class TreeNode<K,V> extends Node<K,V> {
	TreeNode<K,V> parent;  // red-black tree links
	TreeNode<K,V> left;
	TreeNode<K,V> right;
	TreeNode<K,V> prev;    // needed to unlink next upon deletion
	boolean red;
}
  和 HashMap 比较，多了内部类 TreeBin，它不存储键值，而是指向 TreeNode 列表和它们的根节点，而 ConcurrentHashMap 也是操作 TreeBin。此外，TreeBin 还维护了读写锁状态，这会使得在树重组之前，持有锁的写操作会等待未持锁的读操作完成。

// 指向TreeNode列表和它们的根节点，
static final class TreeBin<K,V> extends Node<K,V> {
	TreeNode<K,V> root;
	volatile TreeNode<K,V> first;
	volatile Thread waiter;
	volatile int lockState;
	static final int WRITER = 1; // 持有写锁时
	static final int WAITER = 2; // 等待写锁时
	static final int READER = 4; // 用来设置读锁的增量值
}
  如何做到线程安全的呢？

  1. sizeCtl：控制表的初始化和重建。负数表示表正在初始化或者重建：
    -1表示在初始化；
    -(1+N)表示有N个正在进行重建的线程；
  2. 节点哈希值表示的状态
    MOVED=-1 表示 forward 节点；
    TREEBIN=-2 表示 treeBin 的根节点；
  3. V put(K key, V value) 操作
    如果表为空，则初始化表；
    如果hash位置为空，则通过CAS设置值；
    如果hash=-1，则帮组扩容；
    如果节点既不为空，也不等于-1，那么锁住该节点，在链表或者红黑树上添加值；
  4. void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) 扩容
    初始化新表，容量是原表的2倍；
    复制元素到新表，处理一个节点就把节点设置为forward；
    当这个节点为空时，通过CAS来设置forward；
    当节点值为-1时，表示forward已经处理过了；
    当节点不为空且不为-1时，锁住节点进行处理；
    其他线程看到节点为forward时，向后遍历来完成；
  5. V get(Object key) 操作
    如果hash值匹配，则直接获取；
    如果hash值小于0，则从树上查找；
    否则，遍历链表寻找；
  6. mappingCount()（推荐，因为其返回值时long） 和 size()，都是调用 sumCount() 来计算
    定义了内部类 CounterCell 来计数，计算总数时，就是把 CounterCell[] 数组的值累加起来即可；

// put 源码
Node<K,V> f; int n, i, fh;
if (tab == null || (n = tab.length) == 0)
	tab = initTable(); // 表长度为空时，初始化表
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
	if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
		break;                   // hash的位置为空时，通过CAS设置值
}
else if ((fh = f.hash) == MOVED)
	tab = helpTransfer(tab, f); // 如果节点是 forward 节点，帮助扩容
else {
	synchronized (f) { // 不为空，不是 forward 节点时，synchronized 锁住节点
		if (tabAt(tab, i) == f) { // 锁住后再次判断节点有没有变化
			if (fh >= 0) { 
				... // 表示要操作链表节点
			}
			else if (f instanceof TreeBin) {
				... // 表示操作的是TreeBin节点
			}
		}
	}
	if (binCount != 0) {
		if (binCount >= TREEIFY_THRESHOLD)
			treeifyBin(tab, i); // 链表长度大于8，转成红黑树
	}
}

// 并发扩容的方法
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
	int n = tab.length, stride;
	if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
		stride = MIN_TRANSFER_STRIDE; // subdivide range
	if (nextTab == null) {            // 初始化新的表，容量是原表的2倍
		Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
		nextTab = nt;
		nextTable = nextTab; // nextTable 是定义的临时表，仅在表重建时不为空
		transferIndex = n; // 表重建时的下一个表的索引
	}
	int nextn = nextTab.length; // 扩容后的表长度
	ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
	boolean advance = true; // true 表示该节点已处理
	boolean finishing = false; // 确保已经完成了
	for (int i = 0, bound = 0;;) {
		if (i < 0 || i >= n || i + n >= nextn) {
			int sc;
			if (finishing) {
				... // 完成了
				return;
			}
			if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) { // sizeCtl-1,表示多了一个线程来扩容
				if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
					return;
				finishing = advance = true;
				i = n; // recheck before commit
			}
		}
		else if ((f = tabAt(tab, i)) == null)
			advance = casTabAt(tab, i, null, fwd); // 节点位置是空的，通过CAS设置值为forward
		else if ((fh = f.hash) == MOVED)
			advance = true; // 这个位置是forward节点，表示已经处理了
		else {
			synchronized (f) { // 节点不为空，且不是forward节点，锁住该节点再处理
				... // 类似put的操作
			}
		}
	}
}

// get 源码
if ((eh = e.hash) == h) {
	if ((ek = e.key) == key || (ek != null && key.equals(ek)))
		return e.val; // 直接获得值
}
else if (eh < 0)
	return (p = e.find(h, key)) != null ? p.val : null; // 在树上查找
while ((e = e.next) != null) {
	if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
		return e.val; // 遍历链表查找
}
// 计数方法
private transient volatile CounterCell[] counterCells; // 数组，存储统计值
@sun.misc.Contended static final class CounterCell {
	volatile long value;
	CounterCell(long x) { value = x; }
}
final long sumCount() {
	CounterCell[] as = counterCells; CounterCell a;
	long sum = baseCount;
	if (as != null) {
		for (int i = 0; i < as.length; ++i) {
			if ((a = as[i]) != null)
				sum += a.value; // 统计值累加
		}
	}
	return sum;
}
 2 ConcurrentSkipListMap
  基于跳表实现，按照 key 自然排序，key 不能为 null，类似 TreeMap。
  利用 volatile+CAS 来保证线程安全。

static final class Node<K,V> {
    final K key;
    volatile Object value;
    volatile Node<K,V> next;
}
boolean casValue(Object cmp, Object val) {
    return UNSAFE.compareAndSwapObject(this, valueOffset, cmp, val);
}
 3 ConcurrentSkipListSet
  使用 ConcurrentSkipListMap 实现。

 4 CopyOnWriteArrayList
  基于数组实现，相当于支持并发的 ArrayList，刚创建时初始化为长度0的数组。
  利用写时复制来保证线程安全。
  写时复制：数组是 volatile 类型的，修改数组时，首先 ReentrantLock 加锁，然后复制一个副本数组，对副本进行修改完成后，把原来的数组指向这个新的数组完成赋值。读时不用加锁。

private transient volatile Object[] array;
public boolean add(E e) {
// 加锁进行写时复制
final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 拷贝新数组，长度+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e; 
        // set给volatile的array
        setArray(newElements);
        return true;
  } finally {
        lock.unlock();
    }
}

 5 CopyOnWriteArraySet
  使用 CopyOnWriteArrayList 实现。去重的，但是按照插入顺序排序的。

### 1.3.2 非阻塞队列
 1 ConcurrentLinkedQueue
  基于链表实现的无界的线程安全的非阻塞队列，遵循 FIFO，利用 volatile+CAS 来保证线程安全。

private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
}
// 初始化 head 和 tail
private transient volatile Node<E> head;
private transient volatile Node<E> tail;
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
// 利用 CAS 修改链表
private boolean casTail(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, tailOffset, cmp, val);
}

 2 ConcurrentLinkedDeque
  基于双向链表实现的无界的线程安全的非阻塞队列，实现方式类似 ConcurrentLinkedQueue。

// 双向链表
static final class Node<E> {
    volatile Node<E> prev;
    volatile E item;
    volatile Node<E> next;
}

### 1.3.3 阻塞队列
  实现：通过 ReentrantLock 和 Condition 实现的等待通知模型来实现阻塞队列。

 1 ArrayBlockingQueue
  基于数组实现的阻塞队列，需要指定容量。

// poll 类似
public boolean offer(E e) {
	final ReentrantLock lock = this.lock;
	lock.lock(); // 加锁
	try {
		if (count == items.length)
			return false; // 超过长度，返回false，数据丢失
		final Object[] items = this.items;
		items[putIndex] = x; // putIndex表示下一次加元素的索引
		if (++putIndex == items.length)
			putIndex = 0; // 达到长度后，索引位归零
		count++; // 计数+1
		notEmpty.signal(); // 通知可以取值了
		return true;
	} finally {
		lock.unlock(); // 解锁
	}
}
 2 LinkedBlockingQueue
  基于链表实现的阻塞队列，默认容量为 Integer.MAX_VALUE。
  实现类似 ArrayBlockingQueue，计数用的原子类 AtomicInteger。

 3 PriorityBlockingQueue
  基于二叉小顶堆实现的阻塞队列，保证取出的元素是最小的，默认初始化容量11。

 4 DelayQueue
  基于数组实现的延迟阻塞队列。使用时必须实现 Delayed。

原子操作类
  以 AtomicInteger 为例，利用 volatile+CAS 来保证原子操作，直接看源码注释

private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

private volatile int value;

// 直接获取 volatile 变量
public final int get() {
    return value;
}
// 通过 Unsafe 的 CAS 原子操作 volatile 变量
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
// 通过 Unsafe 的 CAS 原子操作 + 1
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

### 1.3.4并发工具类

 1 CountDownLatch
  功能：指定 N 个线程等待全部完成后，继续执行。
  实现：内部类 Sync 实现了 AQS 同步器，初始化时设置 AQS 的同步状态来表示 countDown 的数量，await() 方法把当前线程加入到 AQS 等待队列，让当前线程阻塞住，执行 countDown() 方法会把同步状态减1，当减到0时，唤醒等待队列中的线程。

 2 CyclicBarrier
  功能：类似 CountDownLatch，但是支持 reset() 重置状态，能指定到达数量时执行的动作。
  实现：基于 ReentrantLock 和 Condition 实现，核心源码如下

private int dowait(boolean timed, long nanos) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 加锁，保护 count
    try {
        

        if (Thread.interrupted()) {
            breakBarrier(); // 使用 signalAll 唤醒所有线程
            throw new InterruptedException();
        }
    
        int index = --count; // 线程数量递减
        if (index == 0) {  // 如果线程数量到达 0
            final Runnable command = barrierCommand;
            if (command != null)
                command.run(); // 执行 barrierAction
            return 0;
        }
    
        // 此时线程数量还没到 0
        for (;;) {
            try {
                if (!timed)
                    trip.await(); // 调用 Condition 的 await 进行等待
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos); // 待超时的等待
            }
        }
    } finally {
        lock.unlock(); // 释放锁
    }
}

### 1.3.5线程池

 ThreadPoolExecutor 参数说明：

  1. 核心线程池
  2. 最大线程池
  3. 线程空闲时间
  4. 线程空闲时间单位
  5. 阻塞队列
  6. 线程工厂：创建具有相同特性的一组线程。
  7. 拒绝策略
   CallerRunsPolicy：重试添加当前的任务，会自动重复调用 execute() 方法，直到成功。
   AbortPolicy：对拒绝任务抛弃处理，并且抛出异常。
   DiscardPolicy：对拒绝任务直接无声抛弃，没有异常信息。
   DiscardOldestPolicy：对拒绝任务不抛弃，而是抛弃队列里面等待最久的一个线程，然后把拒绝任务加到队列。

   线程池数量理论值 -> CPU 密集型：N+1；IO 密集型：2N+1

   线程的提交方式：
   1. execute()：用于提交不需要返回值的任务
   2. submit()：用于提交需要返回值的任务，返回future对象。

   线程池线程的执行流程：核心 -> 队列 -> 最大 -> 拒绝策略
   1. 如果当前运行的线程少于核心线程池时，则创建新的线程执行任务；
   2. 如果当前运行的线程大于等于核心线程池时，则把任务加入阻塞队列；
   3. 如果阻塞队列已经满了，则创建新的线程执行任务；
         4. 如果线程数超过了最大线程数，则调用拒绝策略；

# 第二章 Stream

```
+--------------------+       +------+   +------+   +---+   +-------+
| stream of elements +-----> |filter+-> |sorted+-> |map+-> |collect|
+--------------------+       +------+   +------+   +---+   +-------+
```

## 2.1什么是stream

​	流就像是两个水容器之间的管道（这也意味着对数据的操作是块状的（如集合元素，数组元素），不同于IO流的字节流，但跟NIO的管道缓冲区很像）。这个管道中间，我们可以加一个个过滤器（筛选，排序，映射，这些统一叫做中间操作），然后接一个水龙头（收集操作）。

```java
List<Integer> transactionsIds = 
widgets.stream()                 //流式风格，fluent style
             .filter(b -> b.getColor() == RED)
             .sorted((x,y) -> x.getWeight() - y.getWeight())
             .mapToInt(Widget::getWeight)
             .sum();
```



## 2.2 特点

- 元素是特定类型的对象，形成一个队列。 **Java中的Stream并不会存储元素（使用的是访问者模式，实现数据结构与数据操作分离的分离）**，而是按需计算。
- **数据源** 流的来源。 可以是集合，数组，I/O channel， 产生器generator 等。
- **聚合操作** 类似SQL语句一样的操作， 比如filter, map, reduce, find, match, sorted等。

**和以前的Collection操作不同， Stream操作还有两个基础的特征**：

- **Pipelining**: 中间操作都会返回流对象本身。 这样多个操作可以串联成一个管道， 如同流式风格（fluent style）。 这样做可以对操作进行优化， 比如延迟执行(laziness)和短路( short-circuiting)。
- **内部迭代**： 以前对集合遍历都是通过Iterator或者For-Each的方式, 显式的在集合外部进行迭代， 这叫做外部迭代。 Stream提供了内部迭代的方式， 通过访问者模式(Visitor)实现。

# 第三章 IO/NIO

![img](C:\Users\Arpgalaxy\Desktop\md文件\images\iostream2xx.png)

## 3.1 什么是NIO

​	区别于**BIO（传统IO流+多线程）**，**NIO使用 channel + 多路复用器（selector）的方式**实现同步非阻塞的网络通信。

​	channel和传统io流都会最终调用**read0（）**的本地方法，**在io过程中都要主动等待结果返回**，所以都是同步的。

**总结:**

​	1.同步：**指的是在操作过程中等待结果与否**（无论是BIO还是NIO都是同步的）

​	2.阻塞：指的是线程状态处于等待状态，无法继续执行，等待条件满足才执行。

​					在bio中，一个连接一个线程，并且这个线程一直被占用，即使没有传输数据，也会一直被连接占用				（线程无法被其他连接使用，等待当前连接有IO请求，这条件满足才执行），等于线程处于阻塞等待状					态。

​					在NIO中，多路复用器会监听哪个socket channnel条件满足，即有io请求，就会让线程去处理这个请					求，线程一直处于可用状态，所以是非阻塞的。

## 3.2 NIO操作

### 3.2.1 Buffer

1.四大属性

​				capacity，limit，position，mark

2.**ByteBuffer.allocate（capacity）**获取非直接缓冲区，jvm内存申请，需copy到操作系统主内存

​	**ByteBuffer.allocateDirect(capacity)**获取直接缓冲区，直接在操作系统主内存申请，**FileChanel.map()**也是	直接内存 ，该方法返回一个MappedByteBuffer，也就是说只有bytebuffer支持，

​	这种方法就直接对MappedByteBuffer操作读写，不用Channel读写

​	capacity=limit

3.函数：

​			put（b）存数据position=b.length

​			filp（）=》limit=position；position=0；

​			get（）取数据

​			mark（）记录当前时刻得其他三大属性

​			reset（）恢复上面mark标记。

​			rewind()	=》 position=0；

4.提高效率: 直接缓冲区直接在操作系统中存取，减少了jvm与操作系统内存映射，提高了效率。

​					而非直接缓冲区相当于传统BufferInputStream,BufferoutputStream。

### 3.2.2 channel

相对于DMA，channel有独立的指令，不用cpu计算地址，所以更快

1.分类

​			1.FileChannel

​			2.SocketChannel（TCP）

​			3.ServerSocketChannel（TCP）

​			4.DatagramChannel（UDP）

**2.获取通道**

​	getChannerl（）：

​	本地IO：

​		|--FileInputStream/FileOutputStream

​		|--RandomAccessFile

​	网络IO：

​		|--Socket

​		|--ServerSocket

​		|--DatagramSocket

​		|--Pip.***

​	在JDK1.7中的NIO.2针对各个通过提供了静态方法open()

​	在JDK1.7中的NIO.2的Files工具类的newByteChannel()

​	Channles工具类中提供了静态方法newChannel()。

### 3.2.3 Pipe

Java NIO 管道是2个线程之间的单向数据连接。Pipe有一个source通道和一个sink通道。数据会被写到sink通道，从source通道读取



![img](https://img-blog.csdn.net/20170612201506115?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmFpeWVfeGluZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



## 3.3示例代码

### 3.3.1 channel 和 buffer



```java

package com.atguigu.nio;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.FileChannel.MapMode;
import java.nio.charset.CharacterCodingException;
import java.nio.charset.Charset;
import java.nio.charset.CharsetDecoder;
import java.nio.charset.CharsetEncoder;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;
import java.util.Map;
import java.util.Map.Entry;
import java.util.Set;

import org.junit.Test;

/*
 * 一、通道（Channel）：用于源节点与目标节点的连接。在 Java NIO 中负责缓冲区中数据的传输。Channel 本身不存储数据，因此需要配合缓冲区进行传输。
 * 
 * 二、通道的主要实现类
 * 	java.nio.channels.Channel 接口：
 * 		|--FileChannel
 * 		|--SocketChannel
 * 		|--ServerSocketChannel
 * 		|--DatagramChannel
 * 
 * 三、获取通道
 * 1. Java 针对支持通道的类提供了 getChannel() 方法
 * 		本地 IO：
 * 		FileInputStream/FileOutputStream
 * 		RandomAccessFile
 * 
 * 		网络IO：
 * 		Socket
 * 		ServerSocket
 * 		DatagramSocket
 * 		
 * 2. 在 JDK 1.7 中的 NIO.2 针对各个通道提供了静态方法 open()
 * 3. 在 JDK 1.7 中的 NIO.2 的 Files 工具类的 newByteChannel()
 * 
 * 四、通道之间的数据传输
 * transferFrom()
 * transferTo()
 * 
 * 五、分散(Scatter)与聚集(Gather)
 * 分散读取（Scattering Reads）：将通道中的数据分散到多个缓冲区中
 * 聚集写入（Gathering Writes）：将多个缓冲区中的数据聚集到通道中
 * 
 * 六、字符集：Charset
 * 编码：字符串 -> 字节数组
 * 解码：字节数组  -> 字符串
 * 
 */
public class TestChannel {
	
	//字符集
	@Test
	public void test6() throws IOException{
		Charset cs1 = Charset.forName("GBK");
		
		//获取编码器
		CharsetEncoder ce = cs1.newEncoder();
		
		//获取解码器
		CharsetDecoder cd = cs1.newDecoder();
		
		CharBuffer cBuf = CharBuffer.allocate(1024);
		cBuf.put("尚硅谷威武！");
		cBuf.flip();
		
		//编码
		ByteBuffer bBuf = ce.encode(cBuf);
		
		for (int i = 0; i < 12; i++) {
			System.out.println(bBuf.get());
		}
		
		//解码
		bBuf.flip();
		CharBuffer cBuf2 = cd.decode(bBuf);
		System.out.println(cBuf2.toString());
		
		System.out.println("------------------------------------------------------");
		
		Charset cs2 = Charset.forName("GBK");
		bBuf.flip();
		CharBuffer cBuf3 = cs2.decode(bBuf);
		System.out.println(cBuf3.toString());
	}
	
	@Test
	public void test5(){
		Map<String, Charset> map = Charset.availableCharsets();
		
		Set<Entry<String, Charset>> set = map.entrySet();
		
		for (Entry<String, Charset> entry : set) {
			System.out.println(entry.getKey() + "=" + entry.getValue());
		}
	}
	
	//分散和聚集
	@Test
	public void test4() throws IOException{
		RandomAccessFile raf1 = new RandomAccessFile("1.txt", "rw");
		
		//1. 获取通道
		FileChannel channel1 = raf1.getChannel();
		
		//2. 分配指定大小的缓冲区
		ByteBuffer buf1 = ByteBuffer.allocate(100);
		ByteBuffer buf2 = ByteBuffer.allocate(1024);
		
		//3. 分散读取
		ByteBuffer[] bufs = {buf1, buf2};
		channel1.read(bufs);
		
		for (ByteBuffer byteBuffer : bufs) {
			byteBuffer.flip();
		}
		
		System.out.println(new String(bufs[0].array(), 0, bufs[0].limit()));
		System.out.println("-----------------");
		System.out.println(new String(bufs[1].array(), 0, bufs[1].limit()));
		
		//4. 聚集写入
		RandomAccessFile raf2 = new RandomAccessFile("2.txt", "rw");
		FileChannel channel2 = raf2.getChannel();
		
		channel2.write(bufs);
	}
	
	//通道之间的数据传输(直接缓冲区)
	@Test
	public void test3() throws IOException{
		FileChannel inChannel = FileChannel.open(Paths.get("d:/1.mkv"), StandardOpenOption.READ);
		FileChannel outChannel = FileChannel.open(Paths.get("d:/2.mkv"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);
		
//		inChannel.transferTo(0, inChannel.size(), outChannel);
		outChannel.transferFrom(inChannel, 0, inChannel.size());
		
		inChannel.close();
		outChannel.close();
	}
	
	//使用直接缓冲区完成文件的复制(内存映射文件)
	@Test
	public void test2() throws IOException{//2127-1902-1777
		long start = System.currentTimeMillis();
		
		FileChannel inChannel = FileChannel.open(Paths.get("d:/1.mkv"), StandardOpenOption.READ);
		FileChannel outChannel = FileChannel.open(Paths.get("d:/2.mkv"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);
		
		//内存映射文件
		MappedByteBuffer inMappedBuf = inChannel.map(MapMode.READ_ONLY, 0, inChannel.size());
		MappedByteBuffer outMappedBuf = outChannel.map(MapMode.READ_WRITE, 0, inChannel.size());
		
		//直接对缓冲区进行数据的读写操作
		byte[] dst = new byte[inMappedBuf.limit()];
		inMappedBuf.get(dst);
		outMappedBuf.put(dst);
		
		inChannel.close();
		outChannel.close();
		
		long end = System.currentTimeMillis();
		System.out.println("耗费时间为：" + (end - start));
	}
	
	//利用通道完成文件的复制（非直接缓冲区）
	@Test
	public void test1(){//10874-10953
		long start = System.currentTimeMillis();
		
		FileInputStream fis = null;
		FileOutputStream fos = null;
		//①获取通道
		FileChannel inChannel = null;
		FileChannel outChannel = null;
		try {
			fis = new FileInputStream("d:/1.mkv");
			fos = new FileOutputStream("d:/2.mkv");
			
			inChannel = fis.getChannel();
			outChannel = fos.getChannel();
			
			//②分配指定大小的缓冲区
			ByteBuffer buf = ByteBuffer.allocate(1024);
			
			//③将通道中的数据存入缓冲区中
			while(inChannel.read(buf) != -1){
				buf.flip(); //切换读取数据的模式
				//④将缓冲区中的数据写入通道中
				outChannel.write(buf);
				buf.clear(); //清空缓冲区
			}
		} catch (IOException e) {
			e.printStackTrace();
		} finally {
			if(outChannel != null){
				try {
					outChannel.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
			
			if(inChannel != null){
				try {
					inChannel.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
			
			if(fos != null){
				try {
					fos.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
			
			if(fis != null){
				try {
					fis.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}
		
		long end = System.currentTimeMillis();
		System.out.println("耗费时间为：" + (end - start));
		
	}

}

```

### 3.3.2 网络通信

```java
package com.atguigu.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

import org.junit.Test;

/*
 * 一、使用 NIO 完成网络通信的三个核心：
 * 
 * 1. 通道（Channel）：负责连接
 * 		
 * 	   java.nio.channels.Channel 接口：
 * 			|--SelectableChannel
 * 				|--SocketChannel
 * 				|--ServerSocketChannel
 * 				|--DatagramChannel
 * 
 * 				|--Pipe.SinkChannel
 * 				|--Pipe.SourceChannel
 * 
 * 2. 缓冲区（Buffer）：负责数据的存取
 * 
 * 3. 选择器（Selector）：是 SelectableChannel 的多路复用器。用于监控 SelectableChannel 的 IO 状况
 * 
 */
public class TestBlockingNIO {

	//客户端
	@Test
	public void client() throws IOException{
		//1. 获取通道
		SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9898));
		
		FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
		
		//2. 分配指定大小的缓冲区
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		//3. 读取本地文件，并发送到服务端
		while(inChannel.read(buf) != -1){
			buf.flip();
			sChannel.write(buf);
			buf.clear();
		}
		
		//4. 关闭通道
		inChannel.close();
		sChannel.close();
	}
	
	//服务端
	@Test
	public void server() throws IOException{
		//1. 获取通道
		ServerSocketChannel ssChannel = ServerSocketChannel.open();
		
		FileChannel outChannel = FileChannel.open(Paths.get("2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
		
		//2. 绑定连接
		ssChannel.bind(new InetSocketAddress(9898));
		
		//3. 获取客户端连接的通道
		SocketChannel sChannel = ssChannel.accept();
		
		//4. 分配指定大小的缓冲区
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		//5. 接收客户端的数据，并保存到本地
		while(sChannel.read(buf) != -1){
			buf.flip();
			outChannel.write(buf);
			buf.clear();
		}
		
		//6. 关闭通道
		sChannel.close();
		outChannel.close();
		ssChannel.close();
		
	}
	
}

```

```java
package com.atguigu.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

import org.junit.Test;

public class TestBlockingNIO2 {
	
	//客户端
	@Test
	public void client() throws IOException{
		SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9898));
		
		FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
		
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		while(inChannel.read(buf) != -1){
			buf.flip();
			sChannel.write(buf);
			buf.clear();
		}
		
		sChannel.shutdownOutput();
		
		//接收服务端的反馈
		int len = 0;
		while((len = sChannel.read(buf)) != -1){
			buf.flip();
			System.out.println(new String(buf.array(), 0, len));
			buf.clear();
		}
		
		inChannel.close();
		sChannel.close();
	}
	
	//服务端
	@Test
	public void server() throws IOException{
		ServerSocketChannel ssChannel = ServerSocketChannel.open();
		
		FileChannel outChannel = FileChannel.open(Paths.get("2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
		
		ssChannel.bind(new InetSocketAddress(9898));
		
		SocketChannel sChannel = ssChannel.accept();
		
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		while(sChannel.read(buf) != -1){
			buf.flip();
			outChannel.write(buf);
			buf.clear();
		}
		
		//发送反馈给客户端
		buf.put("服务端接收数据成功".getBytes());
		buf.flip();
		sChannel.write(buf);
		
		sChannel.close();
		outChannel.close();
		ssChannel.close();
	}

}

```

```java
package com.atguigu.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Date;
import java.util.Iterator;
import java.util.Scanner;

import org.junit.Test;

/*
 * 一、使用 NIO 完成网络通信的三个核心：
 * 
 * 1. 通道（Channel）：负责连接
 * 		
 * 	   java.nio.channels.Channel 接口：
 * 			|--SelectableChannel
 * 				|--SocketChannel
 * 				|--ServerSocketChannel
 * 				|--DatagramChannel
 * 
 * 				|--Pipe.SinkChannel
 * 				|--Pipe.SourceChannel
 * 
 * 2. 缓冲区（Buffer）：负责数据的存取
 * 
 * 3. 选择器（Selector）：是 SelectableChannel 的多路复用器。用于监控 SelectableChannel 的 IO 状况
 * 
 */
public class TestNonBlockingNIO {
	
	//客户端
	@Test
	public void client() throws IOException{
		//1. 获取通道
		SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9898));
		
		//2. 切换非阻塞模式
		sChannel.configureBlocking(false);
		
		//3. 分配指定大小的缓冲区
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		//4. 发送数据给服务端
		Scanner scan = new Scanner(System.in);
		
		while(scan.hasNext()){
			String str = scan.next();
			buf.put((new Date().toString() + "\n" + str).getBytes());
			buf.flip();
			sChannel.write(buf);
			buf.clear();
		}
		
		//5. 关闭通道
		sChannel.close();
	}

	//服务端
	@Test
	public void server() throws IOException{
		//1. 获取通道
		ServerSocketChannel ssChannel = ServerSocketChannel.open();
		
		//2. 切换非阻塞模式
		ssChannel.configureBlocking(false);
		
		//3. 绑定连接
		ssChannel.bind(new InetSocketAddress(9898));
		
		//4. 获取选择器
		Selector selector = Selector.open();
		
		//5. 将通道注册到选择器上, 并且指定“监听接收事件”
		ssChannel.register(selector, SelectionKey.OP_ACCEPT);
		
		//6. 轮询式的获取选择器上已经“准备就绪”的事件
		while(selector.select() > 0){
			
			//7. 获取当前选择器中所有注册的“选择键(已就绪的监听事件)”
			Iterator<SelectionKey> it = selector.selectedKeys().iterator();
			
			while(it.hasNext()){
				//8. 获取准备“就绪”的是事件
				SelectionKey sk = it.next();
				
				//9. 判断具体是什么事件准备就绪
				if(sk.isAcceptable()){
					//10. 若“接收就绪”，获取客户端连接
					SocketChannel sChannel = ssChannel.accept();
					
					//11. 切换非阻塞模式
					sChannel.configureBlocking(false);
					
					//12. 将该通道注册到选择器上
					sChannel.register(selector, SelectionKey.OP_READ);
				}else if(sk.isReadable()){
					//13. 获取当前选择器上“读就绪”状态的通道
					SocketChannel sChannel = (SocketChannel) sk.channel();
					
					//14. 读取数据
					ByteBuffer buf = ByteBuffer.allocate(1024);
					
					int len = 0;
					while((len = sChannel.read(buf)) > 0 ){
						buf.flip();
						System.out.println(new String(buf.array(), 0, len));
						buf.clear();
					}
				}
				
				//15. 取消选择键 SelectionKey
				it.remove();
			}
		}
	}
}

```

```java
package com.atguigu.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.util.Date;
import java.util.Iterator;
import java.util.Scanner;

import org.junit.Test;

public class TestNonBlockingNIO2 {
	
	@Test
	public void send() throws IOException{
		DatagramChannel dc = DatagramChannel.open();
		
		dc.configureBlocking(false);
		
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		Scanner scan = new Scanner(System.in);
		
		while(scan.hasNext()){
			String str = scan.next();
			buf.put((new Date().toString() + ":\n" + str).getBytes());
			buf.flip();
			dc.send(buf, new InetSocketAddress("127.0.0.1", 9898));
			buf.clear();
		}
		
		dc.close();
	}
	
	@Test
	public void receive() throws IOException{
		DatagramChannel dc = DatagramChannel.open();
		
		dc.configureBlocking(false);
		
		dc.bind(new InetSocketAddress(9898));
		
		Selector selector = Selector.open();
		
		dc.register(selector, SelectionKey.OP_READ);
		
		while(selector.select() > 0){
			Iterator<SelectionKey> it = selector.selectedKeys().iterator();
			
			while(it.hasNext()){
				SelectionKey sk = it.next();
				
				if(sk.isReadable()){
					ByteBuffer buf = ByteBuffer.allocate(1024);
					
					dc.receive(buf);
					buf.flip();
					System.out.println(new String(buf.array(), 0, buf.limit()));
					buf.clear();
				}
			}
			
			it.remove();
		}
	}

}

```

```java
package com.atguigu.nio;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.Pipe;

import org.junit.Test;

public class TestPipe {

	@Test
	public void test1() throws IOException{
		//1. 获取管道
		Pipe pipe = Pipe.open();
		
		//2. 将缓冲区中的数据写入管道
		ByteBuffer buf = ByteBuffer.allocate(1024);
		
		Pipe.SinkChannel sinkChannel = pipe.sink();
		buf.put("通过单向管道发送数据".getBytes());
		buf.flip();
		sinkChannel.write(buf);
		
		//3. 读取缓冲区中的数据
		Pipe.SourceChannel sourceChannel = pipe.source();
		buf.flip();
		int len = sourceChannel.read(buf);
		System.out.println(new String(buf.array(), 0, len));
		
		sourceChannel.close();
		sinkChannel.close();
	}
	
}

```

```java
package com.atguigu.nio;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.SeekableByteChannel;
import java.nio.file.DirectoryStream;
import java.nio.file.Files;
import java.nio.file.LinkOption;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.nio.file.StandardOpenOption;
import java.nio.file.attribute.BasicFileAttributes;
import java.nio.file.attribute.DosFileAttributeView;

import org.junit.Test;

public class TestNIO_2 {
	
	
	//自动资源管理：自动关闭实现 AutoCloseable 接口的资源
	@Test
	public void test8(){
		try(FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
				FileChannel outChannel = FileChannel.open(Paths.get("2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.CREATE)){
			
			ByteBuffer buf = ByteBuffer.allocate(1024);
			inChannel.read(buf);
			
		}catch(IOException e){
			
		}
	}
	
	/*
		Files常用方法：用于操作内容
			SeekableByteChannel newByteChannel(Path path, OpenOption…how) : 获取与指定文件的连接，how 指定打开方式。
			DirectoryStream newDirectoryStream(Path path) : 打开 path 指定的目录
			InputStream newInputStream(Path path, OpenOption…how):获取 InputStream 对象
			OutputStream newOutputStream(Path path, OpenOption…how) : 获取 OutputStream 对象
	 */
	@Test
	public void test7() throws IOException{
		SeekableByteChannel newByteChannel = Files.newByteChannel(Paths.get("1.jpg"), StandardOpenOption.READ);
		
		DirectoryStream<Path> newDirectoryStream = Files.newDirectoryStream(Paths.get("e:/"));
		
		for (Path path : newDirectoryStream) {
			System.out.println(path);
		}
	}
	
	/*
		Files常用方法：用于判断
			boolean exists(Path path, LinkOption … opts) : 判断文件是否存在
			boolean isDirectory(Path path, LinkOption … opts) : 判断是否是目录
			boolean isExecutable(Path path) : 判断是否是可执行文件
			boolean isHidden(Path path) : 判断是否是隐藏文件
			boolean isReadable(Path path) : 判断文件是否可读
			boolean isWritable(Path path) : 判断文件是否可写
			boolean notExists(Path path, LinkOption … opts) : 判断文件是否不存在
			public static <A extends BasicFileAttributes> A readAttributes(Path path,Class<A> type,LinkOption... options) : 获取与 path 指定的文件相关联的属性。
	 */
	@Test
	public void test6() throws IOException{
		Path path = Paths.get("e:/nio/hello7.txt");
//		System.out.println(Files.exists(path, LinkOption.NOFOLLOW_LINKS));
		
		BasicFileAttributes readAttributes = Files.readAttributes(path, BasicFileAttributes.class, LinkOption.NOFOLLOW_LINKS);
		System.out.println(readAttributes.creationTime());
		System.out.println(readAttributes.lastModifiedTime());
		
		DosFileAttributeView fileAttributeView = Files.getFileAttributeView(path, DosFileAttributeView.class, LinkOption.NOFOLLOW_LINKS);
		
		fileAttributeView.setHidden(false);
	}
	
	/*
		Files常用方法：
			Path copy(Path src, Path dest, CopyOption … how) : 文件的复制
			Path createDirectory(Path path, FileAttribute<?> … attr) : 创建一个目录
			Path createFile(Path path, FileAttribute<?> … arr) : 创建一个文件
			void delete(Path path) : 删除一个文件
			Path move(Path src, Path dest, CopyOption…how) : 将 src 移动到 dest 位置
			long size(Path path) : 返回 path 指定文件的大小
	 */
	@Test
	public void test5() throws IOException{
		Path path1 = Paths.get("e:/nio/hello2.txt");
		Path path2 = Paths.get("e:/nio/hello7.txt");
		
		System.out.println(Files.size(path2));
		
//		Files.move(path1, path2, StandardCopyOption.ATOMIC_MOVE);
	}
	
	@Test
	public void test4() throws IOException{
		Path dir = Paths.get("e:/nio/nio2");
//		Files.createDirectory(dir);
		
		Path file = Paths.get("e:/nio/nio2/hello3.txt");
//		Files.createFile(file);
		
		Files.deleteIfExists(file);
	}
	
	@Test
	public void test3() throws IOException{
		Path path1 = Paths.get("e:/nio/hello.txt");
		Path path2 = Paths.get("e:/nio/hello2.txt");
		
		Files.copy(path1, path2, StandardCopyOption.REPLACE_EXISTING);
	}
	
	/*
		Paths 提供的 get() 方法用来获取 Path 对象：
			Path get(String first, String … more) : 用于将多个字符串串连成路径。
		Path 常用方法：
			boolean endsWith(String path) : 判断是否以 path 路径结束
			boolean startsWith(String path) : 判断是否以 path 路径开始
			boolean isAbsolute() : 判断是否是绝对路径
			Path getFileName() : 返回与调用 Path 对象关联的文件名
			Path getName(int idx) : 返回的指定索引位置 idx 的路径名称
			int getNameCount() : 返回Path 根目录后面元素的数量
			Path getParent() ：返回Path对象包含整个路径，不包含 Path 对象指定的文件路径
			Path getRoot() ：返回调用 Path 对象的根路径
			Path resolve(Path p) :将相对路径解析为绝对路径
			Path toAbsolutePath() : 作为绝对路径返回调用 Path 对象
			String toString() ： 返回调用 Path 对象的字符串表示形式
	 */
	@Test
	public void test2(){
		Path path = Paths.get("e:/nio/hello.txt");
		
		System.out.println(path.getParent());
		System.out.println(path.getRoot());
		
//		Path newPath = path.resolve("e:/hello.txt");
//		System.out.println(newPath);
		
		Path path2 = Paths.get("1.jpg");
		Path newPath = path2.toAbsolutePath();
		System.out.println(newPath);
		
		System.out.println(path.toString());
	}
	
	@Test
	public void test1(){
		Path path = Paths.get("e:/", "nio/hello.txt");
		
		System.out.println(path.endsWith("hello.txt"));
		System.out.println(path.startsWith("e:/"));
		
		System.out.println(path.isAbsolute());
		System.out.println(path.getFileName());
		
		for (int i = 0; i < path.getNameCount(); i++) {
			System.out.println(path.getName(i));
		}
	}
}

```

# 第四章 集合框架

![img](C:\Users\Arpgalaxy\Desktop\md文件\images\2243690-9cd9c896e0d512ed.gif)



arraylist  扩容为1.5倍，初始默认容量为10



hashmap 

1. 初始容量为16（16-1=1111），
2. 指定容量也会被转化为2的倍数（4-1=0011，8-1=0111），其中n=cap-1,n|=n>>>1,n|=n>>>2,n|=n>>>4,n|=n>>>8,n|=n>>>16或运算，有一个为1就为1，保证传入的cap-1的最高位，并通过位移将其他低位转化为1，因此3=>3-1=2（0010=>0011)+1=4,  5=>5-1=4(0100=>0111)7+1=8等等，容量被转化为2的倍数。
3. 关于扩容为2倍，1111=》11111，方便（hash&cap）运算。
4. 关于加载因子0.75，0.5太小，16容量8个就扩容，会导致频繁扩容，根据注释的泊松分布计算：离散概率分布，第一次就命中的概率高达0.6，也就是说，16容量的hashmap会很轻松全部命中。但是16个才扩容，就有可能会出现饿死现象，因为可能有一个位置永远不会命中，就会加大链表长度，导致红黑树时，频繁命中并自旋以求自平衡。
5. 关于链表<8 ,>6.根据源码注释得知，当hash值命中次数达到8次以后，再次命中的概率就会成为一亿分之6左右，命中次数达到7次以后，概率也在一亿分之9几左右。但是当命中次数在6以内时，概率时十万分之1点几。实际生产中极容易达到的次数。因此在<6时会可能有频繁的添加删除操作，不适合红黑树，会转化为链表，链表长度大于8就可以转换为红黑树，提高查找效率。当链表转换成8个节点的红黑树以后，删除操作使得节点数>6,<8,也就是=7的时候，不会进行上转或下转。

set是用hashmap的key作值因此不可重复但是可以

# 第五章 JVM

![img](C:\Users\Arpgalaxy\Desktop\md文件\images\java\jvm)

![img](C:\Users\Arpgalaxy\Desktop\md文件\images\java\jdk8jvm)

![img](C:\Users\Arpgalaxy\Desktop\md文件\images\java\jvm.png)

# 第6章 常用类

## 1.String


## 2. 日期

1. **int compareTo(Date date)**
   比较当调用此方法的Date对象和指定日期。两者相等时候返回0。调用对象在指定日期之前则返回负数。调用对象在指定日期之后则返回正数。

2. **long getTime( ) ** 返回自 1970 年 1 月 1 日 00:00:00 GMT 以来此 Date 对象表示的毫秒数。

3. **void setTime(long time)**

   用自1970年1月1日00:00:00 GMT以后time毫秒数设置时间和日期。

## 5. Optional  (jdk8)

减少空指针

1. Optional.ofNullable(value1)允许传递为 null 参数
2. Optional.of(value2)如果传递的参数是 null，抛出异常 NullPointerException
3. Optional.isPresent（） 判断值是否存在
4. new Optional().orElse(object) - 如果值存在，返回它，否则返回默认值
5. Optional.get - 获取值，值需要存在

## 6. 6大包装类



## 7.Arrays

```java
@Test
public void test(){
    int a[]={1,2,3,4,5};
    List<Integer> integers = Arrays.<Integer>asList(1, 2, 3, 4);

    int[] ints = Arrays.copyOf(a, 1);//length=1
    int[] ints1 = Arrays.copyOfRange(a, 1, 3);//length=2
    int b[]=a;
    Arrays.equals(a, b);//true

    Arrays.fill(a,0);//a数组用0填充所有位置

    IntStream stream = Arrays.stream(a);
    stream.filter(c -> c == 0).forEach(System.out::println);
    
    System.out.println(Arrays.toString(a));//显示a值
}
```

