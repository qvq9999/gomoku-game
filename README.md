# 联机五子棋（gomoku-online）

手机端网页双人联机五子棋 —— 基于 Node.js + Express + ws 的实时对战服务，前端零构建、零框架，纯原生 HTML5 / CSS / Canvas 实现。

> 当前版本：`2.3.0`，数据库已升级为 SQLite（`data/gomoku.db`）
> 项目效果预览：http://47.104.238.201:3000/
---

## 一、核心功能特性

- **账号系统**：注册 / 登录 / 登出，会话持久化，自定义头像（存储于 SQLite）。
- **好友系统**：添加好友、搜索用户、好友在线状态、好友邀请对战。
- **双人联机对战**：WebSocket 实时落子，房间制（创建房间 / 好友邀请 / 快速匹配）。
- **挑战 AI**：三档棋力梯队（novice / medium / hard）。hard 级由 Rapfi 引擎驱动，自研引擎兜底。
- **段位与排位**：初始 1000 分，按先后手补偿计分，连胜 / 连败修正；练习赛与段位赛双模式。
- **经济系统**：门票、金币、任务、商城（皮肤 / 落子特效等），数值集中在 `lib/ai_economy.js`。
- **悔棋**：双方同意、每局限定次数、超时自动放弃。
- **互动与体验**：表情气泡聊天浮于棋盘、Web Audio 合成音效、移动端震动反馈（haptics）、断线重连（默认 120 秒宽限）。
- **观战与快速匹配**：详见 `docs/观战与快速匹配-设计方案.md`（设计文档）。
- **站内信**：左下角入口，消息存储于 `lib/mail_store.js`。
- **对局回放**：前端 `public/js/replay.js` 支持战绩回放。

> 说明：以上为代码与文档中已落实或已设计实现的能力；少量细节（如商城具体 SKU）以 `lib/ai_economy.js` 与 `public/` 实际内容为准，未在此展开处标注「待补充」。

---

## 二、技术栈与运行环境要求

| 类别 | 选型 |
| --- | --- |
| 运行时 | Node.js **≥ 18.0.0**（见 `package.json` `engines`） |
| 后端 | Express 4、ws（WebSocket）、原生 `http` / `zlib` |
| 持久化 | SQLite（WAL 模式，`better-sqlite3` 原生模块） |
| 前端 | 原生 HTML / CSS / JavaScript / Canvas，**无构建步骤、无前端框架** |
| AI 引擎 | 自研 `lib/gobang-ai`（移植自开源 `lihongxun945/gobang`）+ Rapfi（`lib/rapfi`，hard 级） |
| 进程加速 | `worker_threads`（`lib/gobang-worker.js`）分担 AI 搜索，降低主线程抖动 |

**环境注意**

- `better-sqlite3` 为原生模块，安装时可能需本地编译工具链（Linux：`build-essential` + `python3`；macOS：Xcode CLT）。安装失败请使用与生产系统一致的镜像预编译，或先装好编译依赖再 `npm install`。
- Rapfi 引擎二进制（`engine/rapfi/pbrain-rapfi-*`）仅提供 **Linux** 可执行文件（sse / avx2 / avx512vnni 三档指令集）。**硬棋力 AI 仅在 Linux 服务器生效**；非 Linux 环境下需设置 `GOBANG_AI_ENGINE=gobang` 强制使用自研引擎兜底。
- 本机 CPU 不支持 `avx512vnni` 时，请勿选用对应二进制（会触发非法指令崩溃），引擎会自动选取可用的最高指令集文件。

---

## 三、依赖安装

项目无额外第三方前端依赖，仅需安装 Node 依赖：

```bash
# 进入项目目录（本 README 所在目录）
cd gomoku

# 安装生产依赖（含 better-sqlite3）
npm install --production

# 校验原生模块可加载
node -e "require('better-sqlite3'); console.log('ok')"
```

> 开发 / 测试依赖同属 `dependencies`（项目未拆分 `devDependencies`），直接 `npm install` 即可。

---

## 四、本地运行 / 构建 / 测试

### 运行（开发或生产）

```bash
# 方式一：直接启动
npm start
# 等价于：node server.js

# 方式二：PM2 守护（生产推荐）
npm run pm2:start        # 启动，命名为 gomoku
npm run pm2:stop         # 停止
npm run pm2:restart      # 重启（修改 server.js / lib/ 后必须重启）
npm run pm2:logs         # 查看日志
```

启动后：

- 主服务：`http://<HOST>:<PORT>`（默认 `0.0.0.0:3000`，WebSocket 路径 `/ws`）
- 管理接口：仅本机 `http://127.0.0.1:<ADMIN_PORT>`（默认 `PORT+100 = 3100`，需 `ADMIN_KEY` 鉴权）
- 健康检查：`GET /healthz`（返回 `guard` / `ai` / `storage` 状态）

### 构建

**无需构建**。前端为 `public/` 下的静态文件，由 Express 直接托管；`index.html` 内的资源版本号（`?v=...`）由 `server.js` 按文件 mtime 自动注入，部署时 `index.html` 须与 `server.js` 同批更新。

