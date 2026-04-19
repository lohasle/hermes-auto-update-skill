# Hermes & Hermes-WebUI 本机部署 + 每日自动升级

> 给其他 agent 的指令：直接按步骤执行，所有路径自动探测，无需人工配置。

## 核心原则

1. **不用 Docker** — 容器挂载导致数据丢失、cron/tools 模块 500
2. **本机 systemd 运行** — 数据全在 ~/.hermes，永不丢失
3. **复用 hermes-agent 的 venv** — webui 的 cron/tools 模块直接可用
4. **每日自动升级** — cron job 检查更新，有才 pull，不盲目重启

---

## 步骤 1：探测环境

用 terminal 执行以下命令，收集所有路径信息：

```bash
echo "USER_NAME=$(whoami)"
echo "USER_HOME=$(eval echo ~$(whoami))"
echo "HERMES_HOME_ENV=${HERMES_HOME:-UNSET}"
echo "HERMES_BASE=$(eval echo ~$(whoami))/.hermes"
# 探测 profile
if [ -d "$(eval echo ~$(whoami))/.hermes/profiles" ]; then
    FIRST_PROFILE=$(ls -d $(eval echo ~$(whoami))/.hermes/profiles/*/ 2>/dev/null | head -1 | sed 's:/$::')
    echo "FIRST_PROFILE=${FIRST_PROFILE}"
fi
# 探测已有安装
echo "HERMES_AGENT_EXISTS=$([ -d $(eval echo ~$(whoami))/.hermes/hermes-agent ] && echo YES || echo NO)"
echo "HERMES_WEBUI_EXISTS=$([ -d $(eval echo ~$(whoami))/hermes-webui ] && echo YES || echo NO)"
echo "HERMES_VENV_PYTHON=$([ -f $(eval echo ~$(whoami))/.hermes/hermes-agent/venv/bin/python3 ] && echo YES || echo NO)"
# 探测 hermes-agent 的主分支
if [ -d "$(eval echo ~$(whoami))/.hermes/hermes-agent" ]; then
    cd $(eval echo ~$(whoami))/.hermes/hermes-agent && git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'
fi
```

根据输出确定以下变量（后续步骤引用）：

| 变量 | 确定规则 |
|------|----------|
| USER_NAME | whoami 结果 |
| USER_HOME | 用户 home 目录 |
| HERMES_HOME | 优先取环境变量 $HERMES_HOME → 否则取 profiles 下第一个子目录 → 否则用 ~/.hermes |
| HERMES_AGENT_DIR | $USER_HOME/.hermes/hermes-agent |
| HERMES_WEBUI_DIR | $USER_HOME/hermes-webui |
| VENV_PYTHON | $HERMES_AGENT_DIR/venv/bin/python3 |
| AGENT_BRANCH | hermes-agent 仓库 HEAD branch（通常 main 或 master） |

---

## 步骤 2：安装 hermes-agent

仅在 HERMES_AGENT_EXISTS=NO 时执行克隆：

```bash
mkdir -p $USER_HOME/.hermes
git clone https://github.com/NousResearch/hermes-agent.git $USER_HOME/.hermes/hermes-agent
```

仅在 VENV_PYTHON=NO 时执行安装：

```bash
cd $USER_HOME/.hermes/hermes-agent
python3 -m venv venv
source venv/bin/activate
pip install -e .
mkdir -p $USER_HOME/.local/bin
ln -sf $USER_HOME/.hermes/hermes-agent/venv/bin/hermes $USER_HOME/.local/bin/hermes
```

验证：

```bash
$USER_HOME/.hermes/hermes-agent/venv/bin/python3 -c "from cron.jobs import list_jobs; print('cron OK')"
$USER_HOME/.hermes/hermes-agent/venv/bin/python3 -c "from tools.skills_tool import skills_list; print('tools OK')"
```

如果这两个 import 报错，说明 venv 安装不完整，重新 `pip install -e .`。

---

## 步骤 3：安装 hermes-webui

仅在 HERMES_WEBUI_EXISTS=NO 时执行：

```bash
git clone https://github.com/nesquena/hermes-webui.git $USER_HOME/hermes-webui
```

