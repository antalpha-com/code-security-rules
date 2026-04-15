# Code Security & Ops Rules — 完整模板与脚本参考

> 本文件包含 SKILL.md 中各规则条款对应的完整代码模板和脚本示例。
> SKILL.md 定义"做什么"，本文件定义"怎么做"。

---

## 一、凭证读取模板

### Python

```python
import os
import sys

def get_required_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        print(f"错误：环境变量 {name} 未设置。请联系运维配置。", file=sys.stderr)
        sys.exit(1)
    return value

API_KEY = get_required_env("MY_SERVICE_API_KEY")
```

### Bash

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${MY_API_KEY:?错误：环境变量 MY_API_KEY 未设置，请联系运维配置}"
: "${DB_PASSWORD:?错误：环境变量 DB_PASSWORD 未设置}"
```

---

## 二、安全 HTTP 请求模板

```python
import requests
import sys

def safe_api_call(url: str, api_key: str, payload: dict) -> dict:
    headers = {"Authorization": f"Bearer {api_key}"}
    try:
        resp = requests.post(
            url, json=payload, headers=headers,
            timeout=30, verify=True
        )
        resp.raise_for_status()
        return resp.json()
    except requests.exceptions.RequestException as e:
        print(f"API 调用失败: {e}", file=sys.stderr)
        raise
```

---

## 三、安全文件路径模板

```python
import os

SAFE_BASE_DIR = os.path.abspath("./data")

def safe_file_path(user_input_filename: str) -> str:
    clean_name = os.path.basename(user_input_filename)
    full_path = os.path.abspath(os.path.join(SAFE_BASE_DIR, clean_name))
    if not full_path.startswith(SAFE_BASE_DIR):
        raise ValueError("非法文件路径")
    return full_path
```

---

## 四、Shell 脚本通用模板

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="/var/log/$(basename "$0" .sh).log"
LOCK_FILE="/tmp/$(basename "$0" .sh).lock"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
die() { log "FATAL: $*"; exit 1; }

usage() {
    cat <<EOF
Usage: $(basename "$0") [OPTIONS]
Options:
  --dry-run    只检查不执行
  --help       显示帮助
EOF
}

cleanup() {
    rm -f "$LOCK_FILE"
    log "清理完成"
}
trap cleanup EXIT

exec 200>"$LOCK_FILE"
flock -n 200 || die "另一个实例正在运行"

: "${REQUIRED_VAR:?错误：环境变量 REQUIRED_VAR 未设置}"

log "开始执行..."
```

---

## 五、systemd unit 模板

```ini
[Unit]
Description=MyService
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=myservice
Group=myservice
WorkingDirectory=/opt/myservice
ExecStartPre=/opt/myservice/check-config.sh
ExecStart=/opt/myservice/run.sh
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s
StandardOutput=journal
StandardError=journal
Environment=LANG=en_US.UTF-8
ProtectSystem=strict
ProtectHome=yes
NoNewPrivileges=yes
ReadWritePaths=/opt/myservice/data /var/log/myservice

[Install]
WantedBy=multi-user.target
```

---

## 六、防火墙配置模板

```bash
#!/usr/bin/env bash
set -euo pipefail

ALLOWED_CIDRS=("1.2.3.0/24" "5.6.7.8/32")

ufw --force reset
ufw default deny incoming
ufw default allow outgoing

for cidr in "${ALLOWED_CIDRS[@]}"; do
    ufw allow from "$cidr" to any port 22 proto tcp
    ufw allow from "$cidr" to any port 80 proto tcp
    ufw allow from "$cidr" to any port 443 proto tcp
done

ufw --force enable
ufw status verbose
```

---

## 七、健康检查脚本模板

