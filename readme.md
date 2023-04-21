## RocksDB Eight Legged

RocksDB 是一个有无数用户的大型项目，最初是 fork 自 leveldb 的一个对 SSD 友好的 LSM-KV 项目，如今已经是用户无数、用例无数、功能无数的银河战舰。如今，RocksDB 的参数和调参都需要技术人员阅读半天代码和技术文档，才能进行玄学调参。同时，虽然 RocksDB 大部分功能的稳定性久经社区考验，但是一些奇妙新功能可能还是没那么多很好的实践。

本项目作为一个收集器，尝试汇总 RocksDB 的材料。

## Official

* RocksDB Wiki: https://github.com/facebook/rocksdb/wiki
* RocksDB Blogs: https://rocksdb.org/blog/

Papers:

* CIDR'17: [Optimizing Space Amplification in RocksDB](https://research.facebook.com/publications/optimizing-space-amplification-in-rocksdb/)
* FAST'20: [Characterizing, Modeling, and Benchmarking RocksDB Key-Value Workloads at Facebook](https://research.facebook.com/publications/characterizing-modeling-and-benchmarking-rocksdb-key-value-workloads-at-facebook/)
* FAST'21: [Evolution of Development Priorities in Key-value Stores Serving Large-scale Applications: The RocksDB Experience](https://research.facebook.com/publications/evolution-of-development-priorities-in-key-value-stores-serving-large-scale-applications-the-rocksdb-experience/)
* TOS'21: [RocksDB: Evolution of Development Priorities in a Key-value Store Serving Large-scale Applications](https://research.facebook.com/publications/rocksdb-evolution-of-development-priorities-in-a-key-value-store-serving-large-scale-applications/)
* SIGMOD'23: **Disaggregating RocksDB: A Production Experience**

## Interesting use cases

* MyRocks
  * ERUOSYS'18: [Reducing DRAM Footprint with NVM in Facebook](https://research.facebook.com/publications/reducing-dram-footprint-with-nvm-in-facebook/)
  * VLDB'20: [MyRocks: LSM-Tree Database Storage Engine Serving Facebook's Social Graph](https://research.facebook.com/publications/myrocks-lsm-tree-database-storage-engine-serving-facebooks-social-graph/)
* TiKV:
  * WriteStall 相关的博客：https://www.pingcap.com/blog/how-to-troubleshoot-rocksdb-write-stalls-in-tikv/
  * https://github.com/tikv/rfcs/blob/master/text/0067-substitute-rocksdb-write-stall.md
  * https://github.com/tikv/rfcs/blob/master/text/0093-rocksdb-per-region.md