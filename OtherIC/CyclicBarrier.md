# CyclicBarrier
java.util.concurrent.CyclicBarrier类是一种同步机制，它允许一组线程等待其他所有线程到达共同屏障点。CyclicBarriers在涉及固定大小的线程方的程序中非常有用，这些线程必须偶尔相互等待。CyclicBarrier被称为循环，因为它可以在等待的线程被释放之后重新使用。
换句话说，所有线程必须等待，直到所有线程到达之前，才能继续任何线程。以下是图示：

<img src="./assets/cyclic-barrier.png" alt="CyclicBarrier" style="zoom:75%;" />

