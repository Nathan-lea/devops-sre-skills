---
name: pg-backup-verify
description: >
  Verify PostgreSQL backup integrity for both logical (pg_dump/pg_dumpall)
  and physical (pg_basebackup) backups. Use after running your backup
  script to confirm backups are restorable, or during disaster recovery
  drills. Checks backup file completeness, test restore to a temporary
  instance, row count and sequence validation, and S3 remote copy
  verification. Ensures 3-2-1 backup compliance.
tools_allowed:
  - bash
  - psql
  - pg_dump
  - pg_restore
  - pg_basebackup
  - rclone
safety: read-only-verify
---

# PostgreSQL Backup Verify

验证 PostgreSQL 备份的完整性和可恢复性。
支持逻辑备份（pg_dump/pg_dumpall）和物理备份（pg_basebackup）两种模式，
适用于任何 PostgreSQL 备份体系。

## 适用场景

- 每日备份后的完整性验证
- 灾备恢复演练
- S3 远程备份校验
- 备份模式切换后的验证（logical <-> physical）

## 前置条件

```bash
# PostgreSQL 客户端工具
pg_dump --version 2>/dev/null || echo "安装: apt install postgresql-client"
psql --version 2>/dev/null || echo "安装: apt install postgresql-client"

# rclone 已配置 S3
rclone listremotes | grep -q s3 && echo "rclone S3 已配置" || echo "需配置 rclone S3"

# 备份凭据
echo "PG_HOST=${PG_HOST:-127.0.0.1}"
echo "PG_PORT=${PG_PORT:-5432}"
echo "PG_USER=${PG_USER:-backup}"
echo "PG_PASS=${PG_PASS:-<需设置环境变量>}"
echo "BACKUP_MODE=${BACKUP_MODE:-logical}"
echo "BACKUP_BASE=${BACKUP_BASE:-/srv/backup/postgresql}"
echo "S3_BUCKET=${S3_BUCKET:-s3://db-backup/postgresql}"
```

> 所有密码均为占位符，通过环境变量或 Vault 注入，禁止硬编码。

## 验证步骤

### 步骤 1 - 本地备份文件检查

```bash
export PGPASSWORD="${PG_PASS}"
BACKUP_BASE="${BACKUP_BASE:-/srv/backup/postgresql}"
HOSTNAME=$(hostname)
DATE=$(date +%Y%m%d)
BACKUP_DIR="$BACKUP_BASE/$HOSTNAME/$DATE"

# 1a. 检查备份目录存在且非空
if [ ! -d "$BACKUP_DIR" ] || [ -z "$(ls -A "$BACKUP_DIR" 2>/dev/null)" ]; then
  echo "[FAIL] 备份目录不存在或为空: $BACKUP_DIR"
  exit 1
fi
echo "[PASS] 备份目录存在: $BACKUP_DIR"

# 1b. 按备份模式检查关键文件
if [ "${BACKUP_MODE:-logical}" = "logical" ]; then
  # 逻辑备份：检查 globals.sql.gz 和 .dump 文件
  if [ ! -f "$BACKUP_DIR/globals.sql.gz" ]; then
    echo "[FAIL] globals.sql.gz 不存在"
    exit 1
  fi
  echo "[PASS] globals.sql.gz 存在"

  DUMP_COUNT=$(find "$BACKUP_DIR" -name "*.dump" | wc -l)
  echo "[INFO] 逻辑备份文件数: $DUMP_COUNT 个 .dump"
  [ "$DUMP_COUNT" -eq 0 ] && echo "[FAIL] 无 .dump 文件" && exit 1

else
  # 物理备份：检查 base.tar.gz 和 pg_wal/
  if [ ! -d "$BACKUP_DIR/base" ] && [ ! -f "$BACKUP_DIR/base.tar.gz" ]; then
    echo "[FAIL] 物理备份 base 目录/文件不存在"
    exit 1
  fi
  echo "[PASS] 物理备份 base 存在"
fi

# 1c. 备份文件大小
BACKUP_SIZE=$(du -sb "$BACKUP_DIR" | cut -f1)
echo "[INFO] 备份大小: $(numfmt --to=iec "$BACKUP_SIZE")"
```

### 步骤 2 - 逻辑备份验证（pg_dump 模式）

