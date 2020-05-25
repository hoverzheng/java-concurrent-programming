
# BlockingQueue接口
## 介绍
java.util.concurrent包中的Java BlockingQueue接口表示一个线程可以安全地插入并从中获取实例的队列。

## BlockingQueue接口要点
* 支持当队列变为非空时取回元素，也支持当队列有剩余空间时往队列中存放元素。
* BlockingQueue可能是容量限制的。 在任何给定的时间，它可能具有剩余容量，超过这个容量不能在没有阻塞的情况下放置其他元素。
* 没有任何内在容量约束的BlockingQueue总是报告Integer.MAX_VALUE的剩余容量。
* BlockingQueue不接受null元素。在add，put，或offer一个空元素时，将会抛出NullPointerException异常。
* 使用null作为哨兵值来指示轮询操作失败。
* BlockingQueue实现被设计为主要用于生产者 - 消费者队列，但另外支持Collection接口。
* 因此，例如，可以使用remove（x）从队列中删除任意元素。但这样的操作通常不能非常有效地执行，并且仅用于偶尔使用，诸如当排队的消息被取消时。

&emsp;&emsp;BlockingQueue的实现是线程安全的。所有排队方法都使用内部锁或其他形式的并发控制来实现其效果。 但是，批量Collection 操作addAll，containsAll，retainAll和removeAll不一定以原子方式执行，除非在实现中另有指定。因此，例如，在c中添加部分元素后，addAll(c)可能失败（抛出异常）。

&emsp;&emsp;BlockingQueue不支持严格意义上的close或shutdown操作，来表示再没有元素会添加到队列中。这些功能的需求和使用往往依赖于实现。例如，一个常见的策略是生产者插入特殊的流结尾符(end-of-stream)和位置对象，在消费者使用时会被相应地解释。

&emsp;&emsp;使用示例代码：基于典型的生产者-消费者场景。请注意，BlockingQueue可以安全地与多个生产者和多个消费者一起使用。

&emsp;&emsp;还可以访问BlockingQueue中的所有元素，而不仅仅是开始和结束处的元素。例如，假设您已排队处理对象，但是您的应用程序决定取消它。然后，您可以调用remove（o）删除队列中的特定对象。但是，这并不是非常有效，所以你不应该使用这些Collection方法，除非你真的必须


以下代码是官方的示例：

```
class Producer implements Runnable {
   private final BlockingQueue queue;
   Producer(BlockingQueue q) { queue = q; }
   public void run() {
     try {
       while (true) { queue.put(produce()); }
     } catch (InterruptedException ex) { ... handle ...}
   }
   Object produce() { ... }
 }

 class Consumer implements Runnable {
   private final BlockingQueue queue;
   Consumer(BlockingQueue q) { queue = q; }
   public void run() {
     try {
       while (true) { consume(queue.take()); }
     } catch (InterruptedException ex) { ... handle ...}
   }
   void consume(Object x) { ... }
 }

 class Setup {
   void main() {
     BlockingQueue q = new SomeQueueImplementation();
     Producer p = new Producer(q);
     Consumer c1 = new Consumer(q);
     Consumer c2 = new Consumer(q);
     new Thread(p).start();
     new Thread(c1).start();
     new Thread(c2).start();
   }
 }

```

## BlockingQueue方法说明
&emsp;&emsp;BlockingQueue有四组不同的方法来插入，删除和检查队列中的元素。在所请求的操作无法立即执行的情况下，每组方法的行为不同。这是一个表的方法：


|  method   |Throws Exception|	Special Value|	Blocks  |	Times Out 				|
|     -     |    -           |       -       |    -     |      -                    |
|Insert     |add(o)			 |	offer(o)	 |	put(o)	|offer(o, timeout, timeunit) |
|Remove	    |remove(o)		 |	poll()		 |	take()	|poll(timeout, timeunit)|
|Examine    |element()		 |	peek()		 | 	 		| |


这四种不同的行为意味着：

* 抛出异常：
	如果尝试的操作不可能立即，抛出异常。
* 特殊值：
	如果尝试的操作不可能立即执行，则返回一个特殊值（通常为true / false）。
* 阻塞：
	如果尝试的操作不可能immedidately，方法调用阻塞，直到它被通知。
* 超时：
	如果尝试的操作不可能immedidately，方法调用阻塞，直到它条件满足，但不等于给定的超时。
返回一个特殊的值来告诉操作是否成功（通常为true / false）。不能将null插入到BlockingQueue中。

如果尝试插入null，BlockingQueue将抛出NullPointerException。

