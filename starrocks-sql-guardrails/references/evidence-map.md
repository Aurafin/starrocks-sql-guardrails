# StarRocks SQL Guardrails 证据地图

当需要解释规则来源、区分通用写法问题和版本 bug 时，优先看这些公开代码和文档位置。处理真实客户问题时，还要用目标版本/tag 的源码、`EXPLAIN` 和 Profile 验证。

## 官方文档锚点

- `docs/zh/best_practices/partitioning.md`
  - 分区裁剪目的、分区/tablet 数量风险、复合分区和 FE/Compaction 开销。
- `docs/zh/best_practices/table_clustering.md`
  - 读取路径中的分区裁剪、tablet 裁剪、短键索引、Zone Map，以及排序键选择。
- `docs/zh/best_practices/bucketing.md`
  - Hash bucketing 与 random bucketing 的差异；分桶裁剪、Colocate Join、本地聚合能力。
- `docs/zh/best_practices/query_tuning/query_planning.md`
  - 如何读 `EXPLAIN`：scan 谓词、partition/tablet 比例、JOIN、聚合/排序、数据移动、谓词下推。
- `docs/zh/best_practices/query_tuning/query_profile_tuning_recipes.md`
  - `NESTLOOP`/`CROSS` 非等值回退、Sort spilling、Wide window partitions、内存和 spill 指标。
- `docs/zh/best_practices/query_tuning/query_hint.md`
  - hint 会影响 join reorder；只在验证后使用。
- `docs/zh/using_starrocks/async_mv/use_cases/query_rewrite_with_materialized_views.md`
  - MV rewrite 限制：`ORDER BY`、`LIMIT`、`UNION`、窗口函数、外表 MV 一致性等。
- `docs/zh/using_starrocks/sorted_aggregate.md`
  - 高基数聚合可通过有序输入降低 hash 聚合内存，但有分区/分桶/部署限制。
- `docs/zh/sql-reference/sql-statements/table_bucket_part_index/DELETE.md`
  - DELETE 会把数据标记为 Deleted，Compaction 后物理回收；Compaction 完成前可能降低查询效率。
- `docs/zh/sql-reference/sql-statements/table_bucket_part_index/TRUNCATE_TABLE.md`
  - `TRUNCATE TABLE ... PARTITION(...)` 可清空指定分区，不会对查询性能造成影响，但数据不可恢复。

## 源码和测试锚点

- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/QueryOptimizer.java`
  - 谓词下推、MV rewrite、重复 partition prune、external table partition prune 前的规划顺序。
- `fe/fe-core/src/test/java/com/starrocks/sql/optimizer/rule/transformation/FurtherPartitionPruneTest.java`
  - 同时包含可裁剪和不可裁剪的函数/谓词形态，证明不能简单按函数名下结论。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/transformation/pruner/ExtractRangePredicateFromScalarApplyRule.java`
  - `convert_tz` scalar-subquery 范围谓词抽取逻辑。
- `fe/fe-core/src/test/java/com/starrocks/sql/plan/ExtractRangePredicateFromScalarApplyRuleTest.java`
  - `convert_tz` scalar-subquery 对 DATE/DATETIME/string 分区列的计划验证。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/implementation/HashJoinImplementationRule.java`
  - Hash Join implementation 要求非 cross join 且有可提取等值谓词。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/implementation/NestLoopJoinImplementationRule.java`
  - 没有 equal-conjuncts 且 Hash Join 不能处理时才选择 NestLoop Join。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rewrite/OptExternalPartitionPruner.java`
  - Hive 外表分区裁剪后检查 `scan_hive_partition_num_limit`，超限抛错。
- `fe/fe-core/src/main/java/com/starrocks/qe/SessionVariable.java`
  - `scan_hive_partition_num_limit` 默认 0，表示不限制；这是保护线，不是 SQL 优化。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/transformation/PartitionColumnMinMaxRewriteRule.java`
  - `min/max(partition_column)` 的边界分区优化；`checkPartitionPrune()` 的核心条件包括 duplicate table、无 filter/limit 且 `!table.hasDelete()`，实际还受表类型、分区列、显式 partition/tablet/replica hint 等条件约束；单独 min 或 max 可能转 `TOP-N`。
