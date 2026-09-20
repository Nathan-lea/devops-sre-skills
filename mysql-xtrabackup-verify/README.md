# mysql-xtrabackup-verify

验证 Percona XtraBackup 产生的 MySQL 备份的完整性和可恢复性。

## 用途

- 每日备份后完整性验证（xtrabackup_info 检查 + prepare + 测试恢复）
- 灾备恢复演练
- S3 远程备份一致性校验
- 行数级数据对比

## safety 契约

`read-only-verify` - 创建临时实例验证后自动清理，不修改线上数据。

## 使用方式

```bash
export BACKUP_BASE=/srv/backup/mysql
export MYSQL_HOST=127.0.0.1
export BACKUP_USER=backup
export BACKUP_PASS=<via-vault>
export S3_BUCKET=s3://db-backup/mysql
# 按 SKILL.md 步骤执行验证
```

## 依赖

| 工具 | 必需 | 说明 |
|------|------|------|
| xtrabackup | 是 | prepare + copy-back 恢复 |
| mysql client | 是 | 行数校验 |
| rclone | 是 | S3 远程校验 |

## License

MIT
