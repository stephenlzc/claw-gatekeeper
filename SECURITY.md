# OpenClaw Guardian 安全部署指南

> **Human-in-the-Loop Security Mode** | 人机回环安全模式

---

## 🛡️ 安全模式概览

| 模式 | 自动允许 | 人工确认 | 适用场景 |
|-----|---------|---------|---------|
| **默认 (Default)** | LOW + MEDIUM | HIGH + CRITICAL | 便捷使用 |
| **加固 (Hardened)** | ❌ 无 | ✅ 全部 | **推荐** |
| **紧急 (Emergency)** | ❌ 无 | ✅ 全部 + 延迟确认 | 高危环境 |

---

## 🚀 快速加固（3步）

### Step 1: 备份当前配置
```bash
cd ~/.openclaw-guardian/scripts
./deploy-secure.sh --check
```

### Step 2: 应用加固配置
```bash
./deploy-secure.sh --apply
```

### Step 3: 验证部署
```bash
./deploy-secure.sh --check
```

---

## 📋 加固配置说明

### 核心安全设置

```json
{
  "strict_mode": true,           // 严格模式
  "auto_allow_low": false,       // 禁止自动允许低风险
  "auto_allow_medium": false,    // 禁止自动允许中风险
  "auto_allow_high": false,      // 禁止自动允许高风险
  "medium_requires_approval": true,  // 中等风险需确认
  "confirmation_timeout": 300    // 5分钟确认超时
}
```

### 风险等级行为

| 风险等级 | 分值范围 | 加固模式行为 |
|---------|---------|------------|
| 🔴 **CRITICAL** | 80-100 | 始终单独确认，无会话批准 |
| 🟠 **HIGH** | 60-79 | 始终确认，可会话批准 |
| 🟡 **MEDIUM** | 30-59 | 始终确认，可会话批准 |
| 🟢 **LOW** | 0-29 | **始终确认**（与默认模式不同） |

### 黑名单增强

加固配置包含 **20+ 危险命令模式**：
- 系统破坏：`rm -rf /`, `mkfs`, `dd if=`
- 管道执行：`curl | sh`, `wget | bash`
- 代码注入：`eval $(`, `python -c 'import os'`
- 反向 shell：`nc -e`, `bash -i >& /dev/tcp`

### 敏感目录保护

```json
"blacklist_paths": [
  "~/.ssh",      // SSH 密钥
  "~/.aws",      // AWS 凭证
  "~/.kube",     // Kubernetes 配置
  "~/.gnupg",    // GPG 密钥
  "~/.docker",   // Docker 配置
  "~/.openclaw"  // OpenClaw 自身配置
]
```

---

## 🔐 数据脱敏（可选）

### 使用 Sanitizer

```bash
# 检查文件是否包含敏感数据
./scripts/sanitizer.sh --check session.json --verbose

# 脱敏处理
./scripts/sanitizer.sh --file session.json > sanitized.json

# 管道使用
cat conversation.txt | ./scripts/sanitizer.sh --stdin > clean.txt
```

### 支持的敏感模式

| 类别 | 示例 |
|-----|------|
| 凭证 | Password, API Key, Secret Token |
| 云服务 | AWS Key, GitHub Token, OpenAI Key |
| 数据库 | MongoDB URI, PostgreSQL URI |
| 个人信息 | Email, Credit Card, SSN |
| 加密货币 | Bitcoin, Ethereum 地址 |
| 证书 | JWT, RSA Key, SSH Key |

---

## 📝 审计日志安全

### 日志位置
```
~/.openclaw-guardian/sessions/Operate_Audit.log
```

### 权限设置
```bash
chmod 700 ~/.openclaw-guardian/sessions
chmod 600 ~/.openclaw-guardian/sessions/Operate_Audit.log
```

### 自动清理（30天保留）

加固脚本会自动添加 cron 任务：
```cron
0 0 * * * find ~/.openclaw-guardian/sessions -name '*.log' -mtime +30 -delete
```

### 手动验证
```bash
# 查看日志大小
ls -lh ~/.openclaw-guardian/sessions/Operate_Audit.log

# 查看最近记录
tail -20 ~/.openclaw-guardian/sessions/Operate_Audit.log

# 检查权限
ls -la ~/.openclaw-guardian/sessions/
```

---

## 🔄 恢复默认配置

```bash
cd ~/.openclaw-guardian/scripts
./deploy-secure.sh --restore
```

---

## 🎯 安全最佳实践

### 1. 最小权限原则
```bash
# 配置文件权限
chmod 600 ~/.openclaw-guardian/config.json

# 目录权限
chmod 700 ~/.openclaw-guardian
chmod 700 ~/.openclaw-guardian/sessions
```

### 2. 定期审计
```bash
# 检查是否有异常拒绝
python3 ~/.openclaw-guardian/scripts/policy_config.py stats

# 查看高风险操作记录
grep "🔴 CRITICAL" ~/.openclaw-guardian/sessions/Operate_Audit.log
```

### 3. 会话管理
- **会话批准**：MEDIUM/HIGH 风险可一次性批准会话内同类操作
- **超时机制**：30分钟无活动自动过期
- **CRITICAL 操作**：始终单独确认，不纳入会话批准

### 4. 网络隔离（高级）
如果担心 LLM 数据外泄：
```bash
# 配置本地 Ollama 作为决策后端
export GUARDIAN_MODEL_PROVIDER=ollama://localhost:11434
```

---

## 📊 安全对比

| 安全指标 | 默认模式 | 加固模式 | 提升 |
|---------|---------|---------|------|
| 人机回环覆盖率 | 40% | **100%** | +60% |
| 危险命令拦截 | 5种 | **20+种** | +300% |
| 敏感目录保护 | 4个 | **10+个** | +150% |
| 自动允许操作 | LOW + MEDIUM | **无** | -100% |
| 确认超时 | 无 | **5分钟** | 新增 |
| 日志保留 | 永久 | **30天** | 合规 |

---

## ⚠️ 重要提示

### 安全权衡

加固模式提供了最高安全性，但会带来以下影响：

1. **操作效率**：所有操作都需要确认，可能降低工作效率
2. **用户体验**：频繁的确认提示可能显得繁琐
3. **误报可能**：某些安全操作也可能被拦截

### 建议

- **日常使用**：标准模式 + 关键目录保护
- **处理敏感数据**：加固模式
- **团队协作**：统一加固配置，避免安全配置漂移

---

## 🔗 相关文件

| 文件 | 用途 |
|-----|------|
| `config/config.hardened.json` | 加固配置文件 |
| `scripts/deploy-secure.sh` | 一键加固部署 |
| `scripts/sanitizer.sh` | 对话脱敏工具 |
| `sessions/Operate_Audit.log` | 审计日志 |

---

## 🆘 故障排除

### 问题：所有操作都被拒绝
```bash
# 检查模式设置
python3 ~/.openclaw-guardian/scripts/policy_config.py mode

# 如果是 emergency 模式，切换回 standard
python3 ~/.openclaw-guardian/scripts/policy_config.py mode standard
```

### 问题：日志文件过大
```bash
# 手动清理旧日志
find ~/.openclaw-guardian/sessions -name '*.log' -mtime +7 -delete
```

### 问题：恢复误删的配置
```bash
# 查看备份列表
ls -la ~/.openclaw-guardian/backups/

# 手动恢复
cp ~/.openclaw-guardian/backups/config.json.bak.XXXX ~/.openclaw-guardian/config.json
```

---

**Last Updated**: 2026-03-12  
**Version**: v0.1.0-hardened
