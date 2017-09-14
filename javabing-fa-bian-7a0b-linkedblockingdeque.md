
# LinkedBlockingDeque要点
* 该队列是基于双向链表的有界阻塞队列。
* 构造函数的可选参数是容量的界限，用来阻止容量过度扩展的一种方法。也就是说，虽然是双向链表，但也可以指定其容量大小。
* 容量（如果未指定）等于Integer.MAX_VALUE。若节点数量没有到达容量的界限，链接节点在每次插入时都会动态创建。
* 大多数操作在恒定时间运行，异常包括remove，removeFirstOccurrence，removeLastOccurrence，contains，iterator.remove()和批量操作，所有这些都是以线性时间运行的。
* 可以指定元素的插入位置：头部还是尾部。


# LinkedBlockingDeque实现原理

## LinkedBlockingDeque类变量
以下代码是LinkedBlockingDeque的变量定义，从代码实现来看，可以得出以下要点：

* 创建一个Node节点，该节点有prev和next两个“指针”，所以通过该节点，可以创建一个双向链表。（具体实现看代码分析）
* 通过ReentrantLock锁和条件变量来进行元素访问的同步。
* 可以在构造LinkedBlockingDeque时指定队列的最大容量，也可以不指定，若不指定容量大小为：Integer.MAX_VALUE。

```
public class LinkedBlockingDeque<E>
    extends AbstractQueue<E>
    implements BlockingDeque<E>,  java.io.Serializable {

    private static final long serialVersionUID = -387911632671998426L;

    /** Doubly-linked list node class */
    static final class Node<E> {
        /**
         * The item, or null if this node has been removed.
         */
        // 插入的元素不能为空
        E item;

        /**
         * One of:
         * - the real predecessor Node
         * - this Node, meaning the predecessor is tail
         * - null, meaning there is no predecessor
         */
        // 前一个节点的引用
        Node<E> prev;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head
         * - null, meaning there is no successor
         */
        // 下一个节点的引用
        Node<E> next;

        // 为该节点赋值
        Node(E x) {
            item = x;
        }
    }

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    // 指向链表的头节点
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    // 指向链表的尾节点
    transient Node<E> last;

    /** Number of items in the deque */
    // 目前双向链表的元素的数量
    private transient int count;

    /** Maximum number of items in the deque */
    // 双向链表的最大元素个数
    private final int capacity;

    /** Main lock guarding all access */
    // 用来进行同步的ReentrantLock锁
    final ReentrantLock lock = new ReentrantLock();

    /** Condition for waiting takes */
    // 基于锁的条件变量，用来判断队列是否为空
    private final Condition notEmpty = lock.newCondition();

    /** Condition for waiting puts */
    // 基于锁的条件变量，用来判断队列是否已满
    private final Condition notFull = lock.newCondition();

```

## LinkedBlockingDeque类的构造函数

1.没有任何参数的构造函数(默认构造函数)
此时最大容量个数是Integer.MAX_VALUE。

```
    public LinkedBlockingDeque() {
        this(Integer.MAX_VALUE);
    }

```

2.指定容量大小的构造函数
此时，容量大小是我们指定的大小。

```
    public LinkedBlockingDeque(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
    }
```

3.通过复制一个集合元素构造LinkedBlockingDeque

此时需要遍历该集合的元素，并为集合的每个元素创建一个Node，并把该Node添加到LinkedBlockingDeque中。

```
public LinkedBlockingDeque(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock lock = this.lock;
        lock.lock(); // Never contended, but necessary for visibility
        try {
        	// 遍历集合元素，把元素添加到LinkedBlockingDeque的尾部
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (!linkLast(new Node<E>(e)))
                    throw new IllegalStateException("Deque full");
            }
        } finally {
            lock.unlock();
        }
    }
```

## LinkedBlockingDeque类的内部函数

### 函数的返回值总结

 |  operation|Throws exception | Special value | Blocks | Times out |
 |   -  |    -     |      -        |  -     |    -     |
 |Insert |addFirst(e) | offerFirst(e) | putFirst(e) | offerFirst(e, time, unit)  |
 |Remove |removeFirst() |pollFirst() | takeFirst()  | pollFirst(time, unit) |
 |Examine |getFirst() |peekFirst()   |not applicable| not applicable |


### 基础函数
* private boolean linkLast(Node<E> node)
    * 向队列的尾部添加一个节点：node。
	* 目前的元素个数大于等于最大容量数，该函数直接返回false
	* 新节点被添加到双向链表的末尾，作为末尾节点
	* 成功后，向在条件变量notEmpty上等待的线程发送信号

