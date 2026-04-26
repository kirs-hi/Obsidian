
## 一、简历项目描述

> 本文件是「供应链实时数仓体系建设」项目的架构升级部分，面试时作为第二阶段讲述：先讲原有 Flink+Kafka+Doris 体系怎么建的，再讲为什么要升级、怎么用 Paimon 升级的，形成完整的技术成长叙事。

### 项目名称：供应链实时数仓体系建设（Paimon 湖仓架构升级）

**项目背景：** 旭日供应链体系持续扩展至 13 个子系统（PMS 采购/WMS 仓储/TMS 运输/MMS 维修/OMS 订单/CST 客服等），覆盖骑行、充电宝、备件等硬件设备的全生命周期管理。业务侧，供应链控制塔 SCCT 从早期的"经营数据看板"升级为"一线作战指挥 + 驾驶指挥舱"双模式——一线人员（仓管/物流/维修）需要秒级刷新的作战看板指导日常操作，管理层需要分钟级的指挥舱进行全局资源调度。在原有 Flink+Kafka+Doris 架构支撑下，系统已稳定运行但暴露出三个瓶颈：一是 Kafka 承担层间数据流转导致中间态数据不可查，线上问题排查和修数成本极高（需从 ODS 重放全链路，耗时数小时）；二是去重和乱序处理依赖 ROW_NUMBER + Doris Sequence 列 + Redis 外部状态，链路长、组件多、State 开销大；三是 13 个子系统的维度数据分散在 Redis/Cellar/MySQL 等多套存储中，管理混乱且一致性难保障。基于以上痛点，引入 Apache Paimon 构建流式湖仓，对核心链路进行架构升级。

**职责描述：** 负责 Paimon 湖仓架构的技术选型、方案设计与核心场景落地，推动四大场景从原有架构迁移至 Paimon 方案，完成生产验证与效果评估。

**技术方案：**

- **==场景一：去重与乱序治理==：** 原方案采用 Flink SQL ROW_NUMBER() 窗口函数做去重，每条数据需全局排序 + Shuffle，State 膨胀严重，同时依赖 Redis 缓存时间戳做乱序判断，每条数据一次网络 I/O，高并发下成为瓶颈。新方案采用 Paimon 主键表（deduplicate merge-engine）+ sequence.field 机制：定义主键即完成去重，Paimon 基于 LSM Tree 在写入时自动按 event_time 保留最新记录，乱序数据自动过滤，无需 Redis 外部状态，无需 Flink 维护去重窗口。Flink 去重算子 State 降为零，相关任务 Checkpoint 大小下降约 75%，CPU 和内存开销显著降低。
- **==场景二：流读替代 Kafka 中间层==：** 原方案 ODS→DWD→DWS 每层用独立 Kafka Topic 做数据流转，中间态数据是纯日志流不可查，排查问题需从源头重新消费，修数需重跑全链路。新方案将层间流转迁移至 Paimon 表，下游任务通过 Changelog 流读消费变更数据，同时支持 SQL 点查和时间旅行回溯（Time Travel）。中间态数据从"黑盒"变为"可查可回溯"，Kafka Topic 数量减少约 40%，问题排查效率提升 5 倍以上（直接 SQL 查 Paimon 快照定位问题分区和时间点）。
- **==场景三：聚合下沉存储层==：** SCCT 控制塔的天粒度实时指标（库存周转率、履约时效、设备在线率、人效等）原来依赖 Flink Cumulate Window 做滚动聚合，窗口状态随时间线性增长，Checkpoint 膨胀严重且依赖 Watermark 处理乱序。新方案采用 Paimon Aggregate Table（aggregation merge-engine），按主键自动合并数据并执行预定义聚合函数（SUM/LAST_NON_NULL 等），聚合逻辑从计算层下沉到存储层。消除 Flink 窗口 State，聚合类任务 Checkpoint 大小下降约 90%，任务恢复速度提升 70%，且不再依赖 Watermark，天然支持乱序数据的正确聚合。
- **==场景四：统一维表与批处理修数==：** 旭日 13 个子系统的维度数据（设备信息、仓库信息、供应商信息、物流网络等）原来分散在 Redis/Cellar/MySQL 中，不同系统的维度表更新频率和存储方式各异，关联逻辑散落在各个 Flink 任务中。新方案将维度数据统一收口至 Paimon 维表，事实表通过 Lookup Join（Local PrimaryKey Partial Lookup 模式，基于 LSM Tree 有序性定位查找）进行关联，减少外部组件依赖，维表管理从"各自为政"变为"统一存储、统一关联"。同时利用 Paimon 的批处理修数能力：停掉下游 Flink 任务 → Spark SQL 直接对 Paimon 主键表执行修复逻辑 → 修复完成后 Flink 从指定时间戳恢复增量消费。修数时间从小时级（ODS 重放全链路）缩短至分钟级，修数资源消耗降低约 80%。