# ArrayBlockingQueue
## ArrayBlockingQueue要点
* 是有数组实现的有界的阻塞队列。
* 该队列按先进先出(FIFO: first-in-first-out)排列元素。
* 队列的头部是队列中最老的元素，队列尾部的是队列中时间最短的元素。
* 新元素插入队列的尾部，队列检索操作获取队列头部的元素。

* 这是一个经典的“有界缓冲区”，其中固定大小的数组保存由生产者插入的元素并由消费者提取。
* 创建后，容量无法更改。尝试将元素放入已满的队列将导致操作阻塞;尝试从空队列中获取元素将同样阻止。
* 此类支持一个可选的公平策略用来对等待的生产者和消费者进行排序。
* 默认情况下该顺序是不保证的，但在构建队列时可以设置公平策略：所有等待线程按FIFO的顺序访队列。但公平策略可能会降低队列的性能。

## ArrayBlockingQueue函数说明
* boolean add(E e)
	* 若目前的元素个数没有超过队列的容量，则可能立即把指定的元素插入到队列的尾部。
	* 如果此队列已满，则返回true并抛出IllegalStateException。
	* 成功返回true，失败返回false
* public boolean offer(E e)
	* 该函数在队列的尾部插入指定元素，如果元素个数没有超过队列容量，可能会立即执行，成功返回true，失败返回false。
	* 该方法通常比方法add(E)更好，因为当插入失败时add(E)只能通过抛出异常来插入元素。
* public E take()
	* 返回并删除队列的头部元素，可能会等待直到有元素可用。


## ArrayBlockingQueue实现原理

### 类申明和变量定义

```
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /**
     * Serialization ID. This class relies on default serialization
     * even for the items array, which is default-serialized, even if
     * it is empty. Otherwise it could not be declared final, which is
     * necessary here.
     */
    private static final long serialVersionUID = -817911632652898426L;

    /** The queued items */
    // 注意：这里是Objectd的数组，也就是说队列的长度不会变化
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    // 下一个读取(获取)元素的位置
    int takeIndex;

    /** items index for next put, offer, or add */
    // 下一放(写)元素的位置
    int putIndex;

    /** Number of elements in the queue */
    // 目前队列的元素个数
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    // 使用ReentrantLock来进行队列的同步
    final ReentrantLock lock;
    /** Condition for waiting takes */

    // 使用ReentrantLock的条件变量来进行线程间的同步
    private final Condition notEmpty;
    /** Condition for waiting puts */
    private final Condition notFull;
```

### 构造函数

```
   public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

说明：
* 阻塞队列的同步是通过ReentrantLock来实现的。
* 阻塞队列的线程公平调度策略是通过ReentrantLock的公平调度来实现的。


### 函数实现

#### 内部函数
* inc(int i)函数
该函数是一个工具函数，它会把位置参数加1，若位置到达队列尾部，回绕到队列的第一个元素的0位置。
注意：这是典型的环形缓冲区的实现方式。

```
   /**
     * Circularly increment i.
     */
    final int inc(int i) {
        return (++i == items.length) ? 0 : i;
    }
```

* private void insert(E x)
该函数向队列中添加一个元素，若到达队列的末尾直接绕回到队列的第一个位置，也就是索引为0的位置，继续放元素。
注意：该函数不会管队列是否已满，若队列已满该函数不会阻塞，会继续向putIndex写入元素。若队列的该位置已有数据，会覆盖该位置的数据。

```
   /**
     * Inserts element at current put position, advances, and signals.
     * Call only when holding lock.
     */
    private void insert(E x) {
    	// 不管队列是否已满，直接把数据放入到putIndex位置，若该位置有数据，则覆盖原来位置的数据。
        items[putIndex] = x;
        // 计算下一个放数据的位置
        putIndex = inc(putIndex);
        // 现有元素的个数+1
        ++count;
        // 发送notEmpty信号，通知在该条件变量上等待的线程，已经有数据放入队列。
        notEmpty.signal();
    }
```

* checkNotNull：空元素检查
阻塞队列的元素不能为空。

```
    private static void checkNotNull(Object v) {
        if (v == null)
            throw new NullPointerException();
    }
```

* private E extract()函数
功能：
	* 从位置takeIndex处，取出一个元素，并把位置的数据置为null。
	* takeIndex的位置到达队列尾部，也会回绕到位置0。

```
    private E extract() {
        final Object[] items = this.items;
        // 取出队列中takeIndex位置的数据
        E x = this.<E>cast(items[takeIndex]);
        // 数据已经取出，需要把该位置的数据重置为Null
        items[takeIndex] = null;
        // 计算下一个放数据的位置
        takeIndex = inc(takeIndex);
        // 元素个数-1
        --count;
        // 通知在notFull条件变量等待的线程
        notFull.signal();
        return x;
    }