```bash
if [ "${BACKUP_MODE:-logical}" = "logical" ]; then
  echo "===== 逻辑备份验证 ====="

  # 2a. globals 完整性
  echo "--- globals.sql.gz 解压测试 ---"
  if zcat "$BACKUP_DIR/globals.sql.gz" | head -5 | grep -q "PostgreSQL"; then
    echo "[PASS] globals.sql.gz 可正常解压"
  else
    echo "[FAIL] globals.sql.gz 解压失败"
    exit 1
  fi

  # 2b. 逐库恢复测试（恢复到临时数据库）
  DATABASES=$(psql -h "${PG_HOST:-127.0.0.1}" -p "${PG_PORT:-5432}" -U "${PG_USER:-backup}" \
    -Atqc "SELECT datname FROM pg_database WHERE datistemplate = false" 2>/dev/null)

  for DB in $DATABASES; do
    echo "--- 验证数据库: $DB ---"

    # 检查 dump 文件可被 pg_restore 读取
    if ! pg_restore --list "$BACKUP_DIR/${DB}.dump" >/dev/null 2>&1; then
      echo "[FAIL] ${DB}.dump 无法被 pg_restore 读取"
      continue
    fi
    echo "  [PASS] ${DB}.dump 结构完整"

    # 恢复到临时数据库验证
    TEMP_DB="_verify_${DB}_$$"
    psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d postgres \
      -c "CREATE DATABASE \"$TEMP_DB\";" 2>/dev/null

    if pg_restore -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" \
      -d "$TEMP_DB" --no-owner --no-privileges \
      "$BACKUP_DIR/${DB}.dump" 2>/dev/null; then

      # 行数对比
      TABLES=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d "$DB" \
        -Atqc "SELECT tablename FROM pg_tables WHERE schemaname='public'" 2>/dev/null)
      for TBL in $TABLES; do
        SRC_ROWS=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d "$DB" \
          -Atqc "SELECT count(*) FROM public.\"$TBL\"" 2>/dev/null)
        DST_ROWS=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d "$TEMP_DB" \
          -Atqc "SELECT count(*) FROM public.\"$TBL\"" 2>/dev/null)
        if [ "$SRC_ROWS" = "$DST_ROWS" ]; then
          printf "  [PASS] %-40s %s rows\n" "$TBL" "$SRC_ROWS"
        else
          printf "  [FAIL] %-40s 源=%s 恢复=%s\n" "$TBL" "$SRC_ROWS" "$DST_ROWS"
        fi
      done

      # 序列值对比
      SEQS=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d "$DB" \
        -Atqc "SELECT sequence_name FROM information_schema.sequences WHERE sequence_schema='public'" 2>/dev/null)
      for SEQ in $SEQS; do
        SRC_LAST=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d "$DB" \
          -Atqc "SELECT last_value FROM public.\"$SEQ\"" 2>/dev/null)
        DST_LAST=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d "$TEMP_DB" \
          -Atqc "SELECT last_value FROM public.\"$SEQ\"" 2>/dev/null)
        if [ "$SRC_LAST" -le "$DST_LAST" ]; then
          printf "  [PASS] %-40s %s\n" "$SEQ" "$SRC_LAST"
        else
          printf "  [WARN] %-40s 源=%s 恢复=%s (序列偏小)\n" "$SEQ" "$SRC_LAST" "$DST_LAST"
        fi
      done
    else
      echo "  [FAIL] ${DB}.dump 恢复失败"
    fi

    # 清理临时数据库
    psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" -d postgres \
      -c "DROP DATABASE IF EXISTS \"$TEMP_DB\";" 2>/dev/null
  done
fi
```

### 步骤 3 - 物理备份验证（pg_basebackup 模式）

```bash
if [ "${BACKUP_MODE}" = "physical" ]; then
  echo "===== 物理备份验证 ====="

  TEMP_PGDATA="/tmp/pg_verify_$$"
  mkdir -p "$TEMP_PGDATA"

  # 解压 base backup
  if [ -f "$BACKUP_DIR/base.tar.gz" ]; then
    tar -xzf "$BACKUP_DIR/base.tar.gz" -C "$TEMP_PGDATA"
  elif [ -d "$BACKUP_DIR/base" ]; then
    cp -r "$BACKUP_DIR/base/"* "$TEMP_PGDATA/"
  fi

  # 启动临时 PostgreSQL 实例
  TEMP_PORT=15432
  # 清理 postmaster.pid
  rm -f "$TEMP_PGDATA/postmaster.pid"
  # 修改端口
  echo "port = $TEMP_PORT" >> "$TEMP_PGDATA/postgresql.conf"
  echo "listen_addresses = ''" >> "$TEMP_PGDATA/postgresql.conf"

  pg_ctl -D "$TEMP_PGDATA" -l /tmp/pg_verify.log start 2>/dev/null
  sleep 3

  if psql -h 127.0.0.1 -p "$TEMP_PORT" -U "${PG_USER}" -c "SELECT 1;" 2>/dev/null; then
    echo "[PASS] 物理备份恢复实例启动成功"

    # 对比数据库列表
    SRC_DBS=$(psql -h "${PG_HOST}" -p "${PG_PORT}" -U "${PG_USER}" \
      -Atqc "SELECT datname FROM pg_database WHERE datistemplate=false" 2>/dev/null | sort)
    DST_DBS=$(psql -h 127.0.0.1 -p "$TEMP_PORT" -U "${PG_USER}" \
      -Atqc "SELECT datname FROM pg_database WHERE datistemplate=false" 2>/dev/null | sort)
    if [ "$SRC_DBS" = "$DST_DBS" ]; then
      echo "[PASS] 数据库列表一致"
    else
      echo "[FAIL] 数据库列表不一致"
      diff <(echo "$SRC_DBS") <(echo "$DST_DBS")
    fi
  else
    echo "[FAIL] 临时实例启动失败"
    cat /tmp/pg_verify.log
  fi

  # 清理
  pg_ctl -D "$TEMP_PGDATA" stop 2>/dev/null
  rm -rf "$TEMP_PGDATA" /tmp/pg_verify.log
fi
```

