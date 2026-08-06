---
name: migration-verify
description: >
  Verify data consistency after database or file migrations (MySQL to TiDB,
  PostgreSQL to TiDB, cross-cloud database migration, K8s cross-cloud,
  Web server migration). Use to confirm source and target are in sync
  before traffic cutover. Covers row count comparison, CRC32 checksum,
  sequence validation, file-level rsync dry-run, and generates a go/no-go
  report. Works with any migration verify script or manual commands.
tools_allowed:
  - bash
  - mysql
  - psql
  - rsync
  - ssh
  - python3
safety: read-only-verify
---

# Migration Verify

验证数据库迁移和文件迁移后的数据一致性，作为流量切换前的 go/no-go 门禁。
可配合你的迁移校验脚本使用，也支持手动执行各校验步骤。

## 适用场景

- MySQL -> TiDB 迁移后数据校验
- PostgreSQL -> TiDB 迁移后数据校验
- 数据库跨云迁移后数据校验
- Web 静态资源迁移后文件校验
- 灰度切换前的最终一致性确认

## 前置条件

```bash
# MySQL 客户端（MySQL/TiDB 校验用）
mysql --version 2>/dev/null || echo "安装: apt install mysql-client"

# PostgreSQL 客户端（PG 校验用）
psql --version 2>/dev/null || echo "安装: apt install postgresql-client"

# rsync（文件校验用）
rsync --version 2>/dev/null | head -1 || echo "安装: apt install rsync"

# SSH 免密已配置（文件校验需 SSH 到源/目标主机）
ssh -o BatchMode=yes -o ConnectTimeout=5 <user>@SOURCE_IP echo "SSH ok" 2>/dev/null || echo "需配置 SSH 免密"

# 环境变量
echo "SOURCE_IP=${SOURCE_IP:-<源端IP>}"
echo "TARGET_IP=${TARGET_IP:-<目标端IP>}"
echo "VERIFY_TYPE=${VERIFY_TYPE:-mysql}  # mysql|pg|file"
```

> 所有 IP、密码均为占位符，通过环境变量或 Vault 注入。

## 校验步骤

### 步骤 1 - 执行数据校验

```bash
# 方式一：使用你的迁移校验脚本（如已封装）
# bash <your-verify-script.sh> \
#   --source "${SOURCE_IP}" \
#   --target "${TARGET_IP}" \
#   --type "${VERIFY_TYPE:-mysql}"

# 方式二：按下方步骤 2-5 手动执行校验
```

校验退出码含义：
- `0` = 全部一致，可以切换流量（go）
- `0`（含警告）= 存在警告项，需人工确认
- `1` = 存在不一致项，禁止切换流量（no-go）

### 步骤 2 - MySQL 行数 + CRC32 校验（手动执行）

```bash
# 适用于 MySQL -> TiDB 或 MySQL 跨云迁移
SOURCE_IP="${SOURCE_IP:-10.0.2.10}"
TARGET_IP="${TARGET_IP:-10.1.2.10}"
MYSQL_USER="verify_user"
MYSQL_PASS="${MYSQL_PASS}"

# 2a. 获取数据库列表
DBS=$(mysql -h"$SOURCE_IP" -u"$MYSQL_USER" -p"$MYSQL_PASS" \
  -e "SHOW DATABASES;" -s --skip-column-names 2>/dev/null | \
  grep -vE '^(information_schema|performance_schema|mysql|sys|INFORMATION_SCHEMA|PERFORMANCE_SCHEMA|METRICS_SCHEMA)$')

for DB in $DBS; do
  echo "===== 数据库: $DB ====="

  # 2b. 获取表列表
  TABLES=$(mysql -h"$SOURCE_IP" -u"$MYSQL_USER" -p"$MYSQL_PASS" \
    -e "SHOW TABLES;" -s --skip-column-names "$DB" 2>/dev/null)

  for TBL in $TABLES; do
    # 行数对比
    SRC_ROWS=$(mysql -h"$SOURCE_IP" -u"$MYSQL_USER" -p"$MYSQL_PASS" \
      -e "SELECT count(*) FROM \`$TBL\`;" -s --skip-column-names "$DB" 2>/dev/null)
    DST_ROWS=$(mysql -h"$TARGET_IP" -u"$MYSQL_USER" -p"$MYSQL_PASS" \
      -e "SELECT count(*) FROM \`$TBL\`;" -s --skip-column-names "$DB" 2>/dev/null)

    # CRC32 校验（使用 pt-table-checksum 或手动）
    SRC_CRC=$(mysql -h"$SOURCE_IP" -u"$MYSQL_USER" -p"$MYSQL_PASS" \
      -e "SELECT COALESCE(SUM(CRC32(CONCAT_WS('#', \`$TBL\`.*))),0) FROM \`$TBL\`;" \
      -s --skip-column-names "$DB" 2>/dev/null)
    DST_CRC=$(mysql -h"$TARGET_IP" -u"$MYSQL_USER" -p"$MYSQL_PASS" \
      -e "SELECT COALESCE(SUM(CRC32(CONCAT_WS('#', \`$TBL\`.*))),0) FROM \`$TBL\`;" \
      -s --skip-column-names "$DB" 2>/dev/null)

    if [ "$SRC_ROWS" = "$DST_ROWS" ] && [ "$SRC_CRC" = "$DST_CRC" ]; then
      printf "  [PASS] %-40s rows=%-10s crc32=%s\n" "$TBL" "$SRC_ROWS" "$SRC_CRC"
    else
      printf "  [FAIL] %-40s src_rows=%s dst_rows=%s src_crc=%s dst_crc=%s\n" \
        "$TBL" "$SRC_ROWS" "$DST_ROWS" "$SRC_CRC" "$DST_CRC"
    fi
  done
done
```

