
## synchronized的原理和使用

### synchronized一般语法
```
synchronized (expr) {
	statements
}
```
注意:这里的expr的值必须是某个对象的引用。而当expr表达式为空时，其实是作用在this指针上的。

### synchronized的原理
  * 要是在某个对象上使用synchronized，那么会先获取这个对象的锁，然后执行方法体，最后释放这个对象上的锁。
  * 注意：无论synchronized的包括的语句中发生任何异常，错误终止，对象上的锁都会被释放。有时候可以利用它的这个特性来进行设计。
  * 注意：synchronized的锁是作用在某个对象上，要是作用在一个类的不同对象上，不同对象上的synchronized是不起作用的。
  * 构造器在创建任何对象时，是单线程的，所以它不需要也不能被声明为synchronized。

### synchronized方法和synchronized语句
* synchronized方法
在方法名前面添加synchronized关键字，表示在整个方法的代码块加锁。加锁的粒度比较粗，大多数情况下并不太合适。

```
// 在方法中使用synchronized，将会在整个方法区加锁。
public synchronized long getBalance() {
	return balance;
}
```

* synchronized语句
把需要同步的语句块，使用synchronized(expr){} 包括起来，实现部分语句块的加锁。使用这种方式，加锁的粒度可以根据自己的需要进行控制，比较灵活。

```
// 对部分代码块进行加锁
public static void doArray(int[] values) {
	synchronized (values) {
		for (int i = 0; i < values.length; i++) {
			if (values[i] < 0)
				values[i] = -values[i];
		}
	}
	// 可以添加其他不需要同步的处理语句
}
```

### 静态synchronized方法
* 可以把synchronized添加到静态方法上，因为每个类都有与其关联的Class对象，静态同步方法就是它所属类的Class对象上的锁。
* 两个线程不能同时执行同一个类的静态同步方法，和两个线程不能同时在同一对象上执行同步方法一样。
* 注意：在静态同步方法中获取Class对象上的锁，不会对这个类的任何对象产生影响，也就是说，当一个线程在执行静态同步方法时，任然可以在这个类的对象上调用并执行该同步方法。

静态synchronized方法的锁和以下代码效果相同：
```
class Body() {
    synchronized(Body.class) {
        idNum = nextID++;
    }
}
```

* 使用静态synchronized的例子：

```
package DoThread;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;


public class DoStaticSync {
    public static void main(String[] args) {
        DoThreadPool();
    }

    // 通过线程池来管理线程
    public static void DoThreadPool() {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        // 提交执行线程
        executor.execute(new DoStaticSync.PrintChar());
        executor.execute(new DoStaticSync.PrintNum());
        executor.shutdown();
    }


    private static class PrintChar implements Runnable {
        public void run() {
            Worker.PrintChars();
        }
    }

    private static class PrintNum implements Runnable {
        public void run() {
            Worker.PrintNums();
        }
    }

    // inner class: worker class
    private static class Worker {
        public final static int times = 1000;

        public static synchronized void PrintChars() {
            for (int i = 0; i < times; i++)
                System.out.print('a');
        }

        public static synchronized void PrintNums() {
            for (int i = 0; i < times; i++)
                System.out.print(""+1);
        }
    }
}
```


### 对synchronized的理解总结
* 首先，java中没有操作系统中的临界区、互斥量的概念，java中的原子操作是通过synchronized关键字来实现。
* java的synchronized()法类似于操作系统概念中的互斥内存块，在JAVA中的Object类型中，都是带有一个内存锁的，在有线程获取该内存锁后，其它线程无法访问该内存，从而实现JAVA中简单的同步、互斥操作

  一个日本作者-结成浩的《java多线程设计模式》有这样的一个列子：
  

```
  pulbic class Something() {
  	public synchronized void isSyncA(){}
  	public synchronized void isSyncB(){}
  	public static synchronized void cSyncA(){}
  	public static synchronized void cSyncB(){}
  }
```

   那么，若有Something类的两个实例a与b，那么下列组方法何以被1个以上线程同时访问呢

```
   a.   x.isSyncA()与x.isSyncB()
   b.   x.isSyncA()与y.isSyncA()
   c.   x.cSyncA()与y.cSyncB()
   d.   x.isSyncA()与Something.cSyncA()
```

    这里，很清楚的可以判断：
   * a，都是对同一个实例的synchronized域访问，因此不能被同时访问
   * b，是针对不同实例的，因此可以同时被访问
   * c，因为是static synchronized，所以不同实例之间仍然会被限制,相当于Something.isSyncA()与   Something.isSyncB()了，因此不能被同时访问。
   * 那么，第d呢?，书上的 答案是可以被同时访问的，答案理由是synchronzied的是实例方法与synchronzied的类方法由于锁定（lock）不同的原因。

