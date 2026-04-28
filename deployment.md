# DEPLOYMENT：n8n-001 部署手册

从零到 SC-001~SC-010 全过的逐步操作手册。

> 预计耗时：1.5-2 小时（首次构建 Playwright 镜像约 5-10 分钟，等代理健康检查刷新约 5 分钟，其余都是点击 + 复制粘贴）。
> 建议：开两个浏览器（一个 CMS、一个 n8n）+ 一个 SSH 终端窗口同时操作。

---

## 总览

```
A. Cloudflare R2 准备       （浏览器 5 分钟）
B. 2 个 SOCKS5 代理         （已有的话跳过）
C. SSH 进 VPS              （Windows 终端 1 分钟）
D. MySQL 准备              （VPS 终端 5 分钟）
E. 宝塔 nginx 反代          （宝塔面板 10 分钟）
F. 克隆仓库 + 填 .env       （VPS 5 分钟）
G. docker compose up + 冒烟 （VPS 15 分钟）
H. Directus collection 配置（CMS 浏览器 30 分钟）
I. n8n workflow 配置        （n8n 浏览器 20 分钟）
J. Directus Flow            （CMS 浏览器 15 分钟）
K. 导出配置 + git 提交      （VPS 5 分钟）
L. 跑 SC-001~SC-010 验证   （10 分钟）
M. 合并 PR + /finish 收口  （5 分钟）
```

每一步都标了【在哪做】【做什么】【点哪里/敲什么】【看到什么算成功】【出错处理】。

---

# Phase A：Cloudflare R2 准备

## A1. 创建 R2 桶（如果还没有）

【在哪做】浏览器：https://dash.cloudflare.com/

【操作】
1. 左侧栏点 **R2**
2. 顶部右上角 **Create bucket**
3. Bucket name 填 `seemods-com`（必须**完全这个名字**，spec 里写死了）
4. Location: **Automatic** 即可
5. 点 **Create bucket**

【预期】R2 概览页能看到 `seemods-com` 这一行，size 0 B。

## A2. 拿 Account ID

【操作】R2 主页右上角，标 **Account ID** 的字符串（32 位 16 进制）。点击右边的 📋 复制。

【记下来】粘贴到记事本，标注为 `R2_ACCOUNT_ID`。

## A3. 创建 R2 API Token

【操作】
1. R2 主页右侧栏点 **Manage R2 API Tokens**
2. 右上 **Create API Token**
3. **Token name**：`apkmirror-scraper`
4. **Permissions**：选 **Object Read & Write**
5. **Specify bucket(s)**：选 **Apply to specific buckets only** → 勾选 `seemods-com`
6. **TTL**：留 default（永不过期）
7. **Client IP filtering**：留空
8. 拉到底点 **Create API Token**

【预期】跳到一个**只显示一次**的页面，含三个值：
- **Access Key ID**（约 32 位）
- **Secret Access Key**（约 64 位）
- **Use jurisdiction-specific endpoints for S3 clients**（一串 URL）

【关键】这个页面**关掉就再也看不到 Secret 了**，立刻全部复制到记事本：
```
R2_ACCESS_KEY_ID = 上面的 Access Key ID
R2_SECRET_ACCESS_KEY = 上面的 Secret Access Key
R2_ENDPOINT = https://<R2_ACCOUNT_ID>.r2.cloudflarestorage.com
```
> R2_ENDPOINT 用 jurisdiction-specific 那串里**域名部分**，但**去掉** `/<bucket>`，只保留到 `.cloudflarestorage.com`。

【出错处理】如果不小心关掉了 Secret 页面没记下来：回到 Manage R2 API Tokens，找到刚才创建的 token，**Roll**（重置）它，会再给你一次新的 Secret。

---

# Phase B：2 个 SOCKS5 代理

如果你已经有 2 个住宅 SOCKS5 代理凭证，跳到 Phase C。

如果还没买：常见家用代理服务商（IPRoyal / Bright Data / Smartproxy / 911s5 等）都行，要的是**住宅 IP**（不要数据中心 IP，会被 GP 直接封）。

【最终需要 2 个完整 URL】：
```
socks5://用户名1:密码1@host1:port1
socks5://用户名2:密码2@host2:port2
```

记到记事本，标 `PROXY_URL_1` 和 `PROXY_URL_2`。

---

# Phase C：SSH 进 VPS

## C1. 打开 Windows 终端

【在哪做】Windows 本地

【操作】Win+R 打开 PowerShell 或 Git Bash 都行。

```bash
ssh root@<你的 VPS IP>
```
按提示输入 root 密码（或用密钥文件）。

【预期】提示符变成 `root@<vps-hostname>:~#`。

## C2. 确认基础环境

【在哪做】VPS 终端

【操作】依次粘下面 5 条，**每条单独执行**，把输出贴回来给我看（如果有任一报错的话）：
```bash
docker --version
docker compose version
mysql --version
which openssl
df -h /
```

【预期】
- docker ≥ 20.10
- compose 是 v2.x（`Docker Compose version v2.x.x`）
- mysql 已装（任意版本）
- openssl 路径返回（一般 `/usr/bin/openssl`）
- 根盘剩余空间 > 10 GB（Playwright 镜像约 1.5 GB）

【出错处理】
- `docker: command not found`：宝塔 → 软件商店 → Docker 管理器，安装
- `docker compose version` 报错而 `docker-compose` 能跑：说明是老 v1 版本。`apt install docker-compose-plugin` 升到 v2
- 根盘 < 10 GB：先清理 / 扩容

---

# Phase D：MySQL 准备

## D1. 用 root 登 MySQL

【在哪做】VPS 终端

【操作】
```bash
mysql -uroot -p
```
输 root 密码。提示符变成 `mysql>`。

【出错处理】不知道 root 密码：
- 宝塔面板 → 数据库 → root 密码可以查/重置
- 或者：`grep "temporary password" /var/log/mysqld.log` 看初始密码

