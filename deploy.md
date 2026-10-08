# AI-Novel 生产部署指南（Ubuntu 24.04 + 宝塔面板）

本文面向 **云服务器：Ubuntu 24.04 + 宝塔面板（宿主机自带 nginx）** 的部署场景，
使用 `infra/docker-compose.yml` 一键拉起 数据库(PostgreSQL) / 向量库(Qdrant) / 后端(api) / 前端(web) 全套服务。

---

## 一、环境与架构评估

### 关键约束
- 宿主机 **80/443 端口由宝塔 nginx 占用**，容器不能直接抢占这两个端口。
- 因此本方案：`web` 容器只绑定 **`127.0.0.1:9100`**（容器内部仍监听 8080，宿主机映射到 9100），由宝塔 nginx 作为对外入口反向代理到它，并在宝塔侧完成 SSL 证书。
- **宿主机对外暴露的端口统一使用 9100-9199 段**，避开宝塔 nginx 的 80/443 与其他已跑在 Docker 里的项目。
- `db` / `qdrant` / `api` **不对外暴露任何公网端口**，只在 Docker 内部网络可达，最大化安全面。
- 前端 API 请求走 `web` 容器内 nginx 的 **同域 `/api` 反代**（`/api → api:3000`），浏览器全程同源，无需处理 CORS。

### 服务拓扑
```
浏览器 ──HTTPS(443)──▶ 宝塔 nginx ──HTTP──▶ 127.0.0.1:9100 (宿主机端口) ─▶ web 容器 nginx (8080)
                                                ├── 静态 SPA (前端产物)
                                                └── /api ──▶ api:3000 (Express)
                                                                ├── db:5432   (PostgreSQL)
                                                                └── qdrant:6333 (向量库)
```

### 组件清单
| 服务 | 镜像/来源 | 说明 |
| --- | --- | --- |
| db | `postgres:16-alpine` | 关系数据库，数据存 `pg_data` 卷 |
| qdrant | `qdrant/qdrant:v1.15.4` | RAG 向量库，数据存 `qdrant_storage` 卷 |
| migrate | `Dockerfile.api` 产物 | 一次性任务，启动前部署 Prisma 表结构 |
| api | `Dockerfile.api` | Express 后端，图片存 `api_storage` 卷 |
| web | `Dockerfile.web` | nginx-unprivileged 托管前端 + `/api` 反代 |

---

## 二、前置准备：安装 Docker

> 以下所有命令均在**服务器终端**执行（宝塔面板 → 终端，或 SSH）。
> 涉及安装/系统操作，请你自己确认后执行。

### 方式 A：宝塔面板安装（推荐，最省事）
宝塔「软件商店」→ 搜索 **Docker**（或「Docker 管理器」）→ 安装。
安装后确认：
```bash
docker --version
docker compose version   # 需为 v2.x，输出类似 Docker Compose version v2.x
```

### 方式 B：官方脚本安装
```bash
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
docker compose version
```

如果 `docker compose`（带空格，v2 插件）不可用，需安装插件：
```bash
apt-get update && apt-get install -y docker-compose-plugin
```

### 内存与磁盘建议
- 至少 **2GB 内存**（Qdrant + PostgreSQL + Node 并发时更稳），推荐 4GB。
- 首次构建镜像 + 拉取基础镜像约需 **10–15GB 空闲磁盘**，用 `df -h` 确认。

---

## 三、上传代码到服务器

假设部署目录为 `/www/wwwroot/z-ai-novel`（可按需替换）。

### 方式 A：Git 拉取（推荐，便于后续更新）
```bash
mkdir -p /www/wwwroot && cd /www/wwwroot
git clone <你的仓库地址> z-ai-novel
cd z-ai-novel
```

### 方式 B：打包上传
把本地代码（**排除 `node_modules/`、`.env`**）上传到 `/www/wwwroot/z-ai-novel`，
宝塔「文件」管理上传 zip 后解压即可。

---

## 四、配置环境变量

compose 读取的是 `infra/.env`。从模板复制并填写：

```bash
cd /www/wwwroot/z-ai-novel/infra
cp .env.production.example .env
nano .env    # 或 vim/nano，或用宝塔文件管理器编辑
```

**必须修改/填写的项：**

1. 数据库密码（务必改成强密码）：
   ```env
   POSTGRES_PASSWORD=<一个强密码>
   ```
2. 至少配置一组可用的模型 API Key（否则 AI 功能不可用）：
   ```env
   OPENAI_API_KEY=sk-...
   # 或 DEEPSEEK_API_KEY / SILICONFLOW_API_KEY / 等，任选你在用的
   ```
3. Embedding（RAG 用）。若暂时不想启用知识库检索，先关掉跑通主流程：
   ```env
   RAG_ENABLED=false      # 准备好 embedding 模型后再改回 true
   ```

> 说明：`DATABASE_URL`、`AI_NOVEL_DATABASE_MODE`、`QDRANT_URL`、`HOST`、`PORT`、
> `IMAGE_STORAGE_ROOT` 已由 `docker-compose.yml` 自动注入，**无需在 .env 里填写**。
> `infra/.env` 已在 `.gitignore` 中忽略，不会误提交密钥。

---

## 五、启动服务

在 **`infra/` 目录下**执行（compose 项目目录即 `infra/`）：

```bash
cd /www/wwwroot/z-ai-novel/infra
docker compose up -d --build
```

- 首次执行会拉取基础镜像并构建 `api` / `web` 镜像，视机器性能约 **5–20 分钟**。
- 启动顺序：`db`(健康) + `qdrant` → `migrate`(建表成功后退出) → `api` → `web`。

