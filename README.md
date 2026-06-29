# StarRocks SQL Guardrails Skill

面向 AI / Codex / MCP / Agent 的 StarRocks SQL 预审 skill。

它的目标不是替代 `EXPLAIN`、Profile 或人工 review，而是让 AI 在生成、审查、迁移或调优 StarRocks SQL/DDL 时，先主动避开一批高频且代价很高的坑：扫描数据量过大、同表重复扫描、缺少分区谓词、分区裁剪失效、DELETE 后分区列 `min/max` 退化、改列注释误触发 Schema Change、全局排序、错误 JOIN、`NOT IN` + `NULL`、隐式类型转换、`SELECT *`、外表 planning 膨胀、高 fanout 写入和 MV rewrite 限制等。

## 适用场景

- 让 AI 写 StarRocks 查询、DDL、INSERT、CTAS、物化视图或外表 SQL。
- Review BI / 报表 / 任务平台自动生成的 SQL。
- 给 MCP、Agent、Copilot 类工具增加 StarRocks SQL guardrails。
- 把 SQL 性能事故里的通用经验沉淀成可复用检查清单。

## 安装

### Codex

```bash
git clone https://github.com/Aurafin/starrocks-sql-guardrails.git
cd starrocks-sql-guardrails
mkdir -p ~/.codex/skills
ln -s "$(pwd)/starrocks-sql-guardrails" ~/.codex/skills/starrocks-sql-guardrails
```

也可以直接复制内层目录：

```bash
cp -R starrocks-sql-guardrails ~/.codex/skills/starrocks-sql-guardrails
```

### 其他 AI / MCP / Agent

只要系统支持加载 Markdown instructions，就可以把以下内容作为一组规则注入：

- `starrocks-sql-guardrails/SKILL.md`
- `starrocks-sql-guardrails/references/sql-pitfalls.md`
- `starrocks-sql-guardrails/references/evidence-map.md`

建议让 agent 遵守 `SKILL.md` 的 Reference Routing：普通 SQL review 先读 `sql-pitfalls.md`；需要解释版本、代码证据或边界时再读 `evidence-map.md`。

## 使用方式

安装后可以直接让 AI：

```text
用 starrocks-sql-guardrails review 这条 SQL，指出性能风险、语义风险和需要补充的 EXPLAIN/Profile 信息。
```

输出通常应包含：

- 结论：SQL 是否存在明显风险。
- 风险：分区、JOIN、排序、投影、写入、MV、外表等问题。
- 建议：更安全的 SQL 形态或改写方向。
- 验证：最小必要的 `EXPLAIN` / Profile / DDL 检查项。
- 还缺：不能证明结论时需要的最小信息。

## 校验

如果你也使用 Codex 的 skill-creator，可以在仓库根目录运行：

```bash
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py starrocks-sql-guardrails
```

## License

Apache-2.0