## D2. 建库 + 建用户 + 授权

【在哪做】MySQL 终端（提示符 `mysql>`）

**先想一个强密码**给 `apkmirror_user`，记到记事本标 `MYSQL_PASSWORD`，下面 SQL 把 `REPLACE_WITH_YOUR_PASSWORD` 换成这个值。

```sql
CREATE DATABASE apkmirror_data DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'apkmirror_user'@'%' IDENTIFIED BY 'REPLACE_WITH_YOUR_PASSWORD';
GRANT ALL PRIVILEGES ON apkmirror_data.* TO 'apkmirror_user'@'%';
FLUSH PRIVILEGES;
SHOW DATABASES;
EXIT;
```

【预期】`SHOW DATABASES` 输出里能看到 `apkmirror_data`。`EXIT` 回到 root shell。

## D3. 让 MySQL 监听 0.0.0.0（容器才连得上）

【在哪做】VPS 终端（root shell）

【操作】先看当前监听：
```bash
ss -ltn | grep 3306
```

【看输出做判断】
- 如果是 `0.0.0.0:3306` → ✅ 跳到 D4
- 如果是 `127.0.0.1:3306` → 需要改 bind-address，继续下面：

```bash
# 找 bind-address 配置在哪个文件
grep -rl "bind-address" /etc/mysql/ /www/server/mysql/etc/ 2>/dev/null
```

【根据输出找到那个文件】常见的有：
- `/etc/mysql/mysql.conf.d/mysqld.cnf`（apt 装的）
- `/www/server/mysql/etc/my.cnf`（宝塔装的）

```bash
# 编辑（把 <FILE> 替换成上面找到的路径）
vi <FILE>
```

vi 操作：
- 按 `i` 进入插入模式
- 找到 `bind-address = 127.0.0.1` 改成 `bind-address = 0.0.0.0`（前面别有 `#`）
- 按 `Esc`，敲 `:wq` 回车保存退出

```bash
# 重启 MySQL（任选其一，看你怎么装的）
systemctl restart mysql      # apt 装的
# 或
systemctl restart mysqld     # rpm 装的
# 或宝塔面板 → 数据库 → MySQL 服务 → 重启

# 再次确认
ss -ltn | grep 3306
```

【预期】这次显示 `0.0.0.0:3306` 或 `*:3306`。

## D4. 防火墙允许 Docker 网段访问 MySQL

【在哪做】VPS 终端

【操作】
```bash
# 检查防火墙状态
ufw status 2>/dev/null || iptables -L INPUT -n | grep 3306
```

如果 ufw 是 active：
```bash
ufw allow from 172.16.0.0/12 to any port 3306 comment "docker -> mysql"
ufw status
```

【宝塔用户额外操作】宝塔面板 → 安全 → 系统防火墙：放行端口 3306，**Source IP** 填 `172.16.0.0/12`（Docker 默认网段范围）。

## D5. 测试从一个临时容器能否连上 MySQL

【在哪做】VPS 终端

【操作】
```bash
docker run --rm -it \
  --add-host=host.docker.internal:host-gateway \
  mysql:8 \
  mysql -h host.docker.internal -u apkmirror_user -p apkmirror_data \
  -e "SELECT 'OK' AS result"
```
输 D2 设的密码。

【预期】打印一行：
```
+--------+
| result |
+--------+
| OK     |
+--------+
```

【出错处理】
- `Can't connect to MySQL server`：D3 没生效或 D4 防火墙没放
- `Access denied for user`：D2 的密码错了，重做 D2（用 `DROP USER` 后重建）

---

# Phase E：宝塔 nginx 反代（HTTPS）

你说"现在已经是 https"——这一步是确认两个域名指向对、且 n8n 那个开了 WebSocket。如果没开会导致 webhook 链接断开。

## E1. 确认两个域名都已经在宝塔配好反代

【在哪做】浏览器：宝塔面板 → 网站

【预期】列表里有两条：
- `cms.<你的域名>` → 反代到 `127.0.0.1:8055`
- `n8n.<你的域名>` → 反代到 `127.0.0.1:5678`

如果**只有一个或都没有**，按下面 E2/E3 各建一个。如果**都有**，跳到 E4 检查 WebSocket。

## E2. 给 cms 域名建反代（如果缺）

【在哪做】宝塔面板 → 网站 → 添加站点

【操作】
1. 域名：`cms.<你的域名>`（确保 DNS 已经 A 记录到 VPS IP）
2. 数据库：不需要
3. PHP 版本：纯静态
4. 创建站点
5. 进入这个站点的 设置 → **反向代理**
6. 添加反向代理：
   - 名称：`directus`
   - 目标 URL：`http://127.0.0.1:8055`
   - 发送域名：`$host`
7. 保存
8. 设置 → **SSL** → Let's Encrypt → 申请证书 → 强制 HTTPS

## E3. 给 n8n 域名建反代（如果缺）

同 E2，但目标 URL 是 `http://127.0.0.1:5678`。

## E4. 给 n8n 反代加 WebSocket（关键，否则 webhook 断）

【在哪做】宝塔面板 → 网站 → `n8n.<域名>` → 设置 → 配置文件

【操作】找到 `location ^~ /` 这个块（反代规则块），改成下面这样：

```nginx
location ^~ / {
    proxy_pass http://127.0.0.1:5678;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket 关键三行
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
    proxy_buffering off;
}
```
保存。

【预期】保存时宝塔会自动 `nginx -t`，看到绿色"配置文件保存成功"。

## E5. 确认两个域名 DNS 已生效

【在哪做】VPS 终端

【操作】
```bash
curl -I -k https://cms.<你的域名> 2>&1 | head -5
curl -I -k https://n8n.<你的域名> 2>&1 | head -5
```

【预期】看到 `HTTP/...` 状态行（502/504 也算 DNS 通了，因为后面容器还没起），不要看到 `Could not resolve host`。

