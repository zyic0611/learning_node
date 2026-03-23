Q:ArrayList和LinkedList的区别及使用场景？

A:

底层结构：

Arraylist 底层是动态数组 连续内存 linkedlist是双向链表 非连续内存。

操作性能：

ArrayList 支持按照索引随机访问 时间复杂度为O（1） LinkedLIst查询元素需要顺序遍历 时间复杂度为O（n） 在中间插入/删除的时候 ArrayList需要批量移动元素  时间复杂度为O（n）。LInkedList虽然只需要修改指针 时间复杂度为O（1） 但是找到目标节点 依然需要 O（n） 所以LinkedList主要是头尾插入极快。

场景：

ArrayList适合读多写少 随机访问 日常开发基本都是使用ArrayList

LinkedList适合 头尾频繁增删 不知道数据量大小。

Q:HashMap JDK 1.8的底层数据结构？

A:底层采用数组加链表加红黑树

<u>Q:HashMap扩容因子为什么是0.75？</u>

A: 

1. 如果设置为1 空间利用率高了 但是哈希冲突为很严重 链表变长 查询效率从O(1)退化 
2. 如果设置为0.5 哈希冲突少了 空间利用率低 频繁触发扩容 rehash非常消耗性能
3. 0.75是柏松分布计算出来的理想值 保证链表长度>8 -->红黑树的概率小

Q:HashMap为什么选红黑树而不是AVL树 ？

A:avl 树需要时刻严格保持平衡的二叉树，虽然查询效率高，如果频繁的插入删除节点，保持平衡操作性能差。红黑树只需要保持红黑平衡，是弱平衡树，允许一定的树高度差，用轻微的查询性能，换取了插入删除节点的性能好，适合hashmap这种频繁写的操作。

Q:ConcurrentHashMap的实现原理（1.7 vs 1.8）？

A:jdk 1.7以前是 将数组划分为16段segment，用reentrantlock 上锁。并且底层采用数组加链表的数据结构。 jdk 1.8以后，锁的细粒度减小，只对数组的每一个节点上锁。并且当添加元素的时候，如果要添加位置为空，采用cas 方式添加元素，如果非空，则对链表或者红黑树的头节点上锁。上的事synchronized，底层采用数组➕红黑树➕链表的数据结构。

<u>Q:ConcurrentHashMap的size()方法如何保证准确性？</u>

A:

1. JDK1.7 先尝试两次不加锁计算 if前后两次modCount 修改数据 说明没并发 直接返回 if不一致 就锁住所有的segment锁 在计算
2. JDK1.8 使用baseCount 和 CounterCell[]数组 没有竞争的时候 用CAS累加到basecount上 高并发有竞争的时候 并发压力分散的coutercell数组的不同元素上 size 就是basecount和coutercell数组的总和

Q:MySQL的事务隔离级别有哪些？

A:读未提交，读已提交 可重复读 串行化

Q:MySQL默认隔离级别是什么？解决了哪些问题？

A: 默认是可重复读，解决了脏读和不可重复读的问题 在InnoDB引擎下 <u>通过next key lock 也很大程度上解决了幻读的问题。</u>

<u>Q:可重复读如何解决幻读？</u>

A：

1. 针对快照读（普通select) 依靠1mvcc 解决幻读。幻读底层依靠隐藏字段、undolog版本链 readview 视实现。在RR级别下 事务在第一次select生成一个readview 之后一直沿用。别人新增的数据 都看不到。
2. 针对当前读 (select for update,insert,update) 依靠间隙锁(gap lock)和记录锁(record lock)组成 next-key lock 锁住索引之间的间隙 防止其他事务在这个区间内插入新的数据

Q:MVCC的原理是什么？

A:多版本并发控制 以及上面的两种操作 

Q:事务ACID特性分别如何保证？

A: 

1. A原子性 通过Undo Log 回滚日志 失败的时候 用它回滚事务
2. I 隔离性 由MVCC和锁机制保证
3. D 持久性 由redo log 保证 宕机也能恢复数据
4. C 一致性 最终目的 由上述三个特性+业务代码实现

Q:Redo Log和Bin Log有什么区别？

A：

1. redo log 是记录物理行为的日志 主要是记录内存是如何修改的 他是以覆盖写的方式 当写满了 回到开头开始覆盖写
2.  bin log 记录的是逻辑行为的日志 存储的是具体的sql语句 他不以覆盖写的方式存储 当满了就会新开一个文件存储 并且redo log 是先于binlog存储的

Q：Redo Log和Bin Log的一致性如何保证？

A：是一个两阶段提交的方式保证的 当redolog写入后 打上一个prepare的标记 只有当binlog写入后 prepare的标志才会变成commit 此时才是真正提交 当redolog写入后断电 来不及写binlog需要断电回复的时候 发现redolog里是prepare状态 并且找不到对应的binlog 就不会恢复redolog的内容 保证主从一致 如果是binlog写完来不及吧prepare标记修改为commit 此时找得到redolog对应的binlog 可以保证主从一致 会自动修改为commit 然后恢复数据

