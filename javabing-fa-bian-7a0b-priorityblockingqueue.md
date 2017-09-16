
# PriorityBlockingQueue要点

* PriorityBlockingQueue类实现了BlockingQueue接口。
* PriorityBlockingQueue是一个无界并发队列。它使用与java.util.PriorityQueue类相同的排序规则。
* 也就是说，进入该队列的元素会自动排序，在获取该队列的元素时该队列的元素是排好序的。
* 不能在此队列中插入null。
* 虽然此队列在逻辑上是无界的，但由于资源耗尽（导致OutOfMemoryError），尝试添加可能会失败。  
* 依靠自然排序的优先级队列也不允许插入不可比较的对象（这样做会导致ClassCastException）。
* 插入到PriorityBlockingQueue中的所有元素都必须实现java.lang.Comparable接口。因此，根据您在可比较实施中决定的任何优先级，这些元素将自动排序。
* 请注意，PriorityBlockingQueue不对具有相同优先级的元素（compare（）== 0）强制执行任何特定行为。
* 该类及其迭代器实现了Collection和Iterator接口的所有可选方法。
* 方法iterator（）中提供的迭代器不能保证以任何特定的顺序遍历PriorityBlockingQueue的元素。如果需要有序遍历，请考虑使用Arrays.sort（pq.toArray（））。
* 此外，方法drainTo可以用于按优先级顺序删除一些或所有元素，并将它们放在另一个集合中。
* 这个类的操作不会保证同等优先级的元素的顺序。若要实现同等优先级元素的顺序，需要自己实现类进行比较：
* 例如你要实现按fifo对元素排序的机制：

官方给出的代码如下：
```
 class FIFOEntry<E extends Comparable<? super E>>
     implements Comparable<FIFOEntry<E>> {
   static final AtomicLong seq = new AtomicLong(0);
   final long seqNum;
   final E entry;
   public FIFOEntry(E entry) {
     seqNum = seq.getAndIncrement();
     this.entry = entry;
   }
   public E getEntry() { return entry; }
   public int compareTo(FIFOEntry<E> other) {
     int res = entry.compareTo(other.entry);
     if (res == 0 && other.entry != this.entry)
       res = (seqNum < other.seqNum ? -1 : 1);
     return res;
   }
 }

```

## 返回值

|  method   |Throws Exception|  Special Value   |  Blocks  |   Times Out  |
|     -     |    -           |       -          |    -     |      -       |
|Insert     |                |  put(o),offer(o),add(o) |          |              | 
|Remove     |                |  poll(),remove(o)| take()   |  poll(t)     |
|Examine    |                |  peek()          |          |              |

请注意PriorityBlockingQueue的返回值情况。
* 由于PriorityBlockingQueue队列可以自动扩容，所以插入元素时，不用等待阻塞。
* 获取元素时，若队列为空，需要阻塞等待。


# PriorityBlockingQueue实现原理分析
* PriorityBlockingQueue使用基于数组的二叉堆，和一个公共的锁来实现。

## 变量定义

```
// 数组的默认容量大小
private static final int DEFAULT_INITIAL_CAPACITY = 11;

// 数组的最大容量，源码注释：有的虚拟机可能会使用数组的头部一个字节。
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

// 数组，节点queue[n]的儿子节点是 queue[2*n+1]和queue[2*(n+1)]。
// 注：最小堆的定义和使用，详细见《算法导论》
private transient Object[] queue;

// 队列中元素的个数
private transient int size;

// 比较器，用来排序
private transient Comparator<? super E> comparator;

// 对操作进行同步的锁
private final ReentrantLock lock;

// 基于锁的条件变量
private final Condition notEmpty;

// 自旋锁，在分配新的空间时进行同步
private transient volatile int allocationSpinLock;

// 优先队列变量，用来向前兼容
private PriorityQueue q;

```

## 构造函数

* public PriorityBlockingQueue(int initialCapacity, Comparator<? super E> comparator)
指定容量大小和对比器。该构造函数构造一个空的初始化数组的队列。

```
    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }
```

