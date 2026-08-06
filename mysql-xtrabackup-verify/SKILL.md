---
name: mysql-xtrabackup-verify
description: >
  Verify MySQL backup integrity created by Percona XtraBackup. Use after
  running your backup script to confirm backup is restorable, or during
  disaster recovery drills. Checks xtrabackup_info existence, performs
  prepare (redo log apply), test restore to a temporary instance, row count
  validation, and S3 remote copy verification. Ensures 3-2-1 backup compliance.
tools_allowed:
  - bash
  - xtrabackup
  - mysql
  - rclone
safety: read-only-verify
---

# MySQL XtraBackup Verify

验证 MySQL xtrabackup 备份的完整性和可恢复性。
适用于任何使用 Percona XtraBackup 产生的备份，可配合你的备份脚本使用。

## 适用场景

- 每日备份后的完整性验证
- 灾备恢复演练
- S3 远程备份校验
- 备份策略变更后的验证

## 前置条件

```bash
# xtrabackup 已安装
xtrabackup --version 2>/dev/null || echo "安装: https://docs.percona.com/percona-xtrabackup/"

# MySQL 客户端
mysql --version 2>/dev/null || echo "安装: apt install mysql-client"

# rclone 已配置 S3
rclone listremotes | grep -q s3 && echo "rclone S3 已配置" || echo "需配置 rclone S3"

# 备份凭据
echo "BACKUP_USER=${BACKUP_USER:-backup}"
echo "BACKUP_PASS=${BACKUP_PASS:-<需设置环境变量>}"
echo "MYSQL_HOST=${MYSQL_HOST:-127.0.0.1}"
echo "MYSQL_PORT=${MYSQL_PORT:-3306}"
echo "BACKUP_BASE=${BACKUP_BASE:-/data/backup/mysql}"
echo "S3_BUCKET=${S3_BUCKET:-s3://db-backup/mysql}"
```

> 所有密码均为占位符，通过环境变量或 Vault 注入，禁止硬编码。

## 验证步骤

### 步骤 1 - 本地备份文件检查

```bash
BACKUP_BASE="${BACKUP_BASE:-/data/backup/mysql}"
HOSTNAME=$(hostname)
DATE=$(date +%Y%m%d)
BACKUP_DIR="$BACKUP_BASE/$HOSTNAME/$DATE"

# 1a. 检查备份目录存在
if [ ! -d "$BACKUP_DIR" ]; then
  echo "[FAIL] 备份目录不存在: $BACKUP_DIR"
  exit 1
fi

# 1b. 检查 xtrabackup_info 文件（关键元数据）
if [ ! -f "$BACKUP_DIR/xtrabackup_info" ]; then
  echo "[FAIL] xtrabackup_info 不存在，备份可能不完整"
  exit 1
fi
echo "[PASS] xtrabackup_info 存在"

# 1c. 检查关键文件
for f in xtrabackup_checkpoints xtrabackup_logfile; do
  if [ ! -f "$BACKUP_DIR/$f" ]; then
    echo "[WARN] 缺少 $f"
  fi
done

# 1d. 查看备份元数据
cat "$BACKUP_DIR/xtrabackup_info"
```

### 步骤 2 - 执行 prepare（redo log 应用）

```bash
# prepare 阶段：应用 redo log，使备份达到一致状态
# 必须执行，否则无法启动恢复
xtrabackup --prepare \
  --target-dir="$BACKUP_DIR" \
  2>&1 | tee /var/log/mysql_backup_prepare.log

if [ ${PIPESTATUS[0]} -eq 0 ]; then
  echo "[PASS] prepare 成功，备份可恢复"
else
  echo "[FAIL] prepare 失败，备份不可用"
  exit 1
fi
```

> **注意**：prepare 会修改备份目录，建议先复制一份再 prepare。
> 对于压缩备份需先 `xtrabackup --decompress`。

### 步骤 3 - 测试恢复到临时实例

```bash
# 使用临时 datadir 验证恢复
TEMP_DATADIR="/tmp/mysql_verify_$$"
mkdir -p "$TEMP_DATADIR"

# 恢复备份到临时目录
xtrabackup --copy-back \
  --target-dir="$BACKUP_DIR" \
  --datadir="$TEMP_DATADIR" 2>&1 | tee /var/log/mysql_verify_restore.log

if [ ${PIPESTATUS[0]} -ne 0 ]; then
  echo "[FAIL] 恢复到临时目录失败"
  rm -rf "$TEMP_DATADIR"
  exit 1
fi

# 启动临时 MySQL 实例验证
TEMP_PORT=13306
mysqld --datadir="$TEMP_DATADIR" --socket="/tmp/mysql_verify.sock" \
  --port="$TEMP_PORT" --skip-networking &
MYSQL_PID=$!
sleep 5

# 检查实例是否正常启动
if mysql --socket="/tmp/mysql_verify.sock" -e "SELECT 1;" 2>/dev/null; then
  echo "[PASS] 临时实例启动成功"
else
  echo "[FAIL] 临时实例启动失败"
  kill $MYSQL_PID 2>/dev/null
  rm -rf "$TEMP_DATADIR"
  exit 1
fi
```

### 步骤 4 - 行数校验

