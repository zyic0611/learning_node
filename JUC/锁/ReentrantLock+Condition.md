# ReentrantLock+Condition

1. synchronized:

- 只有一个窗口,随机叫号。

2. reentrantlock+condition:

- lock锁可以手动开关，condition是不同的等候区，signal可以精准让人来持有锁。

核心优势为精准唤醒。

## 核心方法

#### 1. ReentrantLock（可重入锁）

- **手动档**：必须手动 `lock()` 和 `unlock()`。为了防止出意外死锁，**必须写在 `try-finally` 块里**。
- **可重入**：同一个线程拿到了这把锁，可以不停地再次进入（计数器加 1），不会把自己锁死。
- **公平性选择**：你可以设置成“公平锁”（先来的先办），而 `synchronized` 只能是非公平的。

#### 2. Condition（等待条件/等待区）

- 它是通过 `lock.newCondition()` 创建出来的。
- **`await()`**：对应 `wait()`，让线程去特定的等待区休息，**并释放锁**。
- **`signal()`**：对应 `notify()`，只唤醒在这个 Condition 上等待的线程。

## 对比

![image-20260302105941211](/Users/zhuzhucheng/Library/Application Support/typora-user-images/image-20260302105941211.png)

