# 联机五子棋 · 部署说明

技术栈：Node.js + Express + ws。零构建、零前端框架，部署即"拷贝目录 + 装依赖 + 起进程"。

> **用宝塔面板？直接看第 0 节**，那是公网访问的最短路径。
> 第 1～9 节是通用手动部署（命令行 / Docker 宿主 / 非宝塔环境）参考。

> :warning: **国内服务器 + 域名必读**：若服务器在中国大陆且域名**未做 ICP 备案**，`80 / 443` 端口会被运营商拦截，导致站点打不开、Let's Encrypt 也申请不到证书。
> 这种情况请用**路线 A（`http://服务器IP:3000`）**，或先完成备案再走路线 B。

---

## SQLite 升级部署（2026-10-04）

### 迁移内容

- `users.json`、`sessions.json`、`stats.json`、`matches.json`、`friends.json`、`ai_economy.json`、`mail.json` 首次启动时自动导入 `data/gomoku.db`。
- 新用户、会话、好友、统计、战绩、经济和站内信写入 SQLite（WAL 模式），并发写入由 SQLite 事务保护。
- 战绩不再限制数据库里的历史总量；旧前端接口仍显示最近 50 条，新增 REST 接口可查全量。
- 自定义头像改存 SQLite 的 `user_avatars` 表；启动时若发现旧版 `public/uploads/avatars/` 文件，会自动导入数据库。

### 部署步骤

1. 停止旧进程并备份数据：

   ```bash
   npm run pm2:stop
   cp -a data "data-backup-$(date +%Y%m%d%H%M%S)"
   ```

2. 安装依赖（包含 `better-sqlite3`）：

   ```bash
   npm install --production
   node -e "require('better-sqlite3')"
   ```

   若服务器没有预编译二进制且编译失败，先安装 `build-essential` / `python3`，或使用与生产系统一致的镜像构建。

3. 启动新版本，让服务自动完成 JSON → SQLite 迁移：

   ```bash
   npm run pm2:start
   curl -fsS http://127.0.0.1:3000/healthz | node -e "let s='';process.stdin.on('data',c=>s+=c).on('end',()=>{const j=JSON.parse(s);console.log(j.ok,j.storage)})"
   ```

   预期 `storage.engine` 是 `sqlite`。首次迁移后可检查核心表行数：

   ```bash
   node -e "const S=require('./lib/sqlite_store.js');['users','sessions','friends','friend_requests','stats','matches'].forEach(t=>console.log(t,S.db.prepare('select count(*) c from '+t).get().c))"
   ```

4. 保持旧 JSON 文件作为迁移来源，不要在迁移后手改它们。之后的运行时数据（含自定义头像、经济、站内信）以 `data/gomoku.db`、`data/gomoku.db-wal`、`data/gomoku.db-shm` 为准；日常备份需包含这三个文件（更安全的做法是先停服再拷贝）。确认新版本运行正常并另有备份后，旧 JSON 才可作为迁移来源删除。

5. 全量战绩查询：

   ```bash
   curl -H "Authorization: Bearer <登录token>" \
     "http://127.0.0.1:3000/api/matches?limit=50&offset=0&mode=ai&result=win&opponent=AI&from=1790000000000&to=1800000000000"
   ```

### 回滚

1. 停止新进程。
2. 恢复上一个版本代码。
3. 用部署前备份的 `data-backup-*` 目录恢复旧 JSON。
4. 再启动旧进程。不要把升级后只存在于 SQLite 的新战绩导回旧 JSON，除非另做导出。

---

## 0. 宝塔面板部署（推荐）

### 0.0 先选路线

| 路线 | 访问地址 | 是否需要域名 | WebSocket | 适用场景 |
|---|---|---|---|---|
| **A. 端口直连** | `http://服务器IP:3000` | 不需要 | `ws://` | 临时验证、内网/自用，5 分钟搞定 |
| **B. 域名 + HTTPS** | `https://gomoku.你的域名.com` | 需要 | `wss://` | 正式对外、手机浏览器长期可用 |

> 建议：**先按 A 跑通，再按 B 上域名和证书**。这样出问题能快速定位是"服务没起来"还是"反代/证书问题"。

---

### 0.1 上传代码

本地把 `gomoku` 目录**排除 `node_modules`** 打成 zip（目录里只有 4 个文件：`package.json`、`server.js`、`public/`、`*.md`）。

宝塔 → **文件** → 进入 `/www/wwwroot/` → **上传** → 上传 zip → 右键 **解压** → 得到 `/www/wwwroot/gomoku/`。

---

### 0.2 安装 Node.js 运行环境

宝塔 → **软件商店** → 搜索 **「Node.js版本管理器」** → 安装 → 打开它 → **安装 Node 版本 v20**（或 v18+，选一个 LTS）。

> 老版本宝塔没有这个插件的话，装 **「PM2管理器」** 也可以，它会自动带一个 Node 版本。

---

### 0.3 安装依赖

宝塔 → **终端**（或 文件 → 终端），执行：

```bash
cd /www/wwwroot/gomoku
npm install --production
# 国内服务器慢可用镜像：
# npm install --production --registry=https://registry.npmmirror.com
ls node_modules | head   # 看到 express、ws 即成功
```

> `ws` 有两个可选依赖（bufferutil / utf-8-validate）需要编译，**装不上也不影响运行**，npm 会自动跳过。

---

### 0.4 启动服务（宝塔 Node 项目，自带 PM2 守护）

宝塔 → **网站** → **Node项目** → **添加Node项目**：

| 配置项 | 填什么 |
|---|---|
| 项目名称 | `gomoku` |
| 项目目录 | `/www/wwwroot/gomoku` |
| 启动文件 | `server.js` |
| 项目端口 | `3000` |
| Node版本 | 选刚装的 v20 |
| 运行方式 | 默认即可 |
| 开机启动 | 开启（建议） |

提交后状态应变为「运行中」。点 **日志** 能看到：

```
五子棋服务已启动：http://0.0.0.0:3000  (WebSocket 路径 /ws)
```

> 若你的宝塔没有「Node项目」菜单，用 **PM2管理器** → 添加项目：目录 `/www/wwwroot/gomoku`，启动文件 `server.js`，端口 `3000`。

---

### 0.5 放行端口（**最容易漏，漏了就是"页面打不开"**）

需要**两处**都放行 `3000`：

1. **宝塔** → **安全** → **系统防火墙** → 放行端口 → 填 `3000` → 备注「五子棋」
2. **云厂商控制台** → **安全组** → 添加入站规则：TCP `3000`，来源 `0.0.0.0/0`

（阿里云叫安全组、腾讯云叫防火墙、华为云叫安全组，位置都在云服务器实例详情页里。）

验证：本机浏览器打开 `http://服务器IP:3000/healthz`，应返回 `{"ok":true,...}`。

---

### 0.6 路线 A 完成：手机直接访问

手机浏览器打开 `http://服务器IP:3000` → 建房 → 把链接发给朋友即可开打。

> HTTP 页面下前端自动走 `ws://`，功能完全一致。缺点是地址带端口、且不是加密连接。

---

### 0.7 路线 B：域名 + HTTPS + wss

#### ① 域名解析

域名服务商后台添加 **A 记录**：`gomoku` → `服务器公网IP`，TTL 默认。等 1～5 分钟生效。

#### ② 宝塔建站

宝塔 → **网站** → **添加站点**：

| 配置项 | 填什么 |
|---|---|
| 域名 | `gomoku.你的域名.com` |
| 根目录 | `/www/wwwroot/gomoku/public` |
| PHP版本 | **纯静态**（本项目是 Node，不需要 PHP） |
| 数据库 | 不创建 |
| FTP | 不创建 |

#### ③ 申请 SSL 证书

站点 → **设置** → **SSL** → **Let's Encrypt** → 选域名 → 申请 → 打开 **强制 HTTPS**。

> 申请失败多为 80 端口未放行或解析未生效，先确认 `http://域名` 能打开再申请。

#### ④ 配置反向代理（关键）

**更省事的做法**：直接回到 **网站 → Node项目 → gomoku → 设置 → 域名管理**，填入 `gomoku.你的域名.com` 并开启「外网映射」，宝塔会自动生成反代。然后**跳到第 ⑤ 步补长连接配置**即可。

**标准做法**（推荐，可控性更好）：站点 → **设置** → **反向代理** → **添加反向代理**：

| 配置项 | 填什么 |
|---|---|
| 代理名称 | `gomoku` |
| 目标 URL | `http://127.0.0.1:3000` |
| 发送域名 | `$host` |
| 代理目录 | `/` |
| 缓存 | **关闭** |

#### ⑤ 补 WebSocket 长连接配置（必须，否则一段时间就掉线）

站点 → **设置** → **配置文件**，在对应的 `server { ... }` 里加上下面这段（放在 `listen 443 ...` 那个 server 块内）：

```nginx
    # ===== WebSocket 支持：五子棋实时对局依赖它 =====
    location /ws {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 3600s;    # 不加这行，空闲 60s 会被 Nginx 掐断
        proxy_send_timeout 3600s;
    }
```

同时在 `server {` 之前（文件或 nginx 的 `http` 块里）确认有这个 map，没有就加上：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

保存后宝塔会校验并自动重载 Nginx。若提示 `map` 放错位置，把它挪到 **软件商店 → Nginx → 配置修改** 里 `http {` 的开头。

