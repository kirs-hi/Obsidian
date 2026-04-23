---
title: "Paimon在58交易数据的应用和实践"
source: "https://mp.weixin.qq.com/s/ZU-_ngAzebNgKdUGfgCbtQ"
author:
  - "[[张云浩,高剂斌]]"
published:
created: 2026-04-23
description: "Paimon在交易数据的应用和实践"
tags:
  - "clippings"
---
![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2VY3NksPSaFjKrJFpibBdibmdWgQfBsYmoibPhe3Me6pGXibW9w0ztKPlT1SzD3U1rjiaCs6bIfCStw0V7nfbqG6AEQ/0?wx_fmt=jpeg)


1.1 业务背景和原有实时数仓架构

交易数据相关的实时需求主要有两部分: 一部分是需要给业务提供实时业绩数据，支持业务方业绩冲刺；另一部分是需要提供实时的当月新续会员数。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2VY3NksPSaFjKrJFpibBdibmdWgQfBsYmoClMJdtCvwIWrTy2q3UJzRmV8Ro8nFBIiaIyoyEISja5gTDkjnO1vIZQ/640?wx_fmt=webp&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

图 1：原数据架构

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2VY3NksPSaFjKrJFpibBdibmdWgQfBsYmozHFaxF7VQjgRnXEdJjIDucz5K5Mial3AhWzRCQrCoDgbaGqBKWsp9CA/640?wx_fmt=webp&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=1)

图 2：原数据流转过程

上图展示了我们之前的实时数仓计算架构。前面从ods至app的部分是实时链路，主要由 Flink、Kafka 和 Hbase、Mysql 组成。此部分链路中的数据来自在线系统生成的交易数据日志。数据经过wmb(公司自研的消息队列系统)，使用Flink streaming对其进行自定义的数据转换操作，将数据下发到Kafka，后续流程使用Flink sql构建实时数仓，最终结果落在Mysql中，供下游系统使用。

在App应用层，实时系统主要聚焦于实时当天数据的处理与分析。由于实时计算链路出于系统稳定性或资源优化的考量，通常无法存储完整的周期性数据，这导致实时计算产出的数据结果存在时效性限制。针对此类场景下关键指标的精确性要求，我们采用离线批处理与实时流计算相结合的混合架构，通过离线补算机制完善数据缺口，最终生成完整准确的最终结果。

1.2 原架构存在的问题:

- 需求灵活多变：首先，在业务层面，随着业务需求的变化，每个环节都需要维护和开发，这在灵活性和效率上构成了重大挑战
- 开发成本高：其次，需要维护的组件较多(Kafka、Hbase、Redis、MySQL等)，另外每次任务逻辑变更都会导致状态丢失， 重复回溯数据验证数据逻辑费时费力
- 资源浪费：第三，数据在MySQL和Hive分别保存一份，造成数据冗余，另外，Kafka中的数据不能点查，为了方便问题排查，我们每个流程中数据的中间计算结果都落在了MySQL保存一份用于排查问题，间接地导致了资源浪费
- 数据口径不一致：第四，数据口径问题，如果在某个业务系统中忘记或未能及时更新数据口径，那么输出的数据就会出现问题
- 运维成本高：最后，运维方面，数据出现问题，排查需要投入大量时间和成本

2

**基于Paimon构建流式湖仓**

  

2.1 关于Paimon

Paimon 是一个实时数据湖格式，具备以下核心功能：

- 实时更新：基于 LSM 树结构，支持主键表的高效流式更新（如去重、部分更新、聚合），并提供变更日志（Changelog）生成能力，简化流式分析。
- 流批一体：统一支持流式与批处理操作，兼容 Flink、Spark、Hive 等引擎，实现数据湖的实时写入与离线分析。
- OLAP 优化：列式存储（默认 ORC 格式）、Z-order/Hilbert 排序、数据跳过（minmax 索引）等技术，加速复杂查询。
- 数据湖能力：支持 ACID 事务、时间旅行（版本回溯）、可扩展元数据（PB 级存储）及模式演变。
- 分支管理：无锁创建数据分支，支持独立测试、验证，通过快照和标签实现分支同步（Fast Forward）。
- 低成本与生态兼容：依托对象存储（如 OSS/S3），降低存储成本，并无缝集成主流计算框架，构建 Streaming Lakehouse 架构。

