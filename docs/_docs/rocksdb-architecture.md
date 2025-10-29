---
docid: rocksdb-architecture
title: RocksDB Architecture
layout: docs
permalink: /docs/rocksdb-architecture.html
---

# RocksDB 架构 (RocksDB Architecture)

## 概述 (Overview)

RocksDB 是一个基于 Log-Structured Merge Tree (LSM) 架构的嵌入式持久化键值存储引擎。它由 Facebook 开发和维护，构建在 Google 的 LevelDB 之上，专为快速存储（尤其是闪存设备）进行了优化。

RocksDB is an embedded persistent key-value storage engine based on the Log-Structured Merge Tree (LSM) architecture. Developed and maintained by Facebook, it builds upon Google's LevelDB and is optimized for fast storage, especially flash devices.

## 核心架构组件 (Core Architecture Components)

### 1. LSM 树结构 (LSM Tree Structure)

RocksDB 采用分层存储架构，数据从内存逐步迁移到磁盘：

RocksDB uses a tiered storage architecture where data gradually migrates from memory to disk:

```
┌─────────────────────────────────────────┐
│          Write Ahead Log (WAL)          │
│         (Durability guarantee)          │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│            MemTable (Active)            │
│         (In-memory write buffer)        │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         Immutable MemTables             │
│    (Waiting to be flushed to disk)     │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         SST Files (Level 0)             │
│         (Sorted String Tables)          │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         SST Files (Level 1)             │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         SST Files (Level 2-N)           │
│        (Compacted and merged)           │
└─────────────────────────────────────────┘
```

### 2. 主要组件说明 (Main Components)

#### MemTable (内存表)
- **功能 (Function)**: 内存中的数据结构，接收所有新的写入操作
- **实现 (Implementation)**: 默认使用跳表 (Skip List)，也支持哈希表、向量等其他实现
- **容量 (Capacity)**: 可配置大小，默认通常为 64MB
- **特点 (Features)**: 有序存储，支持高效的读写操作

The in-memory data structure that receives all new write operations. By default, it uses a skip list implementation, but also supports hash tables, vectors, and other implementations. It maintains sorted storage and supports efficient read and write operations.

#### Write-Ahead Log (WAL / 预写日志)
- **功能 (Function)**: 确保数据持久性的日志文件
- **作用 (Purpose)**: 每次写入首先写到 WAL，然后写入 MemTable
- **恢复 (Recovery)**: 系统崩溃后可通过 WAL 重建 MemTable
- **位置 (Location)**: 存储在磁盘上，与数据文件分开

Ensures data durability through log files. Each write is first written to the WAL, then to the MemTable. After a system crash, the MemTable can be rebuilt from the WAL.

#### Immutable MemTable (不可变内存表)
- **功能 (Function)**: 当 MemTable 达到容量限制时，转变为 Immutable MemTable
- **状态 (State)**: 只读，不再接受新的写入
- **处理 (Processing)**: 等待后台线程将其刷新到磁盘

When a MemTable reaches its capacity limit, it becomes an Immutable MemTable. It is read-only and waits for background threads to flush it to disk.

#### SST Files (Sorted String Table / 排序字符串表)
- **功能 (Function)**: 磁盘上的不可变数据文件
- **格式 (Format)**: 
  - Data blocks: 存储实际的键值对
  - Index blocks: 快速定位数据块
  - Filter blocks: 布隆过滤器，快速判断键是否存在
  - Meta blocks: 元数据信息
- **分层 (Levels)**: 组织成多个层级 (Level 0 到 Level N)

Immutable data files on disk, organized into multiple levels (Level 0 to Level N). They contain data blocks, index blocks, filter blocks (Bloom filters), and metadata blocks.

## 写入路径 (Write Path)

```
User Write
    ↓
┌──────────────┐
│ Write to WAL │ ← Durability
└──────────────┘
    ↓
┌─────────────────┐
│Write to MemTable│ ← In-memory
└─────────────────┘
    ↓
Return Success
```

写入流程 (Write Flow):

1. **写入 WAL**: 首先将数据追加写入 WAL，确保持久性
2. **写入 MemTable**: 然后将数据写入活动的 MemTable
3. **返回成功**: 向客户端返回写入成功
4. **后台刷新**: 当 MemTable 满时，后台线程异步将其刷新到 SST 文件

The write process: First writes to WAL for durability, then to the active MemTable, and returns success. When the MemTable is full, background threads asynchronously flush it to SST files.

## 读取路径 (Read Path)

```
User Read
    ↓
┌─────────────────┐
│ Check MemTable  │ ← Fastest
└─────────────────┘
    ↓ (if not found)
┌──────────────────────┐
│Check Immutable Tables│
└──────────────────────┘
    ↓ (if not found)
┌─────────────────┐
│Check Level 0    │ ← Use Bloom filters
└─────────────────┘
    ↓ (if not found)
┌─────────────────┐
│Check Level 1-N  │ ← Binary search
└─────────────────┘
    ↓
Return Result
```

读取流程 (Read Flow):

1. **查找 MemTable**: 首先在活动 MemTable 中查找
2. **查找 Immutable MemTables**: 按时间倒序查找
3. **查找 SST 文件**: 
   - Level 0: 可能重叠，需要检查多个文件
   - Level 1+: 使用二分查找定位文件
4. **使用优化**:
   - Block Cache: 缓存热数据块
   - Bloom Filters: 快速排除不存在的键
   - Index Blocks: 快速定位数据块位置

The read process searches from newest to oldest: active MemTable, immutable MemTables, and SST files in levels. It uses Block Cache, Bloom filters, and index blocks for optimization.

