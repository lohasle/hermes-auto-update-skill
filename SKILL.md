---
name: hermes-auto-update
version: 1.1.0
description: 本机部署 hermes-agent + hermes-webui（不用 Docker），并配置每日自动升级 cron job。所有路径自动探测，零人工配置。
category: devops
triggers:
  - 部署 hermes
  - 安装 hermes-webui
  - hermes 自动更新
  - hermes 升级
  - dashboard 重启
---

# Hermes Auto Update Skill

本机部署 hermes-agent + hermes-webui + dashboard，每日自动升级。不用 Docker。

## 前置条件

- macOS 或 Linux 系统
- Python 3.11+
- 网络可访问 github.com



## 关键路径（自动探测）

```bash
USER_NAME=$(whoami)
USER_HOME=$(eval echo ~${USER_NAME})
HERMES_BASE="${USER_HOME}/.hermes"

if [ -n "${HERMES_HOME:-}" ]; then
    HERMES_HOME="${HERMES_HOME}"
elif [ -d "${HERMES_BASE}/profiles" ] && [ "$(ls -d ${HERMES_BASE}/profiles/*/ 2>/dev/null | head -1)" != "" ]; then
    HERMES_HOME=$(ls -d ${HERMES_BASE}/profiles/*/ | head -1 | sed 's:/$::')
else
    HERMES_HOME="${HERMES_BASE}"
fi

HERMES_AGENT_DIR="${HERMES_BASE}/hermes-agent"
HERMES_WEBUI_DIR="${USER_HOME}/hermes-webui"
VENV_PYTHON="${HERMES_AGENT_DIR}/venv/bin/python3"
HERMES_CLI="${HERMES_AGENT_DIR}/venv/bin/hermes"
```

## 组件架构

| 组件 | 端口 | 说明 |
|------|------|------|
| hermes-webui | 8787 | 第三方 Web UI（nesquena/hermes-webui），launchd 管理 |
| hermes dashboard | 9119 | agent 内置 dashboard（`hermes dashboard`，FastAPI+uvicorn） |

升级 agent 后，dashboard 必须重启才能生效。

## 执行步骤

### Step 1: 探测环境

```bash
USER_NAME=$(whoami)
USER_HOME=$(eval echo ~${USER_NAME})
HERMES_BASE="${USER_HOME}/.hermes"

if [ -n "${HERMES_HOME:-}" ]; then
    HERMES_HOME="${HERMES_HOME}"
elif [ -d "${HERMES_BASE}/profiles" ] && [ "$(ls -d ${HERMES_BASE}/profiles/*/ 2>/dev/null | head -1)" != "" ]; then
    HERMES_HOME=$(ls -d ${HERMES_BASE}/profiles/*/ | head -1 | sed 's:/$::')
else
    HERMES_HOME="${HERMES_BASE}"
fi

HERMES_AGENT_DIR="${HERMES_BASE}/hermes-agent"
HERMES_WEBUI_DIR="${USER_HOME}/hermes-webui"
VENV_PYTHON="${HERMES_AGENT_DIR}/venv/bin/python3"
```

### Step 2: 安装 hermes-agent

```bash
if [ ! -d "${HERMES_AGENT_DIR}" ]; then
    mkdir -p "${HERMES_BASE}"
    git clone https://github.com/NousResearch/hermes-agent.git "${HERMES_AGENT_DIR}"
fi

if [ ! -f "${VENV_PYTHON}" ]; then
    cd "${HERMES_AGENT_DIR}"
    python3 -m venv venv
    source venv/bin/activate
    pip install -e .
    mkdir -p "${USER_HOME}/.local/bin"
    ln -sf "${HERMES_AGENT_DIR}/venv/bin/hermes" "${USER_HOME}/.local/bin/hermes"
fi
```

验证：

```bash
${VENV_PYTHON} -c "from cron.jobs import list_jobs; print('cron OK')"
${VENV_PYTHON} -c "from tools.skills_tool import skills_list; print('tools OK')"
```

两个都必须输出 OK。

### Step 3: 安装 hermes-webui

```bash
if [ ! -d "${HERMES_WEBUI_DIR}" ]; then
    git clone https://github.com/nesquena/hermes-webui.git "${HERMES_WEBUI_DIR}"
fi
```

### Step 4: 创建 macOS launchd 服务（替代 systemd）

plist 文件: ~/Library/LaunchAgents/com.hermes.webui.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermes.webui</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/lohas/.hermes/hermes-agent/venv/bin/python3</string>
        <string>server.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/lohas/hermes-webui</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>HERMES_HOME</key>
        <string>/Users/lohas/.hermes</string>
        <key>HERMES_WEBUI_HOST</key>
        <string>0.0.0.0</string>
        <key>HERMES_WEBUI_PORT</key>
        <string>8787</string>
        <key>HERMES_WEBUI_STATE_DIR</key>
        <string>/Users/lohas/.hermes/webui</string>
        <key>HERMES_WEBUI_AGENT_DIR</key>
        <string>/Users/lohas/.hermes/hermes-agent</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/lohas/.hermes/logs/webui-stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/lohas/.hermes/logs/webui-stderr.log</string>