* public PriorityBlockingQueue(Collection<? extends E> c)
把已有队列中的数据复制到优先队列中。需要分两种情况对集合中的数据进行处理：
    * 若参数的结合类型是SortedSet或PriorityBlockingQueue，则不需要进行元素的调整（堆化）。


```
public PriorityBlockingQueue(Collection<? extends E> c) {
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();

    boolean heapify = true; // true if not known to be in heap order
    boolean screen = true;  // true if must screen for nulls

    // 若参数集合c是SortedSet类型，不需要堆化
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        heapify = false;
    }
    else if (c instanceof PriorityBlockingQueue<?>) {
    	// 若是参数是PriorityBlockingQueue类型的，也不需要堆化
        PriorityBlockingQueue<? extends E> pq =
            (PriorityBlockingQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        screen = false;
        if (pq.getClass() == PriorityBlockingQueue.class) // exact match
            heapify = false;
    }

    // 把参数中的元素复制到Object[]数组中，然后对该数组进行堆化
    Object[] a = c.toArray();
    int n = a.length;
    // If c.toArray incorrectly doesn't return Object[], copy it.
    if (a.getClass() != Object[].class)
        a = Arrays.copyOf(a, n, Object[].class);
    if (screen && (n == 1 || this.comparator != null)) {
        for (int i = 0; i < n; ++i)
            if (a[i] == null)
                throw new NullPointerException();
    }
    this.queue = a;
    this.size = n;
    // 根据情况进行堆化
    if (heapify)
        heapify();
}
```


## 内部函数

* siftUpComparable
	* 该函数在最小堆的位置k中插入一个元素x。
	* 该函数用来调整最小堆中元素的位置，保证插入元素后，满足最小堆的特性。

```
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
    	// 找到父结点的位置
        int parent = (k - 1) >>> 1;
        // 获取父结点的值
        Object e = array[parent];
        // 插入的值大于父节点的值，直接插入
        if (key.compareTo((T) e) >= 0)
            break;
        // 否则和父节点交换，并重复这个过程，直到该节点的值大于父节点。
        array[k] = e;
        k = parent;
    }
    array[k] = key;
}

```

* private static <T> void siftDownComparable(int k, T x, Object[] array, int n)
参数说明：
    * k: 需要插入元素的位置
    * x：需要插入的元素值
    * array: 保存堆的数组
    * n：目前数组中的元素个数
功能：
    * 把x插入到堆中。该函数会不断调整堆的结构，让所有的孩子节点的值小于父节点的值。
    * 具体的堆调整的步骤详见《算法导论》的堆排序。

```
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        // 数组的中间位置一定是叶子节点
        int half = n >>> 1;           // loop while a non-leaf
        while (k < half) {
            // 获取左边的叶子节点
            int child = (k << 1) + 1; // assume left child is least
            Object c = array[child];
            // 右边的儿子节点
            int right = child + 1;
            // 若右边的索引数小于元素个数,且左边的大于右边的儿子节点的值
            // 把右边的儿子节点的值赋给变量c
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            // 若插入的值小于c，k应该作为父节点，否则k应该作为儿子节点
            if (key.compareTo((T) c) <= 0)
                break;
            // c是父节点
            array[k] = c;
            k = child;
        }
        // 插入的值作为父节点
        array[k] = key;
    }
}
```

* private E dequeue()
功能说明：该函数获取堆的顶点的元素，也就是最小的元素。然后再进行堆化。

```
    private E dequeue() {
        int n = size - 1;
        if (n < 0)
            return null;
        else {
            Object[] array = queue;
            // 获取堆的顶点元素
            E result = (E) array[0];
            // 取数组的最后一个元素，插入到堆的顶点，并进行堆化
            E x = (E) array[n];
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            if (cmp == null)
                siftDownComparable(0, x, array, n);
            else
                siftDownUsingComparator(0, x, array, n, cmp);
            size = n;
            return result;
        }
    }
```


* private void tryGrow(Object[] array, int oldCap)

```
private void tryGrow(Object[] array, int oldCap) {
        lock.unlock(); // must release and then re-acquire main lock
        Object[] newArray = null;
        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {
            try {
                // 分配这么多新的空间
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;
            }
        }
        if (newArray == null) // back off if another thread is allocating
            Thread.yield();
        lock.lock();
        // 复制原来的数组的值到新的数组中
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
```