```bash
#!/usr/bin/env bash
set -euo pipefail

check_service() {
    local name="$1" port="$2"
    if ss -tlnp | grep -q ":${port} "; then
        echo "[OK] ${name} listening on :${port}"
    else
        echo "[CRITICAL] ${name} NOT listening on :${port}" >&2
        return 1
    fi
}

check_disk() {
    local threshold="${1:-80}"
    local usage
    usage=$(df / --output=pcent | tail -1 | tr -d ' %')
    if [ "$usage" -ge "$threshold" ]; then
        echo "[WARN] 磁盘使用率 ${usage}% >= ${threshold}%" >&2
        return 1
    fi
    echo "[OK] 磁盘使用率 ${usage}%"
}

check_cert_expiry() {
    local domain="$1" days_warn="${2:-30}"
    local expiry
    expiry=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
        | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
    if [ -z "$expiry" ]; then
        echo "[CRITICAL] 无法获取 ${domain} 证书信息" >&2
        return 1
    fi
    local expiry_epoch now_epoch days_left
    expiry_epoch=$(date -d "$expiry" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$expiry" +%s)
    now_epoch=$(date +%s)
    days_left=$(( (expiry_epoch - now_epoch) / 86400 ))
    if [ "$days_left" -le "$days_warn" ]; then
        echo "[WARN] ${domain} 证书将在 ${days_left} 天后过期" >&2
        return 1
    fi
    echo "[OK] ${domain} 证书剩余 ${days_left} 天"
}

ERRORS=0
check_service "openclaw-gateway" "443" || ((ERRORS++))
check_disk 80 || ((ERRORS++))
check_cert_expiry "your-domain.com" 30 || ((ERRORS++))

if [ "$ERRORS" -gt 0 ]; then
    echo "共 ${ERRORS} 项异常" >&2
    exit 1
fi
echo "所有检查通过"
```

---

## 八、logrotate 配置模板

```
/var/log/myservice/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0644 myservice myservice
    postrotate
        systemctl reload myservice 2>/dev/null || true
    endscript
}
```

---

## 九、安全巡检脚本模板

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="/var/log/security-audit.log"
ALERT_WEBHOOK="${ALERT_WEBHOOK_URL:-}"
BASELINE_DIR="/opt/security-baseline"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

