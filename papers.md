# Papers

对 RocksDB 提到的论文的一些总结

## FAST'21

描述了 RocksDB 在 2012 - 2020 的演进。Feature 只到 2020 年. 这里就贴我之前写的博客，不做进一步展开了。

https://blog.mwish.me/2023/01/22/Fast21-RocksDB/

## SIGMOD'23

这论文最近正好有好人做了个翻译，见：https://www.cyningsun.com/06-01-2025/disaggregating-rocksdb-a-production-experience-cn.html  。总的来说这篇文章理论部分价值不是特别高，但是介绍的都是很中肯的工程实现部分。这个翻译大致事很不错的，不过有的地方感觉会有一点点漏译。

文章大意是将 RocksDB 从 SSD 上迁移到 Meta 的 Tectonic 文件系统上，这里的 Point 本身还是「让 CPU 和存储按需独立配置」，即文章希望实现更高的 TCO，分离计算和存储，降低一部分直接的性能来提高资源利用率。我把文章中的 Purpose 原样复制一下（引用上面的文章）：核心是（硬件之间）网络带宽的提升给应用提供了分离的机会。论文也提出「即使存算分离模式的性能不及本地，但对于 Meta 内大量 RocksDB 应用来说已足够」。

> 几年前，几乎没有 RocksDB 应用能承受将存储放在远程网络上的代价，但近年来网络带宽迅速提升。例如，十年前我们还在将主机从 1Gbps 网卡升级到 10Gbps，而现在至少是 25Gbps，常见的还有 50Gbps 或 100Gbps。另一方面，大多数用户对磁盘 IO 带宽的需求与十年前相似。这一趋势使得分离式存储对越来越多的应用具有吸引力。尽管网络仍是瓶颈，但许多场景受限于空间而非 IOPS，在这种情况下，降低 IOPS 可能是可以接受的权衡。此外，如果分离式存储实现为可靠服务，用户可以以多种方式利用其可靠性，例如提升区域内可用性、减少跨区域数据传输。

这里论文也提出了问题:

1. 引入网络跳数导致延迟增加
2. 以容错方式存储时，能很好的做 Failover，但引入了更大的写放大
3. 在分离式存储下，如果有故障导致切换节点，那么需要有一定的机制做 Fencing，防止旧 master 写入
4. 对远程 io 行为，比如超时等，做出特殊处理

（此外，笔者扩展一下 3，之前 Backup 节点读会强制 open 所有文件做读，现在需要维护别的机制来保证写入节点不会）

### 决策

> 然而，NFS 过于复杂，难以高效利用；而块设备接口下，不同主机间的数据共享也很困难。我们意识到，通用的分布式块设备或文件系统过于泛化，可能错失针对性优化机会，从而限制了系统的潜力。RocksDB 的数据和日志文件采用顺序写入，写入后即不可变。任何允许随机写的方案都不可避免地引入额外开销和复杂性。假设数据块不可变的文件系统能实现更高效率，这正是 Tectonic 文件系统的设计假设。

Tectonic 对部分 (1) 延迟关键服务 (2) 不接受容量有小膨胀的服务 还是有影响的。这里文章没展开，但我觉得容量需要某种程度上仔细考虑：对于原先一副本的，这里实际上相当于一副本变 EC；原来备份或者 Raft 的，如果愿意接受某种程度的架构变化，那么存储是 2/3 份变成多台机器的 1.2-1.5 倍的 EC 存储。

### 优化

#### 优化 I/O 尾延迟

这里论文的部分基本在 Pangu 的论文都有提到，而且有相对细一点的介绍。

1. **Dynamic Eager Reconstructions**: 对副本来说备份读比较关键，但是对 EC 来说，这部分第二部分读取涉及多个并行 IO，影响较大；另外如果读来自于某些节点的健康状况问题，那么在这些 EC 节点上备份读只会加重这种状况。所以要 Tracking 存储节点的 Latency 百分位，然后根据百分位来动态判定具体流程。（本篇论文这一块几乎没有很细的介绍，Pangu 的论文这一块写的比较细）。
2. **Dynamic Append Timeouts**: FS 写，包括 EC 写入，基本上都是走一个写 Quorum 的模式的。如果这里服务器节点上有维护操作，可能前台写入就会变慢，这里如果发现 timeout 或者延迟高，这里可能会尝试 track append 的延迟百分位，然后尝试走 seal 的流程，seal 会更新 meta，然后这里会在低延迟的一组机器上开启下一个 chunk。
3. **Hedged Quorum Full Block Writes**: 这里对大块写入做了 append 外的优化，这里核心问题是 append 需要再上一次 append 的一组节点上继续 append，但是比较大块的写（我感觉和 chunk size 或者类似概念有关）可以自己单独宣一组低延迟的服务器，直接写一组 sealed chunk。（总感觉这种技术在 S3 Part 上传或者 S3 上传的时候非常有用，在 RocksDB 这个场景不知道效果如何）

#### 元数据优化

> 底层 RocksDB 目录在任一时刻只会被一个进程访问和修改，因此元数据可以主动缓存。

这个地方似乎 Backup 之类的也需要 Catch 一下数据变更流，但结论似乎依然是成立的。