- `fe/fe-core/src/test/java/com/starrocks/sql/plan/PartitionPruneTest.java`
  - `testMinMaxPrune_PartitionPrune()` 覆盖无 DELETE 时 `max(c1)` 只扫边界分区，以及 DELETE 后 `select min(c1)` 退回 `partitions=5/5`。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/transformation/RewriteSimpleAggToMetaScanRule.java`
  - 普通 simple min/max/count 改写为 MetaScan 的条件包含 duplicate table、无 filter/group/limit 且 `!table.hasDelete()`。
- `fe/fe-core/src/main/java/com/starrocks/catalog/TableProperty.java`
  - `hasDelete` 注释说明 `false` 才能保证 BE segment 没有 delete conditions，`true` 表示可能仍有 delete conditions。
- `fe/fe-core/src/main/java/com/starrocks/load/DeleteJob.java`、`fe/fe-core/src/main/java/com/starrocks/load/DeleteMgr.java`、`fe/fe-core/src/main/java/com/starrocks/catalog/OlapTable.java`
  - 非主键表 DELETE 提交后通过 `DeleteMgr.updateTableDeleteInfo()` 设置 `OlapTable.hasDelete()`。
- `be/src/storage/tablet_reader.cpp`、`be/src/storage/rowset/rowset.cpp`、`be/src/storage/rowset/segment_iterator.cpp`
  - BE 查询读取 delete predicates，并在 segment iterator 里过滤逻辑删除行。
- `be/src/storage/olap_meta_reader.cpp`、`be/src/storage/meta_reader.cpp`
  - MetaScan 的普通 min/max 依赖 footer/segment zonemap，解释了有 DELETE 时普通 SQL 不能无条件信元数据。
- `fe/fe-core/src/main/java/com/starrocks/analysis/JoinOperator.java`、`fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/transformation/LargeInPredicateToJoinRule.java`
  - `NOT IN` 对应 `NULL AWARE LEFT ANTI JOIN`，大 IN/NOT IN 可改写成 semi/anti join，但要保留 NULL-aware 语义。
- `be/src/common/config.h`、`be/src/exec/olap_common.cpp`、`be/src/exec/olap_scan_node.cpp`
  - `max_scan_key_num` 默认 1024；过多 IN/fixed values 可能无法继续展开 short key scan key；Profile 有 `ShortKeyFilterRows`、`ZoneMapIndexFilterRows` 等指标。
- `fe/fe-core/src/main/java/com/starrocks/qe/SessionVariable.java`、`be/src/exec/hash_join_node.cpp`、`be/src/runtime/runtime_filter_worker.cpp`
  - RuntimeFilter 相关变量和 build size 限制；这些是诊断/调参依据，不应替代 SQL 谓词、JOIN key 和统计信息检查。
- `fe/fe-core/src/main/java/com/starrocks/sql/optimizer/rule/transformation/GroupByCountDistinctDataSkewEliminateRule.java`、`GroupByCountDistinctRewriteRule.java`、`RewriteMultiDistinctRule.java`
  - `COUNT(DISTINCT)` 和多阶段/去倾斜改写存在规则支持，具体阶段选择要看统计信息、基数和计划。

## 已核对为版本化/已修复的问题

这些不能写成“所有 StarRocks SQL 都不要这么写”的通用禁忌：

- `convert_tz` scalar-subquery 曾影响分区裁剪，但本地 git log 显示 `#69055` 及 backport 已增加范围谓词抽取。通用规则只能写“复杂表达式必须看 `EXPLAIN` 验证裁剪”。
- Window ranking / PartitionTopN 相关问题属于版本化问题，本地 git log 可验证 `#72848` 及 backport。通用规则只保留宽窗口、排序和资源验证，不给 `row_number/rank` 写禁忌式改写模板。
- MV self-union alias/刷新问题本地 git log 显示 `#58996` 及 backport。通用规则只保留官方 MV rewrite 限制。
- Hudi/Iceberg/Paimon 等外表 stale metadata、缺文件、snapshot 变化问题属于外表元数据诊断，不是 SQL 写法 guardrail。
- SegmentFlushTask、image 持久化、BE/CN/Manager OOM 等事故类问题不是 SQL 生成规则。

## 脱敏支持模式

这些是支持历史中反复出现的抽象模式，不是无条件事实：

- 函数包裹分区列、复杂 `OR`、不稳定日期字符串格式，可能让等价业务条件扫描更多分区。
- 子查询常量、函数常量、表达式分区的裁剪能力随版本和表达式变化，必须看目标版本计划。
- 重复 self-join 的 alias 复制错误会让某个表副本缺过滤，进而放大 join、sort、agg。
- `NESTLOOP JOIN` / `CROSS JOIN` 在大表上默认高风险，但小表或真正需要非等值语义时可能合理。
- 子查询里的 `ORDER BY` 经常没有语义价值，并可能增加代价或影响 MV rewrite。
- varchar 数字、日期字符串和 join key 类型不一致，会导致隐式 cast、统计估算差或业务比较语义错误。
- duplicate/detail 分区表发生 DELETE 后，分区列 min/max 不能再默认认为只扫最新/最老分区；必须看 `EXPLAIN`。
- 整分区清理优先用 `TRUNCATE TABLE ... PARTITION(...)` / `DROP PARTITION`；大批量 DELETE 会留下 delete predicate、增加 compaction/版本压力，还会影响部分元数据优化。`TRUNCATE` 不可恢复。
- `NOT IN` 子查询要先确认 NULL；这类问题常表现为“结果为空/少数据”，但本质是 SQL 语义。
- RuntimeFilter、short key、zonemap、count distinct 阶段选择都可以作为诊断线索，但不能替代分区谓词、类型一致、JOIN key 完整和统计信息验证。

永远用目标版本的 `EXPLAIN`、Profile、源码或 PR 证据确认根因；没有直接证据时，不要下确定性结论。