---

# Phase F：克隆仓库 + 填 .env

## F1. 克隆代码

【在哪做】VPS 终端

【操作】
```bash
cd ~
git clone -b feat/n8n-001-plan-tasks https://github.com/lwang47utas/n8n-aerows101-pro.git n8n-stack
cd n8n-stack
ls
```

【预期】列出 `docker-compose.yml` / `.env.example` / `scraper/` / `migrations/` 等。当前路径 `~/n8n-stack`。

## F2. 生成 3 个随机密钥

【在哪做】VPS 终端

【操作】**直接复制下面整段**粘贴到终端：
```bash
echo "DIRECTUS_KEY=$(openssl rand -hex 32)"
echo "DIRECTUS_SECRET=$(openssl rand -hex 32)"
echo "N8N_WEBHOOK_SECRET=$(openssl rand -hex 24)"
```

【预期】打印 3 行，每行 `KEY_NAME=<一串 hex>`。**全部复制到记事本**。

## F3. 复制模板 + 编辑 .env

【在哪做】VPS 终端

```bash
cp .env.example .env
chmod 600 .env
vi .env
```

vi 编辑：按 `i` 进入插入模式，**逐行替换**下面 17 个字段（**全部要填**）：

```env
MYSQL_HOST=host.docker.internal
MYSQL_PORT=3306
MYSQL_USER=apkmirror_user
MYSQL_PASSWORD=<D2 设的密码>
MYSQL_DATABASE=apkmirror_data

R2_ACCOUNT_ID=<A2 拿的>
R2_ACCESS_KEY_ID=<A3 拿的>
R2_SECRET_ACCESS_KEY=<A3 拿的>
R2_BUCKET=seemods-com
R2_ENDPOINT=https://<R2_ACCOUNT_ID>.r2.cloudflarestorage.com

PROXY_URL_1=<Phase B 的代理 1 完整 URL>
PROXY_URL_2=<Phase B 的代理 2 完整 URL>

N8N_HOST=<你的 n8n 域名，不带 https://>
N8N_PROTOCOL=https
N8N_WEBHOOK_SECRET=<F2 生成的>
N8N_API_KEY=
N8N_BASE_URL=http://n8n:5678
SCRAPE_PIPELINE_WORKFLOW_ID=

DIRECTUS_KEY=<F2 生成的>
DIRECTUS_SECRET=<F2 生成的>
DIRECTUS_ADMIN_EMAIL=<你的邮箱>
DIRECTUS_ADMIN_PASSWORD=<你想用的 Directus 后台密码>
DIRECTUS_PUBLIC_URL=https://<你的 cms 域名>

SCRAPER_LOG_LEVEL=INFO
PROXY_HEALTH_CHECK_INTERVAL=300
PROXY_COOLDOWN_MINUTES=5
PROXY_PROBE_URL=https://www.google.com/generate_204
```

> `N8N_API_KEY` 和 `SCRAPE_PIPELINE_WORKFLOW_ID` 现在留空，**Phase I 之后再回来填**。

按 `Esc`，敲 `:wq` 保存退出。

## F4. 自查 .env 没填错

【操作】
```bash
# 应该列出全部带值的行（除了 N8N_API_KEY 和 SCRAPE_PIPELINE_WORKFLOW_ID）
grep -vE '^#|^$' .env | grep -E '=$' && echo "上面这些是空值的字段" || echo "无空字段"
```

【预期】只列出 `N8N_API_KEY=` 和 `SCRAPE_PIPELINE_WORKFLOW_ID=` 两行（这两行现在就该是空的）。

如果列出更多空行→回 vi 补上再来。

---

# Phase G：拉起容器 + 冒烟测试

## G1. 首次构建 + 启动

【在哪做】VPS 终端 `~/n8n-stack/`

【操作】
```bash
docker compose up -d --build
```

【预期】首次构建 scraper 镜像约 5-10 分钟（要装 Playwright Chromium）。最后输出：
```
[+] Running 4/4
 ✔ Network n8n-stack    Created
 ✔ Container n8n        Started
 ✔ Container scraper    Started
 ✔ Container directus   Started
```

【出错处理】
- `failed to compute cache key`：网络问题，重跑
- `Error response from daemon: pull access denied`：基础镜像拉不下来，配国内镜像源（宝塔 → Docker → 镜像加速）

## G2. 看三容器状态

```bash
docker compose ps
```

【预期】三行 STATUS 都是 `Up X seconds`（不能是 `Restarting`）。如果有 `Restarting`：
```bash
docker compose logs <服务名> --tail=50
```
把日志贴给我。

## G3. 看 scraper 启动日志（最关键）

```bash
docker compose logs scraper --tail=80
```

【预期】依次看到：
```
[entrypoint] running migration: /migrations/001_init.sql
[entrypoint] migration OK: /migrations/001_init.sql
[entrypoint] running migration: /migrations/002_subcatalogue_strategy.sql
[entrypoint] migration OK: /migrations/002_subcatalogue_strategy.sql
[entrypoint] starting uvicorn on 0.0.0.0:8000
INFO:     Started server process [...]
INFO:     Waiting for application startup.
INFO: synced 2 proxies from env
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

【出错处理】
- `migration failed`：MySQL 连不上，回 D5 重测；连得上但 SQL 失败 → 把日志贴给我
- `synced 0 proxies from env`：`.env` 里 PROXY_URL_1/2 没填或格式错

## G4. 验证 MySQL 表都建好了 + 种子数据

```bash
mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data -e "SHOW TABLES; SELECT COUNT(*) AS bigcorp_count FROM bigcorp_keywords; SELECT COUNT(*) AS settings_count FROM app_settings; SELECT COUNT(*) AS strategy_count FROM subcatalogue_strategy;"
```

【预期】
- 9 张表全列出
- bigcorp_count = 13
- settings_count = 4
- strategy_count = 56

## G5. 三个端点冒烟（SC-005 / SC-006 / SC-007）

```bash
# /health 三项 ok
docker compose exec scraper curl -s http://localhost:8000/health | python3 -m json.tool
```

【预期 JSON】（首次 alive 可能是 false，要等 5 分钟代理健康检查跑完才 true。先继续往下走，最后在 Phase L 复测）
```json
{
  "db":      { "ok": true },
  "r2":      { "ok": true, "bucket": "seemods-com" },
  "proxies": { "ok": false, "available": 0, "total": 2, "error": "no alive proxy" }
}
```

```bash
# /proxy-status 2 条记录，url_ref 是变量名，不是 socks5://
docker compose exec scraper curl -s http://localhost:8000/proxy-status | python3 -m json.tool
```

【预期】2 条记录，`url_ref` 是 `PROXY_URL_1` / `PROXY_URL_2`，没有任何 `socks5://`。

