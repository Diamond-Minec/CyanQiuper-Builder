# CyanQiuper 本地版

Minecraft 基岩版 **结构导入工具** 的本地部署版：一个 WebSocket 服务 + 一个网页管理面板，可选接入**网易版**通道。

上传 `.mcstructure` 或投影文件，在游戏里用 `-pos()` 定个原点，工具自动完成 **放置 → 逐块校验 → 缺失补建**，游戏内实时显示多行进度 HUD。

> 本项目基于 [CAIMEOX/FastBuilder](https://github.com/CAIMEOX/FastBuilder)（MIT）扩展而来，特此致谢。
> 与 Mojang、Microsoft、网易无关；使用相关服务时请遵守对应服务条款。

---

## 功能

- **多格式导入**：`.mcstructure`、`.litematic`、`.schem`、`.schematic`、结构方块 `.nbt`
- **三步完整性保证**：放置 → 回读校验 → 缺失补建，失败命令自动重试
- **网页管理面板**：上传文件、查看进度、实时日志、终端命令（`status` / `say` / `cmd` / `eval` 等）
- **游戏内 HUD**：总体/区块进度、文件名/格式、已用时间与剩余时间
- **文件修补**：损坏的 `.mcstructure` 一键修补（补结束标记、解 gzip、修正索引）
- **网易版通道（可选）**：通过你自备的网易机器人 + 事件桥导入（账号由你自己配置）

## 环境要求

- **Node.js ≥ 20**（推荐 22），需要 `npm`
- Windows / Linux / macOS
- Minecraft 基岩版 **1.21+**（支持聊天栏 `/connect ws://...`）

## 快速开始（Windows）

1. 安装 [Node.js](https://nodejs.org/)（安装时勾选 Add to PATH）
2. 双击 **`start.bat`**：会自动 `npm install` 并启动服务
3. 浏览器打开 **http://127.0.0.1:11452**（管理面板）
4. 游戏里连接：

```
/connect ws://127.0.0.1:11451
```

5. 在面板上传结构文件 → 游戏里站到构建点输入 `-pos()` → 点「开始生成」

## 快速开始（Linux / macOS）

```bash
npm install
npm run admin        # 启动管理面板（会同时拉起 WS 服务）
```

后台常驻可用 `deploy/cyanqiuper.service`（systemd 示例）。

## 管理面板密码

默认**本机免登录**。要加密码：

```bash
node tools/set-password.mjs 你的密码
```

会在 `admin/auth.json` 生成 scrypt 哈希（只存哈希，不存明文；建议权限 600、别提交到 git）。
删除 `admin/auth.json` 即恢复免登录。

## 端口

| 端口 | 用途 | 说明 |
|---|---|---|
| 11451 | WS 服务 | 游戏 `/connect` 连这个 |
| 11452 | 管理面板 | 默认只监听 127.0.0.1 |
| 11453 | 控制接口 | 仅本机，面板内部调用 |
| 11454 | 网易版服务（可选） | 仅本机 |

> 如果部署在服务器上并让外部玩家连接，记得放行 11451（以及下面网易版相关端口）。

## 网易版通道（可选，账号自己配置）

本项目 **不包含** 网易机器人本体与网易协议实现（第三方组件，且涉及登录凭据，不随仓库分发）。
使用前提：你已经有可用的网易机器人，能提供 **Unix socket 事件桥 + token**。

### 1. 准备机器人

把机器人放到本目录下（例如 `netease-bot/`，已在 `.gitignore` 中忽略），按其说明可以启动并进服。

### 2. 配置你自己的网易账号

复制模板：

```bash
netease/config.json.example  →  机器人目录/config.json
netease/mc.cfg.example       →  机器人目录/mc.cfg
```

- `config.json`：填 `mode`（`server` 租赁服 / `lobby` 联机大厅 / `domain` 山头 / `nethernet` 本地联机）、`id`（短号/房号/山头码）、`password`（房间密码）
- `mc.cfg`：粘贴**你自己的** 网易版 `sauth` cookie（从你本地网易版客户端导出；不要用别人的）

### 3. 配置本项目的网易服务

```bash
netease/netease.env.example  →  netease/netease.env
```

```
FB_NETEASE_SOCKET=你的机器人 bot.sock 路径
FB_NETEASE_TOKEN=机器人事件桥的 token
FB_NETEASE_PORT=11454
```

### 4. 启动

- Windows：先 `set` 上面这些环境变量，再运行 `npm run netease`
- Linux：用 `deploy/cyanqiuper-netease.service`（改好路径和 EnvironmentFile）

### 5. 开始导入

机器人进服后，在管理面板「网易版」里上传结构、开始导入。
注意网易版会自动限速（`FB_NETEASE_RATE`，默认 5 条/秒），校验用幂等的 `setblock` 而不是 `testforblock`（网易不支持后者）。

> ⚠️ 网易账号 cookie 属于登录凭证，只保存在你自己机器上；不要提交到 git、不要发给别人。

## 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `FB_WS_PORT` | 11451 | WS 端口 |
| `FB_CONTROL_PORT` | 11453 | 本机控制接口 |
| `FB_ADMIN_PORT` | 11452 | 管理面板端口 |
| `FB_ADMIN_HOST` | 127.0.0.1 | 管理面板监听地址 |
| `FB_PUBLIC_WS_URL` | 空 | 面板展示给玩家的 WS 地址（如 `ws://1.2.3.4:11451`） |
| `FB_WS_VERSION` | 17104896 | 基岩版协议版本 |
| `FB_LOG_DIR` | `./logs` | 日志目录 |
| `FB_NO_OPEN` | 空 | 设为 `1` 不自动打开浏览器 |
| `FB_NETEASE_SOCKET` | 空 | 机器人事件桥 socket 路径 |
| `FB_NETEASE_TOKEN` | 空 | 事件桥 token |
| `FB_NETEASE_PORT` | 11454 | 网易版服务端口 |
| `FB_NETEASE_RATE` | 5 | 网易版命令限速（条/秒） |
| `FB_NETEASE_VERIFY_CONC` | 2 | 网易版校验并发 |
| `FB_NETEASE_VERIFY_SAMPLE` | 120 | 网易版校验抽样点数 |

## 目录结构

```
main.ts                WS 服务（游戏连接、结构导入、校验补建、HUD）
admin-server.ts        管理面板（HTTP + 静态页，自动拉起 main.ts）
netease-service.ts     网易版导入服务（可选）
src/                   结构解析、命令生成、文件修补、网易桥
admin/                 管理面板前端（index.html / login.html / assets）
tools/set-password.mjs 设置管理面板密码
netease/               网易账号/房间配置模板（自己填，不入库）
deploy/                systemd 示例
logs/                  运行日志
```

## 常见问题

**游戏连不上？** 确认 WS 服务在跑、端口 11451 可访问；服务器部署时要放行防火墙；地址用 `ws://IP:11451`。

**校验未通过 / 还有缺口？** 大多是大结构远处区块未加载，靠近结构再点「完整性检查」或「补建」。

**端口被占用？** 用 `stop.bat`（Windows）或手动结束占用进程；或改 `FB_WS_PORT`/`FB_ADMIN_PORT`。

**没有 node？** 安装 Node.js 后重开命令行；`start.bat` 会自动检测。

## 安全建议

- 管理面板默认只监听 `127.0.0.1`，如需公网访问请加反向代理 + HTTPS + 强密码
- `admin/auth.json`、`netease/`、`logs/`、`node_modules/` 不要提交
- 不要把自己的网易 cookie 提交到仓库或发给他人

## 许可

MIT License（见 `LICENSE`）。基于 CAIMEOX/FastBuilder 扩展，致谢原作者。
