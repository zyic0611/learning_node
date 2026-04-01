# AQS

abstractqueuesychronizer 抽象队列同步器

**核心定义**：AQS 是一个用来构建锁和同步器的**抽象框架**。JUC 包下的大部分并发工具类（ReentrantLock, Semaphore, CountDownLatch 等）全都是基于它实现的。

1. **两大核心数据结构**：

- **同步状态（State）**：一个 `volatile int` 变量。表示当前的同步状态。线程通过 **CAS** 操作尝试修改它来实现加锁/解锁。
- **同步队列（CLH Queue）**：一个 **FIFO 的双向链表**。当线程获取锁失败时，会被封装成 Node 节点加入队列并挂起（阻塞）。

2. **工作流（获取与释放）**：

- **获取（Acquire）**：尝试通过 CAS 修改 state。成功则拿到锁；失败则进入双向队列尾部，调用 `LockSupport.park()` 挂起自己。
- **释放（Release）**：修改 state 归零。然后检查队列，调用 `LockSupport.unpark()` 唤醒头节点的下一个有效节点（队首的排队者）。

## state：

- 被volatile修饰 保证所有线程可见
- 必须用cas修改
- 重点: **多态化**:在reentrantlock中 state记录的是重入次数 countdownlatch中 是剩余等待线程数  semphore中 是剩余的许可证数量  这就是 AQS 为什么叫“抽象”同步器的原因，它只提供一个数字，具体怎么用由子类自己定。

## CLH 双向链表队列

没抢到锁的线程，会被包装成一个node节点 加入队列中。

**为什么必须是双向的（有 prev 和 next 指针）？** 这是一个极其高频的面试坑题！

- 如果前面的老哥等得不耐烦了，或者被 `interrupt` 中断了，他要放弃排队（取消状态）。有了双向指针，系统就能轻易地把他从链表里剔除，让他前面的节点和后面的节点重新连起来，不会导致队伍断裂。

**怎么睡觉/怎么唤醒？**：底层调用的是 `LockSupport.park()` 让线程陷入沉睡；释放锁时，调用 `LockSupport.unpark(线程)` 把队列头部的下一个节点唤醒。



并且先进先出队列可以保证公平性。

