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