alert() {
    local level="$1" msg="$2"
    log "[$level] $msg"
    if [ -n "$ALERT_WEBHOOK" ] && [ "$level" = "CRITICAL" ]; then
        curl -sf -X POST "$ALERT_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d "{\"text\":\"🚨 安全告警: ${msg}\"}" \
            --timeout 10 || true
    fi
}

ERRORS=0

# --- 可疑进程检查 ---
log "检查可疑进程..."
KNOWN_USERS="root|ubuntu|www-data|openclaw|systemd|nobody|daemon|syslog|messagebus"
SUSPECT_PROCS=$(ps aux | awk 'NR>1 {print $1, $11}' | grep -vE "^($KNOWN_USERS) " | grep -vE '^\[' || true)
if [ -n "$SUSPECT_PROCS" ]; then
    alert "WARN" "发现非预期用户的进程: $(echo "$SUSPECT_PROCS" | head -5)"
    ((ERRORS++))
fi

# --- /tmp /dev/shm 可执行文件检查 ---
log "检查临时目录可执行文件..."
TMP_EXECS=$(find /tmp /dev/shm /var/tmp -maxdepth 2 -type f -executable 2>/dev/null || true)
if [ -n "$TMP_EXECS" ]; then
    alert "CRITICAL" "临时目录发现可执行文件: $(echo "$TMP_EXECS" | head -5)"
    ((ERRORS++))
fi

# --- 挖矿进程检查 ---
log "检查挖矿进程..."
MINING_PATTERNS="xmrig|kdevtmpfsi|kinsing|cryptonight|minerd|cpuminer"
MINING_PROCS=$(ps aux | grep -iE "$MINING_PATTERNS" | grep -v grep || true)
if [ -n "$MINING_PROCS" ]; then
    alert "CRITICAL" "疑似挖矿进程: $(echo "$MINING_PROCS" | head -3)"
    ((ERRORS++))
fi

# --- 异常网络监听端口 ---
log "检查网络监听端口..."
KNOWN_PORTS="22|80|443|18789"
UNKNOWN_LISTENERS=$(ss -tlnp | awk 'NR>1 {print $4}' | grep -oE '[0-9]+$' | grep -vE "^($KNOWN_PORTS)$" || true)
if [ -n "$UNKNOWN_LISTENERS" ]; then
    alert "WARN" "发现非预期监听端口: $UNKNOWN_LISTENERS"
    ((ERRORS++))
fi

# --- 异常外连检查（矿池端口等） ---
log "检查异常外连..."
SUSPECT_PORTS="3333|4444|5555|9999"
SUSPECT_CONNS=$(ss -tnp | grep -E "ESTAB" | awk '{print $5}' | grep -E ":($SUSPECT_PORTS)$" || true)
if [ -n "$SUSPECT_CONNS" ]; then
    alert "CRITICAL" "发现可疑外连（疑似矿池）: $SUSPECT_CONNS"
    ((ERRORS++))
fi

# --- 异常 cron 任务检查 ---
log "检查 cron 任务..."
CRON_HASH_FILE="${BASELINE_DIR}/cron.hash"
CURRENT_CRON_HASH=$(crontab -l 2>/dev/null | sha256sum | awk '{print $1}')
if [ -f "$CRON_HASH_FILE" ]; then
    BASELINE_HASH=$(cat "$CRON_HASH_FILE")
    if [ "$CURRENT_CRON_HASH" != "$BASELINE_HASH" ]; then
        alert "WARN" "crontab 发生变化（与基线不一致）"
        ((ERRORS++))
    fi
else
    mkdir -p "$BASELINE_DIR"
    echo "$CURRENT_CRON_HASH" > "$CRON_HASH_FILE"
    log "[INFO] 首次运行，已生成 cron 基线"
fi

# --- authorized_keys 检查 ---
log "检查 authorized_keys..."
AK_HASH_FILE="${BASELINE_DIR}/authorized_keys.hash"
AK_FILE="$HOME/.ssh/authorized_keys"
if [ -f "$AK_FILE" ]; then
    CURRENT_AK_HASH=$(sha256sum "$AK_FILE" | awk '{print $1}')
    if [ -f "$AK_HASH_FILE" ]; then
        BASELINE_AK=$(cat "$AK_HASH_FILE")
        if [ "$CURRENT_AK_HASH" != "$BASELINE_AK" ]; then
            alert "CRITICAL" "authorized_keys 被修改（与基线不一致）"
            ((ERRORS++))
        fi
    else
        echo "$CURRENT_AK_HASH" > "$AK_HASH_FILE"
        log "[INFO] 首次运行，已生成 authorized_keys 基线"
    fi
fi

# --- /etc/passwd 用户变动检查 ---
log "检查系统用户变动..."
PASSWD_HASH_FILE="${BASELINE_DIR}/passwd.hash"
CURRENT_PASSWD_HASH=$(sha256sum /etc/passwd | awk '{print $1}')
if [ -f "$PASSWD_HASH_FILE" ]; then
    BASELINE_PASSWD=$(cat "$PASSWD_HASH_FILE")
    if [ "$CURRENT_PASSWD_HASH" != "$BASELINE_PASSWD" ]; then
        alert "WARN" "/etc/passwd 发生变化，可能有新增用户"
        ((ERRORS++))
    fi
else
    echo "$CURRENT_PASSWD_HASH" > "$PASSWD_HASH_FILE"
    log "[INFO] 首次运行，已生成 passwd 基线"
fi

# --- 汇总 ---
if [ "$ERRORS" -gt 0 ]; then
    log "巡检完成：发现 ${ERRORS} 项异常"
    exit 1
fi
log "巡检完成：所有检查通过"
```

---

## 十、部署脚本模板

```bash
#!/usr/bin/env bash
set -euo pipefail

DRY_RUN=false
[[ "${1:-}" == "--dry-run" ]] && DRY_RUN=true

BACKUP_DIR="/opt/backups/$(date +%Y%m%d_%H%M%S)"
SERVICE="myservice"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
die() { log "FATAL: $*"; exit 1; }

pre_check() {
    log "前置检查..."
    systemctl is-active --quiet "$SERVICE" || die "$SERVICE 当前未运行，请先排查"
    local disk_usage
    disk_usage=$(df /opt --output=pcent | tail -1 | tr -d ' %')
    [ "$disk_usage" -lt 90 ] || die "磁盘使用率 ${disk_usage}%，空间不足"
    log "前置检查通过"
}

backup() {
    log "备份到 $BACKUP_DIR ..."
    if $DRY_RUN; then
        log "[DRY-RUN] 将备份 /opt/$SERVICE/config/ 到 $BACKUP_DIR"
        return
    fi
    mkdir -p "$BACKUP_DIR"
    cp -a "/opt/$SERVICE/config/" "$BACKUP_DIR/"
    log "备份完成"
}

deploy() {
    if $DRY_RUN; then
        log "[DRY-RUN] 将执行部署步骤..."
        return
    fi
    log "执行部署..."
    systemctl restart "$SERVICE"
    sleep 3
    systemctl is-active --quiet "$SERVICE" || die "部署后服务未启动，请检查日志"
    log "部署完成"
}

rollback() {
    log "回滚到 $BACKUP_DIR ..."
    cp -a "$BACKUP_DIR/config/" "/opt/$SERVICE/config/"
    systemctl restart "$SERVICE"
    log "回滚完成"
}

pre_check
backup
deploy
log "发布成功。如需回滚: $0 --rollback $BACKUP_DIR"
```

---

## 十一、备份脚本模板

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_BASE="/opt/backups"
DATE_TAG=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_BASE}/${DATE_TAG}"
RETENTION_DAYS=7
ALERT_WEBHOOK="${ALERT_WEBHOOK_URL:?错误：ALERT_WEBHOOK_URL 未设置}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

alert_failure() {
    local msg="$1"
    curl -sf -X POST "$ALERT_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "{\"text\":\"备份失败: ${msg}\"}" \
        --timeout 10 || true
}

trap 'alert_failure "脚本异常退出 at line $LINENO"' ERR

mkdir -p "$BACKUP_DIR"

log "备份 OpenClaw 状态..."
openclaw backup create --output "$BACKUP_DIR" --verify

log "备份数据库..."
pg_dump -h localhost -U "$DB_USER" "$DB_NAME" \
    | gzip > "${BACKUP_DIR}/db_${DATE_TAG}.sql.gz"

log "清理 ${RETENTION_DAYS} 天前的备份..."
find "$BACKUP_BASE" -maxdepth 1 -type d -mtime +"$RETENTION_DAYS" \
    -exec rm -rf {} + 2>/dev/null || true

log "备份完成: ${BACKUP_DIR}"
```

---

## 十二、SSH 远程执行模板

```python
import subprocess

def ssh_exec(host: str, key_path: str, command: list[str], timeout: int = 60) -> str:
    cmd = [
        "ssh", "-i", key_path,
        "-o", "StrictHostKeyChecking=accept-new",
        "-o", "ConnectTimeout=10",
        "-o", "ServerAliveInterval=15",
        f"ubuntu@{host}",
        "--",
    ] + command
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout)
    if result.returncode != 0:
        raise RuntimeError(f"SSH 命令失败: {result.stderr.strip()}")
    return result.stdout.strip()
```

---

## 十三、Dockerfile 模板

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/deps -r requirements.txt

FROM python:3.12-slim
RUN groupadd -r app && useradd -r -g app -d /app -s /sbin/nologin app
WORKDIR /app
COPY --from=builder /deps /usr/local/lib/python3.12/site-packages
COPY --chown=app:app . .
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"
ENTRYPOINT ["python", "main.py"]
```

---

## 十四、Python 依赖安全安装流程

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. 生成带哈希的锁文件（开发环境执行一次）
uv pip compile --generate-hashes requirements.in -o requirements.txt

# 2. 漏洞扫描
pip-audit -r requirements.txt --fail-on-vuln

# 3. 带哈希校验的安装
uv pip install --require-hashes -r requirements.txt

# 4. 可选：生成 SBOM（软件物料清单）
cyclonedx-py environment -o sbom.json --format json
```

---

## 十五、Node.js 依赖安全安装流程

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. 确保 .npmrc 安全配置
cat > .npmrc <<'EOF'
ignore-scripts=true
min-release-age=7d
audit-level=high
EOF

# 2. 严格按锁文件安装
npm ci --ignore-scripts

# 3. 漏洞扫描
npm audit --audit-level=high

# 4. 签名验证
npm audit signatures

# 5. 按需为特定包运行脚本
npx --package=esbuild -- esbuild --version
```

---

## 十六、参数化查询模板

```python
import psycopg2

def safe_query(conn, user_id: int) -> list:
    with conn.cursor() as cur:
        cur.execute(
            "SELECT id, name FROM users WHERE id = %s LIMIT 100",
            (user_id,)
        )
        return cur.fetchall()
```

### 反面示例（绝对禁止）

```python
# SQL 注入风险
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")

# 无 WHERE 的删除
cursor.execute("DELETE FROM orders")

# 硬编码连接串
conn = psycopg2.connect("host=10.0.0.1 dbname=prod user=root password=123456")
```

---

## 十七、供应链安全自检清单

在引入新依赖或更新依赖版本时，必须完成以下检查：

```
# ═══ 供应链安全自检 ═══
# [✓/✗] 包名拼写正确，非仿冒包（typosquatting）
# [✓/✗] 包来自官方源（PyPI / npmjs.com）
# [✓/✗] 版本号精确锁定（无 ^、~、>=、*）
# [✓/✗] 锁文件已生成并提交到 Git
# [✓/✗] Python: requirements.txt 包含 --hash（哈希校验）
# [✓/✗] Node.js: package-lock.json 已提交且使用 npm ci
# [✓/✗] 漏洞扫描已通过（pip-audit / npm audit 无 HIGH/CRITICAL）
# [✓/✗] 包的发布时间 > 7 天（冷却期）
# [✓/✗] 包有可审计的 GitHub 仓库、近期有维护
# [✓/✗] 包的下载量合理（非极低下载量的不明包）
```
