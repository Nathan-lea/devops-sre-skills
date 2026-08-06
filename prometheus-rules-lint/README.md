# prometheus-rules-lint

校验 Prometheus 告警规则 YAML 文件的语法、结构完整性、命名规范与 PromQL 正确性。

## 用途

- 新增/修改告警规则后提交前校验
- 排查 `rule group failed to load` 错误
- CI 流水线门禁

## safety 契约

`read-only` — 纯校验，不修改任何系统状态。

## 使用方式

```bash
export RULES_DIR=/path/to/your/rules
# 按需执行 SKILL.md 中各步骤的校验命令
```

## 依赖

| 工具 | 必需 | 说明 |
|------|------|------|
| python3 + pyyaml | 是 | YAML 解析与结构校验 |
| promtool | 否 | 有则额外执行 PromQL 语法检查 |

## License

MIT