### 测试

```bash
# 运行完整回归套件（按 package.json scripts.test）
npm test
```

`npm test` 依次执行（均位于 `tools/`，已核对存在）：

- `tools/protocol-test.js` —— WebSocket 协议 / 房间流程
- `tools/economy-test.js` —— 经济 / 战绩数值（"宪法"级断言）
- `tools/gobang-worker-test.js` —— worker_threads 加速
- `tools/tactics-test.js` —— AI 战术
- `tools/guard-test.js` —— 限流 / 防护
- `tools/smoke-test.js` —— 冒烟
- `tools/compression-test.js` —— 压缩中间件

> 还可单独运行：`node tools/smoke-test.js` 等。其余 `tools/*.js`（如 `lobby-modes-shot.js`、`glass-real-shot.js`、`rapfi-test.js`）为开发期截图 / 排查 / 引擎探针脚本，非 `npm test` 强制项。

---

## 五、关键配置项说明

服务通过 **环境变量** 配置（无独立配置文件；`.env` 已被 `.gitignore` 忽略）。以下为 `server.js` 中实际读取的变量与默认值：

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | `3000` | 主服务监听端口 |
| `HOST` | `0.0.0.0` | 主服务绑定地址 |
| `ADMIN_PORT` | `PORT + 100` | 管理接口端口（仅本机 `127.0.0.1`） |
| `ADMIN_KEY` | 空（未配置则停用） | 管理接口密钥；可写入 `data/admin_key.txt` 首行替代 |
| `DATA_DIR` | `<项目根>/data` | 运行期数据目录（SQLite 等） |
| `GRACE_MS` | `120000` | 玩家断线重连宽限（毫秒） |
| `AI_GRACE_MS` | `30000` | AI 断线重连宽限（毫秒） |
| `GOBANG_DEPTH` | `4` | 自研引擎默认搜索深度 |
| `GOBANG_VCF_DEPTH` | `8` | 纯冲四算杀深度 |
| `GOBANG_BOOK_USE` | 未设置 | 开局库开关（`'1'` 开启） |
| `GOBANG_AI_BUDGET_MS` | `0` | 自研引擎单步时间预算（0 = 不限制） |
| `GOBANG_AI_MAX_DEPTH` | `8` | 自研引擎最大深度 |
| `GOBANG_WORKER` | 开启 | 是否启用 worker_threads（`'0'` 关闭） |
| `GOBANG_WORKER_SIZE` | `1` | worker 池大小 |
| `GOBANG_WORKER_TIMEOUT_MS` | `8000` | worker 单步超时 |
| `GOBANG_WORKER_QUEUE_MS` | `0` | worker 排队等待上限 |
| `GOBANG_SYNC_FALLBACK_MS` | `300` | 主线程同步兜底限时（毫秒） |
| `RAPFI_TIME_MS` | `800` | Rapfi 单步思考时间（毫秒） |
| `RAPFI_POOL_SIZE` | `1` | Rapfi 进程池大小 |
| `RAPFI_QUEUE_WAIT_MS` | `5000` | Rapfi 排队等待上限 |
| `RAPFI_RETRY_MS` | `30000` | Rapfi 启动失败重试间隔 |
| `RAPFI_ENGINE_DIR` | `<项目根>/engine/rapfi` | Rapfi 引擎目录（覆盖默认） |
| `GOBANG_AI_ENGINE` | 未设置（默认 Rapfi） | 设为 `gobang` 强制使用自研引擎 |
| `WS_RATE_CAPACITY` | `60` | 令牌桶容量（每连接） |
| `WS_RATE_REFILL` | `12` | 令牌桶 refill 速率（个 / 周期） |
| `MAX_CONN_PER_IP` | `60` | 单 IP 最大并发连接 |
| `LOGIN_FAIL_MAX` | `8` | 登录失败次数上限 |

> 固定项：`WS_MAX_PAYLOAD = 256KB`（硬性常量，不可经环境变量调低；头像 dataURL 约 109KB，下调会导致拒绝合法请求）。

### 配置示例（`.env` 或启动前 export）

```bash
# 端口与绑定
PORT=3000
HOST=0.0.0.0

# 数据目录（可指向独立磁盘 / 挂载）
DATA_DIR=/var/lib/gomoku/data

# 管理接口密钥（也可写入 data/admin_key.txt 首行）
ADMIN_KEY=your-strong-admin-key

# 非 Linux 开发机：强制自研引擎（Rapfi 为 Linux 二进制）
GOBANG_AI_ENGINE=gobang

# AI 调参（按需）
GOBANG_DEPTH=4
RAPFI_TIME_MS=800
GOBANG_WORKER_SIZE=2

# 防护调参
WS_RATE_CAPACITY=60
WS_RATE_REFILL=12
MAX_CONN_PER_IP=60
```

启动示例：

```bash
PORT=3000 ADMIN_KEY=xxx node server.js
```

---

## 六、目录结构说明

