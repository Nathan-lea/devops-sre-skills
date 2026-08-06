# tidb-br-backup

使用 TiDB BR 工具执行全量备份、完整性校验、灾难恢复与过期清理。

## 用途

- 定时全量备份到 S3/MinIO
- 备份完整性校验（br validate）
- 灾难恢复（从 S3 恢复到新集群）
- 过期备份自动清理

## safety 契约

`destructive-if-restore` - 备份为非破坏性；恢复操作会覆盖目标集群数据。

## 使用方式

```bash
export PD_ENDPOINT=10.0.4.10:2379
export S3_BUCKET=s3://tidb-backup
export S3_ENDPOINT=http://minio:9000
# 按 SKILL.md 步骤执行备份/校验/恢复
```

## 依赖

| 工具 | 必需 | 说明 |
|------|------|------|
| br (tikv-br) | 是 | TiDB Backup & Restore 工具 |
| rclone | 是 | S3/MinIO 对象存储操作 |
| mysql client | 恢复验证用 | 行数校验 |

## License

MIT
