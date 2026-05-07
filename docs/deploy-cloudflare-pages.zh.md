# 使用 Cloudflare Pages 部署用户端（user-ui）

本文说明如何将 [duomihost/user-ui](https://github.com/duomihost/user-ui)（Vue 3 + Vite）部署到 [Cloudflare Pages](https://pages.cloudflare.com/)，并列出常见坑（CORS、SPA 路由、环境变量）。

---

## 前置条件

- Cloudflare 账号，且已将域名托管到 Cloudflare（可选，用于自定义域名）。
- 后端 API 已部署并可公网访问（例如 `https://api.example.com`）。
- 仓库在 GitHub / GitLab / Bitbucket 等 Pages 支持的 Git 托管上（或通过 **Direct Upload** 上传构建产物）。

---

## 一、在 Cloudflare Dashboard 创建项目

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**。
2. 选择托管方并授权，选中 **user-ui** 仓库。
3. 配置构建设置（见下表）。

| 项 | 建议值 |
|----|--------|
| **Production branch** | `main`（或你的主分支） |
| **Framework preset** | `Vite`（或选 None 后手动填下面两项） |
| **Build command** | `npm run build` |
| **Build output directory** | `dist` |
| **Root directory** | `/`（若 monorepo 里子目录才是前端，填该子目录） |

4. 点击 **Save and Deploy**，首次会先跑一次构建。

---

## 二、环境变量（构建时注入）

本项目在构建时会读取 **`VITE_API_BASE_URL`**，用于浏览器直接请求后端（生产环境没有 Vite 开发代理）。

在 Pages 项目 → **Settings** → **Environment variables** 中，为 **Production**（以及需要的 **Preview**）添加：

| 变量名 | 示例值 | 说明 |
|--------|--------|------|
| `VITE_API_BASE_URL` | `https://api.example.com` | **不要**末尾斜杠；与 `src/api/client.ts` 中拼接的 `/api/v1` 一致 |
| `NODE_VERSION` | `20` 或 `22` | 可选；避免默认 Node 过旧导致 Vite 7 / 依赖构建失败 |

修改环境变量后需要 **重新触发一次部署**（Redeploy）才会生效。

本地自检：

```bash
export VITE_API_BASE_URL=https://api.example.com
npm ci
npm run build
npx vite preview
```

---

## 三、SPA 路由（history 模式）

项目使用 `vue-router` 的 **`createWebHistory`**，直接访问或刷新非根路径（如 `/products`）时，静态托管需要把所有路径回退到 `index.html`。

本仓库已在 `public/_redirects` 中加入（构建后会复制到 `dist`）：

```txt
/*    /index.html   200
```

若你 fork 后没有该文件，请在 `public/_redirects` 自行添加上述一行，否则子路径刷新可能 **404**。

---

## 四、与 Cloudflare 相关的已有配置

`vite.config.ts` 中为入口脚本加了 `data-cfasync="false"`，用于减少 Rocket Loader 等对 `type="module"` 脚本的干扰。一般无需再改。

---

## 五、自定义域名与 HTTPS

1. Pages 项目 → **Custom domains** → 按向导添加域名（例如 `shop.example.com`）。
2. 按 Cloudflare 提示完成 DNS（通常自动 CNAME 到 Pages）。
3. 证书由 Cloudflare 自动签发。

---

## 六、CORS 与 `X-Lang`（常见问题）

前端会向 API 携带自定义头 **`X-Lang`**（语言）。当站点在 **`*.pages.dev` 或独立域名**、API 在 **另一域名** 时，浏览器会发 CORS 预检。

**后端必须在 OPTIONS / CORS 响应中允许该头**，例如在 `Access-Control-Allow-Headers` 中包含 `x-lang`（及你实际使用的 `Authorization`、`Content-Type` 等）。否则控制台会出现类似：

> Request header field x-lang is not allowed by Access-Control-Allow-Headers in preflight response

这与 Pages 部署方式无关，是跨域策略问题；**修 API 的 CORS 配置**即可。

同时请将 **`Access-Control-Allow-Origin`** 设为你的前端源（如 `https://shop.example.com`），不要用 `*` 搭配带 Cookie 或部分敏感头。

---

## 七、预览环境与生产环境

- **Preview deployments**：每个 PR/分支可生成独立 `*.pages.dev` 子域；同样配置 `VITE_API_BASE_URL`（可指向测试 API）。
- 若预览域也要调生产 API，记得在 API 的 CORS 里允许对应 **`https://<preview>.pages.dev`**，或使用通配/白名单策略（注意安全边界）。

---

## 八、不连接 Git 的部署方式（可选）

1. 本地：`npm ci && npm run build`。
2. Pages → **Create** → **Direct Upload**，上传 **`dist`** 目录打成的 zip，或使用 [Wrangler](https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/) 在 CI 里上传。

Direct Upload 同样要在 Dashboard 里配置 **环境变量**（或在 wrangler 中指定），否则 `VITE_*` 不会注入。

---

## 九、部署后检查清单

- [ ] 首页能打开，接口无 CORS 报错（Network + Console）。
- [ ] 登录、商品列表等依赖 API 的页面正常。
- [ ] 直接打开或刷新深层链接（非 `/`）不 404。
- [ ] `VITE_API_BASE_URL` 指向正确环境，无混合内容（HTTPS 页不要请求 `http://` API）。

---

## 参考链接

- [Cloudflare Pages — Get started](https://developers.cloudflare.com/pages/get-started/)
- [Cloudflare Pages — Build configuration](https://developers.cloudflare.com/pages/configuration/build-configuration/)
- [Vite — Env Variables](https://vite.dev/guide/env-and-mode.html)