```text
gomoku/
├── server.js                  # 入口：HTTP + WebSocket 服务、路由、房间与游戏主逻辑
├── package.json               # 项目元数据、脚本、依赖（name: gomoku-online, v2.1.0）
├── package-lock.json
├── admin.js / admin.sh         # 管理 / 运维脚本（具体用途见 DEPLOY.md）
├── DEPLOY.md                  # 部署说明（宝塔 / 手动 / Docker 路径、SQLite 迁移）
├── PROTOCOL.md                # WebSocket 协议文档
├── public/                    # 前端静态资源（零构建，由 Express 直接托管）
│   ├── index.html             # 主页 + 游戏页（含 ?v 版本号注入）
│   ├── admin-economy.html     # 经济管理后台页
│   ├── css/                   # tokens.css（主题变量）/ app.css / glass.css（液态玻璃）等
│   ├── js/                    # board.js / game.js / rules.js / sound.js / replay.js 等
│   └── uploads/               # 运行期用户上传（头像等，gitignore）
├── lib/
│   ├── gobang-ai/             # 自研 AI 引擎（移植自 lihongxun945/gobang，含开局库/评测）
│   ├── rapfi/                 # Rapfi 硬棋力引擎封装（pool.js / client.js / config.toml）
│   ├── gobang-worker.js       # worker_threads 池（AI 搜索卸载）
│   ├── gobang-worker-thread.js
│   ├── ai_economy.js          # 经济 / 战绩 / 任务 / 商城数值表（禁止手改 data/*.json）
│   ├── sqlite_store.js        # SQLite 持久化（WAL，启动自动 JSON→SQLite 迁移）
│   └── mail_store.js          # 站内信存储
├── engine/
│   └── rapfi/                 # Rapfi 二进制（Linux only）+ 权重文件（model*.bin / *.lz4）
├── data/                      # 运行期数据（默认；可用 DATA_DIR 覆盖）
│   └── gomoku.db              # SQLite 主库（含 -wal / -shm 同目录）
├── tools/                     # 测试 / 截图 / 排查脚本（见第四节测试清单）
└── docs/                      # 设计文档与截图
    └── 观战与快速匹配-设计方案.md
```

> 根目录另含 `cloudflare-landing/`（着陆页，独立部署）、`rapfi-引擎上传包/`（位于项目外层，为引擎打包产物，非运行必需）。

---

## 七、常见问题与注意事项

**Q1：启动报错 `better-sqlite3` 找不到 / 编译失败？**
安装 `better-sqlite3` 需要原生编译。请先安装编译工具链（Linux：`apt install build-essential python3`；macOS：装 Xcode Command Line Tools），再 `npm install`；或拉取与目标系统一致的预编译二进制。

**Q2：硬棋力 AI（hard）不动 / 报错？**
Rapfi 仅提供 Linux 可执行文件。非 Linux 环境请设置 `GOBANG_AI_ENGINE=gobang` 强制自研引擎；Linux 上若 `engine/rapfi/` 缺失或 CPU 不支持所选指令集（如 avx512vnni），引擎会自动回退。可用 `GET /healthz` 查看 `ai.hardEngine` 是否为 `rapfi` 验证。

**Q3：改了前端没变化 / 缓存不更新？**
`index.html` 的资源版本号由 `server.js` 按 mtime 注入。**修改 `public/` 后必须连同 `index.html` 与 `server.js` 同批部署**；仅更新静态文件而沿用旧 server 会导致长缓存锁死。

**Q4：改了 `server.js` / `lib/` 不生效？**
这类改动需要**重启进程**（`npm run pm2:restart` 或宝塔「网站 → Node 项目 → 重启」）。仅改 `public/` 或 `*.md` 无需重启。

**Q5：WebSocket 连接被频繁断开？**
检查防护参数：`WS_MAX_PAYLOAD`（固定 256KB，不可下调）、令牌桶 `WS_RATE_CAPACITY` / `WS_RATE_REFILL`、单 IP `MAX_CONN_PER_IP`。超限会被限流或断连。

**Q6：数据库备份怎么做？**
运行期数据以 `data/gomoku.db`（及同目录 `-wal` / `-shm`）为准。**建议先停服再整体拷贝 `data/`**；切勿在迁移完成前手改旧 JSON 文件（它们仍是首次迁移来源）。

**Q7：`.env` 没生效？**
`.env` 文件被 `.gitignore` 忽略，但 `server.js` 通过 `process.env` 读取——需由启动环境（shell `export`、PM2 `env`、宝塔环境变量、或 `data/admin_key.txt`）注入，项目本身不读取 `.env` 文件。

### 许可证与合规提示

- `lib/gobang-ai` 移植自 GitHub 仓库 `lihongxun945/gobang`（**未声明许可证**，默认保留所有权利），当前仅用于个人学习与研究；商用 / 公开部署需先获原作者授权或改用明确开源许可的引擎。
- Rapfi 引擎为 **GPL-3.0**，二进制位于 `engine/rapfi/`，**禁止分发到客户端 / 前端包**（如 App 打包时需规避前端引入此引擎）。
---

> 文档与代码如有出入，以 `server.js`、`package.json`、`lib/`、`public/` 实际内容为准。部署细节见同目录 `DEPLOY.md`，协议细节见 `PROTOCOL.md`。