```
	/**
     * Links node as last element, or returns false if full.
     */
    private boolean linkLast(Node<E> node) {
        // assert lock.isHeldByCurrentThread();
        // 若目前的元素个数大于等于最大容量数，直接返回false
        if (count >= capacity)
            return false;
        // 以下代码把新节点添加到链表末尾，新节点作为尾节点
        Node<E> l = last;
        node.prev = l;
        last = node;
        if (first == null)
            first = node;
        else
            l.next = node;
        ++count;

        // 成功把新节点放到双向链表的尾部，现在向在条件变量notEmpty上等待的线程发送信号。
        notEmpty.signal();
        return true;
    }
```

* private boolean linkFirst(Node<E> node)
    * 功能：在队列头部添加一个节点：node
	* 目前的元素个数大于等于最大容量数，该函数直接返回false
	* 新节点被添加到双向链表的头部，作为头节点
	* 成功后，向在条件变量notEmpty上等待的线程发送信号

```
	/**
     * Links node as first element, or returns false if full.
     */
    private boolean linkFirst(Node<E> node) {
        // assert lock.isHeldByCurrentThread();
        // 若元素的个数超过了最大容量的限制，则返回false。
        if (count >= capacity)
            return false;
        // 把新节点添加到双向链表头部，作为头节点
        Node<E> f = first;
        node.next = f;
        first = node;
        if (last == null)
            last = node;
        else
            f.prev = node;
        ++count;
        notEmpty.signal();
        return true;
    }
```

* private E unlinkFirst()
该函数返回双线链表的第一个节点，若该节点的值。若链表为空，返回为Null。

```
    /**
     * Removes and returns first element, or null if empty.
     */
    private E unlinkFirst() {
        // assert lock.isHeldByCurrentThread();
        Node<E> f = first;
        // 链表为空，返回空
        if (f == null)
            return null;
        // 获取第一个节点的值，并删除第一个节点，其中f.next=f这个操作很有java的特点。
        Node<E> n = f.next;
        E item = f.item;
        f.item = null;
        f.next = f; // help GC
        first = n;
        if (n == null)
            last = null;
        else
            n.prev = null;
        --count;
        // 头节点删除成功，向在条件变量：notFull等待的线程发送信号，通知其中的一个线程
        notFull.signal();
        return item;
    }
```

* private E unlinkLast()
功能：返回队列的最后一个节点，并删除。注意删除后，队列有可能为空。

```
    /**
     * Removes and returns last element, or null if empty.
     */
    private E unlinkLast() {
        // assert lock.isHeldByCurrentThread();
        Node<E> l = last;
        // 若队列为null，则返回空
        if (l == null)
            return null;
        // 获取最后一个节点的值并删除
        Node<E> p = l.prev;
        E item = l.item;
        l.item = null;
        l.prev = l; // help GC
        last = p;
        if (p == null)
            first = null;
        else
            p.next = null;
        --count;
        // 删除后，给在条件变量notFull上等待的线程发送信号。
        notFull.signal();
        return item;
    }
```


* public boolean offerFirst(E e)
在队列头部添加一个值为e的节点。

```
   /**
     * @throws NullPointerException {@inheritDoc}
     */
    public boolean offerFirst(E e) {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        // 使用ReentrantLock来进行加锁。
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 向队列的头部添加节点node。
            return linkFirst(node);
        } finally {
            lock.unlock();
        }
    }
```

* public boolean offerLast(E e)
功能：在队列的尾部添加一个值为e的节点。

```
    /**
     * @throws NullPointerException {@inheritDoc}
     */
    public boolean offerLast(E e) {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        // 使用ReentrantLock来进行加锁。
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return linkLast(node);
        } finally {
            lock.unlock();
        }
    }
```


### 对外函数
其中的add,offer,put其实相当于，addLast,offerLast,putLast。

可以把对外接口函数分为几类进行讲解：

#### add,addFirst,addLast 和 remove,removeFirst,removeLast
这两组方法不会阻塞，若是插入不成功，会抛出IllegalStateException异常。

* public void addLast(E e)
```
    public void addLast(E e) {
        if (!offerLast(e))
            throw new IllegalStateException("Deque full");
    }
```


#### offer,offerFirst,offerLast 和 poll,pollFirst,pollLast
    这两组函数都会返回一个值，来代表是否插入数据或获取数据成功。

##### 实现分析

* public boolean offerLast(E e)
    * 该函数在队列的尾部添加一个值为e的节点。
    * 该函数会先加锁，若加锁成功，直接调用linkLast把新节点插入到队列的尾部。
    * 若队列已满(元素个数超过容量限制)，返回false，成功返回true。
    * 注意:offer系列函数在加锁成功后不会阻塞等待。但在等待加锁时可能会因为等待锁而阻塞。
```
 public boolean offerLast(E e) {
        if (e == null) throw new NullPointerException();
        // 生成一个新的节点
        Node<E> node = new Node<E>(e);
        final ReentrantLock lock = this.lock;
        // 尝试加锁，这里可能阻塞
        lock.lock();
        try {
            // 加锁成功，不会阻塞，而是直接插入。成功返回true，失败返回false。
            return linkLast(node);
        } finally {
            lock.unlock();
        }
    }
```