**项目成果：**
- 去重/乱序相关任务 Checkpoint 大小下降 75%，Flink 去重算子 State 降为零
- Kafka Topic 数量减少 40%，中间态数据从不可查变为支持 SQL 点查和时间旅行
- 聚合类任务 Checkpoint 大小下降 90%，任务恢复速度提升 70%
- 修数时间从小时级缩短至分钟级，修数资源消耗降低 80%
- 维表管理从 Redis/Cellar/MySQL 三套收口至 Paimon 统一存储，外部组件依赖减少 60%
- 整体链路组件数从 7 类（Flink/Kafka/Doris/Redis/Cellar/MySQL/HBase）收敛至 4 类（Flink/Paimon/Doris/Spark），运维复杂度大幅降低

---

## 二、技术栈清单

| 技术 | 使用场景 |
|------|---------|
| Apache Paimon | 流式数据湖格式，主键表去重/乱序过滤（deduplicate + sequence.field）、Changelog 流读替代 Kafka 中间层、Aggregate Table 聚合下沉、Lookup Join 维表关联、批处理修数 |
| Flink / Flink SQL | 实时 ETL 主引擎，数据写入 Paimon 表、消费 Changelog 流、Lookup Join 维表关联 |
| Spark SQL | Paimon 离线批处理修数引擎，直接对主键表执行修复逻辑（UPDATE/DELETE） |
| Kafka / Mafka | ODS 层数据采集（Binlog 接入），部分未迁移链路的层间流转 |
| Doris | OLAP 分析引擎，ADS 层查询服务，Unique/Aggregate 模型 |
| Blade | 大状态场景下的外部状态存储（多流 Join 场景，未迁移至 Paimon 的部分） |
| HDFS/对象存储 | Paimon 表底层存储，列式存储（ORC 格式）+ LSM Tree 索引 |

---

## 三、面试深挖问答

### 模块一：Paimon 选型与架构决策

**Q: 为什么选 Paimon 而不是 Iceberg 或 Hudi？**

三者都是数据湖格式，但定位不同。Iceberg 更偏离线分析场景，流式更新能力较弱，不支持主键表的高效 Upsert；Hudi 支持主键更新但 Changelog 生成能力不如 Paimon 原生，且 Hudi 的 MOR 模式在高频更新场景下 Compaction 压力大。Paimon 的核心优势有三点：一是基于 LSM Tree 的高效流式更新，主键表天然支持去重、部分更新、聚合等多种 merge-engine；二是原生的 Changelog 生成能力，下游可以直接流读变更数据，不需要额外的 CDC 组件；三是和 Flink 深度集成（Paimon 本身就是从 Flink Table Store 演进而来），Flink SQL 原生支持，开发体验最好。在我们的场景里，高频流式更新 + Changelog 流读是刚需，所以选了 Paimon。

**Q: Paimon 的 LSM Tree 结构具体是怎么工作的？**