```bash
# POST /scrape 写 run_records
docker compose exec scraper curl -X POST -s http://localhost:8000/scrape | python3 -m json.tool
mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data -e "SELECT id, status, started_at FROM run_records ORDER BY id DESC LIMIT 1"
```

【预期】返回 `{"status":"ok","mode":"mock","run_record_id":N}`，`run_records` 表多一行 `status=success`。

## G6. 触发一次代理健康检查（不等 5 分钟）

```bash
# 重启 scraper 让 sync_from_env + health_check 立即重跑一次
docker compose restart scraper
# 等 30 秒
sleep 30
docker compose exec scraper curl -s http://localhost:8000/proxy-status | python3 -m json.tool
```

【预期】`alive` 至少有一个为 `true`，`latency_ms` 有数字。

【出错处理】两个代理都 `alive=false`：
- 看具体错：`docker compose logs scraper | grep "proxy probe"`
- 容器内手动测代理：
  ```bash
  docker compose exec scraper curl --socks5 user:pass@host:port https://www.google.com/generate_204 -v 2>&1 | tail -10
  ```
- 如果 `curl` 都通不了 → 代理凭证错或代理本身挂了，找代理服务商

---

至此 SC-001 / 005 / 006 / 007 + 部分 010 已通过。剩下 SC-002 / 003 / 004 / 008 / 009 需要 admin UI 配置完才能跑。

---

# Phase H：Directus collection 配置

【在哪做】浏览器：`https://cms.<你的域名>`，用 `.env::DIRECTUS_ADMIN_EMAIL` + `DIRECTUS_ADMIN_PASSWORD` 登录。

## H1. 同步表（FR-030）

【操作】
1. 左下齿轮图标 → 进 Settings
2. 左侧栏点 **Data Model**
3. **如果**右上有 "Sync System with Database" 按钮就点它；**否则**说明 Directus 已经自动扫到了，直接看列表
4. 列表里应有 9 个 collection（见下方 H2 列表）

【预期】Settings → Data Model 显示 9 行：
- apps
- app_versions
- app_hero_images
- comments
- run_records
- proxies
- bigcorp_keywords
- app_settings
- subcatalogue_strategy

## H2. 配置 apps 的显示模板（FR-031）

【操作】
1. Settings → Data Model → 点 **apps**
2. 顶部右侧标签栏中找 **Collection Setup**（或 Display & Layout）
3. 找到 **Display Template** 字段
4. 填入：`{{app_name}} ({{package_id}})`
5. 顶部右上 **Save**（💾 图标）

【验证】
- 左上角 ⌂（Content）回到内容区
- 左侧栏点 **apps**（现在应该是空表）
- 列表上方的标题栏暂时看不到效果（因为表里没数据），等 002 抓数据后才能看到

## H3. 配置 apps 的 O2M 关系（FR-032）

需要在 apps 上加 3 个虚拟关系字段，让详情页能展开 versions / hero_images / comments。

【操作 1：apps → versions】
1. Settings → Data Model → apps
2. 顶部 **Create Field** (+ 图标)
3. 选 **One to Many**（O2M）
4. **Key**: `versions`
5. **Related Collection**: 选 `app_versions`
6. **Foreign Key Field**（应自动检测）: 选 `package_id`（如果没自动选，手动选）
7. **On Delete**: `Set NULL` 即可
8. **Continue in Advanced Field Creation Mode**：可选，跳过
9. 点 **Save**

【操作 2：apps → hero_images】
重复上面，但：
- Key: `hero_images`
- Related Collection: `app_hero_images`
- Foreign Key Field: `package_id`

【操作 3：apps → comments】
重复，但：
- Key: `comments`
- Related Collection: `comments`
- Foreign Key Field: `package_id`

【验证】Settings → Data Model → apps 的字段列表里现在应该多了 3 个 O2M 字段（图标是 →∞ 那种箭头）。

## H4. comments.rating 颜色标签（FR-033）

【操作】
1. Settings → Data Model → comments
2. 点字段 `rating`（TINYINT 类型那个）
3. 切到 **Interface** 标签
4. 把 Interface 改成 **Dropdown**
5. 在 **Choices** 里点 **Add Choice** 五次，每次填：

| Value | Text | Color |
|---|---|---|
| 1 | ★ | `#E35169` |
| 2 | ★★ | `#E68A4F` |
| 3 | ★★★ | `#E6B84F` |
| 4 | ★★★★ | `#5BAD7C` |
| 5 | ★★★★★ | `#2E7D52` |

> Color 字段：点小色块图标，可以直接粘 hex（含 `#`）

6. 切到 **Display** 标签 → 选 **Labels**（这样列表里会显示彩色 chip 而不是数字）
7. 顶部 Save

【验证】回到内容区 → comments → 没数据所以看不到效果，先记下来 Phase L 实测。

## H5. 创建 viewer 角色 + 字段级权限（FR-034 + clarify Q6 B）