### 查看状态与日志
```bash
docker compose ps                       # 各服务应为 running / healthy
docker compose logs -f migrate          # 确认表结构部署成功（无报错退出）
docker compose logs -f api              # 后端启动日志
docker compose logs -f web
```

### 本地连通性自检（服务器内部）
```bash
curl -I http://127.0.0.1:9100           # 前端页面应返回 200
```
> `/api/health` 带鉴权，未带凭据返回 401/403 属正常，说明后端已连通。

---

## 六、宝塔 nginx 反向代理 + SSL

在宝塔「网站」中新建一个**站点**（绑定你的域名，如 `novel.example.com`），
然后为它配置**反向代理**到 `web` 容器。

### 步骤
1. 网站 → 对应站点 → 「反向代理」→ 添加反向代理：
   - 目标 URL：`http://127.0.0.1:9100`
   - 发送域名：`$host`
2. 保存后进入该反向代理的「配置文件」，确保包含 **SSE / 流式响应**所需指令。
   在 `location /` 块内（或代理配置段）加入/核对以下内容：

```nginx
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# SSE / 长连接（模型流式输出 llm-live 等）必需
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_buffering off;
proxy_read_timeout 300s;
proxy_send_timeout 300s;
```

3. 「SSL」→ 申请 Let's Encrypt 证书 → 开启「强制 HTTPS」。
4. 保存后重载 nginx。

> 前端以**同域 `/api`**访问后端，宝塔只需把整站反向代理到 `127.0.0.1:9100` 即可，
> 无需为 API 单独配置域名或处理跨域。

### 验证外网访问
浏览器打开 `https://novel.example.com`，能看到登录/首页即代表 web + api + db 全链路打通。

---

## 七、日常运维命令

均在 `/www/wwwroot/z-ai-novel/infra` 下执行。

```bash
# 查看运行状态
docker compose ps

# 重启全部服务
docker compose restart

# 只重启后端 / 前端
docker compose restart api
docker compose restart web

# 查看实时日志
docker compose logs -f api

# 停止服务（保留数据卷）
docker compose down
```

### 代码更新后重新部署
```bash
cd /www/wwwroot/z-ai-novel
git pull
cd infra
docker compose up -d --build      # 重建镜像并滚动更新；数据卷不受影响
```
> 若更新包含新的数据库迁移，`migrate` 服务会自动再次执行 `prisma migrate deploy`。

### 关于种子数据
后端启动时会自动初始化系统基础/引导数据（`ensureSystemResourceStarterData`），
**常规部署无需手动 seed**。如确需执行 Prisma 种子脚本，注意生产 runtime 镜像
默认面向 `dist` 运行，`db seed` 依赖 `ts-node-dev` 与 `src/db/seed.ts`，可能不在镜像内；
此时改用一次性容器从源码执行，或直接在应用内完成初始化：
```bash
# 仅在确认镜像包含 seed 依赖时使用
docker compose run --rm api node_modules/.bin/prisma db seed --config /app/server/prisma.config.ts
```

### 备份数据库（重要）
```bash
docker compose exec -T db pg_dump -U postgres ai_novel > backup_$(date +%F).sql
```

> ⚠️ **数据销毁警告**：`docker compose down -v` 会**删除 `pg_data`/`qdrant_storage`/`api_storage` 三个卷，
> 即清空全部小说数据、向量库和生成图片**。除非你已确认可丢失并有备份，否则**不要执行 `-v`**。

---

## 八、故障排查

| 现象 | 排查方向 |
| --- | --- |
| `migrate` 反复失败 | `docker compose logs migrate`；多为 `POSTGRES_PASSWORD` 与 db 实际不一致，改了密码需 `docker compose down` 后重建（或首次启动前就设好） |
| `api` 启动即退出 | 看 `logs api`：生产模式缺 `DATABASE_URL` 或数据库不可达会直接抛错 |
| 页面能开但请求失败 | 检查宝塔反代是否把 `/api` 一并转发（同域反代默认转发所有路径）；`curl -I http://127.0.0.1:9100` 自检 |
| 流式输出卡住/超时 | 宝塔反代未加 `proxy_buffering off` 与 `Upgrade` 头，按第六步补齐 |
| 端口 9100 被占用 | `ss -lntp | grep 9100`；改 `docker-compose.yml` 里 web 的宿主机端口映射（保持 9100-9199），并同步宝塔反代目标 |
| 构建内存不足被 kill | 增加 swap 或提升机器内存后重试 `docker compose build` |

---

## 九、涉及的文件（本次新增/修改）

- `infra/docker-compose.yml` — 完整 5 服务编排 + 3 数据卷
- `infra/.env.production.example` — 生产环境变量模板（复制为 `infra/.env` 使用）
- `infra/nginx/ai-novel-web.conf` — web 容器 nginx 增加 `/api` 同域反代（含 SSE Upgrade 头）
- `.gitignore` — 显式忽略 `infra/.env` / `infra/.env.local`
- `deploy.md` — 本部署指南

---

## 十、最小执行清单（速查）

```bash
# 1. 装 Docker（宝塔软件商店 或 curl -fsSL https://get.docker.com | sh）
# 2. 上传/克隆代码到 /www/wwwroot/z-ai-novel
# 3. 配置环境变量
cd /www/wwwroot/z-ai-novel/infra
cp .env.production.example .env && nano .env    # 改数据库密码 + 填 API Key
# 4. 启动
docker compose up -d --build
docker compose ps
# 5. 宝塔新建站点 → 反向代理 http://127.0.0.1:9100 → 补 SSE 头 → 申请 SSL
# 6. 浏览器访问 https://你的域名
```