<u>Q：InnoDB行锁和表锁分别在什么场景下触发？</u>

A：当更新操作 命中了索引 会触发行锁 if where操作 没用使用索引 或者索引失效 行锁--》表锁 性能很差

Q：**间隙锁的作用是什么？**

A：主要是为了解决RR隔离级别下的幻读问题，在进行当前读的时候，并且涉及到一个范围的时候，innodb不仅会锁住具体的行，还回用锁锁住数据间的间隙，阻塞其他事务在这个间隙间进行insert操作，就避免幻读了。

**Q：索引有哪些分类？**

A：

数据结构： B+树索引 hash索引

物理存储：聚簇索引 非聚簇索引

逻辑：主键索引 唯一索引 普通索引 联合索引

Q：InnoDB为什么使用B+树？它的优势是什么？

A：数据库是存放在磁盘里的，所以要尽可能的减少I/O次数。B+树是矮胖的数据结构，所以I/O次数少。并且每个节点只存放主键/索引，不带数据，则每个节点可以存大量的指针，树很宽，就很矮了。B+树叶子结点存放数据，并且用双向链表连接，天然支持范围查找。

Q：联合索引(A,B,C)，WHERE B=1 AND C=1能走索引吗？

A：走不了索引 因为最左前缀原则，所以只能全表扫描

Q：WHERE A=1 AND C=1能走联合索引吗？

A：可以走索引 但是不能走联合索引。因为最左前缀原则，只会靠字段A在二级索引树，找到了一些节点，可以通过索引下推的方式 利用字段C=1排除一些字段，然后剩下的字段回表查找。

Q：SELECT * 和 SELECT A 在索引使用上有什么区别？

A：索引使用是看where里的条件 但是select A可能不会回表 select *一定会回表。

Q：<u>EXPLAIN执行计划重点关注哪些字段？</u>

A：

type 查询类型 性能从好到坏是system>const>eq_ref>ref>range>index>all 至少是range/ref

possible_key和key 预计使用的索引和实际使用的

rows:预计扫描的行数

extra: 最重要 using index说明使用了覆盖索引 性能最好 using filesort/using temporary 需要优化

Q：<u>Redis常见的5种数据类型及使用场景？</u>

A：

String:单值缓存 计数器 分布式锁

List:消息队列 最新文章列表

Hash 对象缓存，比如用户信息 购物车

Set：标签 共同好友【交集并集

Zset：排行榜 延迟队列

Q：<u>Set和ZSet底层数据结构有什么区别？</u>

A：

Set：底层采用intset或者hashtable 

zset底层采用ziplist/listpack 或者skiplist+hashtable 跳表用于按照分数查找

Q：Redis持久化有几种方式？各自优缺点？

A：

rdb快照：文件内存小 恢复快 可能丢失最后一次的数据 

aof最佳日志 文件大 数据安全 回复慢

混合持久化 其实也就是aof的升级模式 先使用快照 快照后的数据使用aof追加到后面 不会丢失数据。

Q：Redis过期键删除策略有哪些？

A：

惰性删除 访问时发现过期才删除

定期删除 后台线程随机抽取检查并删除

Q：如何保证Redis和数据库的数据一致性？

A：最常用的是旁路缓存 cache aside pattern 更新数据的时候 先更新数据库 再删除缓存。如果有强一致要求 延迟双删 或者引入消息队列/canal重试删除

Q：CAS的原理是什么？

A：compare and swap。通过三个参数 预期的值 现在的值 修改后的值。当预期的值等于现在的值 就可以替换成修改后的值。 c

Q：CAS的ABA问题如何解决？

A：加版本号 version

Q：CAS除了ABA问题还有什么问题？

A：有可能会同时有很多线程在不停的自选等待 cpu消耗大。cas只能保证一个共享变量的原子操作 如果是多个变量 得用锁

Q：<u>synchronized和ReentrantLock的区别？</u>

A：

层级与实现：`synchronized` 是 Java 语言的关键字，属于 JVM 层面（底层基于 Monitor 对象，依赖操作系统的 Mutex Lock）；`ReentrantLock` 是 JDK 提供的 API 层面的类（位于 `java.util.concurrent.locks` 包），底层基于 AQS 实现。

释放锁的机制：`synchronized` 代码块执行完毕或发生异常时，JVM 会自动释放锁，一般不会死锁；而 `ReentrantLock` 必须由程序员在 `finally` 块中手动调用 `unlock()` 释放锁，否则容易造成死锁。

公平性：`synchronized` 只能是非公平锁；`ReentrantLock` 默认是非公平锁，但可以在构造函数中传入 `true` 来实现公平锁（按线程排队顺序获取锁）。

