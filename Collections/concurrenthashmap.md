# concurrenthashmap

线程安全的哈希表。

## segment 数组+链表

在jdk1.7以前 采用分段锁segment保证线程安全。

把数组分为16段 每一段都加ReentrantLock ，并发量提高了16倍。

## 数组+链表+红黑树

jdk1.8以后 锁的细粒度提升了，只锁链表或者红黑树的头节点，也就是只锁某一个bucket。 并发度等于数组长度。而且由于1.6以后synchronized优化了 性能也很好 所以使用的是sunchronized。

## jdk1.8后的函数方法如何实现

### put() 存数据

1. 当存数据的时候，先计算出哈希值。然后判断要存在哪个bucket，如果位置为null。则说明此时没有哈希冲突，并且并发程度轻，所以直接使用CAS尝试放入数据，如果位置刚好被人占了 则自旋。
2. 如果位置有人，则吧这个bucket加synchronized悲观锁。然后在桶里找 看是覆盖还是新增。

### get（）取数据

取数据不用加锁 他的结点node里的关键属性 比如 val next都被打了 volatile关键字 保证了内存可见性 读的实时性

### JDK 1.8 为什么要放弃 `ReentrantLock` 改用 `synchronized`

因为自 JDK 1.6 之后，JVM 对 `synchronized` 进行了史诗级优化（引入了偏向锁、轻量级锁、重量级锁的锁升级机制）。在“只锁一个小书架（头节点）”这种**锁竞争非常小、且锁定时间极短**的场景下，优化后的 `synchronized` 性能不仅不输，而且它能节省大量创建 `ReentrantLock` 对象的内存开销

## 限制

不允许 `key` 或 `value` 为 `null`（为了避免并发环境下的“二义性”问题：无法确认是没找到，还是存的就是 null）。

扩容机制：支持**多线程协助扩容**（`transfer` 方法），当某个线程发现数组正在扩容时，它不会干等，而是会主动领一部分搬运任务一起搬，极大提升了并发扩容效率。





## Size()

### jdk1.7:

jdk 1.7以前 底层数据结构式segment数组 默认为16个segment 相当于把一个大的hash表切分成了16个小hash 每个segment自己管自己的数据 自己管自己的锁 。

如果要统计整个map的大小 数据分散在16个segment里 随时有线程在segment里新增或者删除数据。

如果锁住16个segment 则并发性能很差。

采用 乐观试探，先尝试两次不加锁计算。

每个segment内部维护字段 自己存放了多少数据 **count** 还维护一个变量 **modeCount** 发生写操作就+1.

当调用size()方法的时候  

1. 先不加锁
   - 遍历16个segment 累加count 得到总size 并且累加modecount 得到总modecount
   - 再重复一遍
   - 看两遍的size的modecount相同吗
2. 比对
   - 如果相同 则modecount准确
   - 不同 则全部上锁 

### jdk1.8

如果内部只维护一个size字段【比如atomiclong  再极高并发的情况下 几百个线程同时执行put() 操作 他们都会尝试用cas去修改同一个size变量。 则几百个线程都会在cas 消耗cpu资源 这就是cas并发瓶颈

空间换时间 引入基础计数器和数组

- **`baseCount`**：一个普通的 `volatile long` 变量。

- **`CounterCell[]` 数组**：一个专门用来计数的数组。每个 `CounterCell` 对象内部包裹了一个 `volatile long value

当一个线程put() 需要把map大小+1 内部调用addcount()

1. 无竞争状态 ：尝试cas修改basecount  cas成功 修改成功 结束
2. 如果cas失败 说明多个线程在修改basecount 发生冲突  则转向使用countercell[]
3. 通过哈希计算 定位到数组中具体的一个cell 然后用cas修改
4. cas再失败 则会死循环重试 或者对coutercell数组扩容 
5. size()方法则累加所有cell和basecount即可