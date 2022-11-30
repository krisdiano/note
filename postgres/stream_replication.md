# 流复制

## 协议

官方的[流复制协议](https://www.postgresql.org/docs/current/protocol-replication.html)

## wal

参考 [The Internals of PostgreSQL : Chapter 9 Write Ahead Logging — WAL](https://www.interdb.jp/pg/pgsql09.html) 的内容。

其中 9.1 的 3 个 section 分别描述了：

- 为什么需要wal

- 如何使用wal来进行启动恢复

- 如何解决非原子性的写页操作以及为什么wal的重放是幂等的

其中最后一项是比较关键的，PostgreSQL 通过一种 full-page write 的机制了页的撕裂写，即 9.1.3 中提到的备份区块，在 checkpoint 后对数据页的第一次写入会将数据页的内容写到 xlog 中，在恢复时通过 checksum 决定是否使用备份区块的内容替换数据页。而替换数据页后页面的 lsn 变小，会重放替换数据页导致丢失的 xlog ，这样就保证了恢复的正确性。

那么考虑一下，如果没有开启 full-page write 那么会有什么问题？比如说 checkpoint 后对页面 A 进行了插入操作，之后 A 由 buffer pool 刷入了 os page cache ，就认为写入成功了，在之后操作系统写了前 4K 后宕机了，开机恢复阶段，由于写入的前 4K 包含了页面的 lsn ，根据 9.1.2 中提到的比较 xlog 的 lsn 和页面的 lsn 决定是否重放，那么就是跳过，可能跳过的 xlog 是对未落盘的后 4K 的重放。因此就会导致数据库成为不一致的状态。

> 撕裂写：比如 PostgreSQL 设计的页大小为 8K ，而操作系统的页为 4K ，那么操作系统只能保证 4K 的写入是原子性的，如果对于 8K 的页只写了前 4K 或者后 4K ，那么就认为是发生了撕裂写。

## 流复制思路

在知道 wal 日志的重放可以是的数据库自身恢复到之前的一致性状态，那么不同的两个数据库实例是否能达到同样的一致性状态呢？当然是可以的。

以 pg_basebackup 执行流复制为例，具体步骤如下：

1. 打开 full-page write 机制

2. 先拷贝数据库 A 的数据目录作为 B 的数据目录

3. 将数据库 A 的 xlog 不断的传递给 B 进行重放

4. 禁止通过其它的方式对 B 进行写操作

5. 还原之前为 full-page write 机制

其中第 2 步是比较关键的，这个过程是持续的，主要 A 发生而变更就需要将 xlog 传递给 B 进行重放。那么如何发送请求(发送的单位是一条还是一页)和何时响应(写入 os page cache 后还是落盘后回复)就决定了两者之间是否存在不一致？如果发生而不一致两者的差距有多大？

对于流复制来说，是一次发送一条 xlog ，默认情况下是将日志写入 os page cache 同时发送事件通知对应的进程进行重放。因此基本上 A 和 B 是一致的。

> synchronous_commit 参数用来控制如何传递和重放。

为什么流复制阶段需要 A 打开 full-page write 呢？也是为了保证 B 的一致性，比如 A 的操作系统原子写入的单位是 8K ，而 B 的原子写入单位是 4K ，那么如果没有开启 full-page write ，xlog 中就不包含备份区块，所以如果 B 如果第一次恢复过程中遇到了撕裂写，且之后因为异常导致进程退出，那么重新启动后就会因为撕裂写导致数据库不一致。

传输分为两部分：

1. 文件传输
   
   1. 数据目录的文件
   
   2. 日志文件

2. xlog item 流式传输

第一阶段细节也是比较多的，首先 A 强制 xlog 更换新的日志文件进行写入，目的是尽可能只传递 B 恢复到一致性用到的日志。然后做一次 CHECKPOINT 操作，避免 A 当中存在过多的未落盘的数据页，这样可能就只需要传递的比较少的 xlog 。在传输的过程中需要保证没有传输的 xlog 不能被覆盖。在将数据目录的内容传输完后会得到一个 lsn ，这个 lsn 用来告诉 B 收到这个就可以退出进行启动了，把第一阶段的传输的内容恢复完成后再以最后的 lsn 开始第二阶段的流式传输。

更详细的内容可参考：[The Internals of PostgreSQL : Chapter 11 Streaming Replication](https://www.interdb.jp/pg/pgsql11.html)。