### 步骤 3 - PostgreSQL 行数 + 序列校验（手动执行）

```bash
# 适用于 PostgreSQL -> TiDB 或 PG 跨云迁移
export PGPASSWORD="${PG_PASS}"
SOURCE_IP="${SOURCE_IP:-10.0.3.10}"
TARGET_IP="${TARGET_IP:-10.1.3.10}"
PG_USER="verify_user"
DB_NAME="target_db"

# 3a. 表行数对比
TABLES=$(psql -h"$SOURCE_IP" -U"$PG_USER" -d"$DB_NAME" \
  -Atqc "SELECT tablename FROM pg_tables WHERE schemaname='public'" 2>/dev/null)

for TBL in $TABLES; do
  SRC_ROWS=$(psql -h"$SOURCE_IP" -U"$PG_USER" -d"$DB_NAME" \
    -Atqc "SELECT count(*) FROM public.\"$TBL\"" 2>/dev/null)
  DST_ROWS=$(psql -h"$TARGET_IP" -U"$PG_USER" -d"$DB_NAME" \
    -Atqc "SELECT count(*) FROM public.\"$TBL\"" 2>/dev/null)

  if [ "$SRC_ROWS" = "$DST_ROWS" ]; then
    printf "  [PASS] %-40s rows=%s\n" "$TBL" "$SRC_ROWS"
  else
    printf "  [FAIL] %-40s src=%s dst=%s\n" "$TBL" "$SRC_ROWS" "$DST_ROWS"
  fi
done

# 3b. 序列最大值检查（防止迁移后序列溢出导致主键冲突）
SEQS=$(psql -h"$SOURCE_IP" -U"$PG_USER" -d"$DB_NAME" \
  -Atqc "SELECT sequence_name FROM information_schema.sequences WHERE sequence_schema='public'" 2>/dev/null)

for SEQ in $SEQS; do
  SRC_LAST=$(psql -h"$SOURCE_IP" -U"$PG_USER" -d"$DB_NAME" \
    -Atqc "SELECT last_value FROM public.\"$SEQ\"" 2>/dev/null)
  DST_LAST=$(psql -h"$TARGET_IP" -U"$PG_USER" -d"$DB_NAME" \
    -Atqc "SELECT last_value FROM public.\"$SEQ\"" 2>/dev/null)

  if [ "$SRC_LAST" -le "$DST_LAST" ]; then
    printf "  [PASS] 序列 %-30s src=%s dst=%s\n" "$SEQ" "$SRC_LAST" "$DST_LAST"
  else
    printf "  [WARN] 序列 %-30s src=%s dst=%s  目标偏小，需 setval 修正\n" \
      "$SEQ" "$SRC_LAST" "$DST_LAST"
  fi
done
unset PGPASSWORD
```

### 步骤 4 - 文件级 rsync 校验（Web 迁移）

