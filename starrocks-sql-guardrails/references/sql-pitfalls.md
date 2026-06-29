# StarRocks SQL 常见坑检查表

这份检查表用于 AI 生成或审查 StarRocks SQL。每条规则都应通过表结构、`EXPLAIN` 或 Profile 验证，不要只凭关键词判断。

## 1. 扫描预算是全局第一优先级

高风险写法：

```sql
SELECT ...
FROM big_fact
WHERE user_id = 123;

WITH a AS (SELECT ... FROM big_fact WHERE dt >= '2026-06-01'),
     b AS (SELECT ... FROM big_fact WHERE dt >= '2026-06-01'),
     c AS (SELECT ... FROM big_fact WHERE dt >= '2026-06-01')
SELECT ...
FROM a JOIN b ... JOIN c ...;
```

规则：

- 审查任何非小表 SQL 时，先数 scan 节点，再估扫描数据量；后续 JOIN、SORT、AGG、WINDOW 都建立在扫描输入之上。
- 大事实表、事件表、外表、宽表默认要求分区谓词，除非用户明确要全量扫描。
- 没有过滤条件或没有分区谓词时，必须先判断表数据量、分区数、tablet 数和业务是否允许全表扫描；不能直接生成“看起来正确”的事实表全扫 SQL。
- 同一个大表在一个 SQL 中被多次扫描时，要默认高风险。CTE、子查询、`UNION ALL` 分支不应被假定会自动物化或复用，必须看 `EXPLAIN` 是否仍有多个 scan 节点。
- 如果多个分支对同一事实表做相近过滤，优先考虑合并成一次扫描后用条件聚合、`CASE WHEN`、预聚合、半连接或临时/物化中间结果表达；不能合并时要说明每次扫描的必要性。
- 先减少扫描输入，再做 JOIN、排序、窗口和高基数聚合。扫描数据越多，查询越依赖当前集群 CPU、IO、内存和并发状态，稳定性越差。
- 时间分区表优先写半开区间：

```sql
WHERE dt >= '2026-06-01'
  AND dt <  '2026-07-01'
```

- 没有分区谓词时，先追问或补充业务时间、租户、状态等范围；如果确实全扫，只提示风险和运行策略，不要假装普通 SQL 改写可以消除全扫。
- `LIMIT` 只能证明最终返回行数少，不能单独证明扫描量少；必须看 scan 节点是否真的有 limit/predicate 下推或只选中少量分区/tablet。

验证：

- `EXPLAIN` 中先按表统计 scan 节点数量；同一大表出现多个 scan 节点时，要列为主要风险。
- 分区表应看到 `partitions=x/y` 或 `partitionRatio`，且 `x` 明显小于 `y`；tablet 也应看 `tabletRatio`/`tabletsRatio`。
- 看 scan 节点的 `cardinality`/`actualRows`、`avgRowSize`、输出列，估算扫描行数和扫描字节量。
- scan 节点的 `PREDICATES` 应包含分区谓词或等价下推谓词。
- 已执行 SQL 用 Profile 汇总各 scan 节点的 `RowsRead`、`RawRowsRead`、`BytesRead`、`CompressedBytesRead`；如果读入远大于最终输出，优先改扫描范围或重复扫描结构。

## 2. 分区谓词要写成容易裁剪的形态

高风险写法：

```sql
WHERE date(event_time) = '2026-06-01'
WHERE substr(dt, 1, 10) = '2026-06-01'
WHERE event_time >= (SELECT convert_tz(...))
WHERE date_format(event_time, '%Y-%m-%d') >= '2026-06-01'
WHERE (dt >= '2026-06-01' AND dt < '2026-06-02') OR status = 'new'
```

规则：

- 能用分区列直接表达，就不要把函数包在分区列外面。
- 能预计算常量，就把常量算好后用字面量或显式类型比较。
- 避免把分区过滤和非分区过滤用宽泛 `OR` 混在一起；语义允许时拆成 `UNION ALL` 分支。
- 不要写死“某函数一定能裁剪/一定不能裁剪”。StarRocks 对函数裁剪的支持随版本、表达式、分区类型变化。

更稳妥写法：

```sql
WHERE event_time >= '2026-06-01 00:00:00'
  AND event_time <  '2026-06-02 00:00:00'
```

验证：

- 对比改写前后的 `partitions`/`partitionRatio` 和 `tabletsRatio`。
- 如果仍然扫全部分区，不要宣称已经裁剪。