## 压缩机制 (Compaction)

压缩是 RocksDB 的核心后台操作，用于：
- 合并多个 SST 文件
- 删除过期数据和墓碑标记
- 维护层级结构
- 控制读放大、写放大和空间放大

Compaction is a core background operation in RocksDB used to merge multiple SST files, remove obsolete data and tombstones, maintain level structure, and control read, write, and space amplification.

### 压缩类型 (Compaction Types)

#### 1. Minor Compaction (Flush)
- 将 Immutable MemTable 刷新到 Level 0
- 不涉及合并操作
- 快速完成

Flushes Immutable MemTable to Level 0 without merging operations.

#### 2. Major Compaction
- **Level Compaction**: 
  - 默认策略
  - 每层大小是上一层的 10 倍
  - Level 0 特殊处理（文件可能重叠）
  
- **Universal Compaction**:
  - 适合写密集型工作负载
  - 更低的写放大
  - 可能导致更高的空间放大

- **FIFO Compaction**:
  - 简单的先进先出策略
  - 适合时间序列数据
  - 定期删除旧数据

Different compaction strategies: Level Compaction (default, each level 10x larger than previous), Universal Compaction (lower write amplification for write-heavy workloads), and FIFO Compaction (simple strategy for time-series data).

### 压缩触发条件 (Compaction Triggers)

1. **层级大小超限**: 某一层的总大小超过配置的阈值
2. **文件数量过多**: Level 0 文件数量超过阈值
3. **查找开销过大**: 读取需要访问太多文件
4. **手动触发**: 用户显式调用压缩

Compaction is triggered when: layer size exceeds threshold, too many files at Level 0, read cost is too high, or manually triggered by user.

## 关键特性 (Key Features)

### 1. 列族 (Column Families)
- 支持在单个数据库中创建多个列族
- 每个列族有独立的配置和 MemTable
- 原子性跨列族写入
- 独立的压缩和刷新策略

Support for multiple column families in a single database, each with independent configuration, MemTable, and compaction/flush strategies.

### 2. 快照和迭代器 (Snapshots and Iterators)
- **快照**: 提供数据库的时间点视图
- **迭代器**: 支持高效的范围扫描
- **前缀迭代**: 优化的前缀查询

Snapshots provide point-in-time views, iterators support efficient range scans, and prefix iteration is optimized for prefix queries.

### 3. 事务支持 (Transaction Support)
- OptimisticTransactionDB: 乐观并发控制
- TransactionDB: 悲观锁机制
- 支持原子性和隔离性保证

Support for both optimistic and pessimistic transaction mechanisms with atomicity and isolation guarantees.

### 4. 备份和检查点 (Backup and Checkpoints)
- 在线备份功能
- 增量备份支持
- 检查点创建硬链接快照

Online backup functionality, incremental backup support, and checkpoint creation with hard-link snapshots.

### 5. 数据压缩 (Data Compression)
支持多种压缩算法:
- Snappy: 快速压缩，默认选项
- LZ4: 更快的压缩和解压
- Zlib: 更高的压缩率
- ZSTD: 平衡压缩率和速度
- 分层压缩策略

Support for multiple compression algorithms with different trade-offs between compression ratio and speed.

## 性能优化组件 (Performance Optimization Components)

### 1. Block Cache (块缓存)
- LRU 缓存热数据块
- 可配置大小
- 显著减少磁盘 I/O

LRU cache for hot data blocks, significantly reducing disk I/O.

### 2. Bloom Filters (布隆过滤器)
- 快速判断键是否可能存在
- 降低读放大
- 可配置误报率

Quick determination of whether a key might exist, reducing read amplification.

### 3. Table Cache
- 缓存打开的 SST 文件句柄
- 减少文件打开开销
- 加速文件访问

Caches open SST file handles, reducing file open overhead and accelerating file access.

### 4. 多线程压缩 (Multi-threaded Compaction)
- 并发执行多个压缩任务
- 充分利用多核 CPU
- 可配置压缩线程数

Concurrent execution of multiple compaction tasks to fully utilize multi-core CPUs.

## 写放大、读放大和空间放大 (WAF, RAF, SAF)

RocksDB 的设计在三个放大因子之间进行权衡：

RocksDB's design balances three amplification factors:

- **Write Amplification (写放大)**: 实际写入磁盘的数据量 / 用户写入的数据量
- **Read Amplification (读放大)**: 为完成一次读取需要读取的数据量 / 实际返回的数据量
- **Space Amplification (空间放大)**: 磁盘使用空间 / 实际数据大小

这些因子通过压缩策略、层级配置等参数进行调优。

These factors are tuned through compaction strategies, level configuration, and other parameters.

## 总结 (Summary)

RocksDB 的架构设计使其特别适合：
- 写密集型工作负载
- 需要高性能的嵌入式数据库场景
- 闪存存储设备
- 多 TB 级别的数据存储

The architecture makes RocksDB particularly suitable for write-heavy workloads, high-performance embedded database scenarios, flash storage devices, and multi-terabyte data storage.

关键优势包括：
- 高写入吞吐量
- 可预测的查询性能
- 灵活的配置选项
- 强大的压缩机制
- 丰富的功能特性

Key advantages include high write throughput, predictable query performance, flexible configuration options, powerful compaction mechanisms, and rich feature set.

## 参考资源 (References)

- [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki)
- [RocksDB GitHub Repository](https://github.com/facebook/rocksdb)
- [Getting Started Guide](getting-started.html)
- [FAQ](support/faq.html)
