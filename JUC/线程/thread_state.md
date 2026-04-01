# thread_state

1. new 线程创建 没调用start方法 还没启用

2. runnable 包含了操作系统的ready & running。 线程调用了start，正在cpu上运行，或者等待cpu
3. blocked  线程正在等待获取 **Monitor 锁**（即进入 `synchronized` 代码块或方法）时的状态。
4. wait  是持有锁状态主动调用wait()，放弃锁。 正在等待其他线程的某一个操作，比如notify。触发：`Object.wait()`、`Thread.join()`、`LockSupport.park()`。
5. timed_waiting 具有具体时间的等待状态。触发：`Thread.sleep(time)`、`Object.wait(time)`。时间到了自动醒。
6. terminal   线程完成，终止状态



## sleep & wait

sleep是Thread类的静态方法，不需要持有锁，也可以调用。唤醒机制是超时自动回复，如果其持有锁，调用sleep方法，不会自动释放锁。他的用途是暂停线程执行，不涉及锁。sleep会释放cpu 

wait是Object类的实例方法，任何对象都可以调用，需要持有锁，作用是释放锁。



## blocked

只有在等待 **`synchronized` 关键字** 对应的锁（Monitor 锁）时，线程才会进入 `BLOCKED` 状态。

如果你用的是 Java 并发包里的锁（如 `ReentrantLock`），线程等待时进入的是 **`WAITING`** 或 **`TIMED_WAITING`** 状态，而不是 `BLOCKED`