【操作 5.1：创建角色】
1. Settings → **Access Control** （盾牌图标）
2. 右上 **Create Role** (+)
3. **Name**: `viewer`
4. **App Access**: ✅
5. **Admin Access**: ❌
6. **IP Access**: 留空
7. Save

【操作 5.2：配 viewer 的权限】
进入 viewer 角色编辑页。下方有所有 collection 的权限矩阵（行：collection，列：Create/Read/Update/Delete/Share）。

针对每个 collection 的权限，**点对应格子**会循环切换 `No Access` / `All Access` / `Use Custom`。

**对所有 9 个业务 collection**（apps / app_versions / app_hero_images / comments / run_records / proxies / bigcorp_keywords / app_settings / subcatalogue_strategy）：
- **Read**：点到出现绿色 ✅（All Access）
- **Create / Update / Delete / Share**：点到出现红色 ❌（No Access）

**唯一例外** `app_settings` 的 **Update**：
1. 点 app_settings 的 Update 格子，循环到 **Use Custom**（齿轮图标）
2. 点开它配置：
   - Permissions: `{}`（无条件）
   - **Field Permissions**：把开关切到 **Manual**
     - ✅ 勾选：`cron_expression`、`min_comment_count`
     - ❌ 不勾：`qps_limit`、`proxy_rotate_every`、`key`
3. Save

**System collections**（Directus Files / Users / Roles 等灰色那批）：全部 `No Access`。

## H6. 侧栏分组（FR-035）

【操作】
1. Settings → Data Model
2. 顶部右上 **Create Group** (+ 图标边上的下拉)
3. 创建 3 个 group:
   - Name: `Data`
   - Name: `Configuration`
   - Name: `Reference`
4. 把每个 collection 拖到对应 group 下：
   - **Data**: apps / app_versions / app_hero_images / comments / run_records / proxies
   - **Configuration**: bigcorp_keywords / app_settings
   - **Reference**: subcatalogue_strategy

【验证】回到内容区 ⌂ → 左侧栏现在按三个分组折叠显示。

## H7. 测一下 viewer（FR-034 验证）

【操作】先在 admin 下：
1. Settings → User Directory → Create User
2. Email: 自己一个测试邮箱（或自己邮箱前面加 `+viewer`）
3. Password: 设一个
4. **Role**: `viewer`
5. Save

打开**新的浏览器隐身窗口** → `https://cms.<域名>` → 用 viewer 账号登录：
- 进 `bigcorp_keywords`：右上不应该有 **Add Item** 按钮（只读）
- 进 `app_settings`：编辑某条记录，`qps_limit` 字段应该是灰色不可编辑（FR-034 + Q6 B 验证 ✅）

记下来 Phase L 时把这次截图存档（可选）。

## H8. 验证 SC-002 / SC-003 / SC-004

回到 admin 浏览器：
- ✅ SC-002：你已经登进 Directus 了
- ✅ SC-003：左侧栏 9 个 collection 可见
- ✅ SC-004：进 `bigcorp_keywords` 看到 13 条；进 `app_settings` 看到 4 条

---

# Phase I：n8n workflow 配置

【在哪做】浏览器：`https://n8n.<你的域名>`

## I1. 首次访问设 admin

如果是第一次进 n8n，会跳到 setup 页：
- 填邮箱 / 密码 / 名字
- Save

记下这个 n8n admin 账号（和 Directus 不是同一套）。

## I2. 创建 workflow

【操作】
1. 左上角 **Workflows** → 右上 **Add workflow**
2. 顶部点工作流名（默认 "My workflow"）改成 **scrape-pipeline**

## I3. 加节点 1：Schedule Trigger (cron-trigger)

【操作】
1. 中央有个 **+ Add first step** → 点它
2. 搜 `Schedule Trigger`，点选
3. 配置：
   - Trigger Interval：选 **Custom (Cron)**
   - Cron Expression：`0 2 * * *`
4. 顶部 **Back to canvas**（左上箭头）
5. **节点重命名**：右键节点 → Rename → 改成 `cron-trigger`

## I4. 加节点 2：Webhook (webhook-trigger)

【操作】
1. 画布上空白处右键 → Add Node → 搜 `Webhook` → 选 Webhook（不是 Webhook Response）
2. 配置：
   - HTTP Method: `POST`
   - Path: `webhook/scrape-now`
   - Authentication: **Header Auth**
   - 点 Header Auth 旁边的 **Create New Credential**:
     - Name: `webhook-bearer`
     - Header Name: `Authorization`
     - Header Value: 把 `.env::N8N_WEBHOOK_SECRET` 的值取出来，前面加 `Bearer `（注意空格），整个字符串粘进去。例：`Bearer 7f3a9b2c8e1d...`
     - Save
   - Respond: **When Last Node Finishes**
3. Back to canvas
4. 重命名：`webhook-trigger`

## I5. 加节点 3：Merge

【操作】
1. 画布右键 → Add Node → 搜 `Merge`
2. 配置：
   - Mode: **Append**
3. 重命名：`merge`

【连线】
- `cron-trigger` 输出 → `merge` 的 Input 1（拖出来连过去）
- `webhook-trigger` 输出 → `merge` 的 Input 2

## I6. 加节点 4：HTTP Request (call-scraper)

【操作】
1. 画布右键 → Add Node → 搜 `HTTP Request`
2. 配置：
   - Method: `POST`
   - URL: `http://scraper:8000/scrape`
   - Send Body: ❌（关掉）
   - Send Headers: ❌
   - Send Query Parameters: ❌
   - 展开 **Options** （滚到下面） → **Timeout**: `30000`
   - **Response** → Response Format: `JSON`
3. 重命名：`call-scraper`

【连线】`merge` 的输出 → `call-scraper`

## I7. 加节点 5：IF (branch-success)

【操作】
1. 画布右键 → Add Node → 搜 `If`
2. 配置：
   - 第一个值：点小齿轮 → 选 **Expression**，填 `{{ $json.status }}`
   - Operation: `is equal to`（字符串）
   - 第二个值：直接填 `ok`