</dict>
</plist>
```

```bash
mkdir -p ~/.hermes/logs ~/.hermes/webui
launchctl load ~/Library/LaunchAgents/com.hermes.webui.plist
```

### Step 5: 验证

```bash
# 验证 webui
curl -s http://localhost:8787/api/crons    # 必须返回 {"jobs": [...]}
curl -s http://localhost:8787/api/skills   # 必须返回 {"skills": [...]}

# 验证 dashboard（如已启动）
curl -s http://localhost:9119/health
```

### Step 6: 创建每日自动升级 cron job

用 hermes cronjob 工具创建：

**hermes-webui-daily-upgrade**（计划: 0 3 * * *）：

```
Check for hermes-webui updates and upgrade if available. Auto-detect all paths.

IMPORTANT git identity rules:
- For github.com/lohasle/* repos: user.name=lohasle, user.email=lohasle@users.noreply.github.com
  ABSOLUTELY NO @vanke or @onewo in any commit metadata.
- For github.com/onewo-gesc/* repos: user.name=ful04_onewo, user.email=ful04_onewo@onewo.com
  NEVER use lohasle identity for onewo-gesc repos.

Steps:
1. Locate webui repo: find ~ -maxdepth 2 -name "hermes-webui" -type d, or check ~/hermes-webui
2. cd into it, detect branch: git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
3. git fetch origin <branch>
4. git log HEAD..origin/<branch> --oneline — if empty, report "already up to date" and stop
5. If new commits: git stash, git pull origin <branch>
6. Restart webui: launchctl unload ~/Library/LaunchAgents/com.hermes.webui.plist && launchctl load ~/Library/LaunchAgents/com.hermes.webui.plist
7. Wait 3 seconds, then verify: curl -sf http://localhost:8787/api/crons && curl -sf http://localhost:8787/api/skills
8. Report summary
```

**hermes-agent-daily-upgrade**（计划: 0 3 * * *）：

```
Check for hermes-agent updates and upgrade if available. Auto-detect all paths.

IMPORTANT git identity rules:
- For github.com/lohasle/* repos: user.name=lohasle, user.email=lohasle@users.noreply.github.com
  ABSOLUTELY NO @vanke or @onewo in any commit metadata.
- For github.com/onewo-gesc/* repos: user.name=ful04_onewo, user.email=ful04_onewo@onewo.com
  NEVER use lohasle identity for onewo-gesc repos.

Steps:
1. Locate agent repo: ls ~/.hermes/hermes-agent/.git, default ~/.hermes/hermes-agent
2. cd into it, detect branch: git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
3. git fetch origin <branch>
4. git log HEAD..origin/<branch> --oneline — if empty, report "already up to date" and stop
5. If new commits: git stash, git pull origin <branch>
6. Reinstall: source venv/bin/activate && pip install -e .
7. Restart hermes dashboard if running: pkill -f "hermes_cli.web_server" || true; sleep 1; nohup ~/.hermes/hermes-agent/venv/bin/hermes dashboard --no-open --insecure > ~/.hermes/logs/dashboard.log 2>&1 &
8. Wait 3 seconds, then verify: curl -sf http://localhost:8787/api/crons && curl -sf http://localhost:8787/api/skills
9. Report summary
```

## 升级后必须执行

1. **重启 hermes-webui** — launchctl unload + load
2. **重启 hermes dashboard** — 杀掉旧进程后重新 `hermes dashboard`
3. **验证 API** — curl /api/crons 和 /api/skills 确认正常
4. **Push 代码** — 如有修改，按 git 身份规则提交到 GitHub

## 故障排查

| 症状 | 原因 | 修复 |
|------|------|------|
| /api/crons 500 ModuleNotFoundError: cron | webui 没用 agent venv 的 python | ProgramArguments 必须指向 agent venv 的 python3 |
| /api/skills 500 ModuleNotFoundError: tools | 同上 | 同上 |
| dashboard 页面空白/旧版本 | 升级 agent 后未重启 dashboard | 杀掉并重新启动 `hermes dashboard` |
| 打开 webui 要重新 setup | STATE_DIR 不对 | 确保 state dir 在 HERMES_HOME 下 |
| 局域网无法访问 | 绑定 127.0.0.1 | HERMES_WEBUI_HOST=0.0.0.0 |
| git pull 冲突 | 本地有未提交修改 | 升级脚本先 git stash |

## 注意事项

- 不用 Docker，Docker 会导致数据丢失和模块缺失
- 必须用 agent venv 的 python 跑 webui，否则 cron/tools 模块不可用
- HERMES_WEBUI_AGENT_DIR 必须设置
- 分支名不要硬编码，用 git remote show origin 动态获取
- macOS 用 launchd 替代 systemd
- 升级 agent 后必须重启 dashboard，否则前端显示的还是旧版
