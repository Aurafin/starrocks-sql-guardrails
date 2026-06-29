---
name: starrocks-sql-guardrails
description: 当编写、审查、迁移或调优 StarRocks SQL、DDL、INSERT、CTAS、物化视图、外表查询或 AI 生成 SQL 时使用。用通用且可验证的 guardrails 规避扫描数据量过大、同表重复扫描、缺少分区谓词、分区裁剪失败、DELETE 后分区列 min/max 退化、改列注释误触发 Schema Change、全局排序、错误 JOIN/NESTLOOP/CROSS JOIN、NOT IN NULL、隐式类型转换、SELECT *、外表 planning 膨胀、高 fanout 写入和 MV rewrite 限制等常见坑。
---

# StarRocks SQL Guardrails

## 目标

这个 skill 用来让任何 AI 在写 StarRocks SQL 前先做一次“可执行计划验证”的预审，尽量避免简单但代价很高的 SQL 坑。

原则：

- 不凭经验断言 SQL 安全，先看表结构、分区/分桶/排序键和 `EXPLAIN`。
- 全局先控扫描预算：扫描数据量和同一大表扫描次数越高，SQL 越依赖集群即时资源，稳定性越差。
- 不把某个版本的已修 bug 写成永久规则。
- 不默认给生产 SQL 加 hint 或 session 变量；hint 只作为诊断或临时止血，并且要解释副作用。
- 不暴露客户名、群聊链接、用户 ID、query id 或完整私有 SQL。

## 使用流程

1. 先识别上下文：
   - SQL 类型：`SELECT`、`INSERT`、`CTAS`、MV、外表查询、导出、BI 自动生成 SQL。
   - 版本、部署形态、catalog 类型、表模型、分区键、分桶键、排序键、数据规模、事实表/维表边界。
   - 业务是否真的需要全量扫描、全局排序、所有列、跨很多分区写入。

2. 按风险优先级审查 SQL：
   - 先数 scan 节点和同一大表扫描次数，再看扫描量、分区裁剪和过滤下推。
   - 再看 JOIN 形态、类型一致性、排序/窗口、投影宽度。
   - 外表、MV、CTAS/INSERT 用对应的附加规则。
   - 具体规则见 `references/sql-pitfalls.md`。

3. 对非小表 SQL 必须要求计划验证：
   - `EXPLAIN` / `EXPLAIN COSTS` / `EXPLAIN VERBOSE`：按表统计 scan 节点数量，看 `partitions`/`partitionRatio`、`tabletRatio`/`tabletsRatio`、`cardinality`/`actualRows`、`avgRowSize`、`PREDICATES`、输出列、JOIN 类型、`EXCHANGE`、`SORT`、`TOP-N`、`ANALYTIC`。
   - 已执行 SQL：看 Profile 的 `RowsRead`、`RawRowsRead`、`BytesRead`、`CompressedBytesRead`、spill、peak memory、skew、慢 operator。
   - 如果不能运行计划，明确写“还缺 EXPLAIN/Profile/DDL，不能证明安全”。

4. 输出必须可落地：
   - 先给结论和主要风险。
   - 给出更安全 SQL 形态或改写方向。
   - 给出最小验证命令或计划检查项。
   - 证据不足时只问最小必要信息。

## 推荐输出格式

```text
结论：
- ...

风险：
- ...

建议：
- ...

验证：
- ...

还缺：
- ...
```

小问题可以省略不相关段落。不要为了套格式拉长回答。

## Reference Routing

- 写或审查任意 StarRocks SQL：先读 `references/sql-pitfalls.md`。
- 需要解释为什么这么判断，或需要区分版本 bug 与通用规则：读 `references/evidence-map.md`。

## 隐私与边界

本 skill 来自 StarRocks 公开代码/文档证据和脱敏支持模式总结。引用支持案例时只使用抽象模式，不包含客户、人员、链接、message id、query id 或私有 SQL。
