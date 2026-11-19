# 本地开发服务启动手册

本文说明本地前端、Cloudflare Worker 和 Supabase 的启动方式、请求路径与数据落点。开始开发前，先确认自己要使用远端 Preview Worker 还是本地 Worker。

## 快速判断

| 启动方式 | 前端 | API 后端 | 数据库 |
| --- | --- | --- | --- |
| `pnpm dev` | 本地 Vite | 远端 `qimen-preview` Worker | 前端直连的 Supabase 是 `.env.local` 配置的项目；远端 Worker 的数据库和密钥须另行确认 |
| 本地 Worker + `API_PROXY_TARGET` | 本地 Vite | 本地 Wrangler Worker | Worker 仍连接 `worker/.dev.vars` 中指定的 Supabase；本项目默认配置是云端 Supabase，不是本地数据库 |

仅运行 `pnpm dev` **不等于全本地运行**。Vite 配置会把 `/api/*` 代理到 `https://qimen-preview.oceanjustinlin.workers.dev`，除非设置 `API_PROXY_TARGET` 覆盖目标。

## 启动前配置

确保已在项目根目录安装依赖，并准备以下被 `.gitignore` 忽略的本地配置文件：

- `.env.local`：前端 Supabase 配置，使用 `VITE_SUPABASE_URL` 和 `VITE_SUPABASE_ANON_KEY`。
- `worker/.dev.vars`：本地 Worker 服务端配置，至少包含 `SUPABASE_URL`、`SUPABASE_SERVICE_KEY` 和 `LLM_API_KEY`。

前端 URL 与 Worker URL 必须属于同一个 Supabase 项目。`SUPABASE_SERVICE_KEY` 和 `LLM_API_KEY` 是服务端密钥，只能放在 `worker/.dev.vars` 或受控的服务端密钥配置中；不要填入 `VITE_*` 变量、提交到 Git、截图或粘贴到公开对话。

本地 Worker 使用云端 Supabase 时，创建、修改或删除的数据会进入该 Supabase 项目。调试破坏性操作前确认数据库是开发/测试项目；不要因为 Worker 在本地运行就假设数据库也在本地。

## 方式一：前端 + 远端 Preview Worker

在项目根目录运行：

```bash
pnpm dev
```

Vite 启动本地前端，终端会打印访问地址，通常是 `http://localhost:5173`。浏览器发往相对路径 `/api/*` 的请求由 Vite 转发到：

```text
https://qimen-preview.oceanjustinlin.workers.dev
```

这条路径使用远端 Worker 的代码和 Cloudflare 密钥，不使用本机 `worker/.dev.vars`。仓库中的 `worker/wrangler.toml` 将 Preview 环境的 `SUPABASE_URL` 配置为本项目的 Supabase ref，但无法仅凭本地仓库确认远端 Worker 当前部署的代码和服务端密钥是否与工作区一致。发真实或敏感数据前，先确认远端服务的控制权及其 Supabase 配置。

前端 Supabase 客户端另行使用 `.env.local` 中的 URL 和 publishable/anon key，直接连接该 Supabase 项目。也就是说，同一次本地开发会话可能同时访问自己的 Supabase 和不受本机控制的远端 Worker。

## 方式二：前端 + 本地 Worker

使用两个终端，并保持两个服务都在运行。

终端一：从项目根目录进入 `worker/`，启动本地 Worker：

```bash
cd worker
npx wrangler dev --local --ip 127.0.0.1 --port 8787
```

Wrangler 会加载 `worker/wrangler.toml` 和 `worker/.dev.vars`。如果本机尚无 Wrangler，`npx` 可能提示临时下载；确认包名是 `wrangler` 后再继续。此命令启动本地 Worker，不是部署命令。

终端二：在项目根目录启动 Vite，并显式将 API 代理到本地 Worker：

```bash
cd /path/to/qimen
API_PROXY_TARGET=http://127.0.0.1:8787 pnpm dev
```

将 `/path/to/qimen` 换成实际仓库目录。当前机器也可以使用：

```bash
cd /Users/sm2732/self-projects/qimen-bazi/qimen
API_PROXY_TARGET=http://127.0.0.1:8787 pnpm dev
```

浏览器仍访问 Vite 打印的地址（通常为 `http://localhost:5173`），请求路径为：

```text
浏览器 → 本地 Vite → http://127.0.0.1:8787 → 本地 Worker → worker/.dev.vars 指定的 Supabase / LLM 服务
```

此模式不需要 Cloudflare 账号来部署 Worker，但 API 请求仍会访问 `.dev.vars` 中配置的外部 Supabase 和模型服务。停止时，在两个终端分别按 `Ctrl+C`。

## 配置优先级与常见误区

- `API_PROXY_TARGET` 控制 Vite 服务端 `/api/*` 代理目标。未设置时，默认值是远端 `qimen-preview` Worker。
- 部分前端视图读取 `VITE_API_BASE`。若设置了它，相关请求会使用指定的 API base，可能绕过 Vite 代理；本地联调时确认该变量未指向其他环境。
- `.env.local` 的 Supabase 配置供浏览器端 Supabase 客户端使用，不会自动启动数据库，也不会配置 Worker 的服务端密钥。
- `worker/.dev.vars` 只在本地 Wrangler Worker 运行时加载。单独运行 `pnpm dev` 不会加载它。
- Cloudflare Pages Functions（例如 `/auth/*` 和 `functions/_middleware.js`）不由普通 Vite 开发服务器自动运行；涉及这些路由时，应在 Pages 部署环境验证，或单独配置 Pages 本地开发流程。
- `supabase/config.toml` 是 Supabase CLI 的本地项目配置，不会因 `pnpm dev` 自动启动本地 Supabase 数据库。

## 安全与数据检查

每次开发前确认：

1. `.env.local` 的 Supabase URL 指向预期项目。
2. `worker/.dev.vars` 的 Supabase URL 与前端属于同一个项目。
3. 要使用本地 Worker 时，Vite 命令包含 `API_PROXY_TARGET=http://127.0.0.1:8787`，并确认 `8787` 上运行的是本地 Wrangler。
4. 要使用远端 Preview Worker 时，确认其运营方、当前部署版本及该环境的 `SUPABASE_URL` 和服务端密钥归属。
5. 不在浏览器环境、Git、日志或共享截图中暴露服务端密钥。若密钥曾被公开分享，应在对应服务商处撤销并重新生成。

## 相关配置

- Vite API 代理：`vite.config.js`
- 前端 Supabase 客户端：`src/lib/supabase.mjs`
- 页面 API base 选择：`src/views/HomeView.vue`、`src/views/BaziView.vue`
- Worker 部署与环境变量：`worker/wrangler.toml`
- Pages Functions 认证入口：`functions/auth/`