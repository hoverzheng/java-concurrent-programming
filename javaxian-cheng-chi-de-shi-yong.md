
# 线程池简介

在线程池类ThreadPoolExecutor()源码的注释中写到了设计线程池的原因。线程池主要解决两个问题：

* 一方面当执行大量异步任务时候线程池能够提供较好的性能。这是因为使用线程池可以减少每个任务的调用开销（因为线程池的线程是可复用的）。
* 另一方面线程池提供了一种资源限制和管理的手段。比如当执行一系列任务时候对线程的管理，每个ThreadPoolExecutor也保留了一些基本的统计数据，比如当前线程池完成的任务数目。


# 线程池的使用

## 参数说明
每个线程池的创建都会设置一些参数，这些参数对于理解线程池的运作机制是非常重要的，必须要理解这些参数的用法。下面对这些参数进行说明。
各种类型的线程池都是通过：ThreadPoolExecutor来进行创建的，该类的申明如下：

```

ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue)

参数说明；
corePoolSize: 是线程池中活动线程的数量，不管他们是否是空闲的。
maximumPoolSize: 线程池中允许的最大线程数量。
keepAliveTime: 当线程数大于corePoolSize数时，空闲线程终止前等待新任务的最大时间。
TimeUnit: keepAliveTime的时间单位。
workQueue: 当任务数大于线程池的最大线程数时，会先把任务放到阻塞队列中，当线程池中有空闲线程时，会在该队列中取一个任务，并把任务放到该线程中执行。

```

## 线程池实现的总体原理

[task1]\                                            
[task2]--->[task|task|task|task|task|task|task|task]--->[threadpool]
[task3]/                   taskqueue                
...

通过线程池来完成多线程的程序，可以减少每次都要创建一个线程的开销（因为有些线程池是一次创建多个线程），减少对线程的维护操作。
在创建线程池并向线程池中添加任务时(调用execute(Runnable task))，线程池会为每个任务创建一个线程并使用该线程来运行整个任务，当任务数大于线程池的线程个数时，会把这些任务添加到一个阻塞队列中。当线程池中的线程某个任务完成时，线程不会销毁，而是从阻塞队列中获取一个任务，并使用beforeExecute(wt, task);函数把任务挂载到这个线程上执行。这样就减少了线程创建的开销。


## newFixedThreadPool线程池的使用

### 说明

```
   public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```

通过以上的函数实现可以看出，这种类型的线程池的corePoolSize和maximumPoolSize参数值都是nThreads。这意味着什么呢？
意思是，在线程池中保持nThreads个活动线程，而线程池中允许的最大线程数也是nThreads。也就是说常驻线程和总线程数是相同的，这样线程运行完成后将不会被回收，因为要保持线程池中有nThreads个活动的线程。


### 实战

```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * Created by hover on 2017/11/22.
 */

class WorkerThread implements Runnable {
    private String s;

    public WorkerThread(String s) {
        this.s = s;
    }

    public void run() {
        System.out.println(Thread.currentThread().getName() + " (Start) message : " + s);
        processCommand();
        System.out.println(Thread.currentThread().getName() + " (End)");
    }

    private void processCommand() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


public class doFixThreadPool {
    public static void main(String[] args) {

        // 创建一个newFixedThreadPool类型的线程池
        ExecutorService executor = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 10; i++) {
            Runnable worker = new WorkerThread("" + i);
            executor.execute(worker);
        }

        executor.shutdown();
        while (!executor.isTerminated()){   }
        System.out.println("Good Bye!");
    }
}

```

### 实战输出分析

若编译运行以上代码，输出结果如下：

```
pool-1-thread-1 (Start) message : 0
pool-1-thread-5 (Start) message : 4
pool-1-thread-3 (Start) message : 2
pool-1-thread-4 (Start) message : 3
pool-1-thread-2 (Start) message : 1     // 前五个任务立即执行
pool-1-thread-1 (End)                   // 等到其中一个任务结束，注意：线程不会结束，取一个剩下的线程使用该线程的资源。
pool-1-thread-5 (End)
pool-1-thread-1 (Start) message : 5
pool-1-thread-5 (Start) message : 6
pool-1-thread-3 (End)
pool-1-thread-3 (Start) message : 7
pool-1-thread-4 (End)
pool-1-thread-4 (Start) message : 8
pool-1-thread-2 (End)
pool-1-thread-2 (Start) message : 9
pool-1-thread-1 (End)
pool-1-thread-5 (End)
pool-1-thread-3 (End)
pool-1-thread-4 (End)
pool-1-thread-2 (End)
Good Bye!
```

从以上输出可以看出：我们给一个newFixedThreadPool(5)的线程池10个任务，必然会有任务排队。可以看到:
* 前5个任务在线程启动阶段立即执行了，而后面的任务进入任务队列中。
* 当线程池中的线程执行完一个任务时，线程不会被销毁，线程池会从任务队列中取一个任务，并把任务交给该线程，利用该线程的资源继续执行。
* 线程池中始终有5个活动线程，也是最大的线程数，当所有任务都执行完成后，线程池中的线程才被全部回收。