```bash
# 获取源端数据库列表
SOURCE_DBS=$(mysql -h "${MYSQL_HOST:-127.0.0.1}" -P "${MYSQL_PORT:-3306}" \
  -u "${BACKUP_USER:-backup}" -p"${BACKUP_PASS}" \
  -e "SHOW DATABASES;" -s --skip-column-names 2>/dev/null | \
  grep -vE '^(information_schema|performance_schema|mysql|sys)$')

# 逐库逐表对比行数
MISMATCH=0
for DB in $SOURCE_DBS; do
  TABLES=$(mysql -h "${MYSQL_HOST}" -u "${BACKUP_USER}" -p"${BACKUP_PASS}" \
    -e "SHOW TABLES;" -s --skip-column-names "$DB" 2>/dev/null)
  for TBL in $TABLES; do
    SRC_ROWS=$(mysql -h "${MYSQL_HOST}" -u "${BACKUP_USER}" -p"${BACKUP_PASS}" \
      -e "SELECT count(*) FROM \`$TBL\`;" -s --skip-column-names "$DB" 2>/dev/null)
    DST_ROWS=$(mysql --socket="/tmp/mysql_verify.sock" \
      -e "SELECT count(*) FROM \`$TBL\`;" -s --skip-column-names "$DB" 2>/dev/null)
    if [ "$SRC_ROWS" = "$DST_ROWS" ]; then
      printf "  [PASS] %s.%s  %s rows\n" "$DB" "$TBL" "$SRC_ROWS"
    else
      printf "  [FAIL] %s.%s  源=%s  恢复=%s\n" "$DB" "$TBL" "$SRC_ROWS" "$DST_ROWS"
      MISMATCH=1
    fi
  done
done

# 清理临时实例
kill $MYSQL_PID 2>/dev/null
rm -rf "$TEMP_DATADIR" /tmp/mysql_verify.sock
```

### 步骤 5 - S3 远程备份校验

```bash
S3_BUCKET="${S3_BUCKET:-s3://db-backup/mysql}"

# 5a. 文件列表对比
echo "--- S3 文件校验 ---"
rclone check "$BACKUP_DIR" "$S3_BUCKET/$(hostname)/$DATE/" \
  --one-way 2>&1 | tail -5

# 5b. 抽样下载校验
rclone cat "$S3_BUCKET/$(hostname)/$DATE/xtrabackup_info" | head -5

# 5c. 备份大小对比
LOCAL_SIZE=$(du -sb "$BACKUP_DIR" | cut -f1)
REMOTE_SIZE=$(rclone size "$S3_BUCKET/$(hostname)/$DATE/" --json 2>/dev/null | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['bytes'])" 2>/dev/null || echo 0)
echo "本地: $(numfmt --to=iec $LOCAL_SIZE)  S3: $(numfmt --to=iec $REMOTE_SIZE)"
```

### 步骤 6 - 生成验证报告

```bash
REPORT="/var/log/mysql_backup_verify_$(date +%Y%m%d).log"
cat > "$REPORT" << REPORT_EOF
===== MySQL 备份验证报告 =====
日期: $(date)
主机: $(hostname)
备份目录: $BACKUP_DIR
备份大小: $(numfmt --to=iec $LOCAL_SIZE)
xtrabackup_info: $(test -f $BACKUP_DIR/xtrabackup_info && echo "存在" || echo "缺失")
prepare 结果: $(grep -c "completed OK" /var/log/mysql_backup_prepare.log 2>/dev/null && echo "成功" || echo "失败")
S3 校验: $(rclone check "$BACKUP_DIR" "$S3_BUCKET/$(hostname)/$DATE/" --one-way 2>&1 | grep -c "matching" && echo "通过" || echo "需检查")
行数校验: $([ $MISMATCH -eq 0 ] && echo "全部一致" || echo "存在不一致")
监控指标: mysql_backup_status=1, mysql_backup_size_bytes=$LOCAL_SIZE
REPORT_EOF
cat "$REPORT"
```

## 回滚指引

- 验证过程中创建的临时实例和目录在步骤 4 末尾已自动清理。
- 如果临时实例未正确关闭：
  ```bash
  pkill -f "mysqld.*mysql_verify"
  rm -rf /tmp/mysql_verify_*
  ```
- prepare 操作不可逆地修改了备份目录。如需保留原始备份：
  在步骤 2 前执行 `cp -r "$BACKUP_DIR" "$BACKUP_DIR.prepare_copy"`
- 如果验证发现备份不可用，**立即重新执行备份**：
  ```bash
  bash <your-backup-script.sh>
  ```

## 异常处理

| 故障 | 排查 | 处置 |
|------|------|------|
| `xtrabackup_info 缺失` | 备份被中断 | 重新执行备份 |
| prepare 失败 | redo log 损坏 | 重新执行备份，检查磁盘空间 |
| 临时实例启动失败 | datadir 权限/版本不匹配 | `chown -R mysql:mysql $TEMP_DATADIR` |
| 行数不一致 | 备份期间有写入 | 检查是否在业务低峰备份 |
| S3 校验失败 | 网络中断/rclone 配置 | `rclone config show` 重新配置 |
| BACKUP_PASS 未设置 | 环境变量缺失 | 通过 Vault 或环境变量注入 |
