
# 介绍
在java5.0以后添加了一种新的同步机制：Lock接口。Lock接口提供了一组抽象的加锁操作。与内置的加锁机制有以下不同：

- 无条件的
- 可轮训的
- 定时的
- 可中断的
- 所有加锁和解锁的方法都是显示的

## 内置锁(synchronized)的功能缺陷
为什么还需要创建显示锁呢？因为内置锁不是太灵活，有一些功能缺陷：

- 法中断一个正在等待获取锁的线程。
- 无法在请求一个锁时无限等待下去。
- 虽然内置锁不需要显示的解锁操作，但是他却无法实现非阻塞的加锁机制。


## Lock接口
### Lock接口介绍
Lock接口实现了一组抽象的加锁操作。比起synchronized，它提实现了一组更加具有可扩展性的操作。它允许更加灵活的结构，可能设置不同的属性，并且可以支持多个相关的Condition对象。
一个锁是用来控制多线程访问共享资源的工具。在一般情况下，锁是排他性的，也就是同一时候只能有一个线程访问共享资源。
有的锁也提供多个线程同时访问共享资源，比如：ReadWriteLock，他可以让多个线程同时读取共享资源。

为了保证和synchronized一样能够自动释放锁，使用Lock的一般模式是:

```
 Lock l = ...;
 l.lock();
 try {
     // access the resource protected by this lock
 } finally {
     l.unlock();
 }
```

### Lock接口函数

```
* void lock() 
  获取锁。
* void lockInterruptibly() 
  获取锁，若当前线程没有被中断。
Acquires the lock unless the current thread is interrupted. 
* Condition newCondition() 
  返回一个新的条件(Condition)对象，该对象绑定在Lock对象上。
* boolean tryLock() 
  若在调用时该锁是free的，则获取该锁。
* boolean tryLock(long time, TimeUnit unit) 
  如果在给定的等待时间内空闲，并且当前线程未被中断，则获取该锁。
* void unlock() 
  释放锁。
```

## ReentrantLock锁
### ReentrantLock介绍
* ReentrantLock锁被成功获取锁，且没有解锁的线程拥有。
* 当一个线程没有拥有该锁，调用lock方法最终成功时，会返回。若线程已经拥有该锁，则该线程将会立即返回。
* 该锁有一个构造函数，该构造函数的参数用来指定是否接收公平调度。公平调度的意思是，若是有多个线程都在等待该锁，则等待事件最长的线程获得该锁。
* 使用该锁的一般形式是:

```
class X {
   private final ReentrantLock lock = new ReentrantLock();
   // ...

   public void m() {
     lock.lock();  // block until condition holds
     try {
       // ... method body
     } finally {
       lock.unlock()
     }
   }
 }
```
注意：一定要在finally中进行显示的释放锁，这是和内置锁synchronzied不同的地方，虽然很简单，但很容易忘记。

### ReentrantLock类的主要方法

* public boolean tryLock()
	* 在调用时不被另一个线程占用时，获取该锁，并立即返回true，同时把锁的持有计数(holdCount)+1。
	* 即使在创建该锁时，设置了一个公平策略参数，当该函数获取锁可用时，该函数也会立即返回，不管其他线程是否也在等待该锁。
	* 如果当前线程已经获取了该锁，则持有数计数增加1，该方法返回true。
	* 若该锁被其他线程持有，该方法立即返回false。

* public boolean tryLock(long timeout,  TimeUnit unit) throws InterruptedException
	* 如果在给定的等待时间内没有被另一个线程占用并且当前线程未被中断，则获取锁。
	* 如果此锁已设置为使用合理的排序策略，则如果其他线程正在等待锁定，则不会获取可用的锁。这与tryLock（）方法相反。如果你想要一个定时的tryLock，允许在公平的锁上进行驳船，那么将定时和非定时的形式组合在一起：

	```
	if (lock.tryLock() || lock.tryLock(timeout, unit) ) { ... }
	```

* protected Collection<Thread> getQueuedThreads()
  * 返回一个包含可能正在等待获取该锁的线程的集合。
  * 因为在构建此结果时，实际的线程集可能会动态更改，所以返回的集合只是尽力而为的估计。 
  * 返回的集合的元素没有特定的顺序。 
  * 该方法旨在便于构建提供更广泛监控设施的子类。

* public int getWaitQueueLength(Condition condition)
  * 返回与此锁相关联的给定条件等待的线程数的估计。 
  * 请注意，由于超时和中断可能在任何时候发生，估计仅作为实际服务员人数的上限。 
  * 该方法设计用于监视系统状态，不用于同步控制。

