---
name: hermes-auto-update
version: 1.0.0
description: 本机部署 hermes-agent + hermes-webui（不用 Docker），并配置每日自动升级 cron job。所有路径自动探测，零人工配置。
category: devops
triggers:
  - 部署 hermes
  - 安装 hermes-webui
  - hermes 自动更新
  - hermes 升级
---

# Hermes Auto Update Skill

本机部署 hermes-agent + hermes-webui，每日自动升级。不用 Docker。

## 前置条件

- Linux 系统
- Python 3.11+
- sudo 权限（创建 systemd 服务）
- 网络可访问 github.com

## 执行步骤

### Step 1: 探测环境

```bash
USER_NAME=$(whoami)
USER_HOME=$(eval echo ~${USER_NAME})
HERMES_BASE="${USER_HOME}/.hermes"

# 探测 HERMES_HOME
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
# 仅在未安装时执行
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

两个都必须输出 OK，否则 `pip install -e .` 未完成。

### Step 3: 安装 hermes-webui

```bash
if [ ! -d "${HERMES_WEBUI_DIR}" ]; then
    git clone https://github.com/nesquena/hermes-webui.git "${HERMES_WEBUI_DIR}"
fi
```

### Step 4: 创建 systemd 服务

```bash
sudo bash -c "cat > /etc/systemd/system/hermes-webui.service << EOF
[Unit]
Description=Hermes Web UI
After=network.target

[Service]
Type=simple
User=${USER_NAME}
Environment=HERMES_HOME=${HERMES_HOME}
Environment=HERMES_WEBUI_HOST=0.0.0.0
Environment=HERMES_WEBUI_PORT=8787
Environment=HERMES_WEBUI_STATE_DIR=${HERMES_HOME}/webui
Environment=HERMES_WEBUI_AGENT_DIR=${HERMES_AGENT_DIR}
WorkingDirectory=${HERMES_WEBUI_DIR}
ExecStart=${VENV_PYTHON} server.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF"

sudo systemctl daemon-reload
sudo systemctl enable hermes-webui
sudo systemctl start hermes-webui
```

### Step 5: 验证

```bash
sudo systemctl status hermes-webui
curl -s http://localhost:8787/api/crons    # 必须返回 {"jobs": [...]}
curl -s http://localhost:8787/api/skills   # 必须返回 {"skills": [...]}
journalctl -u hermes-webui --no-pager -n 30
```

### Step 6: 创建每日自动升级 cron job

用 hermes cronjob 工具创建：

**hermes-webui-daily-upgrade**（计划: 0 3 * * *）：

```
Check for hermes-webui updates and upgrade if available. Auto-detect all paths.

Steps:
1. Locate webui repo: find ~ -maxdepth 2 -name "hermes-webui" -type d, or check ~/hermes-webui
2. cd into it, detect branch: git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
3. git fetch origin <branch>
4. git log HEAD..origin/<branch> --oneline — if empty, report "already up to date" and stop
5. If new commits: git stash, git pull origin <branch>, sudo systemctl restart hermes-webui, report summary
```

**hermes-agent-daily-upgrade**（计划: 0 3 * * *）：

```
Check for hermes-agent updates and upgrade if available. Auto-detect all paths.

Steps:
1. Locate agent repo: ls ~/.hermes/hermes-agent/.git, default ~/.hermes/hermes-agent
2. cd into it, detect branch: git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
3. git fetch origin <branch>
4. git log HEAD..origin/<branch> --oneline — if empty, report "already up to date" and stop
5. If new commits: git stash, git pull origin <branch>, source venv/bin/activate && pip install -e ., report summary
```

## 故障排查

| 症状 | 原因 | 修复 |
|------|------|------|
| /api/crons 500 ModuleNotFoundError: cron | webui 没用 agent venv 的 python | ExecStart 必须指向 agent venv 的 python3 |
| /api/skills 500 ModuleNotFoundError: tools | 同上 | 同上 |
| 打开 webui 要重新 setup | STATE_DIR 不对 | 确保 state dir 在 HERMES_HOME 下 |
| 局域网无法访问 | 绑定 127.0.0.1 | HERMES_WEBUI_HOST=0.0.0.0 |
| git pull 冲突 | 本地有未提交修改 | 升级脚本先 git stash |

## 注意事项

- 不用 Docker，Docker 会导致数据丢失和模块缺失
- 必须用 agent venv 的 python 跑 webui，否则 cron/tools 模块不可用
- HERMES_WEBUI_AGENT_DIR 必须设置
- 分支名不要硬编码，用 git remote show origin 动态获取