Paimon 的主键表底层用 LSM Tree（Log-Structured Merge Tree）组织数据。写入时数据先进入内存的 MemTable（类似 Write Buffer），积累到一定大小后 Flush 为磁盘上的有序文件（Sorted Run），这就是 L0 层。随着 L0 文件增多，后台 Compaction 线程会将多个 Sorted Run 合并为更大的有序文件并推到更高层级，合并过程中按主键去重并执行 merge-engine 定义的合并逻辑（deduplicate/aggregation/partial-update 等）。读取时需要多层合并读（类似归并排序），但配合 Deletion Vectors（标记已被覆盖的记录）可以跳过无效数据，加速读取。这个结构的好处是写入极快（顺序追加）、天然支持高频更新，代价是读取需要多层合并，所以 Paimon 更适合写多读少的实时数仓场景。

**Q: 引入 Paimon 后整体架构是什么样的？和原架构的关系是什么？**

不是全部推翻重建，而是渐进式迁移。原架构的 Flink+Kafka+Doris 体系继续承担核心计算和服务职能，Paimon 替换的是 Kafka 的"中间层流转"角色和 Redis/Cellar 的"维表存储"角色。具体来说：ODS 层的 Binlog 接入仍然走 Kafka（因为 Binlog 采集组件原生对接 Kafka，替换成本高且收益小），但 DWD/DWS 层间的数据流转从 Kafka Topic 迁移到 Paimon 表的 Changelog 流读；去重和乱序处理从 Flink State + Redis 迁移到 Paimon 主键表；天粒度聚合从 Flink Window 迁移到 Paimon Aggregate Table；维表从 Redis/Cellar/MySQL 收口到 Paimon；ADS 层查询服务仍然用 Doris。整体是一个"Flink 做计算编排 + Paimon 做存储和状态管理 + Doris 做查询服务"的三层架构。

### 模块二：去重与乱序（场景一深挖）

**Q: Paimon 主键表去重和原来 ROW_NUMBER 去重的本质区别是什么？**

ROW_NUMBER 去重是在计算层做的——Flink 需要维护一个按主键分区、按时间排序的窗口状态，每条新数据进来都要和状态里的历史数据比较排序，这个状态会随数据量线性增长。Paimon 主键表去重是在存储层做的——数据写入 LSM Tree 时，相同主键的多条记录在 Compaction 阶段自动合并，只保留最新一条（deduplicate merge-engine）。Flink 端完全不需要维护去重状态，只负责把数据写入 Paimon 表，去重逻辑由存储引擎保证。本质上是把"有状态计算"变成了"无状态写入 + 有状态存储"。

**Q: sequence.field 是怎么解决乱序的？原来用 Doris Sequence 列，为什么还要换？**

sequence.field 的原理很简单：Paimon 在合并相同主键的多条记录时，不是简单地取最后写入的那条，而是比较 sequence.field 指定的字段值（比如 event_time），只保留该字段值最大的记录。如果新写入的数据 event_time 比已有记录小，直接丢弃。这和 Doris Sequence 列的原理是类似的，但区别在于作用位置不同：Doris Sequence 列解决的是"数据已经到了 Doris 之后的乱序"，而 Paimon sequence.field 解决的是"数据在中间层流转过程中的乱序"。原来的链路是 Flink → Kafka → Flink → Kafka → Doris，中间层的乱序靠 Redis 缓存时间戳判断，到了 Doris 再靠 Sequence 列兜底。现在中间层换成 Paimon 后，每一层写入时就已经保证了顺序正确性，到 Doris 时数据已经是有序的，Sequence 列变成了最后一道防线而不是主要依赖。整条链路的乱序防御从"末端兜底"变成了"逐层保证"。

### 模块三：流读与修数（场景二、四深挖）

**Q: Paimon 流读和 Kafka 消费在机制上有什么区别？**

