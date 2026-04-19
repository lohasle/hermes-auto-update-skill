# Hermes Auto Update Skill

> 给 Hermes agent 的快速安装提示词 — 一键获得 hermes-agent + hermes-webui 本机部署与每日自动升级能力

## 快速安装

将以下提示词发给任何 Hermes agent，即可自动完成部署：

```
请从 GitHub 加载技能 hermes-auto-update：

1. 先执行: git clone https://github.com/lohasle/hermes-auto-update-skill.git /tmp/hermes-auto-update-skill
2. 读取 /tmp/hermes-auto-update-skill/SKILL.md，按照其中的步骤逐一执行
3. 执行完毕后，用 hermes 的 skill_manage 工具将 /tmp/hermes-auto-update-skill/SKILL.md 注册为技能 hermes-auto-update
4. 创建两个每日自动升级 cron job（详见 SKILL.md Step 6）

关键提醒：
- 所有路径自动探测，不要问用户
- 不用 Docker
- 必须用 hermes-agent 的 venv python 跑 webui
- 验证 /api/crons 和 /api/skills 都正常才算成功
```

## 这个技能做什么

1. **本机部署 hermes-agent** — git clone + venv + pip install -e .
2. **本机部署 hermes-webui** — git clone，用 agent venv 运行，systemd 托管
3. **每日自动升级** — 两个 cron job，凌晨 3:00 检查更新，有则 pull + 重启

## 为什么不用 Docker

| 问题 | Docker | 本机 |
|------|--------|------|
| 数据持久性 | 容器重建丢数据 | 数据在 ~/.hermes，永不丢 |
| cron/tools API | 需额外挂载 agent 源码，否则 500 | agent venv 直接可用 |
| 升级速度 | rebuild 镜像几分钟 | git pull + restart，秒级 |
| 配置复杂度 | 挂载路径、UID/GID | 自动探测，零配置 |

## 文件说明

- `SKILL.md` — 技能定义，agent 可直接加载执行
- `README-human.md` — 人类阅读的完整部署指南
- `README.md` — 本文件，包含快速安装提示词