> 说明：宝塔 GUI 生成的反代通常已带 `Upgrade` 头，但**默认没有 `proxy_read_timeout`**，不加的话长时间不发消息会被断开。上文把 `/ws` 单独拎出来配，最稳妥。
>
> 手动改完配置文件后，**不要再点反向代理界面的"保存/编辑"**，否则宝塔会用模板覆盖你手写的内容（需要改就再来这里改配置文件）。
>
> 两个兜底：① 如果 `/ws` 这段没生效（请求仍落到 `/`），把它改成 `location ^~ /ws`；② 如果你的宝塔用的是 **Apache** 而不是 Nginx，改用 [第 6 节的 Nginx 配置](./DEPLOY.md) 不通，需要在 Apache 里启用 `mod_proxy_wstunnel` 并加：
> `ProxyPass /ws ws://127.0.0.1:3000/ws`　`ProxyPassReverse /ws ws://127.0.0.1:3000/ws`

#### ⑥ 验收

手机浏览器打开 `https://gomoku.你的域名.com`：

- 页面能打开 → 静态资源 OK
- 建房、朋友加入、落子对方能立刻看到 → WebSocket OK
- 地址栏是锁形图标，F12 → Network → WS 能看到 `/ws` 状态 **101 Switching Protocols**

---

### 0.8 宝塔部署排错清单

| 现象 | 原因 / 处理 |
|---|---|
| 页面打不开、连接超时 | 端口没放行（0.5 的两处）；Node 项目状态不是「运行中」，去看日志 |
| 页面能开，提示"连接已断开，正在重连" | WebSocket 没通：反代缺 `Upgrade` 头，或只配了 80 没配 443 的 `/ws` |
| HTTPS 页面控制台报 mixed content | 说明实际还在用 `ws://`，确认站点已开强制 HTTPS、且你用 `https://` 访问 |
| 玩几分钟后掉线 | `proxy_read_timeout` 没加；或开了 CDN（需在 CDN 控制台开启 WebSocket 支持） |
| `npm install` 报 gyp 错误 | `ws` 的可选依赖编译失败，可忽略；或 `npm install --production --no-optional` |
| 改了代码不生效 | 宝塔 Node项目 → 重启项目（或 `pm2 restart gomoku`）；若改了前端（css/js），记得把 `index.html` 里的版本号 +1（见下方"版本号与缓存"） |
| 想换端口 | 改 Node 项目端口 → 同步改反代目标 URL 和防火墙放行端口 |

---

## 1. 服务器上安装 Node.js（LTS）

### Ubuntu / Debian（推荐用 NodeSource 的 LTS 源）

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v   # 需 >= v18
npm -v
```

### CentOS / RHEL / Rocky

```bash
curl -fsSL https://rpm.nodesource.com/setup_lts.x | sudo bash -
sudo yum install -y nodejs
node -v
```

> 若服务器在国内且访问 NodeSource 慢，可改用 Node.js 官方二进制包：
> `wget https://nodejs.org/dist/v20.18.0/node-v20.18.0-linux-x64.tar.xz` → 解压 → 把 `bin` 目录加入 `PATH`。

---

## 2. 上传代码并安装依赖

```bash
# 本地打包（不要带 node_modules）
#   gomoku/ 目录整体上传即可
scp -r gomoku root@<你的服务器IP>:/opt/

# 服务器端
cd /opt/gomoku
npm install --production     # 只装 express + ws，共约 70 个包
```

目录最终形态：

```
/opt/gomoku
├── package.json
├── server.js
├── public/
│   ├── index.html
│   ├── css/style.css
│   └── js/game.js
└── node_modules/
```

---

## 3. 启动服务

```bash
# 前台试跑（确认能起来）
PORT=3000 node server.js
# 看到：五子棋服务已启动：http://0.0.0.0:3000  (WebSocket 路径 /ws)

# 健康检查
curl http://127.0.0.1:3000/healthz
# {"ok":true,"rooms":0,"uptime":3}
```

可用环境变量：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PORT` | `3000` | HTTP / WebSocket 监听端口 |
| `HOST` | `0.0.0.0` | 监听网卡，一般不用改 |
| `GRACE_MS` | `120000` | 掉线后房间保留多久（毫秒）。手机切后台会被判掉线，房间保留这段时间等玩家回来；调大更不容易掉房，调小回收更快 |

> **重要**：不要把 `GRACE_MS` 设得比 Nginx 的 `proxy_read_timeout` 还长，否则玩家还没来得及重连，代理层连接就先被掐了。默认 120s 对应前面推荐的 3600s，是安全的。

---

## 4. 用 pm2 守护进程（崩溃自动拉起、开机自启）

```bash
npm install -g pm2

cd /opt/gomoku
pm2 start server.js --name gomoku      # 或：npm run pm2:start
pm2 save                                # 保存进程列表
pm2 startup                             # 生成开机自启服务，按提示再执行一次它输出的命令

# 常用运维
pm2 status
pm2 logs gomoku          # 看实时日志
pm2 restart gomoku       # 重启
pm2 stop gomoku          # 停止
pm2 monit                # CPU / 内存面板
```

带环境变量启动：

```bash
PORT=8080 pm2 start server.js --name gomoku --update-env
```

---

## 5. 开放防火墙端口

### 云服务器安全组（最容易漏的一步）

在云厂商控制台（阿里云 / 腾讯云 / 华为云等）的**安全组**里放行 TCP 端口，例如 `3000`（或你改成 `80` / `443`）。仅配系统防火墙而不配安全组，外网依然连不上。

### 系统防火墙

```bash
# Ubuntu / Debian (ufw)
sudo ufw allow 3000/tcp
sudo ufw reload

# CentOS / RHEL (firewalld)
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload

# 或直接 iptables
sudo iptables -I INPUT -p tcp --dport 3000 -j ACCEPT
```

验证（从手机或另一台机器）：

```bash
curl http://<服务器IP>:3000/healthz
```

---

## 6. 用 Nginx 反向代理 + HTTPS → 使用 wss://

**为什么必须 HTTPS：** 浏览器安全策略下，`https://` 页面里不允许建立 `ws://` 连接（会被拦截为 mixed content）。只要你的站点上了 HTTPS，前端代码会自动使用 `wss://`（`game.js` 里已根据 `location.protocol` 自动切换），所以**只要 Nginx 把 HTTPS 终结掉即可，后端代码无需改动**。

### 6.1 Nginx 配置

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name gomoku.example.com;
    return 301 https://$host$request_uri;   # HTTP 全量跳 HTTPS
}

