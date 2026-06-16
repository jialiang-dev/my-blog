# Umami 集成问题排查与解决

## 1. @unhead/vue v3 API 变化

**现象：** 所有页面空白，仅 DarkModeToggle 可见。控制台报错 `useHead() was called without provide context`。查看源代码有 HTML，但 Vue 组件全部渲染失败。

**原因：** `@unhead/vue` v3 将 API 按 SPA/SSR 拆分：

- 旧：`import { createHead } from '@unhead/vue'`
- 新：`import { createHead } from '@unhead/vue/client'`（SPA）/ `@unhead/vue/server`（SSR）

`createUnhead()` 从 `@unhead/vue` 直接导入不是一个合法的 Vue plugin，无法正确 provide context，导致所有 `useHead()` 调用失败。

**修复：** [src/main.js](blog-frontend/src/main.js) — 改用 `@unhead/vue/client`，并在 router 之前注册。

---

## 2. ghcr.io 镜像拉取失败

**现象：** `docker pull ghcr.io/umami-software/umami:mysql-latest` 报 TLS handshake timeout / EOF，大文件层（56MB + 23MB）无法下载。

**原因：** ghcr.io 走 GitHub CDN（`pkg-containers.githubusercontent.com`），国内直连不稳定，通过代理也经常断裂。

**修复：** 改用 UCloud 香港服务器拉取镜像，推送到 UHub（`uhub.service.ucloud.cn/myblog_docker/umami:mysql-latest`），所有环境走 UCloud 内网拉取。

---

## 3. 阿里云 Docker 镜像加速器 403

**现象：** `docker pull nginx:alpine` 报 403 Forbidden from `8l2xjeic.mirror.aliyuncs.com`。

**原因：** 阿里云容器镜像服务个人版已停止对外提供 Docker Hub 镜像加速。

**修复：** 全部基础镜像（node, nginx, mysql, redis, maven, eclipse-temurin）推送到 UHub，Dockerfile 和 compose 统一改用 UHub 地址。

---

## 4. Umami BASE_PATH 不生效

**现象：** 设置 `BASE_PATH=/umami` 后，访问 `/umami/login` 返回 404，访问 `/` 却直接显示 Dashboard。

**原因：** Umami v2.20.2 的 Next.js 配置对 `BASE_PATH` 环境变量的处理存在 bug，该变量被忽略，所有路由始终在 `/` 下注册。

**修复：** 去掉 `BASE_PATH`，由 nginx 负责剥离前缀：
```nginx
location /umami/ {
    proxy_pass http://umami:3000/;  # 剥离 /umami/ → /
}
```

---

## 5. Umami 重定向死循环

**现象：** 浏览器报 `ERR_TOO_MANY_REDIRECTS`。

**原因：** Next.js 默认去掉末尾斜杠：`/umami/` → 308 → `/umami`（无斜杠）。nginx 的 `location /` 收到 `/umami` 后执行 `try_files $uri/`，发出 301 重定向回 `/umami/`，形成死循环。

**修复：** 添加 `location = /umami` 精确匹配，直接代理到 Umami（不经过 SPA catch-all）。

---

## 6. Umami JS 跳转丢失路径前缀

**现象：** 通过 nginx 代理的 Umami 页面，JS 客户端跳转（如 `/login` → `/dashboard`）变成 `/login` 和 `/dashboard`，落到 SPA 的 index.html 空白页。

**原因：** `proxy_redirect` 只能改写 HTTP `Location` 响应头，无法改写 HTML/JS 中的路径引用。Umami 的 Next.js 客户端路由使用浏览器 History API 或 `window.location` 跳转，这些不受 nginx 控制。

**修复：** Umami 后台管理页面改为独立端口直连（`:3000`），不再通过 nginx 路径代理。仅跟踪脚本 `script.js` 和 API 调用保留 nginx 代理（保证同源不被广告拦截器拦截）。

---

## 7. Docker daemon 反复 SIGBUS 崩溃

**现象：** 在 WSL2 中执行长时间 `docker build`（如 `npm install` + `next build`）时，Docker daemon 反复 `SIGBUS: bus error` 崩溃。

**原因：** WSL2 内存/IO 压力下 Docker Desktop 不稳定，尤其是构建阶段的大量文件写入触发内核级 bug。

**修复：** 弃用本地构建，改用 UHub 预构建镜像。UCloud 香港服务器网络无墙，可直接拉取 ghcr.io。

---

## 8. 本地 SSL 证书缺失

**现象：** nginx 容器启动失败：`cannot load certificate "/etc/ssl/cloudflare/jialiang.dev.pem"`。

**原因：** compose 挂载了 `/etc/ssl/cloudflare`，该目录仅在服务器上存在。

**修复：** 本地生成自签证书挂载到 `./ssl` 目录，`.gitignore` 排除。
```bash
openssl req -x509 -nodes -days 1 -newkey rsa:2048 \
  -keyout ssl/jialiang.dev.key -out ssl/jialiang.dev.pem -subj "/CN=localhost"
```

---

## 9. Umami 数据库未创建

**现象：** Umami 容器重启循环，日志显示 `Database 'umami' does not exist`。

**原因：** MySQL 容器重建时，`mysql-init` 脚本仅在首次初始化时执行。已存在的 `mysql_data` 卷不会触发数据库创建。

**修复：** 手动执行建库 SQL：
```sql
CREATE DATABASE IF NOT EXISTS umami CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## 最终架构

```
                    localhost:3000 (Umami 后台直连)
                         ↑
  HTTPS ──────────► frontend (nginx:443)
  localhost          ├─ /            → SPA 静态文件
                     ├─ /api/*       → backend:8080
                     ├─ /ws/*        → backend:8080
                     ├─ /umami/*     → umami:3000 (仅跟踪脚本/API)
                     └─ /assets/     → 静态资源 (1y 缓存)

                    umami:3000 (ghcr → UHub 预构建)
                    backend:8080 (Spring Boot)
                    db:3306 (MySQL 8.0)
                    redis:6379
```

**跟踪脚本接入：** 一个 `<script>` 标签，异步加载，不影响页面。服务挂了静默失败。