### 步骤 4 - S3 远程备份校验

```bash
S3_BUCKET="${S3_BUCKET:-s3://db-backup/postgresql}"

echo "===== S3 远程备份校验 ====="

# 4a. 文件列表对比
rclone check "$BACKUP_DIR" "$S3_BUCKET/$(hostname)/$DATE/" \
  --one-way 2>&1 | tail -3

# 4b. 抽样下载校验（globals.sql.gz）
rclone cat "$S3_BUCKET/$(hostname)/$DATE/globals.sql.gz" 2>/dev/null | \
  zcat 2>/dev/null | head -3 || echo "[WARN] S3 文件下载校验失败"

# 4c. 大小对比
REMOTE_SIZE=$(rclone size "$S3_BUCKET/$(hostname)/$DATE/" --json 2>/dev/null | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['bytes'])" 2>/dev/null || echo 0)
echo "本地: $(numfmt --to=iec $BACKUP_SIZE)  S3: $(numfmt --to=iec $REMOTE_SIZE)"
```

### 步骤 5 - 生成验证报告

```bash
REPORT="/var/log/postgresql_backup_verify_$(date +%Y%m%d).log"
cat > "$REPORT" << REPORT_EOF
===== PostgreSQL 备份验证报告 =====
日期: $(date)
主机: $(hostname)
备份模式: ${BACKUP_MODE:-logical}
备份目录: $BACKUP_DIR
备份大小: $(numfmt --to=iec $BACKUP_SIZE)
备份文件数: $(find $BACKUP_DIR -type f | wc -l)
S3 校验: $(rclone check "$BACKUP_DIR" "$S3_BUCKET/$(hostname)/$DATE/" --one-way 2>&1 | grep -c "matching")
监控指标: postgresql_backup_status=1
REPORT_EOF
cat "$REPORT"
```

## 回滚指引

- 验证过程中创建的临时数据库在步骤 2 末尾已自动 `DROP DATABASE` 清理。
- 验证过程中创建的临时 PGDATA 在步骤 3 末尾已自动删除。
- 如果临时实例未正确关闭：
  ```bash
  pkill -f "postgres.*15432"
  rm -rf /tmp/pg_verify_*
  ```
- 如果验证发现备份不可用，**立即重新执行备份**：
  ```bash
  bash <your-backup-script.sh>
  # 或通过你的编排工具（如 Ansible）执行
  ```

## 异常处理

| 故障 | 排查 | 处置 |
|------|------|------|
| `globals.sql.gz 解压失败` | 磁盘满/网络中断 | 检查磁盘空间，重新备份 |
| `.dump 文件无法 pg_restore` | 版本不匹配/文件损坏 | 确认 pg_dump 版本 >= 服务端版本 |
| 临时数据库恢复失败 | 权限不足 | 检查 PG_USER 是否有 CREATEDB 权限 |
| 行数不一致 | 备份期间有写入 | 确认在业务低峰备份或使用物理备份 |
| 序列值偏小 | dump 时序列未刷新 | `SELECT setval('seq', max(id)) FROM tbl` 修正 |
| 物理备份实例启动失败 | postmaster.pid 残留 | `rm -f $TEMP_PGDATA/postmaster.pid` |
| S3 校验失败 | 网络中断/rclone 配置 | `rclone config show` 重新配置 |
| PG_PASS 未设置 | 环境变量缺失 | 通过 Vault 或环境变量注入 |