* public E pollFirst()
    * 该函数从队列头部获取一个节点，并删除该节点，返回该节点的值。
    * poll 和 pollFirst功能相同
    * 若成功返回该节点的值，若失败返回false
```
   public E pollFirst() {
        final ReentrantLock lock = this.lock;
        // 尝试加锁，这里可能会阻塞
        lock.lock();
        try {
            // 返回并删除第一个节点
            return unlinkFirst();
        } finally {
            lock.unlock();
        }
    }
```

#### put,putFirst,putLast 和 take,takeFirst,takeLast
    这两组函数在放入数据或获取数据时，会判断队列是否已满，或是否为空，若是则这一组函数会等待。

##### 实现分析

* public void putLast(E e)
    * 该函数向队列尾部添加一个值为e的节点
    * put 和putLast的实现相同
    * 若队列已满，该函数会阻塞等待，直到队列有多余的空间。

```
    public void putLast(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 若在队列尾部添加节点失败，则在条件变量notFull上等待，直到有空位
            while (!linkLast(node))
                notFull.await();
        } finally {
            lock.unlock();
        }
    }
```

* public E takeFirst()
    * 获取队列的第一个节点的值，并删除该节点
    * 若队列为空，则阻塞等待
    * take和takeFirst功能相同

```
    public E takeFirst() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E x;
            // 若队列为空，则阻塞等待
            while ( (x = unlinkFirst()) == null)
                notEmpty.await();
            return x;
        } finally {
            lock.unlock();
        }
    }
```

#### peek,peekFirst,peekLast和getFirst,getLast
    * 这一组函数只是取回队列的值，而不会删除队列的节点
    * 若队列为空，getFirst,getLast会抛出异常
    * 若队列为空，peek,peekFirst,peekLast会返回空

##### 实现分析

* public E peekFirst()

```
    public E peekFirst() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 获取第一个节点的值，但不删除节点。若第一个节点为空，返回Null
            return (first == null) ? null : first.item;
        } finally {
            lock.unlock();
        }
    }
```

* public E getFirst()

```
    public E getFirst() {
        // 获取第一个节点的值，但不删除节点
        E x = peekFirst();
        // 若为空，抛出异常
        if (x == null) throw new NoSuchElementException();
        return x;
    }

```

#### 栈方法：push 和 pop 

```
    public void push(E e) {
        addFirst(e);
    }

    /**
     * @throws NoSuchElementException {@inheritDoc}
     */
    public E pop() {
        return removeFirst();
    }
```


# LinkedBlockingDeque使用实战

以下代码实例，使用LinkedBlockingDeque来实现生产者和消费者的同步。
通过该类实现的生产者和消费者模型，非常简洁。

```
package BlockingQueue;

import java.util.Random;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingDeque;

/*
 * 多生产者，多消费者模型，通过LinkedBlockingDeque实现。
 */

// comsumer
class BQConsumer implements Runnable {
    private final BlockingQueue queue;

    Random r = new Random(1000);

    BQConsumer(BlockingQueue queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            while (true) {
                String v = String.valueOf(r.nextInt(10000));
                queue.put(v);
                System.out.println(Thread.currentThread().getName() + " get: " + v);

                Thread.sleep(r.nextInt(10));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


// producer
class BQProducer implements Runnable {
    private final BlockingQueue queue;
    Random r = new Random(100000);

    BQProducer(BlockingQueue queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            while (true) {
                Thread.sleep(r.nextInt(10));
                String value = (String) queue.take();
                System.out.println(Thread.currentThread().getName() + " put value: " + value);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


public class DoLinkedBDeQ {
    public static void main(String[] args) {
        final BlockingQueue q = new LinkedBlockingDeque(1000);

        new Thread(new BQProducer(q), "producer1").start();
        new Thread(new BQProducer(q), "producer2").start();
        new Thread(new BQProducer(q), "producer3").start();
        new Thread(new BQConsumer(q), "consumer2").start();
        new Thread(new BQConsumer(q), "consumer1").start();

        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

## LinkedBlockingDeque和ArrayBlockQueue的对比
* ArrayBlockQueue的队列大小是固定的，LinkedBlockingDeque的队列大小可以固定，也可以不固定。
* LinkedBlockingDeque中添加和删除节点是动态的，有内存的申请和释放，性能可能有损耗。
* 而ArrayBlockQueue只是把某个位置的数据置为空，队列总的容量大小不会变化，变化的队列中的元素个数。
* 在性能要求较高的环境下，建议使用ArrayBlockQueue。
* LinkedBlockingDeque是一个双向队列，尾部或头部都可以插入或获取接节点。