```bash
# 适用于 Web 服务器静态资源迁移
SOURCE_IP="${SOURCE_IP:-10.0.1.11}"
TARGET_IP="${TARGET_IP:-10.1.1.11}"
FILE_PATH="/var/www"
SSH_USER="${SSH_USER:-ops}"

# 4a. 文件数与总大小对比
SRC_SUMMARY=$(ssh -o StrictHostKeyChecking=no "$SSH_USER@$SOURCE_IP" \
  "find '$FILE_PATH' -type f | wc -l; du -sb '$FILE_PATH' | cut -f1" 2>/dev/null)
DST_SUMMARY=$(ssh -o StrictHostKeyChecking=no "$SSH_USER@$TARGET_IP" \
  "find '$FILE_PATH' -type f | wc -l; du -sb '$FILE_PATH' | cut -f1" 2>/dev/null)

SRC_FILES=$(echo "$SRC_SUMMARY" | head -1)
SRC_SIZE=$(echo "$SRC_SUMMARY" | tail -1)
DST_FILES=$(echo "$DST_SUMMARY" | head -1)
DST_SIZE=$(echo "$DST_SUMMARY" | tail -1)

echo "源端: ${SRC_FILES} 文件, $(numfmt --to=iec "$SRC_SIZE")"
echo "目标: ${DST_FILES} 文件, $(numfmt --to=iec "$DST_SIZE")"

[ "$SRC_FILES" = "$DST_FILES" ] && echo "[PASS] 文件数一致" || echo "[FAIL] 文件数不一致"

# 4b. rsync dry-run 差异检查
DIFF=$(rsync -an --stats "$SSH_USER@$SOURCE_IP:$FILE_PATH/" "$SSH_USER@$TARGET_IP:$FILE_PATH/" 2>/dev/null)
DIFF_COUNT=$(echo "$DIFF" | grep -c '^[^ ]' || echo 0)
if [ "$DIFF_COUNT" -le 2 ]; then
  echo "[PASS] rsync 无差异文件"
else
  echo "[WARN] rsync 发现 ${DIFF_COUNT} 个差异项"
  echo "$DIFF" | head -20
fi
```

### 步骤 5 - 生成 go/no-go 报告

```bash
REPORT="/var/log/migration_verify_$(date +%Y%m%d_%H%M%S).log"

cat > "$REPORT" << REPORT_EOF
===== 迁移数据一致性校验报告 =====
日期: $(date)
校验类型: ${VERIFY_TYPE:-mysql}
源端: ${SOURCE_IP}
目标: ${TARGET_IP}
====================================

校验结果:
$(cat /tmp/migration_verify_detail.log 2>/dev/null)

结论:
- 行数校验: $([ "${ROW_MISMATCH:-0}" -eq 0 ] && echo "✅ 全部一致" || echo "❌ 存在不一致")
- CRC32 校验: $([ "${CRC_MISMATCH:-0}" -eq 0 ] && echo "✅ 全部一致" || echo "❌ 存在不一致")
- 序列校验: $([ "${SEQ_MISMATCH:-0}" -eq 0 ] && echo "✅ 全部一致" || echo "⚠️ 需修正")
- 文件校验: $([ "${FILE_DIFF:-0}" -le 2 ] && echo "✅ 无差异" || echo "⚠️ 存在差异")

最终判定: $([ "${ROW_MISMATCH:-0}" -eq 0 ] && [ "${CRC_MISMATCH:-0}" -eq 0 ] && echo "✅ GO - 可以切换流量" || echo "❌ NO-GO - 禁止切换流量")
REPORT_EOF

cat "$REPORT"
echo ""
echo "报告已保存: $REPORT"
```

## 回滚指引

- 校验是只读操作，**不会修改源端或目标端数据**，无需回滚。
- 如果校验发现不一致：
  1. 检查 DM/CDC 同步任务是否正常运行
  2. 检查同步延迟（`SHOW SYNC STATUS` 或 DM dashboard）
  3. 等待同步追平后重新校验
  4. 如果同步无法追平，按你的迁移回滚预案执行
- 如果已执行流量切换后发现数据不一致，按你的回滚 SOP 执行流量回切：
  ```bash
  # 示例：DNS 回切
  bash <your-rollback-script.sh> --type dns --action rollback
  ```

## 异常处理

| 故障 | 排查 | 处置 |
|------|------|------|
| 源端连接失败 | 网络/防火墙/凭据 | 检查端口可达性，确认凭据 |
| 目标端连接失败 | 网络/防火墙/凭据 | 同上 |
| 行数不一致 | 同步延迟/同步中断 | 检查 DM/CDC 状态，等待追平 |
| CRC32 不一致 | 数据内容差异 | 定位差异行，检查字符集/时区设置 |
| 序列偏小 | 迁移后序列未同步 | `SELECT setval('seq', max(id)+1000) FROM tbl` |
| 文件数不一致 | rsync 未完成 | 重新执行 rsync 增量同步 |
| SSH 连接失败 | 免密未配置 | `ssh-copy-id <user>@TARGET_IP` |
| 校验超时 | 大表 count 慢 | 使用近似 count 或分批校验 |