* protected Collection<Thread> getWaitingThreads(Condition condition)
  * 返回包含可能在与此锁相关联的给定条件下等待的线程的集合。
  * 因为在构建此结果时，实际的线程集可能会动态更改，所以返回的集合只是尽力而为的估计。 返回的集合的元素没有特定的顺序。
  * 该方法旨在便于构建提供更广泛的状态监测设施的子类。

* public boolean hasWaiters(Condition condition)
  * 查询是否有任何线程正在等待与此锁相关联的给定条件。 
  * 请注意，由于超时和中断可能会在任何时间发生，真正的返回并不能保证未来的信号会唤醒任何线程。
  * 该方法主要用于监视系统状态。
* public final boolean hasQueuedThread(Thread thread)
  * 查询给定的线程是否等待获取此锁。 
  * 请注意，因为取消可能会在任何时候发生，所以真正的返回并不能保证此线程将获得此锁。 该方法主要用于监视系统状态。


### ReentrantLock锁实战

#### 一个简单的使用ReentrantLock锁的例子

```
package PlayReentrantLock;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Created by hover.
 *
 * 说明：
 * 演示Reentrant的基本用法，可以看到和synchronized的效果相同。
 * 这里的效果是：输出的字母或数字是连续的，没有交叉输出，说明是原子执行的。可以把锁去掉看看反面的效果。
 *
 */
public class SimpleUse {

    public static ReentrantLock rtlock = new ReentrantLock();

    public static void main(String[] args) {
        // run test method
        SimpleUse s = new SimpleUse();
        s.testRTLock();
    }

    private void testRTLock() {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        executor.execute(new PrintChar('s', 1000));
        executor.execute(new PrintNum(1000));
        executor.execute(new PrintChar('b', 1000));

        executor.shutdown();
    }

    // inner class: just print char
    class PrintChar implements Runnable {
        private char aChar;
        private int times;

        PrintChar(char c, int t) {
            aChar = c;
            times = t;
        }

        public void run() {
            rtlock.lock();
            try {
                for (int i = 0; i < times; i++) {
                    System.out.print(aChar);
                }
            } finally {
                rtlock.unlock();
            }

        }
    }

    // inner class: just print number
    class PrintNum implements Runnable {
        private int lastNum;

        public PrintNum(int num) {
            lastNum = num;
        }

        public void run() {

            rtlock.lock();
            try {
                for (int i = 0; i <= lastNum; i++) {
                    System.out.print(" " + i);
                }
            } finally {
                rtlock.unlock();
            }
        }
    }
}

```

对该程序的说明:

* 该程序使用了线程池的服务接口ExecutorService，该接口的原理和实现会专门有章节进行讲解。
* 注意加锁的地方，我们把打印一长串字符或数字的地方进行了加锁，若不加锁，会产生交叉打印数字和字母的现象，可以自己试试。
* 这里的锁定义是一个静态变量，在该例中，若不是静态变量也不会有问题，因为是通过主类对象运行的。
* 虽然输出的字母和数值都是连续的，但整体的顺序不可预知，这是由线程的调度决定的（谁先持有该锁，谁先运行）。比如：
	结果可能是： 
	* sssssssssss...bbbbbbbbb...1 2 3 4...1000
	也可能是:
	* bbbbbbbbbb...sssssssss...1 2 3 4...1000
	或
	* 1 2 3 4 5 ... 1000 ...sssssssss...bbbbbbbb...bbb

对于ReentrantLock，更多的时候还是与条件变量(Condition)结合使用。


## Condition
### Condition要点
通过将它们与使用任意锁(Lock)的实现相结合，Condition将对象监视方法 (wait, notify and notifyAll) 转换为不同的对象来给出每个对象具有多个等待集的效果。Lock替换了synchronized方法和语句，而Condition替换Object监视器方法的使用。

Condition(也称为条件变量)提供了一组方法，该组方法可以让线程阻塞（等待）直到另一个线程通知某种状态为真。
因为访问这些共享状态信息发生在不同的线程中，所以它们必须被保护，因此某种形式的锁与该条件相关联。 等待条件的关键属性是：它原子上释放相关的锁并挂起当前线程，就像Object.wait一样。

Condition对象本质上被绑定到锁，要获取Condition实例对象，先要获取Lock对象，然后使用其newCondition()方法来获取条件变量。

官方提供了一个有界buffer的例子:

```
class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition(); 
   final Condition notEmpty = lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length)
         notFull.await();
       items[putptr] = x;
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }

   public Object take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0)
         notEmpty.await();
       Object x = items[takeptr];
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
   }
 }

```
对于以上程序要注意：
* 以上程序实际上实现了一个环形缓冲区，请看以下实现：
	```
	// 当保存元素的位置和buffer的长度相等，则回绕到buffer的第0个位置
	if (++putptr == items.length) putptr = 0;
	```