3. 重命名：`branch-success`

【连线】`call-scraper` → `branch-success`

## I8. 加 NoOp 节点（成功 + 失败 + dead-letter，共 3 个）

clarify Q2 D：本轮不接通知，所以分支末端用 NoOp 占位（n8n 自带的 No Operation 节点）。

【操作】
1. 画布右键 → Add Node → 搜 `No Operation, do nothing` → 选它，叫它 `noop-success`
2. 再来一次 → `noop-failure`
3. 再来一次 → `noop-deadletter`

【连线】
- `branch-success` 的 **TRUE 分支**（绿色） → `noop-success`
- `branch-success` 的 **FALSE 分支**（红色） → `noop-failure`
- `call-scraper` 节点底部的红色 **Error** 输出（如果默认看不到，点节点 → Settings → "Continue On Fail" 关掉，error 输出会显示出来）→ `noop-deadletter`

## I9. 激活 + 拿 workflow ID

【操作】
1. 顶部右上 **Active** 切换为 ON（绿色）
2. 浏览器 URL 栏现在是 `https://n8n.<域名>/workflow/<一串字符>` —— **这串字符就是 workflow ID**，复制出来

## I10. 创建 n8n API Key

【操作】
1. 右下角你的头像 → **Settings**
2. 左侧 **API**
3. **Create an API key**
4. Label: `directus-flow-cron-sync`
5. Expiration: 选 **No expiration**（或一年也行）
6. **Create**
7. **复制弹出的 API Key**（关掉就再也看不到了）

## I11. 把 ID 和 Key 填回 .env

【在哪做】回到 VPS 终端

```bash
cd ~/n8n-stack
vi .env
```

找到这两行，填上：
```
N8N_API_KEY=<I10 复制的>
SCRAPE_PIPELINE_WORKFLOW_ID=<I9 复制的>
```

保存退出。

```bash
# 重启让新 env 生效
docker compose restart scraper directus
```

## I12. 测试 schedule 触发（SC-008）

【在哪做】浏览器 n8n

【操作】
1. 打开 `scrape-pipeline` workflow
2. 顶部 **Execute Workflow**
3. 应该看到所有节点变绿（除了 noop-failure 和 noop-deadletter，那些是另外的分支）

【验证 run_records】回 VPS 终端：
```bash
mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data \
  -e "SELECT id, status, started_at FROM run_records ORDER BY id DESC LIMIT 1"
```
应看到刚才那条新记录。

## I13. 测试 webhook 触发（SC-009）

【在哪做】VPS 终端

```bash
# 把 .env 里的 secret 取出来
WEBHOOK_SECRET=$(grep '^N8N_WEBHOOK_SECRET=' .env | cut -d= -f2)
N8N_DOMAIN=$(grep '^N8N_HOST=' .env | cut -d= -f2)

# 带 Bearer 应该 200
curl -X POST -H "Authorization: Bearer $WEBHOOK_SECRET" \
  "https://$N8N_DOMAIN/webhook/scrape-now" -d '{}'
echo

# 不带 Bearer 应该 401 或 403
curl -X POST "https://$N8N_DOMAIN/webhook/scrape-now" -d '{}'
echo
```

【预期】带 Bearer 返回 mock JSON；不带 401。`run_records` 表又多一行。

【出错处理】
- `404 webhook not registered`：n8n UI 里 workflow 没 Active，回 I9 打开
- `502 Bad Gateway`：宝塔 n8n 反代没开 WebSocket，回 E4
- 502 还可能是 webhook URL 路径不对：n8n 默认是 `/webhook/scrape-now`（活跃模式）和 `/webhook-test/scrape-now`（测试模式），确保用前者

## I14. 导出 workflow JSON 入仓

【在哪做】浏览器 n8n

【操作】
1. 打开 scrape-pipeline workflow
2. 顶部右上 **三个点 ⋯** → **Download**
3. 浏览器下载得到 `scrape-pipeline.json`
4. 用 scp / 宝塔文件管理器 / 或者邮件，把这个文件上传到 VPS 的 `~/n8n-stack/n8n-workflows/scrape-pipeline.json`

【最简单的方式】Windows 本地终端：
```bash
# 把下载的文件上传到 VPS（在 Windows PowerShell / Git Bash）
scp ~/Downloads/scrape-pipeline.json root@<VPS-IP>:~/n8n-stack/n8n-workflows/scrape-pipeline.json
```

【验证】VPS：
```bash
ls -la ~/n8n-stack/n8n-workflows/scrape-pipeline.json
```

---

# Phase J：Directus Flow `sync-cron-to-n8n`

【在哪做】浏览器：CMS（admin 账号）

## J1. 创建 Flow

【操作】
1. Settings → **Flows** → **Create Flow** (+)
2. 配置：
   - **Name**: `sync-cron-to-n8n`
   - **Status**: `Active`
   - **Description**: `Push app_settings.cron_expression updates to n8n Schedule Trigger`
   - **Color**: 随便选
   - **Icon**: 随便选（比如 sync）
3. **Continue**（不是 Save，先去配 trigger）

## J2. 配置 Trigger

【操作】
1. Trigger 类型选 **Event Hook**
2. 配置面板：
   - **Type**: `Action (Non-Blocking)`
   - **Scope**: 勾选 `items.update`（其它都不勾）
   - **Collections**: 选 `app_settings`
   - **Filter**（切换到 Raw Editor 模式 / JSON 输入）：
     ```json
     { "keys": { "_in": ["cron_expression"] } }
     ```
3. **Save**

## J3. 配置 Operation 1：Read Cron Value

【操作】
1. 画布上 trigger 后面会有个 **+** 加 operation
2. 选 **Read Data**
3. 配置：
   - **Name**: `read-cron`
   - **Key**: `read_cron`（这是表达式引用名）
   - **Collection**: `app_settings`
   - **IDs**: `cron_expression`（PK 是 `key` 字段，PK 值就是 `cron_expression` 这个字符串）
   - **Permissions**: `Full Access`
