# Docker 使用笔记（docker-demo）

## 结论
这套流程可以拆成 3 件事：
1. 用 Docker 容器做本地开发（跑 Vite，支持热更新）。
2. 用多阶段构建做生产镜像（先构建静态资源，再交给 Nginx）。
3. 用 Nginx 托管前端静态文件，并处理前端路由回退。

## 1. 如何用 Docker 容器做本地开发

### 1.1 目标
本地开发时，我们希望：
- 代码改了马上生效。
- 依赖环境统一，不污染宿主机。
- 所有人用同一套启动命令。

### 1.2 当前项目怎么做
`docker-compose.yml` 里当前配置是：
- 镜像：`node:22-alpine`
- 工作目录：`/app`
- 端口：`3000:3000`
- 挂载：
  - `./:/app`（把本地代码挂到容器）
  - `docker_app_node_modules:/app/node_modules`（依赖单独放在卷里）
- 启动命令：`npm install && npm run dev -- --host 0.0.0.0 --port 3000`

### 1.3 常用命令
```bash
cd /Users/ddbbb/project/my_project/nest-study/docker-demo

# 启动开发容器
docker compose up -d

# 看日志
docker compose logs -f docker_app

# 停止
docker compose down

# 连同依赖卷一起清理（下次会重新安装依赖）
docker compose down -v
```

## 2. 如何做生产构建

### 2.1 为什么用多阶段构建
核心思想：
- 构建阶段需要 Node（安装依赖、执行 `npm run build`）。
- 运行阶段只需要一个静态文件服务器（Nginx）。

这样做的好处：
- 运行镜像更小。
- 不把构建工具带到生产环境。
- 安全面更小，启动更快。

### 2.2 当前 `dockerfile` 做了什么
1. `FROM node:22-alpine AS builder`
2. 拷贝 `package*.json`，执行 `npm ci`
3. 拷贝源码并执行 `npm run build`
4. `FROM nginx:1.27-alpine`
5. 拷贝 `nginx.conf` 到 `/etc/nginx/conf.d/default.conf`
6. 把 `dist` 拷贝到 `/usr/share/nginx/html`

### 2.3 构建命令
```bash
cd /Users/ddbbb/project/my_project/nest-study/docker-demo
docker build -t docker-demo-web:latest -f dockerfile .
```

## 3. Nginx 的使用与原理

### 3.1 在这个项目里，Nginx 负责什么
- 读取 `/usr/share/nginx/html` 下的静态资源（HTML/CSS/JS）。
- 对外提供 HTTP 服务（80 端口）。

### 3.2 当前配置重点
`nginx.conf` 的关键是：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

### 3.3 这行配置的原理
访问一个 URL 时，Nginx 会按顺序尝试：
1. 找有没有这个文件（`$uri`）
2. 找有没有这个目录（`$uri/`）
3. 都没有就返回 `/index.html`

前端单页应用（SPA）里，`/user/1` 这类路径通常是前端路由，不是服务器真实文件。没有这条回退规则时，刷新页面就会 404；有了它，就会回到 `index.html`，由前端路由接管。

## 4. 一句话记忆
- 开发阶段：`docker compose` + 挂载源码 + Vite 热更新。
- 生产阶段：多阶段构建产出 `dist` + Nginx 托管静态资源。
- 路由关键：`try_files ... /index.html` 解决 SPA 刷新 404。