2.2 Paimon流式湖仓架构及应用场景

下面是基于Paimon实现的流式湖仓架构及新数据流转过程：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2VY3NksPSaFjKrJFpibBdibmdWgQfBsYmoBfibWRa8S4JnKrqMAj9B4P2S81kBP5EicTJiaM7QhqKsB6lY0L7ZdicPRg/640?wx_fmt=webp&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=2)

图 3：基于Paimon实现的流式湖仓架构

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2VY3NksPSaFjKrJFpibBdibmdWgQfBsYmoPwGb3iaPDyWgbxyY0yZU72nxQKuU7JFxzktPBTOo6x1ug4T9w8b7XSw/640?wx_fmt=webp&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=3)

图 4：新数据流转过程

项目在构建流式湖仓过程中，主要针对以下几个场景基于Paimon进行了实践：

2.2.1 去重

在构建实时数仓的过程中，数据去重（即确保每条记录在主键维度上唯一）是一个非常关键且常见的需求。早期我们采用的是基于 Flink SQL 的 ROW\_NUMBER() 窗口函数进行去重处理，这种方式虽然实现简单、逻辑清晰，但在面对高并发、大数据量场景时暴露出诸多性能瓶颈和维护难题。

为了提升系统整体的稳定性与效率，我们尝试引入 Paimon 主键表（Primary Key Table） 来替代原有的去重逻辑，并取得了显著的效果。

1）原有方案痛点分析 —— ROW\_NUMBER()

我们通常使用如下结构的 SQL 进行去重：

```
WITH ranked_data AS (
```

该语句通过窗口函数对相同主键的数据按时间排序并编号，只保留最新的一条记录。

存在问题

- 资源消耗大：
- 需要全局排序，Shuffle 数据量巨大
- 内存占用高，尤其在窗口较大或数据倾斜严重时
- Checkpoint 文件体积膨胀，影响任务稳定性
- 开发复杂度高：
- 每次去重都需要写冗长的 SQL，容易出错
- 不同层级的去重逻辑难以统一，维护成本高
- 数据一致性差：
- 如果数据乱序严重，可能造成错误的去重结果
- 下游消费时仍需再次处理，增加链路复杂性

2）新方案探索 —— Paimon 主键表

我们将原来通过 ROW\_NUMBER() 去重生成的中间层表替换为 Paimon 主键表，具体流程如下：

```
CREATETABLEpaimon_table (
```

使用Flink 流任务将原始数据直接写入 Paimon 表，无需手动去重。

```
insert into paimon_table
```

由于 Paimon 主键表的 Merge 引擎会自动保留最后一条记录，因此等价于实现了ROW\_NUMBER()的效果。

3）下游消费方式优化

下游任务可以直接以流的方式读取 Paimon 表的 Changelog，获取最新的变更数据，而无需再做额外的去重操作。

4）实际收益总结

| 具体收益 | 内容总结 |
| --- | --- |
| 资源效率显著提升 | CPU 和内存使用下降明显，资源利用率提升。  Checkpoint 更加稳定，任务恢复速度更快。 |
| 开发维护更高效 | 原来的复杂去重逻辑被完全简化，只需定义主键即可。  减少了多层任务之间的耦合，提升了系统的可维护性。 |

2.2.2 数据乱序

在构建实时数仓的过程中，需要处理数据乱序问题。特别是在事件时间（Event Time）维度下，迟到的数据如果不加以控制，可能会导致状态更新错误或数据不一致，进而影响下游业务的准确性。

早期我们在 Flink 实时任务中通过 Redis 缓存上一条记录的时间戳 来判断当前数据是否为“新”数据，并决定是否进行更新操作。这种方式虽然简单有效，但在实际应用中也暴露出诸多问题。

