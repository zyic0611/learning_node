# ziplist & listpack

压缩列表和紧凑列表

list hash zset如果元素很小 并且都是小整数或者短字符串 redis不回去创建真正的hash或者双向链表 用ziplist存储

## 设计初衷

- linkedlist 每个节点 都需要额外的内存 存prev next 指针 16个字节
- 普通链表节点在内存中是分散地 容易产生内存碎片 对cpu cache不友好
- ziplist 把元素放在连续的内存空间 节约指针



## ziplist的内部结构

它本质上是一个字节数组，结构大概长这样： `[zlbytes总长度] [zltail尾节点偏移量] [zllen节点数量] [entry1] [entry2] ... [entryN] [zlend]`

entry的结构：

- prelen 前一个节点的长度 有了它 就可以从后往前实现反向遍历
- encoding 当前节点数据的类型和长度
- data 真正的数据

### 问题：连锁更新

- redis的规定 如果前一个节点的长度<254 prevlen只分配一个字节来记录 一个字节8位 可以存0-255的信息
- 255呢？redis 把 `255`（即十六进制的 `0xFF`）**专门保留给 `zlend`（整个列表的结尾标志符）** 去用了。为了防止冲突，`prevlen` 最大就只能用到 `254` 啦。
- 如果前一个字节长度>=254 则prevlen分配5字节

情景：

e1 e2 e3 大小都是253字节 则e2 e3 的prevlen都是1字节

当e1前插入一个巨大的节点 则e1的prevlen必须膨胀到5字节 则e1的总长度变成253+4=257 e2的prevlen也需要拓展到5字节 以此类推 

引发了严重的连续内存分配和数据拷贝 这就是连锁更新 由于ziplist这种结构不好数据扩展  最坏情况下 导致严重的性能卡顿 退化到on2



## listpack

redis5.0引入 7.0后完全舍弃ziplist 为了解决连续更新的问题

### entry结构修改

- encoding 当前节点的编码类型
- data 真实数据
- element-tot-len 当前节点的总长度！

记录自己节点的长度，并且放在尾部。由于不记录前一个节点的大小，不管前面插入多大的都关系，后面的节点全部不用改变自己内部的长度字段。

反向遍历

- 指针停在当前节点的最末尾时候，读一下自己节点的总长度 指针剪一下，找到节点的首地址 往前走一个地址 就是到上一个节点了。实现了双向遍历功能。





## 顺序遍历

只要走过了头部 来到entry内部。encoding里包括了data的类型和数据长度。知道了encoding的长度和data的长度，自身的总长度就是encoding占用字节数+data的字节数+prevlen/element_tot_len总字节数。就可以一个entry一个entry的遍历。