Kafka 是纯日志流——消费者通过 Offset 顺序读取消息，读过就没了（除非重置 Offset 回溯），数据本身不支持查询。Paimon 流读是基于快照的变更流——每次 Checkpoint 生成一个新快照（Snapshot），流读消费者读取的是两个快照之间的差异（Changelog），本质上是"增量快照读"。这意味着：一是数据可查，任意时间点的快照都可以用 SQL 直接查询（Time Travel）；二是数据可回溯，不需要重置 Offset，直接指定快照 ID 或时间戳就能读取历史数据；三是天然去重，因为 Changelog 是基于合并后的结果生成的，下游不需要再做去重。代价是时效性受 Checkpoint 间隔影响，通常是秒级到分钟级，不如 Kafka 的毫秒级，但在我们的供应链场景里秒级完全够用。

**Q: 修数流程具体是怎么操作的？停任务不会影响实时性吗？**

修数分四步：第一步，停掉需要修复的 DWD 层及后续的 Flink 实时任务；第二步，用 Spark SQL 读取 Paimon 表，执行修复逻辑（比如 UPDATE SET 某个字段、DELETE 错误数据等），修复结果直接写回 Paimon 主键表，利用主键合并覆盖错误数据；第三步，修复完成后，重启 Flink 任务并指定从某个时间戳开始消费增量数据（Paimon 支持按时间戳定位快照）；第四步，验证修复结果。关于实时性影响，确实会有短暂中断，但在供应链场景里，修数通常安排在业务低峰期（凌晨），而且 Paimon 修数只需要几分钟（直接改目标表），比原来的 ODS 重放（需要重跑整条链路，数小时）快得多，对业务的影响从"半天不可用"变成"几分钟不可用"。

### 模块四：聚合与维表（场景三、四深挖）

**Q: Aggregate Table 和 Flink Cumulate Window 在效果上有什么区别？**

效果上最终结果是一样的，都是按主键做天粒度聚合。区别在于状态管理方式和容错机制。Cumulate Window 的状态在 Flink 内存里，窗口越大状态越大（一天的窗口意味着一天的数据都在内存里），Checkpoint 要把这些状态全部持久化，所以 Checkpoint 大且慢。Aggregate Table 的状态在 Paimon 的 LSM Tree 里，每条新数据写入时和已有记录合并（比如 SUM 就是把新值加到已有值上），Flink 端是无状态的，Checkpoint 几乎为空。另一个关键区别是乱序处理：Cumulate Window 依赖 Watermark 判断数据是否迟到，迟到数据要么丢弃要么进侧输出流二次处理；Aggregate Table 不依赖 Watermark，任何时候来的数据都会正确合并到结果中，天然支持乱序。

**Q: Paimon Lookup Join 和原来用 Redis 做维表关联有什么区别？性能怎么样？**

Redis 做维表关联是"外部点查"模式——每条事实数据来了，Flink 算子发一次 Redis 请求查维度数据，网络 I/O 是性能瓶颈，高并发下 Redis 容易成为热点。Paimon Lookup Join 是"本地缓存"模式——Flink 算子启动时将 Paimon 维表的数据加载到本地（RocksDB 或内存），关联时直接本地查找，没有网络开销。我们用的是 Local PrimaryKey Partial Lookup 模式，不需要全量缓存，只在数据触发时按需加载对应 Bucket 的文件，基于 LSM Tree 的有序性做定位查找。性能上，Lookup 延迟从 Redis 的毫秒级（含网络 RT）降到本地微秒级，但代价是本地磁盘占用增加，且维表更新有一定延迟（依赖 Paimon 快照刷新周期）。在我们的场景里维表更新频率不高（设备信息、仓库信息等基本是天级更新），所以这个延迟完全可以接受。

**Q: Paimon 有什么局限性？哪些场景不适合用？**