## 对外函数

* public E take() throws InterruptedException
    * 该函数获取优先队列头部的元素值，并返回
    * 若队列为空，则阻塞等待

```
   public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        E result;
        try {
            while ( (result = dequeue()) == null)
                notEmpty.await();
        } finally {
            lock.unlock();
        }
        return result;
    }
```

* public void put(E e)
该函数在向队列中插入元素时，不会阻塞或等待。因为队列的长度可以无限增加。

```
    public void put(E e) {
        offer(e); // never need to block
    }
```

* public boolean offer(E e)
    * 该函数向队列中添加一个值为e的元素。
    * 若数组已满，调用tryGrow函数动态增加容量。
    * 注意：主要扩容的代价是很大的，因为要进行数组的全部复制。

```
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    // 大于数组的容量，则需要扩容
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);

    // 把元素e插入，并进行堆化
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```

* public boolean add(E e)
添加一个元素到优先队列中，成功返回true。失败返回false。

```
    public boolean add(E e) {
        return offer(e);
    }
```

* public E poll()
从队列头部获取一个元素，并删除该元素。

```
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```


# PriorityBlockingQueue实战

## 生产者消费者模型

使用PriorityBlockingQueue编写的生产者消费者模型。
注意，生产者不会阻塞等待。

```
package BlockingQueue;

import java.util.Random;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;

class PutTask implements Runnable {
    private BlockingQueue queue;
    private Random r = new Random(1000);

    PutTask(BlockingQueue q) {
        queue = q;
    }

    public void run() {
        while (true) {
            try {
                Thread.sleep(2);

                String value = Integer.toString(r.nextInt(1000));
                queue.put(value);
                System.out.println(Thread.currentThread().getName() + " put value: " + value);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class DealTask implements Runnable {
    private BlockingQueue queue;
    private Random r = new Random(10000);

    DealTask(BlockingQueue q) {
        queue = q;
    }

    public void run() {
        while (true) {
            try {
                Thread.sleep(3);
                String o = (String) queue.take();
                System.out.println(Thread.currentThread().getName() + " get value: " + o);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class DoPriorityBQ {
    public static void main(String[] args) {
        final BlockingQueue q = new PriorityBlockingQueue(1000);
        new Thread(new PutTask(q), "producer3").start();
        new Thread(new DealTask(q), "consumer2").start();
        new Thread(new DealTask(q), "consumer1").start();

        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

## 测试排序功能

以下程序测试优先阻塞队列的排序功能。

```
package BlockingQueue;

import java.util.Random;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;


public class DoPriorityBQ2 {

    public static void main(String[] args) throws InterruptedException {
        final BlockingQueue<Integer> queue = new PriorityBlockingQueue<Integer>(10);
        Random r = new Random(100);

        for (int i = 0; i < 10; i++) {
            int vi = r.nextInt(100);
            System.out.print(" " + vi);
            queue.put(vi);
        }

        System.out.println("\n");

        Integer v;
        while ((v = queue.take()) != null) {
            System.out.print(" " + String.valueOf(v));
        }

    }
}

```
程序解读：
* 该程序会先在队列中插入一些随机数，然后再每次取队列的头节点元素，并输出。
* 可以看到，输出的结果是排好序的。
* 该程序没有使用多线程，主要是为了测试程序的排序功能。

# PriorityBlockingQueue的总结
* 该队列会自动对插入的元素进行排序，也就是说：在读取队列元素时，该队列已经排好序了。
* PriorityBlockingQueue是使用数组实现的二叉堆，使用对排序算法实现。
* 由于队列会自动扩容，所以队列的写入元素方法不会等待，若应用中需要很高的并发，建议不要使用该队列，而是使用ArrayBlockingQueue。
* 由于队列的扩容操作代价很大 ：会申请一个新的队列，并把元素复制到新队列中，所以在实际场景中，应该预先规划好队列的荣容量大小。
* 该队列可能会对每个插入的元素进行堆的调整，所以时间复杂度是O(1)~O(lgn)。