# wazuh-rule-tuning

调优 Wazuh HIDS、Suricata NIDS、fail2ban 与 Prometheus 安全告警规则，降低误报率、提升检测精度。

## 用途

- 识别高频误报并抑制
- 新增自定义威胁检测规则
- 按环境差异化调整安全基线阈值
- fail2ban 封禁参数调优

## safety 契约

`read-only-tuning` - 提供调优建议，实际变更需人工确认后执行。

## 使用方式

```bash
export SECURITY_RULES=/path/to/your/security-rules.yml
# 按 SKILL.md 步骤执行调优
```

## 依赖

| 工具 | 必需 | 说明 |
|------|------|------|
| Wazuh manager | 是 | HIDS 告警源 |
| Suricata | 是 | NIDS 规则调优 |
| fail2ban | 是 | 封禁策略调优 |
| Prometheus + python3 | 是 | 告警阈值校验 |

## License

MIT