4. **Save**

## J4. 配置 Operation 2：PATCH n8n

【操作】
1. Operation 1 后面 **+** 加 → 选 **Webhook / Request URL**
2. 配置：
   - **Name**: `patch-n8n`
   - **Key**: `patch_n8n`
   - **Method**: `PATCH`
   - **URL**: `{{$env.N8N_BASE_URL}}/api/v1/workflows/{{$env.SCRAPE_PIPELINE_WORKFLOW_ID}}`
   - **Headers**（点 Add 两次）：
     | Key | Value |
     |---|---|
     | `X-N8N-API-KEY` | `{{$env.N8N_API_KEY}}` |
     | `Content-Type` | `application/json` |
3. **Request Body** —— 这里**最稳的做法**：先用 curl 拿一份当前完整 workflow body，把它整段贴进 Body，然后只把 cron 字段那块改成 `{{$last.value}}` 表达式。

【先拿 body】回 VPS 终端：
```bash
N8N_API_KEY=$(grep '^N8N_API_KEY=' .env | cut -d= -f2)
WID=$(grep '^SCRAPE_PIPELINE_WORKFLOW_ID=' .env | cut -d= -f2)
N8N_DOMAIN=$(grep '^N8N_HOST=' .env | cut -d= -f2)

curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "https://$N8N_DOMAIN/api/v1/workflows/$WID" \
  | python3 -m json.tool > /tmp/workflow_body.json

# 看一下结构
head -30 /tmp/workflow_body.json
```

【再编辑】用 vi 打开 `/tmp/workflow_body.json`，找到 `cron-trigger` 节点的：
```json
"parameters": {
  "rule": {
    "interval": [
      { "field": "cronExpression", "expression": "0 2 * * *" }
    ]
  }
}
```
把 `"0 2 * * *"` 替换成字面量 `__CRON_PLACEHOLDER__`，保存。

```bash
cat /tmp/workflow_body.json
```
把整个 body 复制（从 `{` 到 `}`）。

【贴回 Directus Flow Operation 2 的 Request Body】把刚才整段贴进去，把 `__CRON_PLACEHOLDER__` 替换成 `{{$last.read_cron.value}}`。

> `$last` 指上一个 operation 的输出。Operation 1 的 key 叫 `read_cron`，输出是 app_settings 表的一行，所以 `$last.read_cron.value` 就是 cron_expression 那条记录的 value 字段。

4. **Save**（整个 Flow）

## J5. 测试 Flow

【在哪做】浏览器 CMS

【操作】
1. 内容区 → app_settings
2. 点 `cron_expression` 那条记录
3. 把 `value` 从 `0 2 * * *` 改成 `*/5 * * * *`
4. **Save**

【验证 Flow 运行】
1. Settings → Flows → `sync-cron-to-n8n`
2. 切到 **Logs** 标签
3. 应有一条**绿色** success 记录（< 5 秒内）

【验证 n8n 这边收到了】
1. 浏览器切到 n8n
2. 打开 `scrape-pipeline` workflow
3. 点 `cron-trigger` 节点
4. **Cron Expression** 字段应已变成 `*/5 * * * *`

【改回去】回 CMS 把 cron 改回 `0 2 * * *`，Flow 又会跑一次。

【出错处理】
- Flow Logs 里红色 error 显示 `401 Unauthorized`：N8N_API_KEY 错了。回 I10 重新创建 + 改 .env + `docker compose restart directus`
- `404 Not Found`：SCRAPE_PIPELINE_WORKFLOW_ID 错了
- 200 但 n8n 那边没变：Body 里 cron 占位符没替换对，或者节点 name 不是 `cron-trigger`

---

# Phase K：导出配置 + git 提交

## K1. 导出 Directus snapshot

【在哪做】VPS 终端

```bash
cd ~/n8n-stack
docker compose exec directus npx directus schema snapshot --yes /directus/snapshots/snapshot.yaml
ls -la directus-config/snapshot.yaml
```

【预期】文件大小 > 10 KB。

## K2. 确认 n8n workflow 也在仓里

```bash
ls -la n8n-workflows/scrape-pipeline.json
```
没有的话回 I14。

## K3. git 提交 + push

```bash
git add directus-config/snapshot.yaml n8n-workflows/scrape-pipeline.json

git commit -m "$(cat <<'EOF'
chore(n8n-001): export directus schema snapshot + n8n scrape-pipeline workflow

完成 Task #7 / #8 / #9 的运行时配置导出：
- directus-config/snapshot.yaml：9 张 collection 的显示模板 / 关系 /
  颜色标签 / 角色权限 / sync-cron-to-n8n Flow
- n8n-workflows/scrape-pipeline.json：schedule + webhook + HTTP +
  IF + NoOp 完整工作流
EOF
)"

git push
```

【预期】push 成功，分支 `feat/n8n-001-plan-tasks` 现在有 4 个提交。

---

# Phase L：跑 SC-001~SC-010 验证清单

【在哪做】VPS 终端

按下面 10 条逐条跑，每条都贴出 ✅ 或 ❌ 状态。

```bash
# SC-001
docker compose ps
# 看到三行 STATUS=Up X 即 ✅
```

```bash
# SC-002
echo "浏览器打开 https://$(grep DIRECTUS_PUBLIC_URL .env | cut -d= -f2 | sed 's|https://||') 用 admin 登录"
# 你登录成功即 ✅
```

```bash
# SC-003 + SC-004
mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data <<'SQL'
SHOW TABLES;
SELECT 'bigcorp_count' AS k, COUNT(*) AS v FROM bigcorp_keywords
UNION ALL SELECT 'settings_count', COUNT(*) FROM app_settings
UNION ALL SELECT 'strategy_count', COUNT(*) FROM subcatalogue_strategy;
SQL
# 9 表 + 13 / 4 / 56 全对即 ✅
```

