# Mini-LSM 仓库功能说明

## 一句话定位

Mini-LSM 是一门"用 Rust 从零写 LSM-Tree 存储引擎"的实战课程仓库：把一个数据库**存储层**拆成 21 个小节，边写代码边跑测试，最终得到一个支持压缩、崩溃恢复、MVCC 和事务的键值存储引擎。

它讲的是数据库 API 之下那一层发生了什么（RocksDB / LevelDB 的核心机制），不涉及 SQL 解析、查询优化、复制、分布式共识。

## 仓库能力清单

| 能力 | 说明 |
| --- | --- |
| 内存表（Memtable） | 有序写缓冲，基于跳表/BTreeMap 的有序键值集合，带冻结与替换逻辑 |
| SST 文件格式 | Block 编码（前缀压缩 + 重启点）、布隆过滤器、元数据与校验和 |
| 读路径 | 点查（get）、范围扫描（scan），跨 memtable 与多层 SST 合并迭代 |
| 写路径 | 写 memtable → 冻结 → 落盘 flush，写放大控制 |
| 迭代器体系 | 单表迭代、双路合并、多路合并（merge iterator）、拼接迭代器 |
| 压缩（Compaction） | 简单分层（Simple Leveled）、Tiered（RocksDB Universal）、Leveled 三种策略 + 后台压缩任务与调度 |
| 持久化与恢复 | Manifest（SST 清单的原子记录）、WAL（预写日志）、批量写与校验和 |
| MVCC | 时间戳键编码、快照读、水印与垃圾回收 |
| 事务 | 基于时间戳的事务 API、乐观并发控制（OCC）、可串行化快照隔离校验 |
| 可观测性 | CLI 交互工具、压缩模拟器、内部状态 dump |

## 目录结构

| 目录 | 作用 |
| --- | --- |
| `mini-lsm/` | 参考实现：第 1~2 周的完整解答（含 memtable、SST、压缩、manifest、WAL） |
| `mini-lsm-mvcc/` | 参考实现：第 3 周 MVCC / 事务部分 |
| `mini-lsm-starter/` | 学员练习用的脚手架，关键函数留空（当前约 50 处 `unimplemented!`、18 处 `TODO`），需要按章节自己补全 |
| `mini-lsm-book/` | 课程正文（mdBook 格式的七章 × 三周讲义与配图） |
| `xtask/` | 课程开发用的自定义 cargo 子命令（复制测试、格式检查、构建电子书、同步 API） |
| `.github/workflows/` | CI：`main.yml`、`pr.yml`，以及本仓新增的跨平台 `matrix.yml` |

## 核心模块（以 `mini-lsm/src` 为例）

| 文件 | 职责 |
| --- | --- |
| `mem_table.rs` | 内存表：写入、冻结、按 key 有序遍历 |
| `block.rs` + `block/` | SST 数据块：builder 编码、iterator 解码、重启点与前缀压缩 |
| `table.rs` + `table/` | SST 读写：builder、iterator、bloom filter |
| `iterators.rs` + `iterators/` | 合并迭代器族：`two_merge`、`merge`、`concat` |
| `compact.rs` + `compact/` | 压缩任务与三种策略实现（simple_leveled / tiered / leveled） |
| `manifest.rs` | 清单文件：记录各层 SST 列表，崩溃后重建状态 |
| `wal.rs` | 预写日志：写入前落盘，重启后回放 |
| `lsm_storage.rs` | 引擎主体：读写入口、状态机、后台线程与任务调度 |
| `lsm_iterator.rs` | 面向用户的扫描游标（含 MVCC 时间戳过滤） |
| `mvcc.rs` + `mvcc/` | 时间戳键、事务对象 `txn.rs`、水印与 GC `watermark.rs` |
| `key.rs` | 键编码：用户键 + 时间戳的组合编码与解码 |
| `tests/` | 按天划分的验收测试（week1_day1 … week3_day7）及回归用例 |

## 课程大纲（21 节）

**第 1 周：存储格式与引擎骨架**

1.1 Memtable · 1.2 Merge Iterator · 1.3 Block · 1.4 SST · 1.5 Read Path · 1.6 Write Path · 1.7 前缀编码与布隆过滤器

**第 2 周：压缩与持久化**

2.1 压缩实现 · 2.2 Simple Leveled · 2.3 Tiered（Universal）· 2.4 Leveled · 2.5 Manifest · 2.6 WAL · 2.7 批量写与校验和

**第 3 周：多版本并发控制**

3.1 时间戳键编码 · 3.2 快照读（memtable 与时间戳）· 3.3 快照读（事务 API）· 3.4 水印与垃圾回收 · 3.5 事务与乐观并发控制 · 3.6 可串行化快照隔离 · 3.7 压缩过滤器

## 常用命令

```bash
# 首次准备工具链与测试
cargo x install-tools
cargo x copy-test --week 1 --day 1

# 检查代码风格 / 跑验收
cargo x scheck
cargo x check

# 交互式体验参考实现
cargo run --bin mini-lsm-cli-ref
cargo run --bin mini-lsm-cli-mvcc-ref

# 压缩策略实验
cargo run --bin compaction-simulator-ref
```

## 本仓（edidada/mini-lsm 分支 `edidada/main`）相对上游的改动

分支比 `origin/main` 多 7 个提交，集中在**跨平台可移植性与 CI**，未改动课程主体逻辑：

- 新增 `.github/workflows/matrix.yml`：Linux / macOS / Windows 多平台构建与测试矩阵
- `table.rs`、`wal.rs`、`manifest.rs`、`lsm_storage.rs` 的文件 IO 改造：目录 fsync、只读句柄、WAL/manifest 截断用"读+写+显式 seek"替代追加模式，修复 Windows 下恢复失败
- CI 侧：macOS 运行器标签从容量受限的 `macos-26` 换成 `macos-latest`，arm64 macOS 任务标记为 best-effort

技术基线：Rust edition 2024，workspace 版本 `0.4.0-alpha.1`，Apache-2.0（课程讲义与图片版权归作者）。

## 参考

- 在线课程：https://skyzh.github.io/mini-lsm
- 上游仓库：https://github.com/skyzh/mini-lsm
- 社区解答合集：仓库根目录 `SOLUTIONS.md`
- 受其启发的项目：SlateDB（对象存储上的 LSM 引擎）、Tonbo（Parquet + LSM 组织）
