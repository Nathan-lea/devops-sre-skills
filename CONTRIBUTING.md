# 贡献指南 - devops-sre-skills

感谢您对 devops-sre-skills 的贡献！本指南帮助您规范地新增或修改 skill。

## Skill 规范

### 必须包含

1. **YAML frontmatter**（`---` 包裹）：

```yaml
---
name: <skill-name>           # 必须与目录名一致
description: >               # 一句话说明用途与触发场景
  <描述>
tools_allowed:               # 允许调用的工具白名单
  - bash
  - python3
safety: <safety-level>       # 安全契约
---
```

2. **safety 契约**（四选一）：

| 值 | 含义 |
|----|------|
| `read-only` | 纯读取/校验，不修改任何系统状态 |
| `read-only-verify` | 读取+临时验证环境，验证后自动清理 |
| `read-only-tuning` | 读取+调优建议，实际变更需人工确认 |
| `destructive-if-restore` | 恢复操作会覆盖数据，需明确警告 |

3. **正文结构**（按顺序）：

```
# Skill 标题
## 适用场景          -- 什么情况下使用
## 前置条件          -- 工具/环境/凭据检查
## 操作步骤          -- 编号步骤，含可执行命令
### 步骤 1 - xxx
### 步骤 2 - xxx
## 验证              -- 如何确认操作成功
## 回滚指引          -- 出错后如何恢复
## 异常处理          -- 常见故障表（故障/排查/处置）
```

### 禁止

- ❌ 硬编码真实 IP/密码/密钥（使用占位符或环境变量）
- ❌ 引用特定项目路径（skill 必须独立可用，不依赖特定项目结构）
- ❌ 无回滚指引的破坏性操作
- ❌ 无 safety 契约的 skill

### 泛化要求

贡献的 skill 不得包含任何项目特定引用：

| 禁止写法 | 应改为 |
|---------|--------|
| `scripts/xxx.sh` | `<your-script.sh>` 或移除引用 |
| `configs/rules/*.yml` | `<your-rules-dir>/*.yml` 或环境变量 |
| `docs/0X-xxx.md` | 移除（项目内部文档引用） |
| `playbooks/xxx.yml` | 移除或改为通用示例 |
| `本仓库` | 移除或改为通用描述 |

## 新增 Skill 流程

```bash
# 1. 创建目录
mkdir -p <your-skill-name>

# 2. 编写 SKILL.md（参照上述规范）
vi <your-skill-name>/SKILL.md

# 3. 编写 README.md（单 skill 说明）
vi <your-skill-name>/README.md

# 4. 校验
python3 -c "
import re, yaml
c = open('<your-skill-name>/SKILL.md').read()
m = re.match(r'^---\n(.*?)\n---\n', c, re.DOTALL)
meta = yaml.safe_load(m.group(1))
assert 'name' in meta and 'safety' in meta
assert '回滚' in c or 'rollback' in c.lower()
print('校验通过')
"

# 5. 检查无项目特定引用
grep -rn "scripts/\|configs/\|docs/0\|docs/1\|playbooks/\|本仓库" <your-skill-name>/
# 应无输出

# 6. 更新 README.md 清单

# 7. 提交
git add -A
git commit -m "feat: 新增 <your-skill-name> skill"
```

## 提交规范

采用 Conventional Commits：

- `feat: 新增 <skill-name> skill`
- `fix: 修复 <skill-name> 的 <问题>`
- `docs: 更新 <skill-name> 文档`
- `refactor: 泛化 <skill-name>，移除项目特定引用`

## License

本仓库使用 MIT 许可证。提交即表示您同意以 MIT 许可证发布您的贡献。