```bash
# SC-005 完整版（这次代理应该至少一个 alive 了）
docker compose exec scraper curl -s http://localhost:8000/health | python3 -m json.tool
# db.ok / r2.ok / proxies.ok 三个都 true 即 ✅
```

```bash
# SC-006
docker compose exec scraper curl -s http://localhost:8000/proxy-status | python3 -m json.tool
# 2 条记录、≥1 alive、url_ref 是 PROXY_URL_X 不是 socks5:// 即 ✅
```

```bash
# SC-007
docker compose exec scraper curl -X POST -s http://localhost:8000/scrape | python3 -m json.tool
mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data \
  -e "SELECT id, status, started_at FROM run_records ORDER BY id DESC LIMIT 1"
# 返回 mock JSON + run_records 多一行即 ✅
```

```bash
# SC-008（n8n UI Execute Workflow，前面 I12 已做）
# 浏览器 n8n → scrape-pipeline → Execute Workflow → 看到全绿 + run_records 多一行即 ✅
```

```bash
# SC-009
WEBHOOK_SECRET=$(grep '^N8N_WEBHOOK_SECRET=' .env | cut -d= -f2)
N8N_DOMAIN=$(grep '^N8N_HOST=' .env | cut -d= -f2)

# 带 Bearer 应 200
curl -s -X POST -H "Authorization: Bearer $WEBHOOK_SECRET" \
  "https://$N8N_DOMAIN/webhook/scrape-now" -d '{}'
echo

# 不带 Bearer 应 401
curl -s -o /dev/null -w "no-auth: %{http_code}\n" -X POST \
  "https://$N8N_DOMAIN/webhook/scrape-now" -d '{}'

mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data \
  -e "SELECT id, status, started_at FROM run_records ORDER BY id DESC LIMIT 2"
# Bearer 那次 200 + run_records 多 1 行；不带 Bearer 是 401 即 ✅
```

```bash
# SC-010
git ls-files | xargs grep -nE 'socks5://[^$]|MYSQL_PASSWORD=[A-Za-z0-9]|R2_SECRET_ACCESS_KEY=[A-Za-z0-9]|DIRECTUS_ADMIN_PASSWORD=[A-Za-z0-9]|N8N_WEBHOOK_SECRET=[A-Za-z0-9]' 2>/dev/null
# 输出**只**含：
#   .env.example: # 格式：socks5://user:pass@host:port
#   spec.md / plan.md / tasks.md / DEPLOYMENT 里的"socks5://user:pass@..."占位说明
#   scraper/db.py: password=settings.MYSQL_PASSWORD（这是 Python 关键字参数，不是凭证字面量）
# 没有任何包含真实凭证的行即 ✅

# 加一道额外保险，确认 proxies 表里也没明文
mysql -h 127.0.0.1 -u apkmirror_user -p apkmirror_data \
  -e "SELECT id, url FROM proxies"
# url 列只有 PROXY_URL_1 / PROXY_URL_2 即 ✅
```

把 10 条的 ✅/❌ 状态贴回来，任一 ❌ 我帮你 debug。

---

# Phase M：合并 PR + 收口本轮

## M1. 开 PR + 合并

【在哪做】浏览器

【操作】
1. 打开 https://github.com/lwang47utas/n8n-aerows101-pro/pull/new/feat/n8n-001-plan-tasks
2. **Title**：`feat(n8n-001): infra + CMS + proxy pool 骨架（Spec-Kit 第 1 期）`
3. **Description**：贴下面这段
   ```
   ## Summary
   - 完成 n8n-001-infra-cms-proxy 全部 10 个 task
   - 三容器骨架（n8n / scraper / directus）+ 9 张表 + 代理池 + Directus CMS + n8n workflow + Cron 同步 Flow
   - 全部 SC-001~SC-010 已在 VPS 验证通过

   ## Test plan
   - [x] SC-001 三容器 running
   - [x] SC-002 Directus 登录
   - [x] SC-003 9 collection 可见
   - [x] SC-004 种子数据 13 + 4
   - [x] SC-005 /health 三项 ok
   - [x] SC-006 /proxy-status 2 代理 ≥1 alive
   - [x] SC-007 /scrape mock + run_records
   - [x] SC-008 n8n schedule → run_records
   - [x] SC-009 webhook → run_records
   - [x] SC-010 grep 无明文凭证
   ```
4. **Create pull request**
5. 自己 review 一遍 → **Merge pull request** → **Confirm merge**

## M2. 本地清理分支

【在哪做】Windows 本地终端，进项目目录

```bash
cd D:/GitRepo/n8n-aerows101-pro
git checkout main
git pull
git branch -d feat/n8n-001-plan-tasks
```

## M3. VPS 也切回 main

【在哪做】VPS 终端

```bash
cd ~/n8n-stack
git checkout main
git pull
git branch -d feat/n8n-001-plan-tasks
```

> VPS 上不要再切到 feature 分支，main 上现在已经合并了。日常运维都在 main 上。

## M4. 回 Claude Code 跑 /finish

【在哪做】Claude Code 对话框

【操作】输入：
```
/finish n8n-001-infra-cms-proxy
```

这会读 spec / clarify / plan / tasks / analyze + 你这次实施时的偏离（spec SC-003 的 8 vs 9 表、FR-062/063 措辞 vs Q2 D 的偏差等），生成 `n8n-001-infra-cms-proxy/finish.md` 作为 n8n-002 的 baseline。

---

# 哪一步出错都贴出来

每个 phase 都有"出错处理"小节，覆盖 90% 的常见坑。剩下 10% 你贴这三样回来我直接定位：
1. 当前 phase / step 编号（例 G3）
2. 你执行的命令或点击的位置
3. 完整错误输出 / 截图

中间随时可以暂停，明天接着跑——`docker compose ps` 和 `git status` 是断点恢复入口。