* 在放数据的时候，若数据的个数已经和buffer的长度相等，则需要等待：
	```
	while (count == items.length)
         notFull.await();
	```
* 同理，当元素的个数为0时，取数据需要等待:
	```
	while (count == 0)
         notEmpty.await();
	```
* 官方批注：（ArrayBlockingQueue类提供此功能，因此没有理由实现此示例使用类。）
	
	* 看了一下ArrayBlockingQueue的实现（专门有章节分析），的确实现了该示例代码的逻辑。可以直接使用，这也是java比较方便的地方。

### Condition设计上的考虑
这里的内容一部分来自于官方。

* 等待条件时，允许发生“虚假唤醒”，一般来说，作为对底层平台语义的让步。这对大多数应用程序几乎没有实际的影响，因为在循环中应始终等待条件，测试正在等待的状态谓词。一个实现可以消除虚假唤醒的可能性，但建议应用程序员总是假定它们可以发生，因此总是等待循环。
	* 注意：虚假唤醒的意思是，await函数返回，但线程还不可运行。


### Condition的主要方法

* void await() 
强制调用线程等待，直到它被信号通知到或被中断。调用该系列函数

* boolean await(long time, TimeUnit unit) 
使得当前线程等待，直到该线程被信号(signal)通知，或被终端，或指定的等待时间过去。

* long awaitNanos(long nanosTimeout) 
和上面的函数相同，不过这里的时间单位是纳秒。

* void awaitUninterruptibly() 
使得当前线程等待，直到该线程被信号(signal)通知，

* boolean awaitUntil(Date deadline) 
使得当前线程等待，直到该线程被信号通知，或被终端，或到达指定的最后期限。

* void signal() 
唤醒等待线程。

* void signalAll() 
唤醒所有的等待线程。

各个方法的设计要点说明：
* 在调用await系列函数时，和该锁关联的Condition会自动释放，线程将阻塞而不能被调度使用。直到发生以下事件：
  * 其他线程通过调用signal方法唤醒该Condition，而目前的线程正好被选中唤醒。
  * 其他线程通过调用signalAll方法唤醒该Condition。
  * 目前的线程被其他线程中断
  * 发生“虚假唤醒”

在所有情况下，在此方法返回之前，当前线程必须重新获取与此条件相关联的锁。 当线程返回时，保证保持此锁。
  * 若目前的线程，在进入该方法时设置了中断状态;
  * 若当前线程，在等待和中止线程挂起时被中断，然后抛出InterruptedException并清除当前线程的中断状态。
    在第一种情况下，没有规定在释放锁之前是否发生中断测试。
  * 实现的考虑：
    * 会假设在调用该方法时，已经持有与该Condition关联的锁。如果没有，可能会抛出异常。


### Condition + ReentrantLock编程实战

#### 基于环形缓冲区的多生产者消费者模型