## 3. 不要假设 `min/max(分区列)` 总能只扫边界分区

高风险写法：

```sql
SELECT max(dt) FROM fact_detail;
SELECT min(dt), max(dt) FROM fact_detail;
```

规则：

- 对 duplicate/detail 分区表，简单 `min(partition_col)` / `max(partition_col)` 在表未发生 DELETE 时，可能被优化成只扫最小/最大非空分区，或在 list 分区上直接由分区值求结果。
- 表做过 DELETE 后，StarRocks 会把表级 `hasDelete` 标记置为 true；这时优化器不会再用“只扫边界分区”的快速路径。原因是逻辑删除行在 BE 读取时通过 delete predicate 过滤，分区边界或 segment 元数据不等于删除后的真实可见行。
- 单独 `min(dt)` 或 `max(dt)` 在有 DELETE 后仍可能被改写成 `TOP-N`，所以不要简单写成“一定普通全表聚合”。更准确的判断是：不会只靠最大/最小分区裁剪，实际扫描范围必须看 `EXPLAIN`。
- 同时求 `min(dt), max(dt)` 时，`TOP-N` 路径也不适用；做过 DELETE 的 duplicate/detail 表更容易退回扫描全部已选分区。
- 显式 `[_META_]` 是特殊元数据查询，不要和普通 SQL 的准确聚合混用；有 DELETE 的表不要用元数据 min/max 承诺准确结果。

建议：

- 如果业务需要“最新有数据日期”，优先在写入链路维护水位表/汇总表，或在查询里写明确的业务时间下界。
- 清理整段历史数据优先 `TRUNCATE TABLE ... PARTITION(...)` / `DROP PARTITION`，避免用大批量 `DELETE` 清分区数据；`TRUNCATE` 会直接删除数据且不可恢复，执行前必须确认业务可接受。

验证：

- 未 DELETE 的简单分区列 `max` 可期待 `partitions=1/N`；做过 DELETE 后如果仍要证明性能，必须看 `EXPLAIN` 是否出现 `TOP-N`、`partitions=N/N` 或其它实际扫描形态。
- 不能只凭 `dt` 是分区字段就断言只扫最新分区。

## 4. 只改列注释时不要带完整列定义

高风险写法：

```sql
ALTER TABLE t MODIFY COLUMN k1 BIGINT NOT NULL COMMENT 'new comment';
```

规则：

- 只修改列注释时，使用 comment-only 语法：

```sql
ALTER TABLE t MODIFY COLUMN k1 COMMENT 'new comment';
```

- 不要为了“保持完整定义”把类型、是否允许 NULL、聚合类型、默认值、列位置等一起写上；这会被解析为普通列定义修改，而不是单纯改注释，可能进入 Schema Change 路径。
- comment-only 语法只更新元数据里的列注释，并写 edit log；不要和 `ADD COLUMN`、`DROP COLUMN`、`ORDER BY(...)` 等其它 alter 子句合并在同一条语句里。
- 这条规则适用于主键列、键列和普通列；前提仍然是目标列存在，且 comment 不是 `NULL`。

验证：

- 语句形态应是 `MODIFY COLUMN <column_name> COMMENT '<new_comment>'`，而不是 `MODIFY COLUMN <full_column_definition>`。
- 如果误写了完整定义，按普通 schema change 风险处理：看是否创建 schema change job、表状态、MV 失效风险和执行窗口。

## 5. 外表查询不能只靠 LIMIT

适用：Hive、Hudi、Iceberg、Paimon、JDBC 等 external catalog。

规则：

- 外表 SQL 更要优先写分区谓词和可下推谓词。
- `LIMIT 10` 只能限制最终返回行数，不能保证跳过 FE planning 阶段的 partition/file/split 元数据读取。
- `scan_hive_partition_num_limit` 这类变量是 fail-fast 保护，不是优化器加速手段。

验证：

- 看外表 scan 的 partition/file/split 数、`PREDICATES`/`PushdownPredicates`。
- 如果 planning 慢或 FE 内存高，优先确认选中分区数和外表元数据规模。
- 如果错误是缺文件、读旧文件、snapshot/manifest 不一致，这是外表元数据诊断问题；只询问 catalog 类型、分区、refresh history，不要把它写成 SQL guardrail。

## 6. JOIN 谓词必须完整、等值、alias 正确

高风险写法：

```sql
FROM fact a
LEFT JOIN fact b
  ON a.id = b.id AND b.dt BETWEEN a.start_dt AND a.end_dt
LEFT JOIN fact c
  ON a.id = c.id AND b.dt BETWEEN a.start_dt AND a.end_dt
```

