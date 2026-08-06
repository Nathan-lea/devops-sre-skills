# migration-verify

验证数据库迁移和文件迁移后的数据一致性，作为流量切换前的 go/no-go 门禁。

## 用途

- MySQL -> TiDB 迁移后数据校验（行数 + CRC32）
- PostgreSQL -> TiDB 迁移后数据校验（行数 + 序列）
- 数据库跨云迁移后数据校验
- Web 静态资源迁移后文件校验（rsync dry-run）
- 生成 go/no-go 报告

## safety 契约

`read-only-verify` - 纯读取校验，不修改源端或目标端数据。

## 使用方式

```bash
export SOURCE_IP=10.0.2.10
export TARGET_IP=10.1.2.10
export VERIFY_TYPE=mysql   # mysql | pg | file
# 按 SKILL.md 步骤执行校验
```

## 依赖

| 工具 | 必需 | 说明 |
|------|------|------|
| mysql client | MySQL 校验 | 行数 + CRC32 对比 |
| psql | PG 校验 | 行数 + 序列对比 |
| rsync + ssh | 文件校验 | dry-run 差异检查 |

## License

MIT