```

#### 对外函数

1.offer(o) 和 poll()
这两个函数提供了ArrayBlockingQueue非阻塞的操作。

* public boolean offer(E e)函数
功能：该函数向队列中添加一个元素，若队列已满，直接返回false。
注意：官方文档说是向队列尾部插入一元素，从实现上来看，这个队列尾部（不是正真的队列尾），这是一个相对位置。

```
    public boolean offer(E e) {
    	// 检查要放入的元素e是否为空，若为空抛出异常
        checkNotNull(e);

        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
        	// 若队列中的元素个数和队列长度相等，说明队列已满，该函数直接返回false。
            if (count == items.length)
                return false;
            else {
            // 若队列没有满，调用insert插入元素。
                insert(e);
                return true;
            }
        } finally {
        	// 无论如何，都要释放锁
            lock.unlock();
        }
    }
```

* public E poll()函数
若队列为空，直接返回null。

```
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : extract();
        } finally {
            lock.unlock();
        }
    }
```

2.put(o) 和 take()
这一对函数实现了一个经典的生产者消费者模型。队列满或队列为空时，都会阻塞等待。

* public void put(E e) throws InterruptedException函数
功能：向队列中添加一个元素，若队列满了，则等待。
注意：插入的位置，也不是严格意义上的队列尾部，也是相对的。因为这是一个环形缓冲区。

```
   public void put(E e) throws InterruptedException {
    	// 检查要放入的元素e是否为空，若为空抛出异常
        checkNotNull(e);

        final ReentrantLock lock = this.lock;
        // 这里调用方法是可中断的lock
        lock.lockInterruptibly();
        try {
        	// 若队列满了，则等待
            while (count == items.length)
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
    }
```

* public E take() throws InterruptedException函数
功能：
	* 从队列中获取一个元素，若队列为空，阻塞，直到队列不为空。
	* 要注意：这里通知的是一个线程，而不是多个。

```
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
        	// 队列为空，等待
            while (count == 0)
                notEmpty.await();
            // 到这里说明队列已经不为空了，而且轮到该线程执行了。
            return extract();
        } finally {
        	// 永远记住，锁用了要释放
            lock.unlock();
        }
    }
```
说明：可以看出take()和put可以配对使用。因为他们都会阻塞。他们配合使用，形成了经典的环形缓冲区的实现。


3.offer(o, timeout, timeunit) 和 poll(timeout, timeunit)

* public E poll(long timeout, TimeUnit unit) 
在获取队列中的元素时，会先等待一会，直到超时。

```
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
        	// 队列空
            while (count == 0) {
            	// 若等待被中断，会意外发生
                if (nanos <= 0)
                    return null;
                // 等待一定时间
                nanos = notEmpty.awaitNanos(nanos);
            }
            return extract();
        } finally {
            lock.unlock();
        }
    }
```

* public boolean offer(E e, long timeout, TimeUnit unit)
在向队列中写数据时，会先等待一会信号，若超时则继续检查条件。

```
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
        	// 队列满了
            while (count == items.length) {
            	// 若等待被中断
                if (nanos <= 0)
                    return false;
                // 等待一段时间
                nanos = notFull.awaitNanos(nanos);
            }
            insert(e);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

## ArrayBlockingQueue实践

以下代码是通过ArrayBlockingQueue实现的多生产者消费者模型。可以看出，由于所有的同步都有该类进行了封装，通过类实现的生产者消费者模型非常简洁。

```
package BlockingQueue;

import java.util.Random;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;


// 消费者类
class Consumer implements Runnable {
    private final BlockingQueue queue;
    Random r = new Random(1000);

    Consumer(BlockingQueue queue) {
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

// 生产者类
class Producer implements Runnable {
    private final BlockingQueue queue;
    Random r = new Random(100000);

    Producer(BlockingQueue queue) {
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

public class DoArrayBQ {
    public static void main(String[] args) {
        final BlockingQueue q = new ArrayBlockingQueue(1000);

        new Thread(new Producer(q)).start();
        new Thread(new Producer(q)).start();
        new Thread(new Consumer(q)).start();

        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 对ArrayBlockingQueue的总结
对ArrayBlockingQueue是具有固定容量的阻塞队列。它也提供了非阻塞的操作函数，需要根据实际情况选择使用。