第二个 join 又过滤了 `b.dt`，`c` 这份表没有自己的时间范围过滤。

规则：

- 重复 self-join 时逐个检查 alias，不能复制上一段 `ON` 后漏改别名。
- 大表 join 尽量有可提取的等值谓词，且 join key 两侧类型一致。
- 复合 join key 要尽量在同一个表别名上表达完整关系，不要让优化器靠等价传递猜。
- 避免逗号 join、隐式 join 和把 join 条件放散到复杂 `WHERE`。
- 纯非等值 join、缺失 `ON`、函数包裹 join key 都可能退化为 `NESTLOOP JOIN` 或 `CROSS JOIN`。

验证：

- `EXPLAIN` 不应出现非预期 `NESTLOOP JOIN` / `CROSS JOIN`。
- 每个大表 scan 都应有预期谓词；join 节点行数估算不能爆炸。

## 7. 不要把 hint 当默认解法

规则：

- `disable_join_reorder`、`join_implementation_mode=hash`、`disable_colocate_join`、join hint 等只适合诊断或临时止血。
- AI 不应自动给生产 SQL 加 hint 或 session 变量，除非用户明确要求，且已有 `EXPLAIN`/Profile 证据说明收益和副作用。
- 加 hint 前先确认 SQL 本身是否缺等值谓词、alias 错误、类型不一致、统计信息缺失。
- `join_implementation_mode=hash` 可能让无法提取等值谓词的 SQL 直接无法出计划，不能无脑加。

验证：

- 先用 hint 对比 good/bad plan，确认症状来自计划选择。
- 生产使用 hint 前说明副作用和回滚方式。

## 8. 全局排序和宽窗口要先减数据

高风险写法：

```sql
SELECT ...
FROM big_result
ORDER BY metric;

SELECT count(*)
FROM (
  SELECT ...
  FROM t
  ORDER BY ts
) q;
```

规则：

- 只有业务语义需要稳定全局有序时才保留顶层 `ORDER BY`。
- 只需要 Top-K 时必须加 `LIMIT`，让计划尽量走 `TOP-N`。
- 子查询里的 `ORDER BY` 如果外层不依赖顺序，通常应该去掉；它还可能影响 MV rewrite。
- 窗口函数要避免超宽 `PARTITION BY` 或无界 `ORDER BY`；能先聚合/过滤/物化就先缩小输入。

验证：

- `EXPLAIN` 看 `SORT`、`TOP-N`、`ANALYTIC`、`MERGING-EXCHANGE`。
- Profile 中 sort/window 的 `SpillBytes`、`MaxBufferedBytes`、`PeakBufferedRows` 异常时必须改写或限流。

## 9. 避免 `SELECT *` 和宽列穿透重算子

规则：

- 大表、外表、JSON/string/array 宽列表、join、sort、window、导出场景不要默认 `SELECT *`。
- 在 join/window/sort 前尽早投影，只保留后续需要的列。
- 不要把大 JSON、长字符串、数组列带进 `GROUP BY`、`ORDER BY` 或窗口阶段，除非业务明确需要。

验证：

- `EXPLAIN VERBOSE` 或逻辑计划中，各 scan/project 输出列应与实际需要一致。

## 10. 类型要显式，尤其是数字字符串、日期和 join key

高风险写法：

```sql
WHERE amount >= '190000' AND amount <= '190000'
WHERE varchar_amount > numeric_amount
WHERE IFNULL(numeric_col, '') = ''
```

规则：

- 数字按数字比较，日期按日期比较，不要依赖隐式 cast 或字符串字典序。
- 配置表、策略表、外部表里常见的 varchar 数字要显式 `CAST` 到业务类型。
- join key 两侧类型、精度、时区语义要对齐。
- 日期字面量尽量使用目标列能稳定解析的格式，例如 `DATE '2026-06-01'` 或 `'2026-06-01 00:00:00'`。

更稳妥写法：

```sql
WHERE amount = CAST('190000' AS DECIMAL(22, 2))
```

验证：

- `EXPLAIN` 谓词里不要出现意外 cast、字符串比较或统计信息缺失。

## 11. 高基数聚合要先过滤再聚合

高风险形态：

- 先对全量大表按高基数字段聚合，再 join 小表过滤。
- `COUNT(DISTINCT ...)`、`GROUPING SETS`、`group_concat`、大 bitmap 聚合在过滤前执行。

