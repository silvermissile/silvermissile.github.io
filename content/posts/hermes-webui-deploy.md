---
title: "给 Hermes Agent 换张脸：hermes-webui 部署实录"
date: 2026-09-01
tags:
  - Hermes
  - WebUI
  - Docker
  - Agent
  - 部署
description: Hermes Agent 自带的 Dashboard 界面实在让人提不起兴致，于是调研并部署了社区项目 hermes-webui，获得了现代化的 Web 聊天体验。记录完整的调研、部署和踩坑过程。
---

## 背景：自带的 UI 太丑了

Hermes Agent 是我目前在一台 Ubuntu 服务器上跑的主力 AI Agent，功能很强大，但官方 Dashboard（:9119）的界面实在让人提不起兴致——本质上就是一个 xterm.js 终端嵌套，操作全靠 TUI 命令。能用，但谈不上好用。

于是开始调研社区有没有更好的 Web 前端。最终锁定了 [hermes-webui](https://github.com/nesquena/hermes-webui) 这个项目，它提供了原生 Web 聊天界面、Markdown 渲染、代码高亮、流式输出、会话管理、文件浏览、可视化配置等一整套现代体验。

这篇文章记录从调研到部署的完整过程，包括踩过的坑。

---

## hermes-webui 是什么

hermes-webui 是 Hermes Agent 生态中最成熟的社区 WebUI 项目，提供完整的浏览器聊天界面，替代官方 Dashboard。

| 属性 | 值 |
| --- | --- |
| 项目地址 | [nesquena/hermes-webui](https://github.com/nesquena/hermes-webui) |
| Docker 镜像 | `ghcr.io/nesquena/hermes-webui:latest` |
| 当前版本 | exp-v0.52.264 |
| 技术栈 | Python 3.12 + vanilla JS（无 React/Vue/Node 构建） |
| 服务端口 | 8787 |
| 许可证 | 开源 |

---

## 为什么选它：功能对比

和官方 Dashboard 相比，hermes-webui 几乎是全方位的升级：

| 功能 | 官方 Dashboard (:9119) | hermes-webui (:8787) |
| --- | --- | --- |
| 聊天界面 | xterm.js 嵌套 TUI（终端风格） | 原生 Web 聊天（Markdown 渲染、代码高亮、流式输出） |
| 会话管理 | 无（依赖 TUI 命令） | 侧边栏列表、搜索、分组、项目归类 |
| 文件浏览 | 无 | 三栏布局含 workspace 文件树 |
| 配置管理 | 无（需 CLI） | 可视化设置面板 |
| 模型切换 | TUI `/model` 命令 | 下拉选择器 |
| Profile 管理 | TUI 命令 | 可视化切换 |
| Skills/MCP | TUI overlay | 可视化面板 |
| 定时任务 | CLI 命令 | 可视化管理 |
| 移动端 | 不支持 | PWA 支持 |
| 主题 | 无 | 多主题 + 深色模式 |
| 认证 | HTTP Basic Auth | 密码 + Passkey + OIDC |

---

## 和 Hermes Studio 的区别

之前也评估过另一个项目 Hermes Studio，但最终放弃了：

| 维度 | hermes-webui | Hermes Studio |
| --- | --- | --- |
| 定位 | Hermes Agent 的 Web 前端 | 多 Agent 工作台平台 |
| 架构 | Python 单进程 | Vue + Koa + SQLite monorepo |
| Agent 运行时 | 复用官方 Hermes | Hermes + Ekko + Claude Code + Codex + Pi |
| 对 ~/.hermes 的侵入 | 新建 `webui/` 子目录，不修改现有文件 | 自动注入 MCP 配置、root 权限写文件 |
| Gateway 冲突 | 无（不启动自己的 Gateway） | 启动独立 Gateway，PID/Lock 冲突 |
| 决策 | **推荐使用** | 已放弃 |

Hermes Studio 的问题在于它太重了——启动独立 Gateway 会和已有的 Agent 冲突，还会自动注入 MCP 配置。hermes-webui 则轻量得多，只作为前端存在，聊天请求转发给现有 Gateway 处理。

---

## 对现有 Agent 的影响

这是我最关心的问题：装个新前端，会不会影响已经在跑的 Agent？

### 数据架构

```
~/.hermes/                          ← 现有 Agent 数据（不被修改）
├── config.yaml                     ← WebUI 读取；用户改设置时才写入
├── state.db                        ← WebUI 以 read-only 打开
├── gateway.pid / gateway.lock      ← WebUI 不触碰
├── sessions/                       ← Agent 原生会话（不触碰）
├── skills/                         ← 共享读取
├── memory/                         ← 共享读取
└── webui/                          ← 【新建】WebUI 专属目录
    ├── sessions/                   ← WebUI 会话 JSON sidecar
    ├── settings.json               ← WebUI 设置
    ├── workspaces.json             ← 已注册的 workspace
    └── projects.json               ← 会话项目分组
```

### 影响矩阵

| 操作 | Gateway 模式（推荐） | In-process 模式 |
| --- | --- | --- |
| 读取 config.yaml | 只读 | 只读 |
| 读取 state.db | read-only 连接 | read-only 连接 |
| 写入 state.db | **不写入** | 写入（标记 source=webui） |
| 启动 Gateway 进程 | **不启动** | **不启动** |
| 占用 8642 端口 | 不占用（连接现有 Gateway） | 不占用 |
| 修改现有会话 | **不修改** | **不修改** |

**结论**：Gateway 模式下对现有 Agent 几乎零影响。WebUI 只读取共享数据，聊天请求转发给现有 Gateway。不会启动新的 Gateway，不会冲突 PID/Lock 文件，不会修改已有会话。

---

## 部署方案

### 架构概览

```
                  Ubuntu Server (内网)
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Native systemd                                     │
│  ┌──────────────────────────────────┐               │
│  │  hermes-gateway.service          │               │
│  │  端口: 127.0.0.1:8642 (API)      │               │
│  │  数据: ~/.hermes/                │               │
│  └──────────────────────────────────┘               │
│  ┌──────────────────────────────────┐               │
│  │  hermes-dashboard.service        │               │
│  │  端口: 0.0.0.0:9119 (官方 Dashboard) │           │
│  └──────────────────────────────────┘               │
│                    ▲                                │
│                    │ HTTP 127.0.0.1:8642            │
│  Docker (host 网络模式)                              │
│  ┌─────────────────┴────────────────┐               │
│  │  hermes-webui 容器               │               │
│  │  端口: 0.0.0.0:8787              │               │
│  │  CHAT_BACKEND=gateway            │               │
│  │  GATEWAY_BASE_URL=               │               │
│  │    http://127.0.0.1:8642         │               │
│  │  挂载: ~/.hermes (共享数据)       │               │
│  └──────────────────────────────────┘               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

核心设计：使用 `network_mode: host` 而非 bridge 网络。原因是 Gateway 仅监听 `127.0.0.1:8642`，bridge 模式下容器通过 `host.docker.internal` (172.17.0.1) 无法访问。host 模式下容器直接共享宿主机网络栈，可直接访问 `127.0.0.1:8642`。

### 前置条件

1. **Gateway API Server 已启用** — 8642 端口在监听，`API_SERVER_KEY` 已配置
2. **Docker 已安装** — Docker 29.7.2 + Compose v5.5.0
3. **防火墙放行 8787 端口** — 服务器启用了 UFW（默认策略 DROP），必须显式放行

### 部署步骤

```bash
# 1. 创建部署目录
mkdir -p ~/hermes-webui && cd ~/hermes-webui

# 2. 准备配置文件
cp <你的仓库路径>/docker-compose.yml ~/hermes-webui/
cp <你的仓库路径>/.env.example ~/hermes-webui/
cp .env.example .env
# 编辑 .env，填入 API_SERVER_KEY、HERMES_WEBUI_PASSWORD、UID/GID

# 3. 拉取镜像（~750MB，首次约 1-2 分钟）
docker compose pull

# 4. 防火墙放行
sudo ufw allow 8787/tcp comment 'Hermes WebUI'

# 5. 启动
docker compose up -d

# 6. 等待初始化（首次启动需安装 agent 依赖，约 1-2 分钟）
docker logs -f hermes-webui
# 看到 "Hermes Web UI listening on http://0.0.0.0:8787" 即启动成功

# 7. 验证
curl -s http://localhost:8787/health | python3 -m json.tool
```

---

## 配置文件说明

### docker-compose.yml 关键设计

- **网络模式**：`network_mode: host`，容器直接使用宿主机网络栈
- **挂载 ~/.hermes**：共享 config.yaml、skills、memory 等；WebUI 数据写入 `~/.hermes/webui/`
- **CHAT_BACKEND=gateway**：聊天由现有 Gateway 处理，WebUI 不运行 Agent runtime
- **密码保护**：端口暴露在 0.0.0.0，必须设密码
- **资源限制**：1GB 内存上限，防止影响 Agent 主进程

### 必须配置的环境变量

| 变量 | 说明 | 示例 |
| --- | --- | --- |
| `API_SERVER_KEY` | Gateway API Key（>=16 字符，需与 Gateway 一致） | `a1b2c3d4e5f6g7h8...` |
| `HERMES_WEBUI_PASSWORD` | WebUI 访问密码 | `your-strong-password` |
| `UID` / `GID` | 宿主机用户 UID/GID | `1000` / `1000` |

### 密码配置位置（易混淆）

**WebUI 密码不在 `~/.hermes/.env` 里。** 两套认证各管各的：

| 服务 | 配置文件 | 变量名 | 端口 |
| --- | --- | --- | --- |
| 官方 Dashboard | `~/.hermes/.env` | `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD` | 9119 |
| hermes-webui | `~/hermes-webui/.env` | `HERMES_WEBUI_PASSWORD` | 8787 |

---

## 使用体验

### 首次访问

1. 浏览器打开 `http://<服务器IP>:8787`
2. 输入密码（注意是 `~/hermes-webui/.env` 中的 `HERMES_WEBUI_PASSWORD`，不是 Dashboard 密码）
3. 进入聊天界面，左侧显示会话列表（包括 CLI/TUI 历史会话，只读）

### 日常操作

| 操作 | 方法 |
| --- | --- |
| 新建对话 | 点击侧边栏 "+" 或快捷键 |
| 切换模型 | 聊天区顶部模型选择器 |
| 查看历史 | 侧边栏会话列表（WebUI + CLI/TUI 会话均可见） |
| 文件浏览 | 右侧 workspace 面板 |
| 修改设置 | 左下角设置按钮 → Control Center |
| 管理 Skills | Control Center → Skills 面板 |
| 管理 MCP | Control Center → MCP 面板 |

### 和官方 Dashboard 的关系

两者可以同时运行、互不干扰：

| 界面 | URL | 用途 |
| --- | --- | --- |
| 官方 Dashboard | `http://<服务器IP>:9119` | TUI 终端体验、系统监控 |
| hermes-webui | `http://<服务器IP>:8787` | 现代 Web 聊天界面、文件浏览、可视化管理 |

---

## 踩坑记录

### 1. bridge 网络无法访问 Gateway

Gateway 绑定 `127.0.0.1:8642`，Docker bridge 网络中容器通过 `host.docker.internal` (172.17.0.1) 访问会被拒绝。

**解决**：改用 `network_mode: host`。

### 2. UFW 防火墙未放行 8787 端口

容器正常运行、端口正常监听，但其他节点浏览器访问超时 (`ERR_CONNECTION_TIMED_OUT`)。

**原因**：`network_mode: host` 时 Docker 不自动管理 iptables 规则（与 bridge 模式不同），需要手动在 UFW 中放行端口。9119 端口在部署 Dashboard 时已放行，但部署 WebUI 时遗漏了 8787。

**修复**：

```bash
sudo ufw allow 8787/tcp comment 'Hermes WebUI'
```

### 3. 首次启动慢

容器首次启动需要 `uv pip install` hermes-agent 的 Python 依赖（约 100 个包），耗时 1-2 分钟。期间 `/health` 端点不可用。后续重启不需要重新安装。

### 4. dashboard-auth-basic 警告

启动日志中出现的 `dashboard-auth-basic` 警告来自 `~/.hermes/.env` 中的 Dashboard Basic Auth 配置，与 WebUI 无关，可以安全忽略。

### 5. WebUI 显示旧会话但无法继续对话

WebUI 以 read-only 模式读取 state.db 中的 CLI/TUI 会话，这些会话无法在 WebUI 中继续。在 WebUI 中新建会话即可开始对话，历史会话仅供查看。

---

## 回退方案

如果哪天不想要了，清理也很干净：

```bash
# 1. 停止并删除容器
cd ~/hermes-webui && docker compose down

# 2. 删除 WebUI 专属数据
rm -rf ~/.hermes/webui/

# 3. 可选：删除 Docker 镜像
docker rmi ghcr.io/nesquena/hermes-webui:latest

# 4. 可选：删除部署目录
rm -rf ~/hermes-webui/
```

临时停用更简单：`docker compose stop`，恢复时 `docker compose start`，数据完整保留。

---

## 已知限制

| 限制 | 说明 | 影响 |
| --- | --- | --- |
| Gateway 模式功能不完整 | 附件上传、工具审批等功能需额外配置 | 核心聊天不受影响 |
| 版本耦合 | WebUI 镜像内置的 agent 版本需与本地 Agent 兼容 | 升级时需注意同步 |
| config.yaml 写入 | 在 WebUI 中修改设置会写入共享的 config.yaml | CLI/TUI 会同步看到变化 |

---

## 总结

hermes-webui 的部署比预期顺利——Docker 单容器 + Gateway 模式 + host 网络，对现有 Agent 几乎零影响。界面从"终端嵌套"升级到"现代 Web 聊天"，日常使用体验提升巨大。

最关键的是回退成本极低：WebUI 的所有数据都在 `~/.hermes/webui/` 下，删掉这个目录就等于什么都没装过。对于这种"锦上添花"的组件，低侵入性比功能丰富更重要。

---

## 参考资料

- [hermes-webui 官方仓库](https://github.com/nesquena/hermes-webui)
- [hermes-webui Docker 文档](https://github.com/nesquena/hermes-webui/blob/master/docs/docker.md)
- [hermes-webui Gateway 模式文档](https://github.com/nesquena/hermes-webui/blob/master/docs/advanced-chat-setup.md)
- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/)
