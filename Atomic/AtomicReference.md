2.AtomicReference



AtomicReference是一个对象引用变量，在对该变量进行读写时会保证其原子性。也就是说，当多个线程同时对AtomicReference变量进行读写时，不会导致变量值不一致。

 AtomicReference还提供了一个方便使用的compareAndSet()方法，它可以将引用与预期值（引用）进行比较，如果它们相等，则在AtomicReference对象内设置一个新的引用。



AtomicReference的实现原理



AtomicReference的基本使用



AtomicReference的使用



AtomicReference的





参考：

* https://www.jianshu.com/p/5521ae322743