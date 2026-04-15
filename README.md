# code-security-rules

公司代码生成与运维安全规范 Skill，确保 OpenClaw 在生成或修改代码、脚本、配置时自动遵守安全与运维最佳实践。

## 文件结构

```
code-security-rules/
├── SKILL.md        # 核心规则（~325 行，AI 自动加载）
├── reference.md    # 完整代码模板和脚本示例（按需查阅）
└── README.md       # 本文件
```

**设计原则**：SKILL.md 控制在 500 行以内，只放规则条款和简短示例，避免占用过多上下文配额。详细的代码模板和脚本放在 `reference.md`，AI 需要时可按需读取。

## 覆盖范围（19 条规则）

| # | 维度 | 核心内容 |
|---|------|----------|
| 1 | **凭证管理** | 禁止硬编码，强制环境变量，Python/Bash 双模板 |
| 2 | **Python 安全** | 禁止危险函数，HTTPS + timeout，路径防逃逸 |
| 3 | **Shell 脚本** | `set -euo pipefail`、变量保护、flock、trap、dry-run |
| 4 | **服务管理** | systemd，非 root 运行，健康检查，优雅停机 |
| 5 | **网络防火墙** | 最小端口，SSH 白名单，管理端口不公网暴露 |
| 6 | **监控告警** | 健康检查、磁盘/证书监控、日志脱敏与轮转 |
| 7 | **安全巡检** | 异常进程/挖矿/反弹 Shell/cron 篡改/SSH key 变动检测 |
| 8 | **变更管控** | 备份→执行→验证→回滚，dry-run，灰度发布 |
| 9 | **备份恢复** | 定期备份+验证+加密+告警，OpenClaw 状态完整备份 |
| 10 | **SSH 安全** | 密钥认证，参数化远程命令，超时设置 |
| 11 | **Docker 安全** | 非 root，锁定版本，多阶段构建，HEALTHCHECK |
| 12 | **Cron 任务** | 脚本封装，flock 防重，日志重定向 |
| 13 | **Skill 配置** | 最小权限声明，禁止提示词注入 |
| 14 | **供应链安全** | Python 哈希校验+pip-audit+冷却期；npm ci+禁用 postinstall+冷却期 |
| 15 | **数据库安全** | 参数化查询，事务，连接池 |
| 16 | **AWS 安全** | IAM 最小权限，禁硬编码 AK/SK |
| 17 | **自检清单** | 每次生成代码附 13 项安全运维检查 |
| 18 | **拒绝策略** | 自动拒绝不安全需求并给出替代方案 |
| 19 | **环境上下文** | AWS EC2/Lightsail，沙箱，多实例管理 |

## 安装

```bash
# 复制到 OpenClaw 工作区（推荐，沙箱安全）
cp -r code-security-rules/ ~/.openclaw/workspace/skills/code-security-rules/

# 重启 Gateway 生效
openclaw gateway restart
```

## 验证

```bash
openclaw skills list
bash audit.sh ~/.openclaw/workspace/skills/code-security-rules/
```

##一些说明
关键问题：是不是每次都会加载？
不一定。取决于你的 description 写法和 OpenClaw 的路由机制：
如果你问"帮我写个 Python 脚本调 API"→ 匹配到 → 会加载（这是预期的）
如果你问"今天天气怎么样"→ 不匹配 → 不会加载（不浪费）
你当前 description 里关键词很多（writing Python code, creating Bash scripts, editing source files...），这意味着大部分写代码的对话都会触发加载。这是有意为之的——你希望所有代码生成都受安全规范约束。

值不值？
简单算一笔账：
成本：每轮对话多消耗约 5K tokens（约 $0.01-0.03，取决于模型）
收益：每次代码生成自动遵守 19 条安全规范，避免一次凭证泄露或生产事故
一次生产环境的密钥泄露造成的损失，远超这个 Skill 运行一整年的 token 消耗。对安全类 Skill 来说，这个消耗是值得的。
如果你确实想进一步降低消耗，可以把 description 缩窄，比如只在"涉及安全审查"或"写部署脚本"时才触发，而不是所有写代码的场景都触发。但这样会降低防护覆盖面，不太建议。
# code-security-rules
