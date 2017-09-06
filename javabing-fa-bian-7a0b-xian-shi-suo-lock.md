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