高级特性（ReentrantLock 特有）：

- 响应中断：调用 `lockInterruptibly()`，正在等待锁的线程可以响应中断请求，而 `synchronized` 只能死等。
- 超时获取：提供 `tryLock(time, unit)`，在指定时间内尝试获取锁，获取不到就返回 false，不会一直阻塞。
- 多条件变量（Condition）：也就是你提到的。`ReentrantLock` 可以绑定多个 `Condition` 对象，实现精准唤醒指定的某个或某些线程；而 `synchronized` 配合 `wait()/notify()` 只能随机唤醒一个或唤醒全部线程。

Q：synchronized的锁升级过程？

A：无锁 偏向锁 轻锁/自旋锁/ 重锁。

Q：Monitor监视器的核心字段有哪些？

A：

_owner 指向当前拿到锁的线程 _EntryList 阻塞队列 _WaitSet 等待队列 调用了wait()的方法在此等待

Q：AQS是什么？核心原理？

A：是抽象阻塞队列  是JUC的基石 核心原理是有一个volatile字段 state 用cas修改他 还维护了一个FIFO的双向链表【CLH队列 阻塞队列。线程阻塞在队列里，依照state的值 决定是否可以出队竞争锁

Q：<u>独占锁和共享锁的区别？</u>

A：

独占锁：每次只有一个线程能拥有锁 其他试图获取锁的线程都会被阻塞 保证了共享资源的绝对独占

代表： synchronized reentrantlock reentrantReadwritelock 的写锁

共享锁：允许多个线程同时获取锁  并发访问共享变量

代表: reentrantreadwritelock的读锁 semaphore信号量 countdownlatch 

底层实现：

独占锁调用的是 `acquire()` 和 `release()` 方法，而共享锁调用的是 `acquireShared()` 和 `releaseShared()` 方法。

<u>Q：CountDownLatch的工作原理？</u>

A：

核心思想 是一个同步辅助类 像一个倒计时 允许一个线程或者多个线程等待其他线程完成一些列操作后再继续执行。

底层原理： 基于AQS的共享锁机制

1. 初始化 创建时 传入一个整数count 会被赋值给aqs的state 代表有count个共享锁
2. 等待：主线程/需要等待的线程 调用await（）方法 如果state>0 说明任务没执行完 会加入aqs的clu中等待
3. 倒数 工作线程完成任务后 调用countdown方法 底层用cas 把state-1
4. 唤醒： state=0时候 aqs会唤醒阻塞队列中所有因调用await()方法阻塞的队列 开始执行

Q：线程池的7个核心参数？

A：核心线程数 最大线程数 阻塞队列 拒绝策略 工厂名称  空闲存活时间 时间单位

Q：核心线程数、最大线程数、队列的配合逻辑？

A：任务来的时候 先去核心线程工作。核心线程满了 再去阻塞队列排队 阻塞队列满了 再根据最大线程数扩充线程 后来的先去扩充的线程里工作

Q：线程池的4种拒绝策略？

A： 抛出异常 【默认 让线程调用者的线程自己运行 默默丢弃不报错 丢弃队列里最老的任务

Q：为什么推荐使用有界队列？

A：如果使用无解队列 消费者跟不上生产者 任务会无限堆积在内存中 最终OOM

Q：内存泄漏和内存溢出有什么区别？

A：内存泄漏是有一部分内存永远无法被释放。内存溢出是内存满了 放不下

Q：JVM运行时内存结构是怎样的？

A：

线程私有：虚拟机栈 本地方法栈 程序计数器【唯一不会OOM的

线程共享 堆【最容易OOM 方法区 

Q：JDK 1.8方法区有什么变化？

A：永久代变成了元空间 并且元空间只是逻辑上再jvm 实际上在本地内存中存储

Q：哪些内存区域会发生OOM？

A：堆 栈【本地方法栈 虚拟机栈 元空间

Q：垃圾回收的判定算法有哪些？

A：可达性分析算法 引用计数法【循环引用问题 java不用

Q：标记清除、标记复制、标记整理的优缺点？

A：

标记清除：速度快 会产生内存碎片

标记复制：无碎片 适合存活率低的年轻代 会浪费一半内存

标记整理：无碎片 不浪费内存 缺点是移动对象速度慢

Q：什么是双亲委派？如何打破？

A：类加载的时候 先未退给父类加载器加载 父类加载不了自己才会加载 好处是保护java核心api不被修改

打破：自定义类加载器 重写loadClass（）方法 比如tomcat jdbc

<u>Q：Spring AOP的原理和应用？</u>

A：

底层原理是动态代理 如果目标类实现了接口 默认用jdk动态代理 没有 就用CGLIB生成子类代理

应用：日志记录 性能统计 安全控制 事务管理