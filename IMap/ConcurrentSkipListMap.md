## 深入浅出ConcurrentSkipListMap

这是一个中会对key进行排序的key-value数据结构。该结构和TreeMap相似，但TreeMap不是线程安全的，而ConcurrentSkipListMap是线程安全的；另外，TreeMap是通过红黑树实现的，而ConcurrentSkipListMap是通过跳转表来实现的。

### ConcurrentSkipListMap要点

* 对于高并发程序，应当使用 ConcurrentSkipListMap，能够提供更高的并发度。
* 在多线程程序中，如果需要对Map的键值进行排序时，请尽量使用ConcurrentSkipListMap，可能得到更好的并发度。
*  调用ConcurrentSkipListMap的size时，由于多个线程可以同时对映射表进行操作，所以映射表需要遍历整个链表才能返回元素个数，这个操作是个O(log(n))的操作。
* 在非多线程的情况下，应当尽量使用TreeMap。
* 此外对于并发性相对较低的并行程序可以使用 Collections.synchronizedSortedMap将TreeMap进行包装，也可以提供较好的效率。

### 和ConcurrentHashMap的不同点

* 根据规范，ConcurrentHashMap不保证其操作的运行时间。其中ConcurrentSkipListMap保证了大多数操作的O(log(n))性能。
* ConcurrentHashMap允许修改线程数以调整并发行为，其中ConcurrentSkipListMap不允许修改并发线程数。
* ConcurrentHashMap不是NavigableMap也不是SortedMap，但是ConcurrentSkipListMap既是NavigableMap又是SortedMap。
* ConcurrentSkipListMap是通过跳转表实现的，key排序的map结构，而CocurrentHashMap不是。

### ConcurrentSkipListMap实战

```java
package ConcurrentCollection;

import java.util.Iterator;
import java.util.NavigableSet;
import java.util.concurrent.ConcurrentNavigableMap;
import java.util.concurrent.ConcurrentSkipListMap;


public class DoConcurrentSkipListMap {
    public static void main(String[] args) {
        ConcurrentNavigableMap<String, String> concurrentSkipListMap = 
          																				new ConcurrentSkipListMap();
        concurrentSkipListMap.put("3", "Apple");
        concurrentSkipListMap.put("2", "Ball");
        concurrentSkipListMap.put("1", "Car");
        concurrentSkipListMap.put("5", "Doll");
        concurrentSkipListMap.put("4", "Elephant");

        System.out.println("ceilingEntry-2: "+ concurrentSkipListMap.ceilingEntry("2"));
        NavigableSet navigableSet = concurrentSkipListMap.descendingKeySet();
        System.out.println("descendingKeySet: ");
        Iterator itr = navigableSet.iterator();
        while (itr.hasNext()) {
            String s = (String) itr.next();
            System.out.println(s);
        }
        System.out.println("firstEntry: " + concurrentSkipListMap.firstEntry());
        System.out.println("lastEntry: " + concurrentSkipListMap.lastEntry());
        System.out.println("pollFirstEntry: " + concurrentSkipListMap.pollFirstEntry());
        System.out.println("now firstEntry: " + concurrentSkipListMap.firstEntry());
        System.out.println("pollLastEntry: " + concurrentSkipListMap.pollLastEntry());
        System.out.println("now lastEntry: " + concurrentSkipListMap.lastEntry());
    }

}
```

### 小结

ConcurrentSkipListMap是基于跳转表实现，key排序的，线程安全的map结构。