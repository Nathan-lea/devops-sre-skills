# devops-sre-skills

> 面向运维 SRE 垂直域的 Codex/Claude Agent Skills 集合。
> 每个 skill 遵循统一规范：YAML frontmatter（含 `tools_allowed`/`safety` 契约）
> + Markdown 正文（含验证步骤与回滚指引），所有示例均使用占位符，无真实密钥。

## 背景

在构建多云异构 Linux 服务器运维体系的过程中，识别出运维 SRE 垂直域（监控、数据库、
安全、备份、迁移）缺少可直接使用的 Agent Skill。为此自研了 6 个 skill 填补缺口，
现泛化后作为独立仓库贡献给社区，使更多 SRE/Ops 团队受益。

## Skill 清单

| Skill | 用途 | safety |
|-------|------|--------|
| [prometheus-rules-lint](prometheus-rules-lint/SKILL.md) | Prometheus 告警规则 YAML 校验（语法/结构/命名/PromQL） | read-only |
| [tidb-br-backup](tidb-br-backup/SKILL.md) | TiDB BR 全量备份、校验、灾难恢复、过期清理 | destructive-if-restore |
| [wazuh-rule-tuning](wazuh-rule-tuning/SKILL.md) | Wazuh/Suricata/fail2ban/Prometheus 安全规则调优 | read-only-tuning |
| [mysql-xtrabackup-verify](mysql-xtrabackup-verify/SKILL.md) | MySQL XtraBackup 备份完整性与可恢复性验证 | read-only-verify |
| [pg-backup-verify](pg-backup-verify/SKILL.md) | PostgreSQL 逻辑/物理备份完整性与可恢复性验证 | read-only-verify |
| [migration-verify](migration-verify/SKILL.md) | 迁移后数据一致性校验（MySQL/PG/文件），go/no-go 门禁 | read-only-verify |

## 设计规范

- **YAML frontmatter**：`name` + `description` + `tools_allowed` + `safety` 四字段
- **safety 契约**：`read-only` / `read-only-verify` / `read-only-tuning` / `destructive-if-restore`
- **正文结构**：适用场景 → 前置条件 → 编号步骤 → 验证 → 回滚指引 → 异常处理
- **占位符**：所有 IP（`10.0.x.x`）、密码（环境变量）、密钥均为占位符，禁止真实凭据
- **独立性**：不依赖任何特定项目结构，所有路径引用使用占位符或环境变量

## 安装

### 方式一：Codex / Claude（Agent Skills 目录）

```bash
git clone https://github.com/<your-org>/devops-sre-skills.git
cd devops-sre-skills
for s in prometheus-rules-lint tidb-br-backup wazuh-rule-tuning \
         mysql-xtrabackup-verify pg-backup-verify migration-verify; do
  ln -sf "$(pwd)/$s" ~/.agents/skills/$s
done

# 验证
ls ~/.agents/skills/*/SKILL.md | grep -E "prometheus|tidb|wazuh|mysql|pg-backup|migration"
```

### 方式二：skills.sh 包管理器

```bash
npx skills add <your-org>/devops-sre-skills -g -y

# 安装单个 skill
npx skills add <your-org>/devops-sre-skills@prometheus-rules-lint
```

## 校验

安装后可验证所有 skill 格式是否合规：

```bash
python3 -c "
import re, yaml, glob
for f in sorted(glob.glob('*/SKILL.md')):
    c = open(f).read()
    m = re.match(r'^---\n(.*?)\n---\n', c, re.DOTALL)
    meta = yaml.safe_load(m.group(1))
    assert 'name' in meta, f'{f}: 缺少 name'
    assert 'safety' in meta, f'{f}: 缺少 safety'
    assert meta['name'] in f, f'{f}: name 与目录名不一致'
    assert '回滚' in c or 'rollback' in c.lower(), f'{f}: 缺少回滚指引'
    print(f'  [PASS] {f} (name={meta[\"name\"]}, safety={meta[\"safety\"]})')
print('全部校验通过')
"
```

## 仓库结构

```
devops-sre-skills/
├── LICENSE                         # MIT
├── README.md                       # 本文件
├── CONTRIBUTING.md                 # 贡献指南
├── prometheus-rules-lint/
│   ├── SKILL.md
│   └── README.md
├── tidb-br-backup/
│   ├── SKILL.md
│   └── README.md
├── wazuh-rule-tuning/
│   ├── SKILL.md
│   └── README.md
├── mysql-xtrabackup-verify/
│   ├── SKILL.md
│   └── README.md
├── pg-backup-verify/
│   ├── SKILL.md
│   └── README.md
└── migration-verify/
    ├── SKILL.md
    └── README.md
```

## License

MIT