三个主要局限。第一是时效性上限：Paimon 的流读时效性受 Checkpoint 间隔制约，通常是 1-10 秒级，做不到 Kafka 的毫秒级，所以对延迟要求极高的场景（比如风控实时拦截）不适合。第二是高频更新场景下的 Compaction 压力：如果单个 Bucket 的写入 QPS 非常高（比如上万 TPS），LSM Tree 的 Compaction 可能跟不上写入速度，导致读放大（Read Amplification）严重，查询变慢。我们通过合理设置 Bucket 数量（按主键哈希分散写入压力）和调优 Compaction 策略来缓解。第三是生态成熟度：Paimon 相比 Hive/Iceberg 生态还年轻，部分 BI 工具和查询引擎的支持不够完善，我们目前 ADS 层查询服务仍然用 Doris，没有直接在 Paimon 上做 OLAP 查询。

---

## 四、和原有简历的衔接话术

面试时建议这样衔接两个阶段：

> 「我们最初基于 Flink+Kafka+Doris 搭建了供应链实时数仓，这套架构解决了从 0 到 1 的问题——实时数据能力从无到有，核心链路稳定性做到了半年零 P0。但随着业务规模从 3 个子系统扩展到 13 个子系统，控制塔从'看数据'升级到'用数据指挥一线作战'，原架构在三个地方遇到了瓶颈：中间态数据不可查导致排查修数成本极高，去重乱序依赖过多外部组件导致链路脆弱，维表管理分散导致一致性难保障。所以我们引入了 Paimon 做架构升级，核心思路是把'状态管理'从计算层下沉到存储层——去重、乱序过滤、聚合这些原来 Flink 用 State 做的事情，现在由 Paimon 的主键表在写入时自动完成，Flink 变成了无状态的计算编排层，稳定性和资源效率都有了质的提升。」

这段话的好处是：讲清了"为什么要升级"（业务驱动而非技术炫技）、"升级了什么"（状态管理下沉）、"效果怎么样"（稳定性和效率提升），同时暗示了你对架构演进有全局视角，不是只会用新技术。

---

## 五、注意事项

第一，Paimon 部分的数字需要和原简历的数字逻辑一致。原简历说"Checkpoint 平均制作时间 < 1 分钟"，Paimon 升级后说"聚合类任务 Checkpoint 下降 90%"，面试官可能会追问"下降 90% 后是多少"，你需要能回答（比如"从原来的 800MB 降到 80MB 左右，制作时间从 40 秒降到 5 秒以内"）。

第二，要想清楚 Paimon 和原方案的边界。面试官一定会问"那原来的 Doris Sequence 列还用吗？Kafka 还用吗？Redis 还用吗？"——答案是都还在，但角色变了：Kafka 从"层间流转主力"退化为"ODS 层接入通道"，Doris Sequence 列从"主要乱序防线"变为"最后兜底"，Redis 从"维表存储 + 乱序状态缓存"收缩为"极低延迟场景的缓存"。不要给面试官"全部替换"的印象，渐进式迁移更真实。

第三，准备一个 Paimon 踩坑的故事。面试官对"一切顺利"的叙述不会太信任。可以准备一个真实的技术挑战，比如："刚上线时 Compaction 策略没调好，某个热点 Bucket 的写入速度超过了 Compaction 速度，导致 L0 文件堆积、流读延迟从秒级退化到分钟级。后来通过增加 Bucket 数量分散热点 + 调大 sort-spill-threshold + 开启 async-file-write 解决了。"这种细节能体现你真正落地过，而不是纸上谈兵。

第四，业务场景要具体化。不要只说"供应链实时指标"，要说清楚是什么指标：SCCT 控制塔的"设备在线率实时监控"（充电宝/骑行设备是否在线）、"仓库收发货进度看板"（WMS 系统的实时出入库）、"维修工单流转时效"（MMS 系统的工单从创建到完成的实时追踪）、"物流配送履约率"（TMS 系统的运输节点实时状态）。这些和你的旭日 13 个子系统直接对应，面试官追问时能一一展开。