为了提升系统的稳定性与可维护性，我们引入了 Paimon 的 sequence.field 特性，实现了一种更优雅、高效且可扩展的数据去重与乱序判断机制。

1）原有方案 —— Redis 缓存时间戳判断乱序

我们的核心逻辑如下：

1. 使用 Flink 算子读取 Kafka 中的数据；
2. 每条数据携带一个事件时间字段（如 event\_time)；
3. 在算子中使用 Redis 缓存该主键对应的最新事件时间；
4. 如果当前数据的 event\_time小于缓存值，则认为是过期数据，跳过处理；
5. 否则，更新 Redis 并继续后续流程。

存在问题

- 性能瓶颈明显：
- 每条数据都需要访问 Redis，造成网络 I/O 压力大
- 高并发场景下容易成为系统瓶颈
- 状态一致性难以保证：
- Redis 是外部存储，无法与 Flink Checkpoint 联邦
- 出现故障恢复时，可能导致状态不一致或重复处理
- 开发耦合严重：
- 逻辑依赖 Redis，不利于组件解耦

2）新方案探索 —— Paimon 的 sequence.field

我们定义一张 Paimon 主键表并设置 sequence.field 参数：

```
CREATETABLEpaimon_table (
```

Paimon 内部会自动根据 event\_time 字段对相同主键的数据进行排序，只保留最新的那条记录。如果新写入的数据 event\_time 更小，则会被忽略，从而实现了乱序过滤+去重的效果。

3）实际收益总结

| 具体收益 | 内容总结 |
| --- | --- |
| 无需额外状态管理 | 所有状态由 Paimon 自动管理，与 Flink Checkpoint 联邦。  支持 Exactly-Once 语义。 |
| 天然支持乱序判断 | 不再依赖 Redis 或其他外部组件。  通过字段比较即可完成数据有效性判断。 |
| 开发简洁易维护 | 只需定义字段名即可启用乱序判断功能。  多个业务模块可复用同一套机制。 |
| 性能更高 | 所有计算都在写入端完成，避免了频繁的外部访问。  LSM Tree 结构支持高效合并与索引查找。 |

2.2.3 Paimon 流读

在原有的实时数仓架构中，Kafka 被用作中间消息队列，承担数据采集、缓冲和分发的角色。Flink 等流计算引擎通过 Kafka 消费原始数据，并进行清洗、聚合等处理后写入下游系统。

1）原有方案痛点分析 —— Kafka 消费架构

传统基于 Kafka 的实时链路如下：

1. 数据源将原始数据写入 Kafka；
2. Flink 任务从 Kafka 中消费数据；
3. 经过 ETL 处理后，写入下游结果表（如 DWD、DWS 层）；
4. 下游任务再次从 Kafka 或其他表中读取结果继续加工。

存在问题

- 数据不可查：
- Kafka 是纯日志型存储，无法直接支持点查或范围查询
- 要想查询历史数据，必须重新消费 Topic，效率低下
- 状态管理复杂：
- Kafka 只保存偏移量，具体的状态逻辑由 Flink 自己维护
- 频繁的 Checkpoint 和恢复操作容易导致状态膨胀或不一致
- 链路过长：
- 每层都需要独立的 Kafka Topic，增加了运维成本
- 数据重复传输浪费资源，影响整体性能
- 难以修复数据：
- 一旦某一层处理出错，需从 Kafka 重新消费重跑整个链路
- 没有统一的“快照”机制，修复效率低

2）新方案探索 —— Paimon 流读表

我们将原本 Kafka 承担的部分职责迁移至 Paimon 表中，构建了一条更简洁、可查、易维护的实时链路。具体流程如下：

