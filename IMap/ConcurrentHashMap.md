## 深入浅出ConcurrentHashMap

它是一个线程安全的数据结构，可以同时对该map进行读写操作。

### ConcurrentHashMap要点

* ConcurrentHashMap实现层面的数据结构是：Hashtable；
* ConcurrentHashMap是线程安全的，即多个线程可以在同一个ConcurrentHashMap对象上进行操作而不会带来任何问题。
* 在同一时间点，任意数量的线程都可用于读取操作。
* 在ConcurrentHashMap中，对象根据并发级别划分为多个段（Segment）。
* ConcurrentHashMap的默认并发级别为16。
* 在ConcurrentHashMap中，在某一时间点任何数量的线程都可以执行search操作，但要进行更新，线程必须锁定要在其中操作线程的特定段，这种锁定机制称为段锁定或存储桶锁定。
* 在ConcurrentHashMap中，不能将null插入为键或值。

### ConcurrentHashMap和Hashtable的区别

* Hashtable在每次访问时，会对整个hash表加锁；虽然是线程安全的，但加锁的粒度过大。在高负载的情况下，会有性能问题。
* 把hash表分成了多个区域（segment），读取数据时不会加锁，在修改或新增数据时，会在某个区域加锁；此时，不会影响其他区域的数据读取，从而提升了并发性。
* ConcurrentHashMap不会在整个hash上加锁，而是维护一个锁的列表，默认情况下：锁的列表有16个，hash表有16个位置，这样每个锁就可以锁定hash表中的单个存储桶。所以，只要在hash表不同的位置进行读取，都是可以同时进行的。

```
[]---> Lock1
[]---> Lock2
```

* 当桶的槽数的占用率到达一定程度时，会对桶的大小进行扩容，一般是扩大一倍，此时锁就会锁定两个槽位的存储桶。

```
[]\
   ---> Lock1
[]/
[]\
   ---> Lock2
[]/
```

### ConcurrentHashMap的缺陷

* 聚合状态方法（包括size，isEmpty和containsValue）的结果通常仅在Map未在其他线程中进行并发更新时才有用。

* ConcurrentHashMap中的key是无序的，因此对于需要排序的情况，ConcurrentSkipListMap是一个合适的选择。

### 实战例子

```scala
package ConcurrentCollection;

import java.util.Iterator;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class DoConcurrentHashMap {
    private final ConcurrentHashMap<Integer,String> conHashMap = new ConcurrentHashMap<Integer,String>();
    public static void main(String[] args) {
        ExecutorService  service = Executors.newFixedThreadPool(3);
        DoConcurrentHashMap ob = new DoConcurrentHashMap();
        service.execute(ob.new WriteThreasOne());
        service.execute(ob.new WriteThreasTwo());
        service.execute(ob.new ReadThread());
        service.shutdownNow();
    }

    class WriteThreasOne implements Runnable {
        @Override
        public void run() {
            for(int i= 1; i<=10; i++) {
                conHashMap.putIfAbsent(i, "A"+ i);
            }
        }
    }
    class WriteThreasTwo implements Runnable {
        @Override
        public void run() {
            for(int i= 1; i<=5; i++) {
                conHashMap.put(i, "B"+ i);
            }
        }
    }

    class ReadThread implements Runnable {
        @Override
        public void run() {
            Iterator<Integer> ite = conHashMap.keySet().iterator();
            while(ite.hasNext()){
                Integer key = ite.next();
                System.out.println(key+" : " + conHashMap.get(key));
            }
        }
    }
}
```



### 小结

虽然可以对ConcurrentHashMap同时进行读写操作，但要注意：对该结构进行聚合操作（比如：size，containsValue）等操作时仅仅返回调用时的值，当返回结果时，可能该返回结果已经是一个过时的值了。

但使用ConcurrentHashMap的目的主要是为了不用对它进行同步处理，因为它是一个线程安全的集合对象。

另外，concurentHashMap不能设置NULL值。