因此，通过对类的方法加上synchronized来实现对该类的对象（代表共享区域）的原子操作,防止对多线程对共享数据的修改造成的不确定性，因此多线程中可以通过将要共享的数据和对数据修改的方法封装到一个类中，从而实现多线程中的并行运行，如：多线程中对同一个文件的写入操作。



## wait 和 notify
* wait方法让一个线程等待，直到某个条件出现。
* notify和notifyAll方法，通知正在等待的线程，发生了某个满足条件的事件。
* 这几个方法都是在Object中定义的，所有类都会继承它。

### 函数详细介绍
* public final void notify()
	通知最多一个正在等待条件发生的线程，若有多个线程会任意选择一个线程来唤醒。若不太确定唤醒满足条件的线程，应该使用notifyAll()。
* public final void notifyAll()
	通知所有等待条件发生变化的线程。
* public final void wait() throws InterruptedException
	与wait(0)类似
* public final void wait(long timeout, int nanos) throws InterruptedException
	更细粒度的wait方法，其超时时间是timeout和nanos的和。总时间按纳秒计算为：1000000*timeout+nanos。

### 标准模式
* wait的标准模式：

```
synchronized void downwhenCondition() {
	while (!condition)
		wait();

	// 到这里说明条件已经满足了，可以执行满足条件的代码了
	... ...
}

```

* 需要注意以下几点：

    * 所有的操作都是在同步代码块内执行。
	* 当调用wait()时，会原子性的释放掉对象上的锁。也就是说：调用wait时会原子性的完成两件事情：(1)挂起线程 (2)释放锁。
	* 条件测试始终在循环中进行，注意这里是while不是if。

* 通知模式
	* 通知也是在同步代码中进行的

```
synchronized void changeCondition() {
	// 改变了在条件测试中的值
	notifyAll();
}
```
注意：notify()会唤醒一个线程，而notifyAll()会唤醒所有的等待线程。

### 设计时要注意的点
* 多个线程可能在等待同一个对象，但每个线程等待的条件不同。此时需要使用notifyAll()来唤醒所有线程，因为notify()可能唤醒不满足条件的线程。
* 使用notify()时的情况：
	* 所有线程等待的是同一个条件。
	* 至少有一个线程可以从被满足的条件中获益。
* 当调用Notify或notifyAll时，若没有线程等待，则该通知会被忽略。
* 若某个线程随后决定等待，先前的通知对它不产生任何影响。只有在wait开始后产生的通知才对等待的线程有影响。
* 以上代码都必须在同步代码中调用。


### 用wait和notify实现的生产者消费者模型

下面的代码是用wait和notify实现的多生产者和消费者模型。
```
package DoThread;


import java.util.LinkedList;
import java.util.Random;

class Worker {
    LinkedList<String> JobQueue = new LinkedList<String>();

    Worker() {}

    // consumer
    public class Consumer implements Runnable {
        String item = null;

        public void run() {
            for ( ; ; ) {
                synchronized (JobQueue) {
                    System.out.println("cons get lock");
                    while (JobQueue.size() == 0) {
                        System.out.println("cons wait . release lock");
                        try {
                            JobQueue.wait();
                        } catch (InterruptedException e) {
                            // TODO Auto-generated catch block
                            e.printStackTrace();
                        }
                    }
                    item = JobQueue.remove();
                    System.out.println("consumer get: " + item);
                }

                // wait for a while
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }

        }
    }

    // producer
    public class Producer implements Runnable {
        Random r;
        StringBuffer sb = new StringBuffer();

        Producer() {
            r = new Random();
        }
        public void run() {
            for ( ; ; ) {
                System.out.println("producer loop ");
                synchronized (JobQueue) {
                    System.out.println("producer get lock");
                    sb.delete(0, sb.length());
                    sb.append("").append(r.nextInt(100));
                    System.out.println("put : " + sb.toString());

                    JobQueue.add(sb.toString());
                    JobQueue.notifyAll();
                }

                System.out.println("producer release lock");
                // wait for a while
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }
    }


    // work produce
    public void run() {
        Producer p = new Producer();
        Consumer c = new Consumer();

        new Thread(p).start();
        new Thread(p).start();

        // consumer
        new Thread(c).start();
        new Thread(c).start();
        new Thread(c).start();
    }
}

public class SyncProducerAndConsumer {
    public static void main(String[] args) throws InterruptedException {
        Worker w = new Worker();

        w.run();
        System.out.println("end main");
        Thread.currentThread().join();
    }
}
```
注意这里的synchronized语句块中使用的锁。