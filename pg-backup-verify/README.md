# pg-backup-verify

验证 PostgreSQL 备份的完整性和可恢复性，支持逻辑备份与物理备份两种模式。

## 用途

- 逻辑备份（pg_dump/pg_dumpall）验证：globals 解压 + 逐库恢复 + 行数/序列对比
- 物理备份（pg_basebackup）验证：临时实例启动 + 数据库列表对比
- S3 远程备份一致性校验
- 灾备恢复演练

## safety 契约

`read-only-verify` - 创建临时数据库/实例验证后自动清理，不修改线上数据。

## 使用方式

```bash
export PG_HOST=127.0.0.1
export PG_USER=backup
export PG_PASS=<via-vault>
export BACKUP_MODE=logical   # logical | physical
export BACKUP_BASE=/srv/backup/postgresql
export S3_BUCKET=s3://db-backup/postgresql
# 按 SKILL.md 步骤执行验证
```

## 依赖

| 工具 | 必需 | 说明 |
|------|------|------|
| psql / pg_dump / pg_restore | 是 | 逻辑备份验证 |
| pg_ctl | 物理模式 | 临时实例启停 |
| rclone | 是 | S3 远程校验 |

## License

MIT
