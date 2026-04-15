# code-security-rules

公司代码生成与运维安全规范 Skill，确保 OpenClaw 在生成或修改代码、脚本、配置时自动遵守安全与运维最佳实践。

## 文件结构

```
code-security-rules/
├── SKILL.md        # 核心规则（16 章 ~256 行，AI 自动加载）
├── reference.md    # 完整代码模板和脚本示例（17 类模板，AI 按需读取）
└── README.md       # 本文件
```

**设计原则**：SKILL.md 控制在 500 行以内，只放规则条款和简短示例，避免占用过多上下文配额。详细的代码模板和脚本放在 `reference.md`，AI 需要时可按需读取。

## 覆盖范围（16 章安全规则）

以下与 SKILL.md 的章节一一对应：

| 章节 | 维度 | 核心内容 |
|------|------|----------|
| 一 | **凭证管理**（最高优先级） | 禁止硬编码密钥/Token/密码，强制环境变量或 Secrets Manager 注入，Python/Bash 双模板 |
| 二 | **Python 安全** | 禁止 eval/exec/pickle.loads/\_\_import\_\_，强制 HTTPS + verify=True + timeout，路径防逃逸 |
| 三 | **Shell 脚本规范** | `set -euo pipefail`、变量双引号保护、flock 防并发、trap 清理、--dry-run |
| 四 | **服务管理与进程安全** | systemd 管理（禁止 nohup/screen），非 root 专用用户，优雅停机，禁止擅自启用数据中间件 |
| 五 | **网络与防火墙安全** | 管理端口走 SSH 隧道禁止公网暴露，数据库端口（3306/5432/6379/27017）禁止对外 |
| 六 | **监控、告警与日志** | 日志时间戳+级别+来源，logrotate 在线保留 30 天，日志禁止输出完整密钥和内部 IP |
| 七 | **变更管控与发布安全** | 变更评审→备份→执行→验证→回滚预案，--dry-run，灰度发布，二次确认 |
| 八 | **SSH 与远程访问安全** | 强制密钥认证，禁用密码登录，ConnectTimeout + ServerAliveInterval，禁止 sshpass |
| 九 | **Docker / 容器安全** | 非 root 运行，锁定镜像版本（禁止 latest），多阶段构建，HEALTHCHECK |
| 十 | **Cron 与定时任务安全** | 任务封装为独立脚本，flock 防重，日志重定向，环境变量显式设置 |
| 十一 | **Skill 配置安全** | name/description 规范，最小权限声明，禁止提示词注入和真实密钥 |
| 十二 | **供应链安全** | Python 哈希校验 + pip-audit + 7 天冷却期；Node.js npm ci + ignore-scripts + 冷却期 |
| 十三 | **数据库操作安全** | 参数化查询，禁止 SQL 拼接，连接串走环境变量，事务 + 连接池 |
| 十四 | **自检清单** | 每次生成代码自动附带 13 项安全运维检查结果 |
| 十五 | **拒绝策略** | 自动拒绝不安全需求（硬编码密钥、eval、chmod 777 等）并给出替代方案 |
| 十六 | **运行环境上下文** | AWS EC2/Lightsail，沙箱隔离，多实例管理，防火墙白名单，Token 轮换 |

## 代码模板参考（reference.md）

`reference.md` 提供了 17 类可直接复用的安全代码模板：

| # | 模板 | 说明 |
|---|------|------|
| 1 | 凭证读取模板 | Python `os.environ.get()` + Bash `${VAR:?}` |
| 2 | 安全 HTTP 请求 | requests + HTTPS + verify + timeout + 异常处理 |
| 3 | 安全文件路径 | `os.path.abspath()` 防路径逃逸 |
| 4 | Shell 脚本通用模板 | set -euo pipefail + log + flock + trap + --dry-run |
| 5 | systemd unit 模板 | 非 root + Restart + ProtectSystem + ReadWritePaths |
| 6 | 防火墙配置模板 | ufw 白名单 CIDR + 最小端口 |
| 7 | 健康检查脚本 | 服务端口 + 磁盘 + 证书过期检测 |
| 8 | logrotate 配置 | daily + rotate 30 + compress |
| 9 | 安全巡检脚本 | 异常进程/挖矿/cron 篡改/authorized_keys 变动检测 |
| 10 | 部署脚本模板 | 前置检查→备份→部署→回滚，支持 --dry-run |
| 11 | 备份脚本模板 | 定时备份 + 保留策略 + 失败告警 |
| 12 | SSH 远程执行模板 | Python subprocess + 超时 + StrictHostKeyChecking |
| 13 | Dockerfile 模板 | 多阶段构建 + 非 root + HEALTHCHECK |
| 14 | Python 依赖安全安装 | uv pip compile --generate-hashes + pip-audit |
| 15 | Node.js 依赖安全安装 | .npmrc 安全配置 + npm ci + npm audit signatures |
| 16 | 参数化查询模板 | psycopg2 参数化 + 反面示例 |
| 17 | 供应链安全自检清单 | 10 项依赖引入检查项 |

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

## 常见问题

### 是不是每次对话都会加载？

不一定。取决于 SKILL.md 中 `description` 的关键词与你的对话是否匹配：

- "帮我写个 Python 脚本调 API" → **匹配** → 会加载（预期行为）
- "今天天气怎么样" → **不匹配** → 不会加载（不浪费 tokens）

当前 `description` 覆盖了 writing Python code、creating Bash scripts、editing source files 等关键词，大部分写代码的对话都会触发加载。这是有意为之——所有代码生成都应受安全规范约束。

### 开销大不大？

| 项目 | 数据 |
|------|------|
| 每轮额外消耗 | ~5K tokens（约 $0.01–0.03，取决于模型） |
| 触发条件 | 仅代码相关对话，日常聊天不受影响 |
| 投入产出比 | 一次生产密钥泄露的损失 >> 此 Skill 运行一整年的 token 消耗 |

如需降低消耗，可缩窄 `description` 关键词（如仅限"安全审查"或"部署脚本"），但会降低防护覆盖面，不建议。