(1）定义 Paimon 表结构

```
CREATETABLEpaimon_ods_table (
```

(2）将原始数据写入 Paimon 表

Flink 流任务直接写入 Paimon 表，替代 Kafka：

```
INSERT INTO paimon_ods_table
```

(3）下游任务流式读取 Paimon 表

Flink 可以像消费 Kafka 一样订阅 Paimon 表的 Changelog 流：

```
createviewview_paimon_changelog_streamAS
```

3）实际收益总结

| 具体收益 | 内容总结 |
| --- | --- |
| 资源效率显著提升 | 减少了 Kafka 的频繁写入和读取压力。  整体任务吞吐量提升了约 40%，Checkpoint 更加稳定。 |
| 开发维护更高效 | 所有数据统一写入 Paimon 表，无需再维护多个 Kafka Topic  流读和快照查询共用一套表结构，减少重复开发。 |

2.2.4 数据聚合

原来在计算天粒度汇总数据的时候，使用到了累积窗口（Cumulate Window）来进行统计当天指标数据。

1）原有方案痛点分析 —— Flink 累积窗口

Flink 的 Cumulate() 函数允许定义最大窗口长度和累积步长。例如：

```
SELECT
```

该语句会按自然日（最大窗口长度）逐步累加每60秒（步长）的订单金额，最终输出每日的累计结果。

存在问题

- 性能瓶颈：
- 每个累积窗口需要频繁触发聚合计算，导致 Shuffle 和状态存储压力大
- 随着时间推移，窗口数量增加，内存占用呈线性增长
- Checkpoint 文件体积膨胀，任务恢复耗时显著增加
- 开发复杂度高：
- 需要手动管理窗口边界和触发逻辑
- 多层嵌套的 SQL 结构难以维护，容易出错
- 数据一致性差：
- 依赖事件时间（Event Time）和水印（Watermark）处理乱序数据
- 若数据延迟严重，可能导致统计结果滞后或重复计算
- 查询效率低：
- 聚合结果存储在 Flink 状态中，无法直接对外提供查询服务
- 下游消费需重复读取原始数据重新计算，形成资源浪费

2）新方案探索 —— Paimon 的 Aggregate Table

Paimon 的 Aggregation Table 支持通过主键和聚合函数定义预聚合逻辑。其核心特性包括：

- 主键 + 聚合函数：相同主键的数据会根据聚合函数（如sum、max等)合并，保留最新结果。
- LSM Tree 结构：高效管理数据文件，支持快速合并和查询。
- Changelog 流式输出：预聚合结果可实时对外提供变更流（Changelog），供下游消费。

定义Paimon Aggregate Table: 

```
CREATETABLEpaimon_table (
```

Flink 流任务将原始数据写入 Paimon 表，无需手动管理窗口逻辑：

```
INSERT INTO paimon_table
```

Paimon 会自动根据主键合并数据，按聚合规则更新score和update\_time。

3）实际收益总结

| 具体收益 | 内容总结 |
| --- | --- |
| 资源效率显著提升 | 消除了 Flink 状态存储的压力，CPU 和内存使用下降明显。  Checkpoint 大小减少 90%，任务恢复速度提升 70%。 |
| 开发维护更高效 | 所有聚合逻辑统一收口，减少重复开发。  下游任务无需再编写复杂的窗口 SQL，直接订阅 Changelog 流即可。 |
| 数据一致性更强 | 主键更新的原子性保证了统计结果的准确性。  不再依赖 Watermark 处理乱序数据，避免因延迟导致的重复计算。 |
| 支持更多业务场景 | 可广泛应用于实时报表（如 T+0 日报）、动态指标监控（如实时 GMV）等场景。  为构建统一的指标中心提供了基础能力支撑。 |

注: 

- 若需求是静态聚合或最终结果，Paimon 的 Aggregate Table 可完全替代 Cumulate Window。
- 若需求涉及时间驱动的累积计算，需结合 Flink 的窗口功能和 Paimon 的聚合能力。

2.2.5 维表 Lookup Join

随着业务变更以及对开发效率的要求，原有基于 HBase、KV 存储或 MySQL 的维度表 Join 方案逐渐暴露出链路复杂、维护成本高、扩展性差等问题。为优化架构、提升效率，我们引入 Paimon 作为新一代统一的数据湖仓存储引擎，尝试重构部分实时链路，替代原有的多系统 Join 操作。

1）原有方案痛点分析 —— 涉及多种数据源Lookup Join

原来构建的实时数仓，维度表依赖 MySQL、HBase 等关系型数据库或 KV 存储系统。

存在问题

- 性能瓶颈
- MySQL 在高并发场景下容易成为 Lookup Join 的性能瓶颈，尤其是在高频维度查询时，状态管理复杂且磁盘随机读频繁
- HBase 等 KV 系统虽然能支撑高并发，但其运维成本高，且无法直接支持复杂查询
- 架构复杂性
- 维度表与事实表分散在不同系统中（如 Kafka + MySQL + HBase），数据治理困难
- 一致性风险
- 维度表更新需通过异步同步机制，存在延迟和数据不一致的风险
- 成本与扩展性
- 维度表存储成本高（如 HBase 的存储开销），且需维护多套存储系统
- 离线批量导入方式导致数据新鲜度低，无法满足时效性要求

2）新方案探索 —— Paimon Lookup Join

Paimon提供了两种Lookup Join缓存构建策略：即FULL Cache和AUTO Cache。其缓存的底层实现是采用了两种方式：本地构建Hash Store的方式以及本地采用RocksDB存储的方式。如果是FULL模式，则在计算节点本地会用RocksDB作为缓存的存储；如果是AUTO模式，则Paimon 会根据表的大小和 内存限制动态决定缓存策略，来决定在计算节点侧采用Hash Store还是RocksDB来存储缓存数据。

Lookup Join 方式划分：

1. FullCacheLookupTable

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 5：Full Cache Lookup Join

表在启动时读 Paimon 表全部数据Load到本地 RocksDB。通过本地 RocksDB 的点查能力来满足维表关联的过程。由于缓存了全部数据，所以 Lookup 效率较高，但是对本地磁盘需求较大，另外由于存在初始化加载的过程，所以在流任务启动的时候加载较慢。

2.Local PrimaryKeyPartialLookupTable

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 6：Local PrimaryKey Partial Lookup Join

由数据触发数据文件的 Load。当接收到数据时，根据 Join Key 分析出 Key 所属的 Partition 和 Bucket，然后基于 LSM Tree 的有序性进行文件的定位和查找。当命中某个文件时，将其 Load 到本地转化为 Lookup File。由于不需要缓存全部数据，所以对磁盘大小不再是强需求，但同样也存在一些缺陷，从上图可以看出，该方式的每个task由于数据到达 Lookup Join 算子比较随机，导致后续每个并发的数据需要不断缓存不同的文件数据，缓存效率相对较差。

适用场景: BucketMode 为HASH\_FIXED 的 PK 表，且 PK 与 JoinKey 相同。

3.Bucket Shuffle LookupTable

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 7：Bucket Shuffle Lookup Join

通过在 Flink 侧支持在 Lookup Join 算子前插入 Custom Shuffle 的策略，一个 Task 只加载一个 Bucket 的数据。每个 Task 所需缓存数据变少了，极大提升了 Paimon 作为维表被 Lookup Join 的可用性。同样也有一定的缺陷，流作业要加层 Shuffle，有多少个 Paimon 表要被 Lookup Join，就要加几层 Shuffle。

适用场景: BucketMode 为HASH\_FIXED 的 PK 表，且 PK 与 JoinKey 相同。

注: 该方式flink1.20版本可用。

根据Paimon Lookup Join的原理，我们目前使用的lookup.cache都是默认配置AUTO，所创建的维度表都是采用Hash固定桶模式且主键和join key完全相同。

Paimon维度表，表结构:

```
CREATETABLE \`dim_table\`(
```

flink 流任务，事实表关联维表：

```
select
```

在Paimon系统中进行维度表分桶配置时，需重点考虑分桶数量与lookup算子并发度的匹配关系。lookup算子的吞吐能力直接受其并发度影响，而并发度的上限则由lookup表的分桶数量所制约。为避免数据分布不均导致的性能问题，建议将lookup算子的并发度设置为分桶数的约数，否则可能引发数据倾斜现象。例如，当lookup表的分桶数为64时，推荐的并发度可设置为64、32或16等因数值，此时增加并发度至超过64并不会提升处理效率。

另外还可以通过配置参数对lookup进行优化，可以增大配置 lookup.cache-max-memory-size(默认256M)或lookup.cache-rows(默认10000)，这可以使得维表的数据更多地保存在内存中，提高查询效率。

3）实际收益总结

使用 Paimon 进行 Lookup 操作后，可以替代部分原本通过 Join HBase 和 KV、mysql 系统完成的任务。这带来了多方面的收益：

| 具体收益 | 内容总结 |
| --- | --- |
| 统一数据管理平台 | 将原本分散在多个系统中的数据集中到数据湖仓中进行管理。  支持标准 SQL 查询和写入操作，显著提升数据处理的便利性与效率。 |
| 精简数据流程架构 | 不再依赖额外的 KV 或 MySQL 系统，减少了数据链路的复杂度。  有效降低开发维护的工作量，节省运维成本。 |

注意，Paimon lookup join目前仍存在一些缺陷：

- 性能与适用场景限制
- 相较于专用的键值存储系统，Paimon 在高频更新或实时性强的场景下表现有限
- 时效性为被 Lookup Join 的 Paimon 表的入湖 Flink 流作业的 Checkpoint 时长，当前更适合用于更新频率较低的维表场景
- 扩展能力受限于计算资源
- 可扩展性依赖于 Flink 计算节点的横向扩展能力
- 如果需要提升并发能力，可能还需重新配置 Paimon 表的 Bucket 分布，增加部署复杂度

2.2.6 离线批处理

在构建实时数仓的过程中，数据修复（Data Repair） 是一个高频且关键的操作。无论是任务 Failover、逻辑迭代，还是上游数据异常，都需要对历史数据进行修正。早期我们依赖 ODS 层数据的重新消费 来完成数据修复，但随着业务复杂度和数据量的增长，这种方式逐渐暴露出效率低、资源消耗大、维护成本高等问题。

为了解决这些问题，我们引入了 Paimon 的批处理能力，通过其主键表（Primary Key Table）、Changelog 流式输出和高效的 LSM Tree 结构，实现了离线批处理修数。

1）原有方案痛点分析 —— ODS 重放修数

在原有方案中，数据修复流程如下：

1. 定位问题时间段：确定需要修复的数据时间范围（如 2024-01-01 00:00 到 2024-01-01 06:00）。
2. 重放 ODS 数据：从 Kafka 或日志系统中读取该时间段内的原始数据。
3. 重新计算逻辑：通过 Flink 任务重新执行修复逻辑（如修正字段值、补全缺失数据）。
4. 覆盖写入下游：将修复后的数据写入目标表，覆盖原有数据。

存在问题

- 资源消耗大：
- 需要重新消费大量原始数据，导致计算和存储资源占用高
- 对 Kafka 等消息队列造成额外压力
- 效率低下：
- 修复时间长（例如 1 天的数据可能需要几十分钟甚至数小时）
- 需要多次全量扫描，无法精准定位变更数据
- 维护复杂：
- 修复逻辑需重复开发，容易引入新错误
- 任务调试和验证成本高，影响业务稳定性
- 数据一致性风险：
- 重放过程中若出现中断，可能导致数据不一致
- 多次覆盖写入可能引发状态冲突

2）新方案探索 —— Paimon 批处理修数

我们将原有的 ODS 重放修数流程迁移至 Paimon 批处理模式，具体流程如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 8：批处理修数流程

离线修数过程我们首先停掉相关的dwd层以及后续Flink实时任务，防止Flink流任务和Spark批任务同时写一张数据表导致底层数据文件冲突，然后从ods表指定对应的修复时间范围开始修复数据，通过 Spark SQL 读取 Paimon 表并执行修复逻辑， 数据逻辑和实时任务数据逻辑一致。

批量数据修复完成后，重启对应的dwd层，从指定的时间戳之后开始消费增量数据，后续重新启动对应Flink实时任务即可。

3）实际收益总结

| 具体收益 | 具体收益 |
| --- | --- |
| 资源效率显著提升 | 消除了 ODS 重放的冗余计算。  修复任务耗时从几十分钟缩短到几分钟，资源利用率提升 80%。 |
| 开发维护更高效 | 修复逻辑直接作用于 Paimon 表，无需重复开发 ODS 重放任务。 |
| 数据一致性更强 | Paimon 的主键表机制确保修复后的数据原子性更新。  Exactly-Once 语义避免因任务中断导致的数据不一致。 |
| 支持更多业务场景 | 为构建统一的数据修复中心提供了基础能力支撑。 |

  

3

## 总结

  

本次基于 Paimon 的实时数仓重构项目，产出结果已提供给下游业务方使用，下游可根据需要流式消费或批量查询，整体项目不仅解决了原有架构中存在的痛点问题，还实现了数据链路的简化和资源成本的大幅优化，也为未来构建高性能、易维护、可扩展的一体化湖仓架构奠定了坚实基础。展望未来，我们将继续深耕湖仓一体方向，不断拓展 Paimon 在各类业务场景中的应用边界，打造更加智能、高效、稳定的数据服务。

  

【作者简介】

张云浩，58同城-TEG大数据部-高级数据开发工程师

高剂斌，58同城-TEG大数据部-资深数据开发工程师

预览时标签不可点

名称已清空

**微信扫一扫赞赏作者**

喜欢作者其它金额

赞赏后展示我的头像

作品

暂无作品

喜欢作者

其它金额

¥

最低赞赏 ¥0

**其它金额**

赞赏金额

¥

最低赞赏 ¥0

1

2

3

4

5

6

7

8

9

0

.

大数据 · 目录

#大数据

上一篇58技术沙龙第40期｜基础架构专场：中间件&amp;云原生

搜索「」网络结果

**

调整当前正文文字大小

**

100%

​

留言

暂无留言

1条留言

已无更多数据

发消息

  写留言:

微信扫一扫  
关注该公众号

继续滑动看下一个

![](http://mmbiz.qpic.cn/mmbiz_png/2VY3NksPSaHwuLn7lR4jliaw4zWYgqe3q20mHq1kCqcxJbibWSTBj1tiaMib8Wq5SOA975IKdicXx4tEMz0A6qkhicfw/0?wx_fmt=png)

58技术

向上滑动看下一个

当前内容可能存在未经审核的第三方商业营销信息，请确认是否继续访问。

继续访问取消

[微信公众平台广告规范指引](javacript:;)

知道了

 微信扫一扫  
使用小程序

取消 允许

取消 允许

取消 允许

 

![作者头像](http://mmbiz.qpic.cn/mmbiz_png/2VY3NksPSaHwuLn7lR4jliaw4zWYgqe3q20mHq1kCqcxJbibWSTBj1tiaMib8Wq5SOA975IKdicXx4tEMz0A6qkhicfw/0?wx_fmt=png)

微信扫一扫可打开此内容，  
使用完整服务

![](https://mmbiz.qpic.cn/mmbiz_png/2VY3NksPSaHwuLn7lR4jliaw4zWYgqe3q20mHq1kCqcxJbibWSTBj1tiaMib8Wq5SOA975IKdicXx4tEMz0A6qkhicfw/300?wx_fmt=png&wxfrom=18)

58技术

： ， ， ， ， ， ， ， ， ， ， ， ， 。   视频 小程序 赞 ，轻点两下取消赞 在看 ，轻点两下取消在看 分享 留言 收藏 听过

可在「公众号 > 右上角 \> 划线」找到划线过的内容

![划线引导图](https://res.wx.qq.com/op_res/opqv3ix6k9E4e64ZzO7uIqE3ZblwIojfmt7u70m59yS1ylFK-hTu6Ra8V_LaWQJ1P4OlUJPdXLfVBtrm3TwRrw)

我知道了

,

,

选择留言身份

**

留言

**

暂无留言

1条留言

已无更多数据

发消息

  写留言:

**

大数据

**

详情

正在加载

## 确认提交投诉

你可以补充投诉原因（选填）

确定

<iframe src="https://open.weixin.qq.com/pcopensdk/frame" allow="local-network-access" style="display: none;"></iframe>