无需额外安装依赖，webui 只依赖 pyyaml，agent venv 里已有。

---

## 步骤 4：创建 systemd 服务

用 terminal（需 sudo）写入：

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
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes-webui
sudo systemctl start hermes-webui
```

---

## 步骤 5：验证

```bash
# 服务状态
sudo systemctl status hermes-webui

# 关键 API（500 = 缺 agent 源码或 venv 不对）
curl -s http://localhost:8787/api/crons
curl -s http://localhost:8787/api/skills

# 日志检查
journalctl -u hermes-webui --no-pager -n 30
```

crons 返回 `{"jobs": [...]}` 且 skills 返回 `{"skills": [...]}` 即成功。

如果 /api/crons 报 ModuleNotFoundError: No module named 'cron'：
- 检查 HERMES_WEBUI_AGENT_DIR 是否正确指向 hermes-agent 源码
- 检查 ExecStart 的 python 是否是 agent venv 里的

如果 /api/skills 报 ModuleNotFoundError: No module named 'tools'：
- 同上，必须用 agent venv 的 python

---

## 步骤 6：配置每日自动升级

使用 Hermes 的 cronjob 工具创建两个定时任务。

### 6.1 hermes-webui 每日升级

```
cronjob action=create
name=hermes-webui-daily-upgrade
schedule=0 3 * * *
deliver=local
prompt=Check for hermes-webui updates and upgrade if available. Auto-detect all paths.

Steps:
1. Run: cd ~ && find . -maxdepth 2 -name "hermes-webui" -type d 2>/dev/null to locate the webui repo. Also check HERMES_WEBUI_DIR env var. Default: ~/hermes-webui
2. cd into the webui repo, detect default branch: git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
3. git fetch origin <branch>
4. git log HEAD..origin/<branch> --oneline — if empty, report "hermes-webui already up to date" and stop
5. If new commits exist:
   a. git stash — save any local changes
   b. git pull origin <branch>
   c. sudo systemctl restart hermes-webui
   d. Report: what was updated + first 10 lines of changelog
```

### 6.2 hermes-agent 每日升级

```
cronjob action=create
name=hermes-agent-daily-upgrade
schedule=0 3 * * *
deliver=local
prompt=Check for hermes-agent updates and upgrade if available. Auto-detect all paths.

Steps:
1. Run: ls ~/.hermes/hermes-agent/.git to locate the agent repo. Also check HERMES_WEBUI_AGENT_DIR env var. Default: ~/.hermes/hermes-agent
2. cd into the agent repo, detect default branch: git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
3. git fetch origin <branch>
4. git log HEAD..origin/<branch> --oneline — if empty, report "hermes-agent already up to date" and stop
5. If new commits exist:
   a. git stash — save any local changes
   b. git pull origin <branch>
   c. cd into repo, source venv/bin/activate && pip install -e . 2>&1 | tail -10
   d. Report: what was updated + first 10 lines of changelog
```

---

## 踩坑记录

### Docker 方式的致命问题

- **数据丢失**：容器重建后 webui-mvp 状态目录丢失，需重新 setup
- **模块缺失**：/api/crons 和 /api/skills 依赖 agent 源码的 cron/tools 模块，Docker 单容器没挂载就 500
- **挂载复杂**：HERMES_HOME 可能是 profiles/minimax，hermes-agent 在上级 ~/.hermes，需额外挂载
- **升级慢**：rebuild 镜像几分钟

### 关键检查点

| 症状 | 原因 | 修复 |
|------|------|------|
| /api/crons 返回 500 ModuleNotFoundError: cron | webui 没用 agent venv 的 python | ExecStart 必须指向 agent venv 的 python3 |
| /api/skills 返回 500 ModuleNotFoundError: tools | 同上 | 同上 |
| 打开 webui 要重新 setup | HERMES_WEBUI_STATE_DIR 不对或丢失 | 确保 state dir 在 HERMES_HOME 下且持久 |
| 局域网无法访问 | 绑定 127.0.0.1 | HERMES_WEBUI_HOST=0.0.0.0 |
| git pull 冲突 | 本地有未提交修改 | 升级脚本先 git stash |
| branch 名不对 | 硬编码 main 或 master | 用 git remote show origin 动态获取 |