server {
    listen 443 ssl http2;
    server_name gomoku.example.com;

    ssl_certificate     /etc/letsencrypt/live/gomoku.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/gomoku.example.com/privkey.pem;

    # ---- WebSocket 代理：关键是 Upgrade / Connection 两个头 ----
    location /ws {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;   # 长连接不要被 60s 默认超时掐断
        proxy_send_timeout 3600s;
    }

    # ---- 静态页面 ----
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 6.2 免费证书

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d gomoku.example.com
# 自动续期已由 systemd timer 接管，可用 certbot renew --dry-run 验证
```

### 6.3 反向代理后的注意事项

- `wss://` 由 Nginx 终结 TLS 后以普通 `ws://` 转发给 `127.0.0.1:3000`，**后端仍是明文 WebSocket，不需要在 Node 里配置证书**。
- 若代理层有请求超时（Nginx 默认 `proxy_read_timeout 60s`），务必按上面改成 3600s，否则空闲 60 秒就会被断开；服务端自身也有 15s 心跳 + 30s 超时兜底。
- 若部署在 CDN / SLB 后面，同样要在控制台开启 WebSocket 支持（部分厂商需手动开）。

---

## 7. 使用方式

1. 玩家 A 打开 `https://gomoku.example.com/` → 点「创建房间」→ 复制邀请链接发给 B。
2. 玩家 B 打开链接（形如 `https://gomoku.example.com/?room=HFXDU3`）→ 自动加入。
3. 或 B 手动在大厅输入 6 位房间号 → 点「加入」。

---

## 8. 常见问题排查

| 现象 | 排查方向 |
|---|---|
| 页面能开，但一直"连接已断开" | 安全组 / 防火墙未放行端口；Nginx 少配 `Upgrade` 头 |
| HTTPS 页面报 mixed content | 说明前端仍走 `ws://`，检查是否用 `https://` 访问站点 |
| 落子没反应 | 看 `pm2 logs gomoku`，服务端会以 `error` 消息回具体原因（非你的回合 / 位置已占用） |
| 手机上双击会放大 | 已用 viewport + `touch-action` + `preventDefault` 处理；若仍出现，清理浏览器缓存确认拿到最新 `index.html` |
| 房间号冲突 | 服务端生成时会 `while` 校验唯一性，不会重复 |

---

## 8.5 版本号与缓存（已全自动，不用再手动改）

### 现状：版本号由服务端按文件 mtime 自动注入

- `index.html` 里的 css/js 引用一律写成 `?v=dev` —— **这是占位符，不要去改它**。
- 服务端返回 HTML 时，把每个 `?v=dev` 替换成该文件 mtime 的 36 进制版本号，
  **每个文件各自独立**。文件一改 mtime 就变，版本号自动跟着变。
- `index.html` 本身 `Cache-Control: no-store`，浏览器每次都拿最新 HTML。
- `css` / `js` 走 `immutable` 长缓存，靠 URL 上的版本号指纹失效。

**所以：以后改完 css/js，什么都不用做，直接上传 + 刷新页面即可。**

### ⚠️ 两个必须注意的点

1. **`server.js` 必须一起更新，不能只传前端。**
   旧版 `server.js` 不认识 `?v=dev` 这套机制，会把 `?v=dev` 原样发给浏览器；
   而静态资源带 `immutable` 长缓存，浏览器会把 `xxx.css?v=dev` **永久缓存**下来，
   之后再怎么更新 css，用户都看不到（必须手动清缓存才能恢复）。
   这是本次更新最容易踩的坑。

2. **文件 mtime 必须真的变化。**
   `git pull` / scp / 宝塔上传都会更新时间戳；
   但如果用 `rsync --times` 且源文件时间戳没变，版本号就不会变，用户也就拿不到新文件。

### 排错

若改完仍不生效：用**无痕窗口或换一个浏览器**访问。新窗口能看到新界面 = 100% 是旧窗口缓存。

---

## 8.7 增量更新：需要新增 / 替换 / 删除哪些文件

### 本次更新（S1+S2 视觉重构）的精确清单

**① 必须新增（6 个文件）**

| 路径 | 说明 |
|---|---|
| `public/css/tokens.css` | 设计令牌 + 三套主题（paper / wood / night） |
| `public/css/app.css` | 全站样式，**替代原来的 style.css** |
| `public/js/tokens.js` | 主题读写与持久化 |
| `public/js/board.js` | 棋盘渲染（木纹底座 / 棋子 / 动画 / 准星） |
| `public/js/haptics.js` | 震动反馈封装 |
| `public/js/share.js` | 战绩分享图合成 |

> ✅ **依赖没有变化**：`package.json` / `package-lock.json` 本次未改动，
> 服务器上**不需要**重新 `npm install`。

**② 必须替换（3 个文件）**

| 路径 | 不替换会怎样 |
|---|---|
| `server.js` | **会出现缓存锁死**（见 8.5 第 1 点）。改完必须重启 Node |
| `public/index.html` | 少了设置抽屉、手数、掉线倒计时的结构，样式引用也还指向旧文件名 |
| `public/js/game.js` | 业务与 UI 逻辑主体，改了音效 / 设置 / 结算 / 倒计时等 |

**③ 必须删除（1 个文件）**

| 路径 | 原因 |
|---|---|
| `public/css/style.css` | 已被 `app.css` 取代。**不删也不会报错**，但会留一份过期样式，日后容易改错文件 |

**④ 不要上传（开发件，与运行无关）**

| 路径 | 说明 |
|---|---|
| `tools/` | `fake-opponent.js`（联调假对手）、`smoke-test.js`（Node 侧冒烟测试） |
| `docs/` | 方案文档、实施规格、验证截图（约 4.5MB） |

> ⚠️ **绝对不要覆盖或删除服务器上的 `data/` 目录** ——
> 账号（`users.json`）、登录会话（`sessions.json`）、好友关系（`friends.json`）
> 都在里面。它已在 `.gitignore` 中排除，`git pull` 不会动它；
> 但如果用「整包上传覆盖」的方式部署，务必先确认没有连 `data/` 一起覆盖。

### 操作步骤

```bash
# 0. 记录当前版本 + 备份账号数据（稳妥做法）
cd /www/wwwroot/gomoku
git log --oneline -1                  # 记下版本号，回退时要用
cp -r data /tmp/gomoku-data-backup    # 备份后再动手

# 1. 取新代码（git 方式；若用宝塔文件管理器上传，跳过这步）
git pull

# 2. 删除已废弃文件
rm -f public/css/style.css

# 3. 重启 Node（server.js 改了，必须重启；只改 html/css/js 可以不重启）
pm2 restart gomoku
#   宝塔面板用户：Node 项目 → 点「重启」

# 4. 验收（最后一条是关键）
curl -s http://127.0.0.1:3000/healthz                          # {"ok":true,...}
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3000/css/app.css   # 200
curl -s http://127.0.0.1:3000/ | grep -E "app.css|board.js"   # 应看到 ?v=xxxxxx，不是 ?v=dev
```

**最后一条命令是本次部署的关键验收点。**
如果输出里还是 `?v=dev`，说明 `server.js` 没生效（没更新或没重启）——
**此时先不要让用户访问**，否则会把 `?v=dev` 这个固定 URL 永久缓存进各人浏览器。

### 如果用了宝塔「整站备份 / 一键迁移」

注意它可能连同 `data/` 一起覆盖或回滚。迁移前单独备份 `data/`，迁移后确认
`data/users.json` 里的账号数量没变。

### 若你的 Nginx 改成过「静态文件直连」

本项目标准的 Nginx 配置（见 6.1）是 `location / { proxy_pass ... }`，
**所有请求都转发给 Node**，所以 mtime 版本号注入正常工作。

如果你后来为了提速，把静态资源改成让 Nginx 直接读磁盘（例如加了
`location ~* \.(css|js)$ { root /www/wwwroot/gomoku/public; }`），
那么 `?v=dev` 就没人替换了，会立刻触发上面说的缓存锁死。
此时要么改回全量代理，要么把 `index.html` 的引用改回手写数字版本号。

---

## 8.6 账号管理（查看 / 删除账号）

游戏有账号系统（注册 / 登录），账号数据保存在项目目录的 `data/users.json`。
管理账号用自带的命令行工具 `admin.js`，在**宝塔「终端」**里执行即可。

### 常用命令

```bash
cd /www/wwwroot/gomoku     # 换成你的实际项目路径
node admin.js              # 不带参数：显示用法 + 账号列表
node admin.js list         # 列出所有账号（用户名 / 在线状态 / 注册时间 / 用户ID）
node admin.js del 小明     # 删除账号（会二次确认，输入 yes 生效）
node admin.js del 小明 -y  # 删除账号（跳过确认）
```

> ⚠️ **宝塔环境注意**：宝塔安装的 Node **不在全局 PATH** 里，直接敲 `node` 会报
> `Command 'node' not found, but can be installed with: apt install nodejs`。
> 不要按提示去 `apt install`（会装出另一个版本造成混乱），用下面两种方式之一：

**方式一（推荐）：用自带的包装脚本，自动定位 node**

```bash
cd /www/wwwroot/gomoku
bash admin.sh list           # 列出账号
bash admin.sh del 小明 -y    # 删除账号
```

**方式二：用完整路径调用 node**

```bash
ls -d /www/server/nodejs/*/bin/node          # 先查看宝塔装了哪些版本
/www/server/nodejs/v18.20.0/bin/node admin.js list   # 版本号换成实际看到的
```

### 删除账号会连带清理

- 该账号的**全部登录会话**（否则旧 token 还能自动登录）
- 该账号的**用户记录**
- 若该账号**当前在线**：立即踢下线；若正在对局中，会正常通知对手

### 关于端口

管理接口使用独立端口（默认 = 游戏端口 + 100，例如游戏 3000 → 管理 3100），
并且**只绑定 127.0.0.1**——公网无法访问，所以不需要额外密码。
如果你自定义过 `PORT`，调用时请保持一致：

```bash
PORT=3000 node admin.js list
# 或直接指定管理端口
ADMIN_PORT=3100 node admin.js list
```

### 注意

- 管理工具依赖**游戏服务正在运行**（它通过本机接口操作，保证内存与文件同步）。
- 不建议用宝塔文件管理器直接编辑 `data/users.json`：格式写错会导致服务读取异常，用 `admin.js` 更安全。

---

## 8.8 增量更新：断线重连身份修复（座位令牌）

**修的是什么**：断线重连有概率与对方**互换身份**（也表现为"本人重连被提示房间已满，卡在旧棋盘上"）。

**根因**：座位原来只表示"哪条连接坐在这儿"，重连时服务端靠"哪个座位空着"倒推身份。
双方同时掉线（都切后台 / 同一条网络抖动）后两个座位都是空的，
**先回来的一方会被无条件分到黑位**，与它原本执什么颜色无关。

**改法**：新增「座位令牌」——每个座位一枚随机令牌，随 `room_created` / `room_joined` / `game_start`
按人下发，前端与房间号成对存进 `localStorage`；重连时带回来即可**精确还座**。
协议细节见 `PROTOCOL.md` §七。

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `server.js` | **替换** | 座位令牌的签发 / 还座 / 顶号；`handlePlayerLeave` 增加"座位是否仍属于这条连接"的校验；补 `SEAT_REQUIRED`、`seat_taken_over` |
| `public/js/game.js` | **替换** | 令牌持久化（`gomoku_seat`）、重连与手动加入都带上 `seat`、处理 `seat_taken_over` / `SEAT_REQUIRED`；加入房间不再提前改写本地房间号 |
| `PROTOCOL.md` | 替换 | 协议文档（新增 §七 座位令牌、`seat_taken_over`、`SEAT_REQUIRED`） |
| `tools/protocol-test.js` | **新增** | 协议回归测试，43 项断言（不需要浏览器、自己拉起服务端） |
| `public/index.html` / `public/css/*` / `tools/smoke-test.js` | 不动 | |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git log --oneline -1                  # 记下当前版本，回退要用
cp -r data /tmp/gomoku-data-backup    # 老规矩：先备份账号数据

git pull                              # 或上传 server.js + public/js/game.js + PROTOCOL.md

pm2 restart gomoku                    # server.js 改了，必须重启（宝塔：Node 项目 → 重启）

# 验收
curl -s http://127.0.0.1:3000/healthz                          # {"ok":true,...}
curl -s http://127.0.0.1:3000/ | grep -E "game.js"             # 应看到 ?v=xxxxxx，不是 ?v=dev
node tools/protocol-test.js                                    # 通过 43 项，失败 0 项
```

> **`server.js` 与 `public/js/game.js` 必须一起上线。**
> 只发前端：新前端会发 `seat` 字段，旧服务端忽略它 → 退回修复前的行为（不会更糟，但也没修好）。
> 只发服务端：旧前端不带令牌 → 单方掉线仍能正确还座，但双方同时掉线时会收到 `SEAT_REQUIRED`
> 并提示回大厅（**不会再静默互换身份**）。
> 两类中间态都不会比现在更差，但仍然建议同时发。

### 上线后的自检（1 分钟）

1. 两个浏览器（或手机 + PC）建一局，各落几手；
2. **两边同时刷新页面**（模拟双方同时掉线），先刷新的那个先回到房间；
3. 双方界面上的「我方」颜色应与刷新前一致 —— 不再互换。

### 已知边界

- 双方都掉线、且两端都清掉了浏览器数据（令牌丢失）时，服务端无法判定身份，
  会明确拒绝（`SEAT_REQUIRED`）而不是瞎猜；此时重新开一局即可。
- 同一浏览器开两个标签页打同一局：后开的窗口会接管座位，先开的窗口收到
  「对局已在别处恢复」并停止重连（这样才不会两个窗口互抢座位）。
- `localStorage` 新增了一个键 `gomoku_seat`（存 `{room, seat}`），主动退出或房间失效时会自动清除。
- 这是**协议扩展**，不需要动 `data/`，也不需要 `npm install`。

---

## 8.9 增量更新：人机对战（AI 房）

**新增能力**：大厅可选「联机对战 / 挑战 AI」；选 AI 时再选难度（简单 / 普通 / 困难），
建房即开局，无需等待对手。AI 在服务端运行（`ai.js`，棋型评分 + alpha-beta），
与真人房共用同一套对局逻辑（悔棋 / 重开 / 断线恢复 / 战绩图全部可用）。
协议细节见 `PROTOCOL.md` §八。

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `ai.js` | **新增** | AI 引擎（纯函数）。三档难度：easy 只防一手败；normal 棋型评分 + 攻防权衡；hard 加 alpha-beta 迭代加深（1.5s 思考预算，`AI_HARD_BUDGET_MS` 可调） |
| `server.js` | **替换** | 抽出 `applyMove` / `performUndo` 内核（真人路径不变）；AI 房创建 / `scheduleAiMove` 调度 / 重开代确认 / 悔棋自动同意 / 掉线即销毁；`game_start` 等报文新增 `mode` / `level` 字段 |
| `public/index.html` | **替换** | 大厅新增模式选择（联机对战 / 挑战 AI）与难度选择（简单 / 普通 / 困难） |
| `public/js/game.js` | **替换** | 模式 / 难度状态、AI 房建房分支、`opponent_thinking` 处理、AI 头像与"思考中"显示 |
| `public/css/app.css` | **替换** | `.lobby__mode`、`.player__avatar--bot`、AI 思考脉冲动画 |
| `tools/protocol-test.js` | **替换** | 新增 [8][9][10] 段 AI 房端到端测试（总计 71 项） |
| `tools/ai-test.js` | **新增** | AI 引擎单元测试，48 项（一手胜 / 必挡 / 挡活三 / 档位棋力 / 时间预算） |
| `tools/ai-match.js` | **新增** | AI 档位对局实验（可选，验证 hard > normal > easy 的胜率单调性） |
| `PROTOCOL.md` / `DEPLOY.md` | 替换 | 协议与部署文档 |
| `data/` | **不动** | 账号数据，绝对不要覆盖 |
| `node_modules` | 不动 | 无新依赖，不需要 `npm install` |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git log --oneline -1                  # 记下当前版本，回退要用
cp -r data /tmp/gomoku-data-backup    # 老规矩：先备份账号数据

git pull                              # 或上传上表中的全部文件（ai.js 必须与 server.js 同级目录）

pm2 restart gomoku                    # server.js 改了，必须重启（宝塔：Node 项目 → 重启）

# 验收（三条都要过）
node tools/ai-test.js                 # 通过 48 项，失败 0 项
node tools/protocol-test.js           # 通过 71 项，失败 0 项
curl -s http://127.0.0.1:3000/ | grep -E "game.js"   # ?v=xxxxxx 而不是 ?v=dev
```

> **`ai.js` 与 `server.js` 必须同时上线**：`server.js` 第一屏就 `require('./ai.js')`，
> 缺了它进程直接起不来（报 `Cannot find module './ai.js'`）。
> 前端三个文件与旧服务端兼容（旧服务端忽略 `mode` 字段，走真人房），可分开发但建议一起。

### 上线后的自检（2 分钟）

1. 打开首页 → 点「挑战 AI」→ 应出现难度选择，大按钮变成「与 AI 对局」；
2. 点「与 AI 对局」→ 直接进入对局页，对手显示「AI·普通」；
3. 落一子 → 约 1 秒内 AI 应招；点「悔棋」→ 立即撤销（无需对方同意）；
4. 点「再来一局」→ 立即重开且黑白互换，AI 执黑时会自动先手落子；
5. 中途刷新页面 → 应回到原对局，轮到 AI 时它会继续落子。

### 已知边界

- AI 房玩家退出 / 掉线后**房间立即销毁**（没有 120s 宽限期）——AI 不会掉线，等待无意义。
- AI 房没有座位令牌语义：房间里只有一个真人，"按空位补位"不会互换身份。
- 困难档 AI 每步最多思考 1.5 秒（服务器可配 `AI_HARD_BUDGET_MS`），不会阻塞其他房间。
- AI 棋力定位：困难 ≈ 业余中上；简单档故意不防活三，供新手练习。

---

## 8.10 增量更新：AI 只保留最高难度（困难）

**改动**：AI 对战去掉难度选择，固定为最高难度（困难档，含 VCF 算杀 + 组合棋型识别 +
alpha-beta 深搜索）。引擎内部仍保留三档实现（供单测与实验），但产品层不再暴露 easy/normal。

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `public/index.html` | **替换** | 删掉难度选择器 `#set-level`；模式提示改为"挑战最高难度 AI" |
| `public/js/game.js` | **替换** | `lobbyAiLevel` 固定 `'hard'`；删掉难度选择绑定与 `elSetLevel` |
| `server.js` | **替换** | `AI_LEVELS` 只留 `hard`；`startAiRoom` / `handleCreateRoom` 固定 hard，忽略客户端 `level`；删掉 `AI_LEVEL_INVALID` 校验 |
| `tools/protocol-test.js` | **替换** | AI 房测试适配 hard 档慢思考（轮询替代固定 sleep）；断言固定 hard |
| `tools/smoke-test.js` | **替换** | 静态检查改为"不再含难度选择器、固定 hard" |
| `PROTOCOL.md` | 替换 | §八 改为"只保留 hard" |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git pull
pm2 restart gomoku        # server.js 改了，必须重启
node tools/ai-test.js     # 通过 63 项
node tools/protocol-test.js  # 通过 68 项
```

> 前端（index.html / game.js）与旧服务端兼容：旧服务端会忽略 `level` 字段。
> 但旧前端若带难度选择，新服务端也一律按困难档建房 —— 所以前后端一起发最稳。

---

## 8.11 增量更新：AI 搜索提速（行评分查表）

**改动**：棋型判定从「拼 9 字符串 + 14 条正则」改为「9 格窗口压成整数查表」，并且叶子节点
不再做全宽候选点评分。**棋型规则与判定结果完全不变**，只是同一件事做得快一个量级。

### 为什么要改（实测数据）

| 指标 | 改前 | 改后 |
|---|---|---|
| 单搜索节点耗时 | 225µs | **49µs**（4.6x） |
| 同等时间搜索节点数 | 1x | **4.0x** |
| 1500ms 预算下平均跑满深度 | 5.96 层 | **7.54 层** |
| 平均每手响应 | 800ms | **498ms** |

之前「深度上限 8」形同虚设 —— 绝大部分局面跑到 6 层就被预算卡断。现在能真正跑满。
（顺带修了一处判定 bug：叶子节点原用 `attack + 0.9*defend >= FIVE`，当 attack=活四、
defend=五连时会把「对方能成五」误判成「我方赢」；现在统一只看攻击侧。）

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `ai.js` | **替换** | 新增 `LINE_SCORE` 查表 + `lineCodeAt`；`patternScoreAt` / `lineScoresAt` 改走查表；`alphaBeta` 叶子节点不再全宽重排；`_internal` 多导出 `PATTERNS` / `lineCodeAt` / `lineScoreTable` 供对拍 |
| `tools/ai-test.js` | **替换** | 新增 [1b] 段：随机盘面 2 万个方向窗口，断言「查表分 === 正则分」 |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git pull
pm2 restart gomoku        # ai.js 是服务端 require 的，必须重启
node tools/ai-test.js     # 通过 65 项
node tools/protocol-test.js  # 通过 68 项
node tools/smoke-test.js     # 通过 80 项
```

### 注意事项

- **首次调用会有约 120ms 的一次性建表开销**（4^9 = 262144 项，占 1MB），进程内只付一次。
  建在第一次 AI 落子时，不影响启动。
- **不要手写快速状态机替换查表**：查表是用同一套 `PATTERNS` 生成的纯缓存，
  规则改了会自动跟着变；手写状态机等于维护第二套判定逻辑，必然漂移。
  `[1b]` 那道断言就是防这个的。
- **不要顺手把搜索宽度 / 深度调大**：实测加宽（根宽 10→12、深层宽 4→6）会让
  跑满深度从 7.67 掉到 6.20、单手耗时 660ms→903ms，广度换不来深度。
  深度上限提到 10 也只有 7.54→7.62，已经饱和。

---

## 8.12 增量更新：AI 防守算杀（检测对手的连续冲四）

**改动**：新增 `defendAgainstVcf`，在 hard 档"自己有 VCF"检测之后、主搜索之前，对**对手**
也跑一次 `vcfMove` —— 命中就按评分枚举落点，第一个"落子后对手杀棋消失"的点作为破杀点返回。

### 为什么要改（实测数据）

之前 `vcfMove` 只检测我方的杀，从不检测对手的杀：当对手有一条比主搜索深度更深的
冲四连杀时 AI 看不到、不去防。实测缺口：**AI 落子后对手仍握有 VCF 必杀的手数占 18.7%**。
补上防守算杀后降到 **7.6%**（下降 59%），单手耗时无回退（约 284ms）。

剩下的 7.6% 属于活三杀（VCT）、或 AI 落子后才成型的杀，是后续叶子算杀 / VCT 的范畴。

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `ai.js` | **替换** | 新增 `defendAgainstVcf`；`pickHard` 在自身 VCF 之后加防守算杀分支；`_internal` 导出 `defendAgainstVcf` |
| `tools/ai-test.js` | **替换** | 新增 [8f] 段：对手有 VCF → 破杀点命中；反例无杀 → 返回 null |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git pull
pm2 restart gomoku        # ai.js 是服务端 require 的，必须重启
node tools/ai-test.js     # 通过 70 项
node tools/protocol-test.js  # 通过 68 项
node tools/smoke-test.js     # 通过 80 项
```

### 注意事项

- 防守算杀只在 hard 档生效（normal/easy 不做算杀，否则三档差异会消失）。
- 破不了的情况会降级走主搜索（对手已是双冲四/活四级必胜时），这是正确行为。

---

## 8.13 增量更新：AI 引擎替换为 gobang

**改动**：AI 房的落子决策从自研 `ai.js` 换成整体移植的 gobang 引擎（`lib/gobang-ai/`，
源自 lihongxun945/gobang，ESM→CJS 转换）。`ai.js` 不再用于落子，只保留台词功能。

### 对比结论

gobang（depth 4、约 150ms/步）vs 原 ai.js（hard 档 400ms 预算）：**9 : 1**。
优势来自科学化评分表（base=10 等比）+ 增量评估 + 纯 alpha-beta。

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `lib/gobang-ai/`（8 文件） | **新增** | gobang 引擎：board/cache/config/eval/minmax/position/shape/zobrist，均 CJS |
| `server.js` | **替换** | AI 房维护 `room.gbBoard`（有状态 Board）随落子同步；落子改调 `gobangMinmax` |
| `tools/compare-gobang.js` | **新增** | 两引擎互搏对比脚本 |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git pull
pm2 restart gomoku        # server.js 改了，必须重启
node tools/ai-test.js        # 通过 70 项
node tools/protocol-test.js  # 通过 68 项
node tools/smoke-test.js     # 通过 80 项
```

### 注意事项

- **`lib/gobang-ai/` 目录必须随 server.js 一起部署**（server.js 第一屏就 `require` 它，缺了起不来）。
- **`enableVCT` 必须保持 false**：gobang 的 VCT（活三+冲四威胁搜索）在"无杀棋/边线局面"单步会
  耗时 5~9.5s 阻塞事件循环。禁用后 ~150ms 稳定，棋力不降（9:1 结论不变）。
- `GOBANG_DEPTH` 默认 4，可用环境变量调（服务器弱调小、要更强调大，但 6 会到秒级）。
- **⚠️ 许可证**：gobang 仓库 `license: null`（保留所有权利），当前仅限个人学习研究。
  商用或公开部署前须联系原作者 `lihongxun945` 授权，或改用明确开源许可的引擎。

---

## 8.14 增量更新：AI 房实时胜率展示

**功能**：AI 每落一子，对局页实时显示玩家方胜率 —— 双色对拉条 + 百分比 + 最近 40 回合趋势折线。

### 胜率的计算逻辑（为什么这样算）

三种可选路线的取舍：
- **历史对战数据**：个人项目样本太少，统计无意义 → 否；
- **蒙特卡洛模拟推演**：每步要跑几十次完整对局，JS 单线程会卡对局 → 否；
- **局面评估映射（采用）**：gobang 引擎每次思考本来就产出局面评估值 `value`（AI 视角分），
  把它经"棋型语义锚点"分段插值映射成百分比 —— **零额外搜索成本**，AI 落子即更新。
  锚点：活二≈53%、活三≈70%、冲四/活四≈92%、接近成五≈99%；输出夹在 [1,99]（不给死局面的
  0/100，保留悬念）。`value` 的**符号方向**决定强势方是谁（v>0 AI 优 / v<0 玩家优），勿丢。

### 触发时机

- **AI 每次落子后**：`scheduleAiMove` 里把 `evalToPlayerRate(value)` 随 `move` 广播下发
  （`winRate` 字段，玩家方视角）。玩家落子不更新（那时局面评估还没有新值）。
- **开局/重开/恢复**：前端收到 `game_start` 时重置为 50%，趋势清空（恢复局面时历史无法重建）。
- **终局**：`game_over` 时定格 —— 赢 100 / 输 0 / 平 50。

### 界面展示

- 组件 `#winrate`：左（我方 %）| 双色对拉条（中间 50% 朱砂刻度线）| 右（对手 %）| 趋势折线(SVG)。
- 颜色取**棋子令牌**（`--c-stone-black-*` / `--c-stone-white-*`），由 `data-self` 切换我方颜色，
  三套主题自动适配；窄屏（<420px）隐藏趋势线保棋盘空间；桌面端进右栏 grid（`winrate` 区域）。
- 仅 AI 房显示（`state.mode === 'ai'` 才摘 `hidden`）；CSS transition 平滑过渡防跳变。

### 性能

- 服务端零额外搜索（复用 AI 思考的 value），映射是几次浮点乘加（微秒级）；
- `move` 报文多一个数字字段，无额外消息往返；
- 前端每回合一次 DOM 样式更新 + SVG polyline 重设 points（≤40 点），无 canvas 重绘。

### 本次改动的文件清单

| 文件 | 动作 | 说明 |
|---|---|---|
| `server.js` | 替换 | 新增 `WINRATE_ANCHORS` / `evalToPlayerRate`；`applyMove` 增 `winRate` 参数并随 move 广播 |
| `public/index.html` | 替换 | 新增 `#winrate` 组件（状态栏下方） |
| `public/css/app.css` | 替换 | 新增 `.winrate` 系列样式（棋子令牌 + 桌面 grid 区域） |
| `public/js/game.js` | 替换 | 新增 `renderWinRate` / `resetWinRate` / `pushWinRate`；接 `move` / `game_start` / `game_over` |
| `tools/protocol-test.js` | 替换 | 新增断言：AI 落子 move 携带 0-100 的 winRate |
| `PROTOCOL.md` | 替换 | `move` 报文补充 `winRate` 字段说明 |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git pull
pm2 restart gomoku        # server.js 改了，必须重启
node tools/protocol-test.js  # 通过 69 项
node tools/smoke-test.js     # 通过 80 项
```

### 注意事项

- `winRate` 是**玩家方视角**；AI 兜底落子（引擎异常）不带该字段，前端保持上一次的值。
- 胜率只是"AI 对局面的看法"，深度仅 4 层，中盘波动大是正常现象，不是 bug。

---

## 8.15 增量更新：删除 AI 对话 + 清理无用文件（2026-09-30）

**改动**：AI 不再说任何话。删掉了四类"AI 对话"：
开局问候、落子后的随机台词（12% 概率）、胜负台词、玩家发消息时的关键词自动回复。
聊天区从此**只有两名真人之间的对话**（AI 房里就是玩家自言自语，不会再有回复）。

同时清理了已经不再使用的文件：旧引擎 `ai.js` 及其单测 / 对局实验脚本、
kevin 移植版与其评测脚本、评测日志、三个临时脚本、浏览器自动化残留的
`tmp-chrome-app/`（9.7MB）。

| 文件 | 动作 | 说明 |
|---|---|---|
| `server.js` | **替换** | 去掉 `require('./ai.js')` 与四处 AI 说话逻辑（落子台词 / 开局问候 / 胜负台词 / 聊天回复） |
| `lib/gobang-ai/*`（含新增的 `opening*.js`） | 不动 | 落子只走 gobang 引擎，与本次改动无关 |
| `ai.js` | **删除** | 旧的无状态引擎（2026-09-29 起已不参与落子，只剩台词） |
| `tools/ai-test.js` | **删除** | 旧引擎的 73 项单测，随引擎一起删 |
| `tools/ai-match.js` / `tools/compare-gobang.js` | **删除** | 旧引擎的对局实验脚本 |
| `tools/kevin-tactics.js` / `kevin-vs-ours.js` / `kevin-blunder.js` | **删除** | kevin 移植版评测脚本（**战术题已另行保留**，见下） |
| `lib/kevin-ai/` | **删除** | kevin2014123/gomoku-ai 的移植版（实测比现行引擎弱一档，详见 `docs/AI-棋力核对-2026-09-30.md`） |
| `tools/_eval/`、`tmp-kevin-*.js`、`tmp-chrome-app/` | **删除** | 评测日志与临时脚本 |
| `tools/tactics-test.js` | **新增** | 战术题回归（12 题 × 2 档）。原 `kevin-tactics.js` 里那两道"破对手连冲四杀"是防守端 VCF 的回归题，不能跟着删，故抽成不依赖 kevin / ai.js 的独立脚本 |

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git pull
pm2 restart gomoku              # server.js 改了，必须重启

# 验收
node tools/tactics-test.js      # 战术题 22 项全过（失败会退出码 1）
node tools/protocol-test.js     # 通过 71 项
node tools/smoke-test.js        # 通过 83 项
```

### 注意事项

- **旧引擎的 `ai-test.js` 不再存在**，自检命令换成 `tools/tactics-test.js`。
- 前端 `public/` 本次没改，但 `server.js` 换了 → 仍要重启。
- App 端：`js/gobang-engine.js` 重新打包（82.2KB，比之前小 40KB，因为不再打旧引擎），
  `js/local-ai.js` 已去掉台词调用 —— 两端都要生效需**在 HBuilderX 重新云打包 APK**。

---

## 8.16 增量更新：难度梯队重排 + 困难档接入 Rapfi 引擎（2026-09-30，**仅网页端**）

**改了什么**：
1. **难度梯队往上整体推一档**：新手(自研 d4) / 中等(自研 d4+VCT算杀+防守VCF) / 困难(**外挂 Rapfi**)。
   原「简单」档删除；原「中等」→ 新手；原「困难」→ 中等。
2. **困难档换成 Rapfi**（Gomocup 冠军级开源引擎，NNUE 评估，走 stdin/stdout 的 Piskvork 协议）。
   实测 Rapfi(300ms/步) vs 自研困难档 = **8:0**，两边单步耗时同量级。
3. **引擎不可用时自动降级**：目录缺失/自检失败/超时/池忙 → 该步回退自研引擎，对局永不卡死。
   未部署引擎时**不需要任何配置**，困难档自动等于中等档配置。
4. **老客户端兼容**：不带 `ladder` 字段的请求按上一代梯队翻译（easy/normal→novice、hard→medium），
   旧 APK 不会被莫名换成 Rapfi。

> ⚠️ **App 端本次不动**（用户 2026-09-30 决定：软件版暂停开发，只维护网页端）。
> App 仍是上一代梯队，靠上面的 legacy alias 继续可用。

### 要上传的文件（网页端）

| 文件 | 动作 |
|---|---|
| `server.js` | **替换**（梯队 + Rapfi 接入 + 异步落子） |
| `lib/rapfi/client.js` | **新增**（协议客户端：看门狗 / 合法性校验 / 噪声过滤） |
| `lib/rapfi/pool.js` | **新增**（进程池 + 启动自检 + 繁忙回退） |
| `lib/rapfi/config.toml` | **新增**（精简配置：只留 freestyle 权重、关分析输出） |
| `public/index.html` | **替换**（难度按钮改为 新手/中等/困难） |
| `public/js/game.js` | **替换**（档位提示文案 + 建房带 `ladder:2`） |
| `lib/gobang-ai/*` | 本次无改动（上一轮已传的不用再传） |
| `tools/*`、`docs/*` | 可选（开发工具与文档，不上传不影响运行） |
| `data/` | **绝对不要覆盖** |
| `gomoku-app/` | **不用传**（App 暂停开发） |

### 引擎准备（可选，但不做则困难档 = 中等档）

把 `rapfi-引擎上传包/engine/rapfi/`（约 15MB，含 3 个指令集版本的 Linux 二进制 + 权重 + config）
整个传到服务器 `/www/wwwroot/gomoku/engine/rapfi/`，然后：

```bash
cd /www/wwwroot/gomoku/engine/rapfi
chmod +x pbrain-rapfi-linux-*

# ① 看 CPU 支持什么指令集，挑最快的那个二进制作 RAPFI_BIN（不设则自动按 vnni→avx2→sse 挑）
grep -o 'avx2\|avx512f\|avx512_vnni' /proc/cpuinfo | sort -u

# ② 手工验证引擎能起来（缺任何一个权重都会 ERROR Unable to open model file）
printf 'START 15\nINFO rule 0\nINFO timeout_turn 300\nBEGIN\nEND\n' | ./pbrain-rapfi-linux-clang-avx2
# 期望：MESSAGE Load config ... / OK / 7,7
```

### 操作步骤

```bash
cd /www/wwwroot/gomoku
git log --oneline -1                  # 记下当前版本，回退要用
cp -r data /tmp/gomoku-data-backup    # 老规矩：先备份账号数据

# 上传上表文件（或 git pull）。engine/ 目录不要求进 git，手工传即可。

pm2 restart gomoku                    # server.js 改了，必须重启

# -------- 验收 --------
node tools/protocol-test.js           # 通过 74 项（含新旧两个梯队 + 老客户端兼容）
node tools/smoke-test.js              # 通过 83 项
curl -s http://127.0.0.1:3000/ | grep -E "game.js"    # ?v=xxxxxx 而不是 ?v=dev
```

### 怎么确认"新引擎真的生效了"（**不需要 pm2**）

> ⚠️ 这台服务器上 `pm2: command not found` 是**正常的**：宝塔的 Node 项目用的是面板自己的
> 进程管理（或 pm2 装在独立目录、不在 root 的 PATH 里）。**不代表部署失败。**

**重启服务的三种办法（任选）**
```bash
# ① 最稳：宝塔面板 → 网站 → Node 项目 → 找到 gomoku → 点「重启」
# ② 如果面板装了「PM2管理器」插件：在里面找到项目点重启
# ③ 命令行找 pm2 真实路径后用它：
ls /www/server/nodejs/*/bin/pm2 2>/dev/null; find /www -maxdepth 4 -name pm2 -type f 2>/dev/null | head
/www/server/nodejs/vXX/bin/pm2 restart gomoku --update-env     # 路径按上一步结果替换
# ④ 完全找不到 pm2 时，直接看进程、用面板重启（别用 kill -9，面板会显示异常状态）：
ps -ef | grep -v grep | grep server.js
```

**先确认一件事：引擎状态是"自检结果"的缓存**
自检发生在**进程启动时**，以及之后每次**引擎不可用时的懒重试**（下一手困难档棋时触发，30 秒冷却）。
所以"我刚在服务器上改了文件，healthz 却没变" 是正常的 —— 重启项目，或开一局困难档下一手即可刷新。

**第 1 层：新代码是否在跑（一条 curl 就能判定）**
```bash
curl -s http://127.0.0.1:3000/healthz
```
- **出现 `ai` 字段 = 新 server.js 已生效**（旧版本没有这个字段）。
- 同时看 `uptime`：个位数说明刚重启过（改完代码必须重启，不重启就还是旧逻辑）。

**⚠️ 启动报 `EACCES` / errno -13（真实踩过的一次）**

```
Error: spawn /www/wwwroot/gomoku/engine/rapfi/pbrain-rapfi-linux-clang-avx512vnni EACCES
errno: -13, code: 'EACCES', syscall: 'spawn ...'
```

原因只有一个：**引擎二进制没有执行权限**（上传/解压后不会自动带 `x` 位）。修法两行：
```bash
cd /www/wwwroot/gomoku/engine/rapfi
chmod +x pbrain-rapfi-linux-*     # 三个二进制都给 x 位（或只给要用的那个）
ls -l pbrain-rapfi-linux-*        # 确认权限位里有 x（-rwxr-xr-x）
```

顺带说明：这类错误**现在不会再把服务打挂**了 —— 客户端已为子进程挂上 `'error'` 监听，
spawn 失败只会让引擎链路**降级**（困难档=中等档配置），服务照常运行，
`/healthz` 的 `reason` 直接写明原因（如 `no-exec: 没有执行权限（… chmod +x …）`）。

> ⚠️ **`chmod` 之后必须让服务重新自检**，否则 `/healthz` 还是显示 `no-exec`。
> 两种办法：
> 1. **重启项目**（宝塔 → Node 项目 → 重启）—— 最直接。
> 2. 什么都不做，**在网页上开一局困难档下一手** —— 新版会在引擎不可用时自动重试自检
>    （30 秒冷却），`/healthz` 的 `initAttempts` 会从 1 变成 2，随后 `available` 转 `true`。
>    （`RAPFI_RETRY_MS` 可调这个冷却毫秒数。）

其它同类启动失败：
- `no-exec` / `EACCES` → 上面这条，`chmod +x` 解决。
- **自愈（2026-09-30 晚新增）**：重新上传/解压引擎目录必丢执行位（已发生两次）——
  现在服务端会**自动补权限**（chmod 755，文件属主须与运行用户一致，宝塔默认 www），
  下一手困难档棋的懒重试即可恢复；只有属主不符（如文件属主 root）或目录在 noexec
  挂载点时才需要手动处理。
- `启动自检失败：... Unable to open model file` → 权重没传全（需要
  `mix9svqfreestyle_bsmix.bin.lz4` **和** `model210901.bin`）。
- 日志里出现多条「候选不可用」→ 正常流程：程序会**逐个真跑一次自检**，
  自动跳过 CPU 不支持的指令集版本（avx512 在旧 CPU 上会 SIGILL），退到 avx2 / sse。

**第 2 层：引擎在不在、困难档到底用的谁**

```bash
curl -s http://127.0.0.1:3000/healthz | python3 -m json.tool | sed -n '/"ai"/,$p'
```
生效时的样子（`hardEngine: "rapfi"`、`available: true`）：
```json
"ai": {
  "ladder": 2,
  "levels": ["novice(新手)", "medium(中等)", "hard(困难)"],
  "hardEngine": "rapfi",
  "hardTimeMs": 800,
  "rapfi": { "available": true, "size": 2, "busy": 0,
             "thinkCount": 0, "fallbackCount": 0, "restarts": 0, "reason": "" }
}
```
没放引擎时的样子（**功能正常，只是困难档=中等档配置**）：
```json
"hardEngine": "gobang(降级)",
"rapfi": { "available": false, ..., "reason": "引擎目录不可用（not-found）：<项目>/engine/rapfi" }
```
`reason` 会直接告诉你缺什么（`not-found` = 目录/二进制没找到；`启动自检失败：...` = 权重或 config 缺文件）。

**第 3 层：引擎链路自测（不经服务器，直接测协议与进程池）**
```bash
RAPFI_ENGINE_DIR=/www/wwwroot/gomoku/engine/rapfi node tools/rapfi-test.js
#   期望：通过 17 项，失败 0 项；引擎没放好时会整体"跳过"并退出码 0
```

**第 4 层：真机对局（最终确认）**
浏览器进大厅 → 挑战 AI → 选「困难」下一手，然后回服务器看：
```bash
curl -s http://127.0.0.1:3000/healthz | grep -E 'thinkCount|fallbackCount'
```
- `thinkCount` 从 0 变成 ≥1 → **Rapfi 真的在给困难档落子**（这是最硬的证据）。
- 若 `thinkCount` 一直是 0 而 `fallbackCount` 在涨 → 引擎起来了但每步都失败，看日志里
  `[rapfi] 本步回退自研引擎：...` 的具体原因。

浏览器验收（真机最好）：大厅选「挑战 AI」→ 三个档位应显示为 **新手 / 中等 / 困难**；
选困难开局下一手，AI 应招明显变慢一拍（Rapfi 每步 800ms）；服务端日志出现 rapfi 的活动痕迹。

### 环境变量（都可不设）

| 变量 | 默认 | 说明 |
|---|---|---|
| `RAPFI_ENGINE_DIR` | `<项目>/engine/rapfi` | 引擎目录（二进制 + 权重 + config 必须同目录） |
| `RAPFI_BIN` | 自动挑 | 指定用哪个二进制（如 `pbrain-rapfi-linux-clang-avx512vnni`） |
| `RAPFI_TIME_MS` | 800 | 困难档每步预算（实测 300ms 就 8:0 碾压自研） |
| `RAPFI_POOL_SIZE` | 1 | 常驻进程数（每个约 100MB：权重 + 置换表）。2 核服务器 1 个即吃满算力，2 个只会互抢核并拖慢主线程；设 2 可回滚 |
| `RAPFI_QUEUE_WAIT_MS` | 5000 | 池忙时排队等待空闲进程的上限（等待不阻塞主线程），等不到才回退自研 |
| `RAPFI_MAX_MEMORY_MB` | 256 | 单个引擎进程的内存上限 |
| `GOBANG_AI_ENGINE` | `rapfi` | 设成 `gobang` 则**完全不启用 Rapfi**（一键回滚） |
| `RAPFI_RETRY_MS` | 30000 | 引擎不可用时的懒重试冷却（毫秒） |

### 回滚（两条路）

```bash
# ① 只回滚引擎（保留新梯队）：困难档改用自研"中等"配置
#    宝塔 Node 项目 → 环境变量加 GOBANG_AI_ENGINE=gobang → 重启；或用 pm2：
GOBANG_AI_ENGINE=gobang pm2 restart gomoku --update-env

# ② 整体回滚到上一版：见第 9 节（git 回退 + pm2 restart）
```

### 注意事项

- **Rapfi 是 GPL-3.0**：服务端自用没问题，但**不要把二进制/wasm 分发进 App 或前端仓库**。
- 引擎进程崩溃/卡死会自动重启并在该步回退自研引擎；`logs` 里 `[rapfi] 本步回退自研引擎` 有限流（10s 一条）。
- 多房并发时进程池上限 `RAPFI_POOL_SIZE` 生效：池忙会先**排队等待**空闲进程（至多 `RAPFI_QUEUE_WAIT_MS`，默认 5s；等待不阻塞主线程），等不到才回退自研中等配置——宁可这一步晚一两秒，也不降棋力。

---

## 8.17 增量更新：AI 排行榜 + 金币/门票系统（2026-09-30，仅网页端）

**新增能力**：主页资产条（金币/门票）、AI 排行榜（TOP50 + 自己名次）、每日签到
（7 天循环 20~100 金币）、门票（中等 1 张/局、困难 2 张/局，**胜局全额返票**）、
排位积分（初始 1000，**按先后手补偿**：中等 先手胜+10/后手胜+25，困难 先手胜+20/后手胜+35，
负 −10/−15，连 3 胜 +5，连 3 负减半，每档每日前 10 局计分）。新手档与游客 = 练习局（免票不计分）。

数值表集中在 `lib/ai_economy.js` 的 `RANK` 常量（87 项单测钉死：`node tools/economy-test.js`）。

| 文件 | 动作 |
|---|---|
| `server.js` | **替换**（结算钩子/建房扣票/WS 四接口 `wallet_get`·`checkin`·`buy_tickets`·`get_rank`） |
| `lib/ai_economy.js` | **新增**（纯函数） |
| `public/index.html`、`public/js/game.js`、`public/css/app.css` | **替换**（资产条/排行入口卡/排行弹窗/签到购票面板/结算反馈） |
| `tools/economy-test.js`、`tools/e2e-rank.js` | 可选（单测与真浏览器截图工具） |
| `data/` | **绝对不要覆盖**。经济数据已迁入 `data/gomoku.db`；`ai_economy.json` 仅作为旧数据迁移来源 |

**老客户端兼容**：`game_start`/`game_over` 只是**追加**字段（`ranked`/`rank`），旧 App 忽略；
票不足时**不拒绝建房**，自动降级练习局（不计分不扣票）。

**操作步骤**
```bash
cd /www/wwwroot/gomoku
cp -r data /tmp/gomoku-data-backup     # 老规矩：先备份账号数据
# 上传上表文件（或 git pull）
pm2 restart gomoku                     # 或宝塔 → Node 项目 → 重启

# 验收
node tools/economy-test.js             # 通过 87 项，失败 0 项
node tools/protocol-test.js            # 通过 90 项（含 [10b] 经济用例 12 条）
# 真机：登录 → 主页出现金币/门票徽标 → 签到 +20 → 开中等房（扣 1 票）→ 打完看结算反馈
#       主页排行入口卡 → TOP50 + 自己的名次
```

**注意事项**
- 测试工具会注册测试账号：`protocol-test` / `e2e-rank` 已默认用 `DATA_DIR` 环境变量指向临时目录，
  **不会**污染生产排行榜（server.js 的 `DATA_DIR` 环境变量即为测试隔离而加）。
- 主动退出排位局 = 弃权判负（扣分不发安慰金币）；断线超时 = 退票不结算。
- 「再来一局」重新扣票（防白嫖）。票不足自动降练习局，不拒绝建房。

### 运维：给玩家加金币 / 门票（三种方式）

| 方式 | 场景 | 用法 |
|---|---|---|
| **网页面板**（推荐） | 在自己电脑的浏览器里操作 | 部署本批代码后，访问 `http://47.104.238.201:3000/admin-economy.html`，填**管理密钥** + 玩家名 + 数值，点「发放」或「查询」 |
| **命令行（远程）** | 在自己电脑的终端 | `node tools/grant.js 玩家名 +100 +2 --host 47.104.238.201 --port 3000 --key 管理密钥` |
| **命令行（本机）** | 已 SSH 到服务器 | `node tools/grant.js 玩家名 +100 +2`（走 127.0.0.1 本机管理接口，无需密钥） |

**必须先做一步：配置管理密钥（两种方式任选其一，改完重启）**

| 方式 | 操作 | 适用 |
|---|---|---|
| ① 密钥文件（最简单，推荐） | 宝塔文件管理 → 进入 `/www/wwwroot/gomoku/data/` → 新建文件 `admin_key.txt` → 内容粘贴你自定的密钥（一行） | 你的面板版本项目设置页没有环境变量输入框时 |
| ② 环境变量 | 项目设置里若有「环境变量」入口 → 新增 `ADMIN_KEY=你自定的密钥` | 面板有该入口时 |

密钥生成：服务器上执行 `node -e "console.log(require('crypto').randomBytes(16).toString('hex'))"`，
把输出（32 位随机串）作为密钥。未配置密钥时，公网管理接口整体停用（403），
只有 SSH 到服务器本机的 CLI 可用；换密钥 = 改文件/环境变量后重启。

安全说明：密钥错误一律 403；接口只开放发放/查询两个动作；页面本身不含任何密钥，
也不会被搜索引擎收录（noindex）。不要手改旧 `data/ai_economy.json`（服务端已读 SQLite，
外部改它不会影响当前数据）。

---

## 8.18 增量更新：开局库棋谱挖掘并入（R3，2026-10-01，仅服务端）

**新增能力**：开局库并入 Gomocup 2024/2025 顶级组棋谱挖掘（3168 局 → 1446 条线、
按出现局数加权），strength 档共 6221 个局面。**默认关闭**；开启后「中等」档 AI 在
前 16 手内优先查表——命中即落子（零搜索成本，走的是顶级引擎棋谱应手）。

**上传/替换（改完必须重启 node；未传新数据就开开关 = 用的还是旧版 57 局面库）**

| 文件 | 动作 |
|---|---|
| `lib/gobang-ai/openingMinedData.js` | **新增**（挖掘数据，209KB，由 `tools/opening-psq-mine.js` 生成） |
| `lib/gobang-ai/openingCompiler.js`、`openingCatalog.js`、`openingBook.js` | **替换** |
| `server.js` | **替换**（含注释更新 + /healthz 开局库观测；R1/R2 的池忙排队与池大小改动也在其中） |
| `tools/opening-psq-mine.js`、`opening-book-match.js`、`opening-book-audit.js` | 可选（本机评测工具） |

**启用 / 切换 / 回滚**（只影响「中等」档人机对战的 AI；困难档走 Rapfi，与本开关无关）

```bash
# 宝塔：Node 项目 → 环境变量 加 GOBANG_BOOK_USE=1 → 重启项目
# 或 pm2：
GOBANG_BOOK_USE=1 pm2 restart gomoku --update-env            # 启用（默认 strength 模式）
GOBANG_OPENING_BOOK=balanced GOBANG_BOOK_USE=1 pm2 restart gomoku --update-env  # 保守档（仅官方 12 开局）
pm2 restart gomoku --update-env                              # 去掉变量重启 = 回滚（回到默认关）
```

- 其他变量：`GOBANG_OPENING_BOOK=strength|balanced|off`（默认 strength）；`GOBANG_BOOK_MAX_PLIES`（默认 16 手，超出不查书）。
- 本地试：PowerShell 里 `$env:GOBANG_BOOK_USE='1'; node server.js`。
- **验收（看 `http://服务器IP:3000/healthz` 的 `ai.book`）**：`enabled` = 开关是否进了进程；
  `positions` = 数据版本（旧库 57 / R3 新库 6221；不是 6221 = 有数据文件没传或没重启）；
  `hits` = 命中次数——和中等档 AI 下几手后刷新，数字涨了即证明在生效。
  另：专项 30 局用书方 14:16（47%，胜负无显著变化），收益主要在开局质量 + 省搜索 CPU。
  审计：`node tools/opening-book-audit.js strength`（加 `OBA_MAX_STONES=8` 只审开局段；全量 6221 个局面很慢）。

---

## 8.19 增量更新：任务中心 + 门票礼包 + 站内信（2026-10-02，v1.6，仅网页端）

一次装了三样东西（同一个 v1.6 版本批次，**公告与功能必须同批部署**）：

### 1）任务中心（每日 / 每周任务 → 领金币）
- 数值全在 `lib/ai_economy.js` 的 `TASKS`（日 30+20+25=75，周 80+70+75=225）。
- 进度只在**正常结算**的局里累加（五连/满盘；弃权/超时不计，与战绩同口径），游客不追踪。
- 协议：`task_get` → `tasks{coins,checkin,d,w}`；`task_claim{scope,id}` → `task_claim_ok` + wallet。
  ⚠️ **客户端 scope 必须是 `'d'` / `'w'`**，不是周期 key（日期串）。
- 周期惰性重置：`rec.tasks.{d,w}` 带 key（date / ISO weekKey），跨天跨周首访自动重建，**无定时器**。
- UI：主页「任务中心」大按钮（可领红点）→ 居中弹窗（顶部融入每日签到卡：7 天轨迹 + 连签角标）。

### 2）商城门票优惠礼包
- `RANK.TICKET_PACK_PROMO{count:10, price:100, weeklyLimit:3}`；单张成本 10 金币（单买 30 / 普通整包 27）。
- 协议：`ticket_pack_buy` → `pack_ok` + wallet；`shop_get` 回包带 `pack{count,price,limit,bought}`。
- 周限购复用任务系统的 ISO 周键惰性重置（`rec.ticketPack{week,bought}`）。
- 「商城」里的皮肤与礼包购买**都有二次确认弹窗**（复用 `#modal`）。

### 3）站内信（运维群发 → 玩家领取）
- 存储：邮件本体已迁入 SQLite `mails` 表（**只存一份**）；玩家已读/已领挂经济记录 `rec.mail`。
- 运维接口（密钥同经济接口：`ADMIN_KEY` 环境变量 或 `data/admin_key.txt`）：
  - `POST /admin_api/mail/send` `{title, body, coins, tickets, adminKey}` —— 群发（奖励可 0 = 纯公告）
  - `GET  /admin_api/mail/list?adminKey=…` —— 已发列表 + 已领人数
  - 运维页 `public/admin-economy.html` 已加「站内信」页签，浏览器里直接写直接发。
- 协议：`mail_get` → `mail{list,unread}`；`mail_read{id}`（幂等）；`mail_claim{id}` → `mail_ok` + wallet。
  群发时**在线玩家即时收到推送**（不用等下次打开）。
- UI：主页**左下角悬浮信封**（`fixed`，不占布局、不影响主页免滚动），未读红点；弹窗关闭即标已读。

### 部署与验收
```bash
# 需要传的文件（本次改动）
server.js
lib/ai_economy.js            # 任务 / 礼包 / 站内信在经济层的纯函数
lib/mail_store.js            # 新增：站内信存储
public/index.html  public/css/app.css  public/js/game.js
public/admin-economy.html    # 运维中心（站内信页签）
# data/ 一律不覆盖（账号、经济、邮件都在 SQLite；旧 JSON 仅作迁移来源）
```
- **改了 `server.js` + `lib/` → 必须重启**（宝塔：网站 → Node 项目 → 重启；本机无 pm2）。
- 验收：① 主页「任务中心」能打开、打完一局进度 +1；② 商城顶部见礼包、点击有确认框；
  ③ 运维页「站内信」页签发一封 → 玩家主页左下角亮红点 → 领取到账。
- 回归命令：`node tools/economy-test.js`（180）→ `node tools/protocol-test.js`（167）→
  `node tools/smoke-test.js`（68）→ `node tools/e2e-rank.js`（15）/ `e2e-about.js`（36）。

---

项目已初始化 Git 仓库（位于 `gomoku/.git`，分支 `main`），**改功能前先提交一次，出问题随时回退**。

### 日常用法

```bash
cd /www/wwwroot/gomoku        # 本地则是 E:\Codex wenjian\五子棋\gomoku

git status                    # 看改了哪些文件
git add -A                    # 暂存全部改动
git commit -m "说明这次改了什么"
git log --oneline             # 看提交历史（每行前面的 7 位字符就是版本号）
```

### 回退（三种场景）

```bash
# ① 改了但还没 commit —— 丢弃全部未提交改动，回到上一次提交
git checkout -- .
git clean -fd                 # 顺带删除新增的未跟踪文件

# ② 已 commit，想回到某个版本（保留历史记录，推荐）
git log --oneline             # 找到目标版本号，如 0f8f0a8
git revert 0f8f0a8            # 生成一个"反向提交"来撤销它

# ③ 已 commit，想硬回到某个版本（之后的提交会丢失，谨慎）
git reset --hard 0f8f0a8
```

### 只想恢复单个文件

```bash
git log --oneline -- server.js        # 看这个文件的改动历史
git checkout 0f8f0a8 -- server.js     # 把 server.js 恢复到那个版本
```

### 部署到服务器后如何回退

```bash
cd /www/wwwroot/gomoku
git reset --hard <版本号>     # 或 git checkout <版本号> -- .
pm2 restart gomoku            # 重启生效
```

### 注意事项

- `node_modules/` 已写入 `.gitignore`，不会进版本库（`package.json` / `package-lock.json` 会进，换机器后 `npm install` 即可还原依赖）。
- 每次**部署前先 commit**，保证服务器上任何时刻都能一键回到上一个可用版本。
- 提交信息写清楚改了什么（例：`fix: 修复重连后棋盘空白`），回退时一眼能找到。

---

## 10. 安全与容量（小规模自用足够）

- 服务端限制房间总数上限 5000（`MAX_ROOMS`），防止被打满内存。
- 所有落子坐标、轮次、胜负均由服务端校验，前端伪造无效。
- 心跳 15s ping / 30s 超时，掉线立即清理房间并通知对手。
- 若要公网长期开放，建议：用 Nginx 限制单 IP 连接数、加 `limit_req`，并只开放 443。