#### SecondaryCache for local store

使用 [SecondaryCache](https://github.com/facebook/rocksdb/wiki/SecondaryCache-%28Experimental%29) Feature，提供一个本地 SSD 缓存，优化读应用的负载。

#### 读写合并调优

> 在 Tectonic 上运行时，通常将合并读取大小设为 4MB 或 8MB，合并写缓冲区设为 64MB 或更大，应用即可获得满意性能。合并延迟仍可能增加——幸运的是，RocksDB 支持并行 memtable 刷新和合并，可帮助吸收延迟。

(这里有个重点感觉官方还没有代码上加上，就是 MultiGet, Iterator 和写入之类的地方 IO 也应该和 CPU 并行起来，这样来打满机器占用的资源。)

这里在 MultiGet 和 Iterator 处理的时候，官方提到了 Adaptive 增大读 io 的模式:

> RocksDB 通过预读减少迭代器路径上的 IO，预读有两种模式：固定配置和自适应。HDD 用户通常用固定预读，但在 Tectonic 上会导致过多网络带宽消耗。自适应模式从读取一个块开始，不断翻倍直到最大 256KB。但这种方式在 Tectonic 上预热太慢。为此，我们让 RocksDB 基于历史统计设置初始预读大小，并使最大值可配置。

这里还在 4.1.5 提到了 `MGet` 的并行访问，我理解本质上是延迟-访问量的 Trade-off。

### Redundancy with low overhead

* SST 占用主要存储空间和写带宽
  * We chose [12,8] encoding (4 parity blocks for 8 data blocks)
* WAL 需要 low tail latency for small (sub-block) appends with persistence
  * We use 5-Way Replica (R5) encoding for WAL and other log files
  * First, Replica encoding provides better tail latencies with no RS-Encoding overhead for small size writes. 
  * Second, unlike RS-Encoding, we don’t need write size alignment or padding for replica encoding. 
  * Finally, we use R5 instead of R3 or other settings, as R5 is sufficient to meet our availability needs based on host failure probabilities.
* WAL 尽量使用条带化 EC
  * 条带化编码需收集一条带（或多条带）数据后再刷新。例如 [12,8] 编码 +8KB 条带时，将 8KB 数据分成 8 个 1KB 数据块，生成 4 个 1KB 校验块，共 12 个 1KB 块分发到 12 个存储节点
  * 每个节点将 1KB 块追加到对应的 XFS 文件（通常为 8MB）。条带大小按文件类型预设。偶尔需刷新非对齐数据时，会用零填充对齐后编码并刷新。
  * 条带较小会降低随机读效率，因为需从多个节点组装并解码，但日志文件几乎总是顺序读取，因此该方案适用。

### cooperative IO Fencing

* RocksDB directory must first ’IO Fence’ the directory using a token (a variable length byte string), and then use the same token for all subsequent operations on that directory and files under it.
* The expectation is that the fencing will be successful if the token is **lexicographically greater than** the previous fencing token used by any other process on that directory.
* Internally, Tectonic executes a sequence of steps to guarantee fencing.
  * First, it updates the token on the directory in the metadata system, provided it compares favorably.
  * Tectonic then iterates over the list of mutable (unsealed) files under the directory, and performs two actions, (a) it updates the token on the file metadata, and (b) seals the tail writable block of the file by connecting to the storage nodes and sealing their corresponding chunks.

本质上还是 cooperative 的去 seal 之前的，然后要求切一些新的，是一个还比较重的 meta 操作。但感觉这里有个问题是，写入的时候也不应该依赖一些 Partial 的操作。

### IO Timeouts / Errors

> 刷盘和合并等操作设置较长的超时时间，而对 Get() 或迭代器等操作则设置亚秒级超时。RocksDB 新增了可配置参数 request deadline，若请求超时则直接返回失败。用户设置的 deadline 会传递给 Tectonic。

本质是分布式 fs 可能 timeout 需要设置比较长，同时一些操作可以容忍慢 IO。前台操作要保证 SLA，后台一些操作还好。超时的时候可能需要一些特殊处理或者直接判定 Fail。

历史上，RocksDB 在 WAL 写入/同步、后台刷盘和合并等关键数据库操作遇到 IO 错误时，会切换为只读模式，以保证数据库一致性。这与 Ext4、XFS 等本地文件系统的做法类似。

在 Tectonic 上，如同大部分分布式系统，IO 错误更多，有的是可以快速恢复的，有的则和单机一样不能恢复。总之，这里的错误率会远比单机的高，但很多也是可以恢复的。这里需要判定更多的 Status: 

> 为此，我们增强了文件系统 API 的返回状态，不仅包含错误码，还包括可重试性、作用域（文件级或全局）及是否永久丢失等元数据。Tectonic 可据此判断错误是否可恢复。RocksDB 侧则聚焦于在文件系统写入错误后恢复数据库一致性并恢复写入。
>
> 对于某些写入失败（如后台刷盘或合并），操作会自动重试且无用户停机。而 WAL 写入失败等情况，则需暂时停止写入，将 memtable 刷盘以保证一致性。