规则：

- 能先 join/where 缩小数据，就不要先对全量数据建巨大聚合 hash table。
- 多对多 join 后再 `DISTINCT` / `COUNT(DISTINCT)` 很容易放大中间结果；语义允许时，先对 join 输入去重/预聚合，或用 `SEMI JOIN` 表达“只要求存在匹配”。
- `COUNT(DISTINCT ...)` 的实现和聚合阶段会受统计信息、基数和 hint/session 变量影响；不要把 `multi_count_distinct`、`new_planner_agg_stage` 写成固定模板，先看计划和数据分布。
- 高频高基数聚合可以提示后续建模方向，例如排序键、分桶键、预聚合表或 MV；这不是普通 `SELECT` 能直接修复的问题。
- 只需要明细去重时，先确认是否可以按业务主键、时间分区拆批。

验证：

- `EXPLAIN` 看 `AGGREGATE` 的位置是否在强过滤之前。
- Profile 看聚合节点的 peak memory、hash table、spill。

## 12. CTAS / INSERT / LOAD / DELETE 避免高 fanout

规则：

- 一条写入不要无意横跨大量分区和 tablet，尤其是多年日分区、多租户复合分区、高 bucket 数表。
- CTAS/INSERT 目标表的分区、分桶、排序键要和查询过滤/写入批次匹配。
- 大范围补数优先按分区拆批，减少一次事务涉及的 tablet 数。
- 大批量清理数据不要默认写 `DELETE FROM table WHERE dt < ...`；如果语义是整分区清理，优先 `TRUNCATE TABLE ... PARTITION(...)` 或 `DROP PARTITION`，减少 delete predicate、版本和 compaction 压力。`TRUNCATE` 不可恢复，不能作为无损替代。
- 明细/duplicate 表发生 DELETE 后还会影响部分元数据/分区列 min-max 优化，见第 3 条。

验证：

- 估算命中分区数、每分区 bucket 数、总 tablet fanout。
- 失败或慢在 commit/publish/version 阶段时，不要只盯执行算子。
- DELETE 后关注 delete predicate、版本数和 compaction 压力；不要只看 SQL 返回耗时。

## 13. `NOT IN` 遇到 NULL 是语义坑

高风险写法：

```sql
WHERE customer_id NOT IN (SELECT customer_id FROM blocklist);
```

规则：

- 如果子查询结果里可能有 `NULL`，`NOT IN` 可能让外层结果被全部过滤或出现看似“少数据”的结果；这是 SQL 三值逻辑语义，不是 StarRocks 性能问题。
- 多列 `(a, b) NOT IN (...)` 也要逐列确认 NULL 语义。
- AI 改写成 `LEFT JOIN ... IS NULL` 前，必须确认 NULL 语义与业务一致。

更稳妥写法：

```sql
WHERE customer_id NOT IN (
  SELECT customer_id
  FROM blocklist
  WHERE customer_id IS NOT NULL
)
```

验证：

- 检查子查询输出列是否可能为 NULL。
- `EXPLAIN` 中 `NULL AWARE LEFT ANTI JOIN` 通常说明优化器在保留 `NOT IN` 的 NULL-aware 语义。

## 14. MV rewrite SQL 避开明确限制

规则：

- 如果目标是透明改写，MV 定义里不要默认放 `ORDER BY`、`LIMIT`、`UNION`、`GROUPING SETS`、窗口函数等文档限制形态。
- 不要把查询需要过滤或排序的列藏在不可见子查询里；需要 rewrite 的列应在 MV 输出中可见。
- 外表 MV 不保证查询结果强一致，不能用来承诺强一致语义。

验证：

- 用 `EXPLAIN` 确认是否命中 MV rewrite，而不是只确认 MV 创建成功。
- MV refresh 慢时看刷新分区数、base table 扫描分区数、是否启用 spill，而不是只调大超时。

## 15. 不把已修 bug 写成通用 SQL 禁忌

规则：

- 如果问题只在某个版本触发，并且已有明确 PR/backport 修复，不要把它写成“所有 StarRocks 都不能这么写”。
- 对 window ranking / PartitionTopN / MV self-union / 外表元数据等历史问题，先查目标版本和修复 PR。
- 通用 skill 只保留仍然有价值的写法 guardrail，例如“宽窗口要看资源”“复杂表达式要看裁剪”“MV rewrite 要看文档限制”。

验证：

- 给出目标版本、`EXPLAIN`/Profile、PR 或源码证据，再下根因结论。
