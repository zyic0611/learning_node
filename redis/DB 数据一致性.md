# DB 数据一致性

## 盘路缓存模式 cache aside

## 

1. 读请求 

   - 读redis 命中 直接返回 
   - 未命中 查mysql 将mysql的数据写入redis 返回数据

2. **写** **核心**

   - 更新数据的时候 要先更新mysql 再删除redis
   - 为什么删除 不是更新
     - 并发写冲突 a b线程同时更新一个数据 a先更新db b再更新db 但是网络抖动情况下 b可能先更新 redis a再更新redis 则按理来说 redis和mysql应该数据都是顺序进行 后面执行的线程b。但是最后redis里是a mysql是b 则脏数据产生。 总结来说就是网络抖动导致并发状况下redis和mysql执行顺序不一致。
     - 性能浪费 lazy load 有些数据计算起来很复杂 如果一更新mysql 就同步更新cache 这个cache半天没人用 浪费cpu 删除cache 有人读再load 。

   

## 两大问题 

### 先删除redis 再更新db

1.   **在并发环境下** 容易产生脏数据
2.   a执行写操作 删除redis b读请求 发现redis为空
3.   b去读mysql 此时 a还没写完mysql b读到的是旧数据
4.   b读旧数据回redis  a写完mysql 由于采用懒加载 所以就算写完mysql 也不会更新redis
5. 数据不一致
5. **总结来说就是更新mysql的过程中 可能有线程查出旧数据写入redis**

### 先更新db 再删除redis （删除失败）

1.  在**网络异常下** 不安全
2. a更新mysql 
3. 准备删除redis 网络卡 或者redis正在full gc 卡顿 导致del redis命令执行失败
4. 则mysql更新成功 redis旧缓存没删除 
5. 数据不一致
5. **总结来说就是没删掉redis 只更新成功mysql**



## 解决方案

1. 对于先删除缓存 采用延迟双删
   - 先删除一次redis
   - 写入mysql
   - 写完休眠一会
   - 再删除redis
   - 休眠是关键 在休眠期间 等待会读取mysql旧数据的线程执行完 写回redis后 再次删除 
2. 对于删除缓存失败 采用 mq重试 保证删除redis缓存
   - 为了防止更新完mysql 由于网络原因导致redis删除失败 引入mq
   - 删除的方式 采用不直接删除redis
   - 给mq发一条消息 我要删除cache
   - 写一个消费者服务监听消息
   - mq自带 确认机制ack和重试机制 保证了一定会删除 保证了 **最终一致性**
3. mysql binlog
   - 业务代码只要去修改mysql
   - 利用阿里的acnnal 把它伪装成了mysql的一个从节点
   - mysql数据只要变化 就会自动产生变更日志【binlog 同步给canal
   - canal解析日志 发现某个table data被修改 则提取对呀key 去redis里执行删除



## 最终一致性

既然运用到了redis 就是性能为主。

上面的这些流程，都会在写入mysql的时候 读数据 读到旧数据，也就是允许短时间内看到旧数据 只保证了最终的一致性。



如果要保证强一致性 那么就得用读写锁。

写请求的时候加锁，所有读请求被阻塞，知道写完成。

除非是处理银行余额这种敏感业务数据 否则很少使用。