```
package PlayReentrantLock;

import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ProductorConsumer {
    // task queue，这里的任务队列使用的是数组，这样不需要每次都分配内存，减少开销
    private static String[] taskQueue = new String[1000];

    // write and read lock
    private static Lock rtlock = new ReentrantLock();
    private static final Condition notFull = rtlock.newCondition();
    private static final Condition notEmpty = rtlock.newCondition();

    // 需要记录读和写的位置
    private static int gpos = 0;   // get pos
    private static int ppos = 0;   // put pos
    private static int count = 0;  // item count

    private Random r = new Random(10000);

    public static void main(String[] args) {
        ProductorConsumer  pc = new ProductorConsumer();
        pc.workproc();
    }

    public void workproc() {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        executor.execute(new Productor());
        executor.execute(new Productor());
        executor.execute(new Consumer());
        executor.shutdown();
    }

    // inner class productor
    private class Productor implements Runnable {
        public void run() {
            for ( ; ; ) {
                rtlock.lock();
                try {
                    taskQueue[ppos] = Integer.toString(r.nextInt(100000));

                    // put item
                    System.out.println(Thread.currentThread().toString() +
                            " put item: [" + ppos + "] = " + taskQueue[ppos]);

                    // 队列满了，回绕到0位置继续放数据，此时会把以前老的数据覆盖
                    if (++ppos == taskQueue.length)
                        ppos = 0;
                    count++;
                    notEmpty.signalAll();
                } finally {
                    rtlock.unlock();
                }

                try {
                    Thread.sleep(r.nextInt(10)*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    // inner class consumer
    private class Consumer implements Runnable {
        public void run() {
            for (; ; ) {
                rtlock.lock();
                try {
                    while (count == 0)
                        notEmpty.await();

                    System.out.println(Thread.currentThread().toString() + " get item " +
                            Integer.toString(gpos) + "  " + taskQueue[gpos]);

                    // roll back
                    if (++gpos == taskQueue.length)
                        gpos = 0;
                    --count;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    rtlock.unlock();
                }

                try {
                    Thread.sleep(r.nextInt(10)*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

程序解读：

* 该程序使用ReentrantLock类的锁和条件变量实现了一个，多生产者和多消费者模型。
* 可以看到以上模型，的生产者是不会等待的，会不断的向任务队列中放任务，即使任务队列已经满了。
* 在生产者放任务时，若到达了队列尾部，会回绕到队列头部继续放任务。
* 以上模型的任务队列被设计成一个数组而不是其他可以动态增加删除节点的结构，这是出于性能的考虑。

为什么要设计成以上模型？是有考虑的：
我们设想一个场景：一个http服务器的设计，可能会通过一个线程来接收请求，并把请求放到一个内存任务队列中，并通知处理请求的线程池，处理请求的线程池接收到请求，从队列处理请求。
在这个场景中，若设计成队列满了就等待的模型，当队列满时新的请求进来会等待。而若按以上设计，则不会等待。到底使用哪一种，需要根据实际需要权很。
再则，若真的到了队列满还处理不完的情况，可能就不是单个服务能解决的，应该使用分布式服务。


#### 基于环形缓冲区的多生产者消费者模型2

以下程序解读：
* 该程序和上一个模型相似，但若队列满了，该程序的模型会让生产者阻塞。这样，有些网络服务器端，就会发生请求的丢失。
* 为了不产生任务丢失，任务队列的size需要根据实际情况进行设计。
* 在以下测试中，生产者和消费者的线程个数可以任意调整，不会影响该模型的运行。

```
package PlayReentrantLock;


import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;


public class ProduceConsumerBlocking {
    // task queue
    private static String[] taskQueue = new String[1000];

    // write and read lock
    private static Lock rtlock = new ReentrantLock();
    private static final Condition notFull = rtlock.newCondition();
    private static final Condition notEmpty = rtlock.newCondition();

    private static int gpos = 0;   // get pos
    private static int ppos = 0;   // put pos
    private static int count = 0;  // item count

    private Random r = new Random(10000);

    public static void main(String[] args) {
        ProductorConsumer  pc = new ProductorConsumer();
        pc.workproc();
    }

    public void workproc() {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        executor.execute(new Productor());
        executor.execute(new Productor());
        executor.execute(new Consumer());
        executor.execute(new Consumer());

        executor.shutdown();
    }

    // inner class productor
    private class Productor implements Runnable {
        public void run() {
            for ( ; ; ) {
                rtlock.lock();
                try {
                    // 若任务队列已经满了，则等待
                    while (count == taskQueue.length)
                        notFull.await();

                    taskQueue[ppos] = Integer.toString(r.nextInt(100000));

                    // put item
                    System.out.println(Thread.currentThread().toString() +
                            " put item: [" + ppos + "] = " + taskQueue[ppos]);

                    // 队列满了，回绕到0位置继续放数据，此时会把以前老的数据覆盖
                    if (++ppos == taskQueue.length)
                        ppos = 0;
                    count++;

                    // 已经放入了一个元素，发送队列不空的信号
                    notEmpty.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    rtlock.unlock();
                }

                try {
                    Thread.sleep(r.nextInt(10)*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    // inner class consumer
    private class Consumer implements Runnable {
        public void run() {
            for (; ; ) {
                rtlock.lock();
                try {
                    while (count == 0)
                        notEmpty.await();

                    System.out.println(Thread.currentThread().toString() + " get item " +
                            Integer.toString(gpos) + "  " + taskQueue[gpos]);

                    // roll back
                    if (++gpos == taskQueue.length)
                        gpos = 0;
                    --count;
                    // 已经取走了一个元素，此时发送队列已满的信号
                    notFull.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    rtlock.unlock();
                }

                try {
                    Thread.sleep(r.nextInt(10)*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

## 总结
* ReentrantLock显示的实现了java内部的锁机制，它使得加锁更加灵活。
* ReentrantLock提供公平加锁的算法，可以在需要的时候进行设置。
* ReentrantLock也有一定的弊端：
  * 在使用的时候必须要显示的解锁，并添加try{}语句块，形式如下：
  ```
  public void m() {
     lock.lock();  // block until condition holds
     try {
       // ... method body
     } finally {
       lock.unlock()
     }
   }
  ```