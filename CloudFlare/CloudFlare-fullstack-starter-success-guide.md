# Cloudflare Fullstack Starter 成功搭建教程（WSL / 脱敏版）

> 目标：从全新 WSL 环境搭建一个可复用的 Cloudflare 全栈模板，覆盖 Pages、Workers、D1、KV、R2、Turnstile、Cloudflare Access、Rate Limit、Tunnel 和 GitHub Actions。

---

## 0. 脱敏约定

| 实际信息 | 文档中统一替换为 |
|---|---|
| 真实主域名 | `example.com` |
| Preview 子域名 | `preview.example.com` |
| Tunnel 子域名 | `dev.example.com` |
| Cloudflare Account ID | `<CLOUDFLARE_ACCOUNT_ID>` |
| Preview D1 Database ID | `<PREVIEW_D1_DATABASE_ID>` |
| Production D1 Database ID | `<PRODUCTION_D1_DATABASE_ID>` |
| Preview KV Namespace ID | `<PREVIEW_KV_NAMESPACE_ID>` |
| Production KV Namespace ID | `<PRODUCTION_KV_NAMESPACE_ID>` |
| Turnstile Site Key | `<TURNSTILE_SITE_KEY>` |
| Turnstile Secret Key | `<TURNSTILE_SECRET_KEY>` |
| Cloudflare API Token | `<CLOUDFLARE_API_TOKEN>` |
| Cloudflare Tunnel Token | `<TUNNEL_TOKEN>` |

---

## 1. 项目目标

本项目最终实现：

```text
Cloudflare Pages        -> 首页 / 前端
Cloudflare Worker       -> /api/*
D1                      -> articles / contact_messages
KV                      -> /api/config
R2                      -> /api/files/:key
Turnstile               -> contact 表单防刷
Cloudflare Access       -> /admin/*
Rate Limiting / WAF     -> /api/contact 防刷
Cloudflare Tunnel       -> dev.example.com 暴露本地 dashboard
GitHub Actions          -> 自动部署
```

推荐域名规划：

```text
https://example.com                     -> Pages 前端
https://example.com/api/*               -> Worker API
https://example.com/admin/*             -> Pages admin 静态页面，由 Access 保护
https://preview.example.com/api/*       -> Preview Worker API
https://dev.example.com                 -> Cloudflare Tunnel 暴露本地服务
```

---

## 2. WSL 环境准备

### 2.1 项目目录要求

项目建议放在 WSL Linux 文件系统内：

```bash
~/projects/cloudflare-fullstack-starter
```

不建议放在：

```bash
/mnt/c/Users/<USER>/...
```

原因：

| 路径 | 说明 |
|---|---|
| `~/projects/...` | 推荐，npm install、Vite、Wrangler、文件监听更稳定 |
| `/mnt/c/...` | 不推荐，跨 Windows/WSL 文件系统，`node_modules` 慢，watch 容易异常 |

---

### 2.2 安装基础工具

```bash
sudo apt update
sudo apt install -y \
  curl \
  git \
  unzip \
  jq \
  ca-certificates \
  build-essential \
  dnsutils
```

---

### 2.3 安装 Node.js

```bash
cd ~

curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"

nvm install 22
nvm use 22
nvm alias default 22

node -v
npm -v
```

---

### 2.4 安装 Wrangler

```bash
npm install -g wrangler
wrangler --version
```

登录 Cloudflare：

```bash
wrangler login
wrangler whoami
```

如果本机环境变量中存在权限不足的 API token，可能覆盖 `wrangler login` 的浏览器登录态：

```bash
env | grep CLOUDFLARE
```

如果看到：

```text
CLOUDFLARE_API_TOKEN=...
CLOUDFLARE_ACCOUNT_ID=...
```

本地调试时建议先临时清除：

```bash
unset CLOUDFLARE_API_TOKEN
unset CLOUDFLARE_ACCOUNT_ID
```

---

## 3. 创建项目结构

```bash
mkdir -p ~/projects
cd ~/projects

mkdir cloudflare-fullstack-starter
cd cloudflare-fullstack-starter

mkdir -p apps/web/src
mkdir -p apps/web/public/admin
mkdir -p apps/api/src/routes
mkdir -p apps/api/src/lib
mkdir -p apps/api/migrations
mkdir -p docs
mkdir -p scripts
mkdir -p .github/workflows
```

最终目录：

```text
cloudflare-fullstack-starter/
├── apps/
│   ├── web/
│   └── api/
├── docs/
├── scripts/
├── .github/
└── package.json
```

---

## 4. 根目录 `package.json`

```bash
cat > package.json <<'EOF'
{
  "name": "cloudflare-fullstack-starter",
  "private": true,
  "workspaces": [
    "apps/*"
  ],
  "scripts": {
    "dev:api": "npm --workspace @cf-starter/api run dev",
    "dev:web": "npm --workspace @cf-starter/web run dev",
    "build:web": "npm --workspace @cf-starter/web run build",
    "typecheck": "npm --workspace @cf-starter/api run typecheck && npm --workspace @cf-starter/web run typecheck",
    "deploy:api:preview": "npm --workspace @cf-starter/api run deploy:preview",
    "deploy:api:production": "npm --workspace @cf-starter/api run deploy:production",
    "deploy:pages": "bash scripts/deploy-pages.sh"
  },
  "devDependencies": {
    "typescript": "latest",
    "wrangler": "latest"
  }
}
EOF
```

---

## 5. 前端 Pages 应用

### 5.1 `apps/web/package.json`

```bash
cat > apps/web/package.json <<'EOF'
{
  "name": "@cf-starter/web",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite --host 0.0.0.0",
    "build": "tsc -b && vite build",
    "preview": "vite preview --host 0.0.0.0",
    "typecheck": "tsc -b"
  },
  "dependencies": {
    "@vitejs/plugin-react": "latest",
    "vite": "latest",
    "typescript": "latest",
    "react": "latest",
    "react-dom": "latest"
  },
  "devDependencies": {
    "@types/react": "latest",
    "@types/react-dom": "latest"
  }
}
EOF
```

---

### 5.2 `apps/web/tsconfig.json`

```bash
cat > apps/web/tsconfig.json <<'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src"]
}
EOF
```

---

### 5.3 Vite 类型声明

创建 `apps/web/src/vite-env.d.ts`：

```bash
cat > apps/web/src/vite-env.d.ts <<'EOF'
/// <reference types="vite/client" />
EOF
```

该文件用于解决：

```text
Property 'env' does not exist on type 'ImportMeta'
Cannot find module or type declarations for side-effect import of './App.css'
```

---

### 5.4 `apps/web/index.html`

```bash
cat > apps/web/index.html <<'EOF'
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Cloudflare Fullstack Starter</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
EOF
```

---

### 5.5 `apps/web/src/main.tsx`

```bash
cat > apps/web/src/main.tsx <<'EOF'
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./App.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
EOF
```

---

### 5.6 `apps/web/src/App.tsx`

```bash
cat > apps/web/src/App.tsx <<'EOF'
import { useEffect, useState } from "react";
import type { FormEvent } from "react";

type Article = {
  id: number;
  slug: string;
  title: string;
  content: string;
  created_at: string;
};

type SiteConfig = {
  siteName?: string;
  description?: string;
  maintenance?: boolean;
};

declare global {
  interface Window {
    onTurnstileSuccess?: (token: string) => void;
  }
}

const API_BASE = import.meta.env.VITE_API_BASE || "";
const TURNSTILE_SITE_KEY = import.meta.env.VITE_TURNSTILE_SITE_KEY || "";

function api(path: string) {
  return `${API_BASE}${path}`;
}

export default function App() {
  const [health, setHealth] = useState<string>("checking...");
  const [articles, setArticles] = useState<Article[]>([]);
  const [config, setConfig] = useState<SiteConfig>({});
  const [turnstileToken, setTurnstileToken] = useState("");
  const [contactResult, setContactResult] = useState("");

  useEffect(() => {
    window.onTurnstileSuccess = (token: string) => {
      setTurnstileToken(token);
    };

    const script = document.createElement("script");
    script.src = "https://challenges.cloudflare.com/turnstile/v0/api.js";
    script.async = true;
    script.defer = true;
    document.head.appendChild(script);

    return () => {
      delete window.onTurnstileSuccess;
      document.head.removeChild(script);
    };
  }, []);

  useEffect(() => {
    fetch(api("/api/health"))
      .then((r) => r.json())
      .then((d) => setHealth(JSON.stringify(d, null, 2)))
      .catch((e) => setHealth(String(e)));

    fetch(api("/api/articles"))
      .then((r) => r.json())
      .then((d) => setArticles(d.data ?? []))
      .catch(() => setArticles([]));

    fetch(api("/api/config"))
      .then((r) => r.json())
      .then((d) => setConfig(d.data ?? {}))
      .catch(() => setConfig({}));
  }, []);

  async function submitContact(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();

    const form = new FormData(e.currentTarget);

    const payload = {
      name: String(form.get("name") || ""),
      email: String(form.get("email") || ""),
      message: String(form.get("message") || ""),
      turnstileToken
    };

    const res = await fetch(api("/api/contact"), {
      method: "POST",
      headers: {
        "content-type": "application/json"
      },
      body: JSON.stringify(payload)
    });

    const data = await res.json();
    setContactResult(JSON.stringify(data, null, 2));
  }

  return (
    <main className="page">
      <section className="hero">
        <h1>{config.siteName || "Cloudflare Fullstack Starter"}</h1>
        <p>
          {config.description ||
            "Pages + Workers + D1 + KV + R2 + Turnstile + Access + Tunnel"}
        </p>
      </section>

      <section className="card">
        <h2>Health</h2>
        <pre>{health}</pre>
      </section>

      <section className="card">
        <h2>Articles from D1</h2>
        {articles.length === 0 ? (
          <p>No articles.</p>
        ) : (
          <ul>
            {articles.map((a) => (
              <li key={a.id}>
                <strong>{a.title}</strong>
                <p>{a.content}</p>
                <small>{a.created_at}</small>
              </li>
            ))}
          </ul>
        )}
      </section>

      <section className="card">
        <h2>Contact</h2>
        <form onSubmit={submitContact}>
          <input name="name" placeholder="name" required />
          <input name="email" placeholder="email" />
          <textarea name="message" placeholder="message" required />

          <div
            className="cf-turnstile"
            data-sitekey={TURNSTILE_SITE_KEY}
            data-callback="onTurnstileSuccess"
          />

          <button type="submit">Send</button>
        </form>

        <pre>{contactResult}</pre>
      </section>

      <section className="card">
        <h2>Admin</h2>
        <a href="/admin/">Open admin dashboard</a>
      </section>
    </main>
  );
}
EOF
```

---

### 5.7 `apps/web/src/App.css`

```bash
cat > apps/web/src/App.css <<'EOF'
.page {
  max-width: 960px;
  margin: 0 auto;
  padding: 40px 20px;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

.hero {
  padding: 32px;
  border-radius: 20px;
  background: #f5f5f5;
  margin-bottom: 24px;
}

.card {
  border: 1px solid #ddd;
  border-radius: 16px;
  padding: 24px;
  margin-bottom: 24px;
}

input,
textarea {
  display: block;
  width: 100%;
  box-sizing: border-box;
  margin: 10px 0;
  padding: 10px;
}

textarea {
  min-height: 120px;
}

button {
  padding: 10px 16px;
  cursor: pointer;
}

pre {
  white-space: pre-wrap;
  word-break: break-word;
  background: #111;
  color: #0f0;
  padding: 12px;
  border-radius: 8px;
}
EOF
```

---

### 5.8 Admin 页面与 Pages redirects

```bash
cat > apps/web/public/admin/index.html <<'EOF'
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Admin Dashboard</title>
  </head>
  <body>
    <h1>Admin Dashboard</h1>
    <p>This page should be protected by Cloudflare Access.</p>
  </body>
</html>
EOF

cat > apps/web/public/_redirects <<'EOF'
/admin/* /admin/index.html 200
/* /index.html 200
EOF
```

---

## 6. Worker API 应用

### 6.1 `apps/api/package.json`

```bash
cat > apps/api/package.json <<'EOF'
{
  "name": "@cf-starter/api",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "deploy:preview": "wrangler deploy --env preview",
    "deploy:production": "wrangler deploy --env production",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "latest",
    "typescript": "latest",
    "wrangler": "latest"
  }
}
EOF
```

---

### 6.2 `apps/api/tsconfig.json`

```bash
cat > apps/api/tsconfig.json <<'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "lib": ["ES2022"],
    "types": ["@cloudflare/workers-types"],
    "noEmit": true
  },
  "include": ["src"]
}
EOF
```

---

## 7. Worker 配置：`wrangler.jsonc`

创建 `apps/api/wrangler.jsonc`：

```bash
cat > apps/api/wrangler.jsonc <<'EOF'
{
  "$schema": "../../node_modules/wrangler/config-schema.json",
  "name": "cf-fullstack-api",
  "main": "src/index.ts",
  "compatibility_date": "2026-05-22",
  "workers_dev": true,

  "vars": {
    "APP_ENV": "local",
    "ALLOWED_ORIGIN": "http://localhost:5173",
    "TURNSTILE_VERIFY_URL": "https://challenges.cloudflare.com/turnstile/v0/siteverify"
  },

  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "cf_fullstack_db_preview",
      "database_id": "<PREVIEW_D1_DATABASE_ID>",
      "migrations_dir": "migrations"
    }
  ],

  "kv_namespaces": [
    {
      "binding": "APP_CONFIG",
      "id": "<PREVIEW_KV_NAMESPACE_ID>"
    }
  ],

  "r2_buckets": [
    {
      "binding": "ASSETS_BUCKET",
      "bucket_name": "cf-fullstack-assets-preview"
    }
  ],

  "env": {
    "preview": {
      "name": "cf-fullstack-api-preview",
      "workers_dev": false,
      "routes": [
        {
          "pattern": "preview.example.com/api/*",
          "zone_name": "example.com"
        }
      ],
      "vars": {
        "APP_ENV": "preview",
        "ALLOWED_ORIGIN": "https://preview.example.com",
        "TURNSTILE_VERIFY_URL": "https://challenges.cloudflare.com/turnstile/v0/siteverify"
      },
      "d1_databases": [
        {
          "binding": "DB",
          "database_name": "cf_fullstack_db_preview",
          "database_id": "<PREVIEW_D1_DATABASE_ID>",
          "migrations_dir": "migrations"
        }
      ],
      "kv_namespaces": [
        {
          "binding": "APP_CONFIG",
          "id": "<PREVIEW_KV_NAMESPACE_ID>"
        }
      ],
      "r2_buckets": [
        {
          "binding": "ASSETS_BUCKET",
          "bucket_name": "cf-fullstack-assets-preview"
        }
      ]
    },

    "production": {
      "name": "cf-fullstack-api-production",
      "workers_dev": false,
      "routes": [
        {
          "pattern": "example.com/api/*",
          "zone_name": "example.com"
        }
      ],
      "vars": {
        "APP_ENV": "production",
        "ALLOWED_ORIGIN": "https://example.com",
        "TURNSTILE_VERIFY_URL": "https://challenges.cloudflare.com/turnstile/v0/siteverify"
      },
      "d1_databases": [
        {
          "binding": "DB",
          "database_name": "cf_fullstack_db",
          "database_id": "<PRODUCTION_D1_DATABASE_ID>",
          "migrations_dir": "migrations"
        }
      ],
      "kv_namespaces": [
        {
          "binding": "APP_CONFIG",
          "id": "<PRODUCTION_KV_NAMESPACE_ID>"
        }
      ],
      "r2_buckets": [
        {
          "binding": "ASSETS_BUCKET",
          "bucket_name": "cf-fullstack-assets"
        }
      ]
    }
  }
}
EOF
```

重点：

```text
代码使用 env.DB / env.APP_CONFIG / env.ASSETS_BUCKET
所以 binding 必须写 DB / APP_CONFIG / ASSETS_BUCKET
```

错误写法：

```jsonc
"binding": "cf_fullstack_db_preview"
```

正确写法：

```jsonc
"binding": "DB"
```

Cloudflare Workers 的 binding 在不同 `env` 下不会自动继承，`preview` 与 `production` 都必须显式配置 D1 / KV / R2。

---

## 8. D1 schema

创建 `apps/api/migrations/0001_init_schema.sql`：

```bash
cat > apps/api/migrations/0001_init_schema.sql <<'EOF'
CREATE TABLE IF NOT EXISTS articles (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  slug TEXT NOT NULL UNIQUE,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS contact_messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT,
  message TEXT NOT NULL,
  ip_hash TEXT,
  user_agent TEXT,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_articles_created_at
ON articles(created_at);

CREATE INDEX IF NOT EXISTS idx_contact_messages_created_at
ON contact_messages(created_at);
EOF
```

---

## 9. Worker 公共代码

### 9.1 `apps/api/src/types.ts`

```bash
cat > apps/api/src/types.ts <<'EOF'
export interface Env {
  DB: D1Database;
  APP_CONFIG: KVNamespace;
  ASSETS_BUCKET: R2Bucket;

  APP_ENV: string;
  ALLOWED_ORIGIN: string;
  TURNSTILE_VERIFY_URL: string;

  TURNSTILE_SECRET_KEY: string;
  IP_HASH_SALT: string;
}
EOF
```

---

### 9.2 `apps/api/src/lib/response.ts`

```bash
cat > apps/api/src/lib/response.ts <<'EOF'
export function json(data: unknown, init: ResponseInit = {}): Response {
  const headers = new Headers(init.headers);
  headers.set("content-type", "application/json; charset=utf-8");

  return new Response(JSON.stringify(data), {
    ...init,
    headers
  });
}

export function ok(data: unknown, init: ResponseInit = {}): Response {
  return json({ ok: true, data }, init);
}

export function fail(status: number, code: string, message: string): Response {
  return json(
    {
      ok: false,
      error: {
        code,
        message
      }
    },
    { status }
  );
}

export function methodNotAllowed(): Response {
  return fail(405, "method_not_allowed", "Method not allowed");
}

export function notFound(): Response {
  return fail(404, "not_found", "Not found");
}
EOF
```

---

### 9.3 `apps/api/src/lib/cors.ts`

```bash
cat > apps/api/src/lib/cors.ts <<'EOF'
import type { Env } from "../types";

export function corsHeaders(request: Request, env: Env): Headers {
  const headers = new Headers();

  const requestOrigin = request.headers.get("origin") || "";
  const allowedOrigin = env.ALLOWED_ORIGIN || "";

  if (allowedOrigin === "*") {
    headers.set("access-control-allow-origin", "*");
  } else if (requestOrigin && requestOrigin === allowedOrigin) {
    headers.set("access-control-allow-origin", requestOrigin);
    headers.set("vary", "Origin");
  } else if (!requestOrigin && allowedOrigin) {
    headers.set("access-control-allow-origin", allowedOrigin);
  }

  headers.set("access-control-allow-methods", "GET,POST,OPTIONS");
  headers.set("access-control-allow-headers", "content-type,authorization");
  headers.set("access-control-max-age", "86400");

  return headers;
}

export function withCors(response: Response, request: Request, env: Env): Response {
  const headers = new Headers(response.headers);

  for (const [k, v] of corsHeaders(request, env)) {
    headers.set(k, v);
  }

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers
  });
}

export function handleOptions(request: Request, env: Env): Response {
  return new Response(null, {
    status: 204,
    headers: corsHeaders(request, env)
  });
}
EOF
```

---

### 9.4 `apps/api/src/lib/turnstile.ts`

```bash
cat > apps/api/src/lib/turnstile.ts <<'EOF'
import type { Env } from "../types";

type SiteVerifyResponse = {
  success: boolean;
  "error-codes"?: string[];
  challenge_ts?: string;
  hostname?: string;
  action?: string;
  cdata?: string;
};

export async function verifyTurnstile(
  token: string,
  remoteIp: string | null,
  env: Env
): Promise<{ success: boolean; errors: string[] }> {
  if (!token) {
    return {
      success: false,
      errors: ["missing_token"]
    };
  }

  const body = new FormData();
  body.append("secret", env.TURNSTILE_SECRET_KEY);
  body.append("response", token);

  if (remoteIp) {
    body.append("remoteip", remoteIp);
  }

  const res = await fetch(env.TURNSTILE_VERIFY_URL, {
    method: "POST",
    body
  });

  if (!res.ok) {
    return {
      success: false,
      errors: [`siteverify_http_${res.status}`]
    };
  }

  const data = await res.json<SiteVerifyResponse>();

  return {
    success: Boolean(data.success),
    errors: data["error-codes"] ?? []
  };
}
EOF
```

---

## 10. Worker routes

### 10.1 `health.ts`

```bash
cat > apps/api/src/routes/health.ts <<'EOF'
import { ok } from "../lib/response";
import type { Env } from "../types";

export async function handleHealth(_request: Request, env: Env): Promise<Response> {
  return ok({
    service: "cf-fullstack-api",
    env: env.APP_ENV,
    ts: new Date().toISOString()
  });
}
EOF
```

---

### 10.2 `articles.ts`

```bash
cat > apps/api/src/routes/articles.ts <<'EOF'
import { fail, methodNotAllowed, ok } from "../lib/response";
import type { Env } from "../types";

export async function handleArticles(request: Request, env: Env): Promise<Response> {
  if (request.method !== "GET") {
    return methodNotAllowed();
  }

  const url = new URL(request.url);
  const limitRaw = Number(url.searchParams.get("limit") || "20");
  const limit = Number.isFinite(limitRaw) ? Math.min(Math.max(limitRaw, 1), 50) : 20;

  try {
    const result = await env.DB.prepare(
      `
      SELECT id, slug, title, content, created_at
      FROM articles
      ORDER BY id DESC
      LIMIT ?
      `
    )
      .bind(limit)
      .all();

    return ok(result.results ?? []);
  } catch (e) {
    return fail(500, "d1_query_failed", e instanceof Error ? e.message : String(e));
  }
}
EOF
```

---

### 10.3 `config.ts`

```bash
cat > apps/api/src/routes/config.ts <<'EOF'
import { fail, methodNotAllowed, ok } from "../lib/response";
import type { Env } from "../types";

const DEFAULT_CONFIG = {
  siteName: "Cloudflare Fullstack Starter",
  description: "Reusable Cloudflare fullstack template",
  maintenance: false
};

export async function handleConfig(request: Request, env: Env): Promise<Response> {
  if (request.method !== "GET") {
    return methodNotAllowed();
  }

  try {
    const value = await env.APP_CONFIG.get("site:config", "json");
    return ok(value ?? DEFAULT_CONFIG);
  } catch (e) {
    return fail(500, "kv_read_failed", e instanceof Error ? e.message : String(e));
  }
}
EOF
```

---

### 10.4 `files.ts`

```bash
cat > apps/api/src/routes/files.ts <<'EOF'
import { fail, methodNotAllowed } from "../lib/response";
import type { Env } from "../types";

export async function handleFiles(request: Request, env: Env): Promise<Response> {
  if (request.method !== "GET") {
    return methodNotAllowed();
  }

  const url = new URL(request.url);
  const prefix = "/api/files/";
  const rawKey = url.pathname.startsWith(prefix) ? url.pathname.slice(prefix.length) : "";
  const key = decodeURIComponent(rawKey);

  if (!key || key.includes("..") || key.startsWith("/")) {
    return fail(400, "invalid_key", "Invalid file key");
  }

  const object = await env.ASSETS_BUCKET.get(key);

  if (!object) {
    return fail(404, "file_not_found", "File not found");
  }

  const headers = new Headers();
  object.writeHttpMetadata(headers);
  headers.set("etag", object.httpEtag);
  headers.set("cache-control", "public, max-age=3600");

  return new Response(object.body, {
    status: 200,
    headers
  });
}
EOF
```

---

### 10.5 `contact.ts`

```bash
cat > apps/api/src/routes/contact.ts <<'EOF'
import { fail, methodNotAllowed, ok } from "../lib/response";
import { verifyTurnstile } from "../lib/turnstile";
import type { Env } from "../types";

type ContactBody = {
  name?: string;
  email?: string;
  message?: string;
  turnstileToken?: string;
  "cf-turnstile-response"?: string;
};

async function sha256Hex(input: string): Promise<string> {
  const data = new TextEncoder().encode(input);
  const digest = await crypto.subtle.digest("SHA-256", data);

  return [...new Uint8Array(digest)]
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

function cleanString(value: unknown, maxLen: number): string {
  if (typeof value !== "string") return "";
  return value.trim().slice(0, maxLen);
}

export async function handleContact(request: Request, env: Env): Promise<Response> {
  if (request.method !== "POST") {
    return methodNotAllowed();
  }

  const contentType = request.headers.get("content-type") || "";
  if (!contentType.includes("application/json")) {
    return fail(415, "unsupported_media_type", "Expected application/json");
  }

  let body: ContactBody;

  try {
    body = await request.json<ContactBody>();
  } catch {
    return fail(400, "bad_json", "Invalid JSON body");
  }

  const name = cleanString(body.name, 80);
  const email = cleanString(body.email, 160);
  const message = cleanString(body.message, 4000);
  const token = cleanString(
    body.turnstileToken || body["cf-turnstile-response"],
    4096
  );

  if (!name) {
    return fail(400, "missing_name", "name is required");
  }

  if (!message) {
    return fail(400, "missing_message", "message is required");
  }

  const ip = request.headers.get("CF-Connecting-IP");
  const turnstile = await verifyTurnstile(token, ip, env);

  if (!turnstile.success) {
    return fail(
      403,
      "turnstile_failed",
      `Turnstile validation failed: ${turnstile.errors.join(",")}`
    );
  }

  const userAgent = cleanString(request.headers.get("user-agent"), 512);
  const ipHash = ip ? await sha256Hex(`${env.IP_HASH_SALT}:${ip}`) : null;

  const result = await env.DB.prepare(
    `
    INSERT INTO contact_messages(name, email, message, ip_hash, user_agent)
    VALUES (?, ?, ?, ?, ?)
    RETURNING id, created_at
    `
  )
    .bind(name, email || null, message, ipHash, userAgent || null)
    .first<{ id: number; created_at: string }>();

  return ok({
    id: result?.id,
    created_at: result?.created_at
  });
}
EOF
```

---

### 10.6 `apps/api/src/index.ts`

```bash
cat > apps/api/src/index.ts <<'EOF'
import { handleOptions, withCors } from "./lib/cors";
import { notFound } from "./lib/response";
import { handleArticles } from "./routes/articles";
import { handleConfig } from "./routes/config";
import { handleContact } from "./routes/contact";
import { handleFiles } from "./routes/files";
import { handleHealth } from "./routes/health";
import type { Env } from "./types";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === "OPTIONS") {
      return handleOptions(request, env);
    }

    const url = new URL(request.url);
    const path = url.pathname;

    let response: Response;

    if (path === "/api/health") {
      response = await handleHealth(request, env);
    } else if (path === "/api/articles") {
      response = await handleArticles(request, env);
    } else if (path === "/api/config") {
      response = await handleConfig(request, env);
    } else if (path.startsWith("/api/files/")) {
      response = await handleFiles(request, env);
    } else if (path === "/api/contact") {
      response = await handleContact(request, env);
    } else {
      response = notFound();
    }

    return withCors(response, request, env);
  }
};
EOF
```

---

## 11. 安装依赖与构建

```bash
cd ~/projects/cloudflare-fullstack-starter

npm install
npm run typecheck
npm --workspace @cf-starter/web run build
```

如果出现：

```text
vite: not found
```

执行：

```bash
rm -rf node_modules package-lock.json
rm -rf apps/web/node_modules apps/api/node_modules

npm install
```

如果出现 React 类型错误：

```text
Could not find a declaration file for module 'react'
Property 'env' does not exist on type 'ImportMeta'
```

确认：

```bash
npm install -w @cf-starter/web -D @types/react @types/react-dom

cat > apps/web/src/vite-env.d.ts <<'EOF'
/// <reference types="vite/client" />
EOF
```

---

## 12. 创建 Cloudflare 资源

进入 API 目录：

```bash
cd ~/projects/cloudflare-fullstack-starter/apps/api
```

---

### 12.1 创建 D1

```bash
npx wrangler d1 create cf_fullstack_db_preview
npx wrangler d1 create cf_fullstack_db
```

把输出的 ID 填入：

```text
<PREVIEW_D1_DATABASE_ID>
<PRODUCTION_D1_DATABASE_ID>
```

执行 migration：

```bash
npx wrangler d1 migrations apply cf_fullstack_db_preview --remote --env preview
npx wrangler d1 migrations apply cf_fullstack_db --remote --env production
```

插入测试文章：

```bash
npx wrangler d1 execute cf_fullstack_db_preview \
  --remote \
  --env preview \
  --command "INSERT OR IGNORE INTO articles(slug,title,content) VALUES('hello','Hello Cloudflare Preview','This article is from D1 preview.');"

npx wrangler d1 execute cf_fullstack_db \
  --remote \
  --env production \
  --command "INSERT OR IGNORE INTO articles(slug,title,content) VALUES('hello','Hello Cloudflare Production','This article is from D1 production.');"
```

查询：

```bash
npx wrangler d1 execute cf_fullstack_db \
  --remote \
  --env production \
  --command "SELECT id, slug, title, content, created_at FROM articles;"
```

踩坑：

```text
UNIQUE constraint failed: articles.slug
```

不是数据库坏了，而是 `slug` 唯一约束导致重复插入。用 `INSERT OR IGNORE` 避免。

---

### 12.2 创建 KV

```bash
npx wrangler kv namespace create cf_fullstack_config_preview
npx wrangler kv namespace create cf_fullstack_config
```

把输出的 ID 填入：

```text
<PREVIEW_KV_NAMESPACE_ID>
<PRODUCTION_KV_NAMESPACE_ID>
```

写入配置：

```bash
npx wrangler kv key put site:config \
'{"siteName":"Cloudflare Fullstack Starter Preview","description":"Preview site using Pages + Workers + D1 + KV + R2","maintenance":false}' \
--binding APP_CONFIG \
--env preview \
--remote

npx wrangler kv key put site:config \
'{"siteName":"Cloudflare Fullstack Starter","description":"Production site using Pages + Workers + D1 + KV + R2","maintenance":false}' \
--binding APP_CONFIG \
--env production \
--remote
```

---

### 12.3 创建 R2

```bash
npx wrangler r2 bucket create cf-fullstack-assets-preview
npx wrangler r2 bucket create cf-fullstack-assets
```

创建待上传文件：

```bash
cd ~/projects/cloudflare-fullstack-starter/apps/api

mkdir -p ./tmp-r2
printf "hello from r2\n" > ./tmp-r2/hello.txt

ls -l ./tmp-r2/hello.txt
cat ./tmp-r2/hello.txt
```

上传远端 R2：

```bash
npx wrangler r2 object put cf-fullstack-assets-preview/demo/hello.txt \
  --file ./tmp-r2/hello.txt \
  --content-type text/plain \
  --remote

npx wrangler r2 object put cf-fullstack-assets/demo/hello.txt \
  --file ./tmp-r2/hello.txt \
  --content-type text/plain \
  --remote
```

踩坑：

```text
Resource location: local
Use --remote if you want to access the remote instance.
```

远端 R2 上传必须加：

```bash
--remote
```

---

## 13. Turnstile 与 Worker Secrets

### 13.1 创建 Turnstile Widget

Cloudflare Dashboard：

```text
Turnstile
-> Add widget
```

填写：

```text
Widget name:
cloudflare-fullstack-starter

Hostnames:
example.com
preview.example.com
localhost

Mode:
Managed
```

获得：

```text
Site key: <TURNSTILE_SITE_KEY>
Secret key: <TURNSTILE_SECRET_KEY>
```

---

### 13.2 前端环境变量

开发环境：

```bash
cat > apps/web/.env.development.local <<'EOF'
VITE_API_BASE=http://localhost:8787
VITE_TURNSTILE_SITE_KEY=<TURNSTILE_SITE_KEY>
EOF
```

生产环境：

```bash
cat > apps/web/.env.production <<'EOF'
VITE_API_BASE=
VITE_TURNSTILE_SITE_KEY=<TURNSTILE_SITE_KEY>
EOF
```

不要让生产包里出现 `localhost:8787`：

```bash
npm --workspace @cf-starter/web run build
grep -R "localhost:8787" apps/web/dist/assets/*.js || echo "OK: no localhost"
```

如果生产包里仍然有 `localhost:8787`，通常是 `.env.local` 被生产构建读取了：

```bash
mv apps/web/.env.local apps/web/.env.development.local 2>/dev/null || true
```

---

### 13.3 Worker secrets

```bash
cd ~/projects/cloudflare-fullstack-starter/apps/api

npx wrangler secret put TURNSTILE_SECRET_KEY --env preview
npx wrangler secret put IP_HASH_SALT --env preview

npx wrangler secret put TURNSTILE_SECRET_KEY --env production
npx wrangler secret put IP_HASH_SALT --env production
```

生成 salt：

```bash
openssl rand -hex 32
```

本地 `.dev.vars`：

```bash
cat > .dev.vars <<'EOF'
TURNSTILE_SECRET_KEY=<TURNSTILE_SECRET_KEY>
IP_HASH_SALT=local-dev-salt-change-me
EOF
```

`.gitignore`：

```bash
cat > .gitignore <<'EOF'
node_modules/
dist/
.dev.vars
.dev.vars*
.env
.env*
.wrangler/
.DS_Store
EOF
```

---

## 14. Pages 部署与 Custom Domain

### 14.1 创建 Pages 项目

如果出现权限问题，先清掉本地 token：

```bash
unset CLOUDFLARE_API_TOKEN
unset CLOUDFLARE_ACCOUNT_ID

npx wrangler logout
npx wrangler login
```

创建 Pages 项目：

```bash
cd ~/projects/cloudflare-fullstack-starter

npx wrangler pages project create cloudflare-fullstack-starter --production-branch main
```

---

### 14.2 部署 Pages

```bash
npm --workspace @cf-starter/web run build

npx wrangler pages deploy apps/web/dist \
  --project-name cloudflare-fullstack-starter \
  --branch main
```

---

### 14.3 配置 Custom Domain

Cloudflare Dashboard：

```text
Workers & Pages
-> cloudflare-fullstack-starter
-> Custom domains
-> Set up a domain
-> example.com
```

等待状态：

```text
Active
```

非常重要：不要只在 DNS 里手动加根域名记录，必须从 Pages 项目的 Custom domains 添加。

---

## 15. Worker 部署与 Route

部署：

```bash
cd ~/projects/cloudflare-fullstack-starter/apps/api

npx wrangler deploy --env preview
npx wrangler deploy --env production
```

如果出现：

```text
Can't deploy routes that are assigned to another worker.
"old-worker" is already assigned to routes:
  - example.com/api/*
```

处理：

```text
Workers & Pages
-> old-worker
-> Settings
-> Domains & Routes
-> 删除 example.com/api/*
```

然后重新部署：

```bash
npx wrangler deploy --env production
```

Worker route 必须接管：

```text
example.com/api/*
```

---

## 16. Cloudflare Access 保护 `/admin/*`

Cloudflare Dashboard：

```text
Zero Trust
-> Access controls
-> Applications
-> Create new application
-> Self-hosted and private
```

建议建两条路径：

```text
Domain: example.com
Path: admin
```

```text
Domain: example.com
Path: admin/*
```

Policy：

```text
Action: Allow
Include: Emails -> user@example.com
```

验证：

```bash
curl -I https://example.com/admin/
curl -I https://example.com/admin/index.html
```

成功结果：

```text
HTTP/2 302
location: https://<team>.cloudflareaccess.com/...
www-authenticate: Cloudflare-Access ...
```

这里 `302` 是正常的，表示未登录用户被 Access 拦截。

---

## 17. Rate Limit / WAF

免费版不要使用过于复杂的表达式，例如：

```text
http.request.method eq "POST"
```

这可能触发 add-on 或升级提示。

建议 Free Plan 使用：

```text
http.request.uri.path eq "/api/contact"
```

配置：

```text
Security
-> WAF
-> Rate limiting rules
-> Create rule
```

推荐参数：

```text
Expression:
http.request.uri.path eq "/api/contact"

Characteristics:
IP

Requests:
3 或 5

Period:
10 seconds

Action:
Managed Challenge 或 Block
```

如果 UI 仍提示购买 add-on，先不买，用 Turnstile + Worker 校验兜底。

---

## 18. Cloudflare Tunnel

### 18.1 本地管理方式

```bash
cloudflared tunnel login
cloudflared tunnel create cf-fullstack-dev-dashboard
cloudflared tunnel route dns cf-fullstack-dev-dashboard dev.example.com
cloudflared tunnel run cf-fullstack-dev-dashboard
```

配置文件：

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

示例：

```yaml
tunnel: <TUNNEL_UUID>
credentials-file: /home/<WSL_USER>/.cloudflared/<TUNNEL_UUID>.json

ingress:
  - hostname: dev.example.com
    service: http://localhost:3000
  - service: http_status:404
```

运行：

```bash
cloudflared tunnel --edge-ip-version 4 --protocol http2 run cf-fullstack-dev-dashboard
```

---

### 18.2 WSL IPv6 坑

如果出现：

```text
dial tcp [2606:4700:...]:443: connect: network is unreachable
```

说明 WSL 没有 IPv6 出口，但 `cloudflared` 选择了 IPv6。

推荐绕过方式：Dashboard 创建 Tunnel，然后 WSL 只运行 token。

Dashboard：

```text
Zero Trust
-> Networks
-> Tunnels
-> Create tunnel
-> Public Hostname:
   dev.example.com -> http://localhost:3000
```

WSL 运行：

```bash
cloudflared tunnel --edge-ip-version 4 --protocol http2 run --token '<TUNNEL_TOKEN>'
```

---

## 19. scripts

### 19.1 `scripts/deploy-worker.sh`

```bash
cat > scripts/deploy-worker.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ENV_NAME="${1:-preview}"

ROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT_DIR/apps/api"

npx wrangler deploy --env "$ENV_NAME"
EOF

chmod +x scripts/deploy-worker.sh
```

---

### 19.2 `scripts/deploy-pages.sh`

```bash
cat > scripts/deploy-pages.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

PROJECT_NAME="${PAGES_PROJECT_NAME:-cloudflare-fullstack-starter}"
BRANCH_NAME="${1:-main}"

ROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT_DIR"

npm --workspace @cf-starter/web run build

npx wrangler pages deploy apps/web/dist \
  --project-name "$PROJECT_NAME" \
  --branch "$BRANCH_NAME"
EOF

chmod +x scripts/deploy-pages.sh
```

---

### 19.3 `scripts/smoke-test.sh`

```bash
cat > scripts/smoke-test.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${1:-http://127.0.0.1:8787}"

echo "[1] health"
curl -fsS "$BASE_URL/api/health"
echo

echo "[2] articles"
curl -fsS "$BASE_URL/api/articles"
echo

echo "[3] config"
curl -fsS "$BASE_URL/api/config"
echo

echo "[4] r2 file"
curl -fsS "$BASE_URL/api/files/demo/hello.txt"
echo
EOF

chmod +x scripts/smoke-test.sh
```

---

### 19.4 `scripts/dev-api.sh`

```bash
cat > scripts/dev-api.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT_DIR/apps/api"

npx wrangler dev --ip 0.0.0.0 --port 8787
EOF

chmod +x scripts/dev-api.sh
```

---

## 20. 本地运行

Terminal 1：

```bash
cd ~/projects/cloudflare-fullstack-starter
bash scripts/dev-api.sh
```

Terminal 2：

```bash
cd ~/projects/cloudflare-fullstack-starter
bash scripts/smoke-test.sh http://127.0.0.1:8787
```

前端：

```bash
cd ~/projects/cloudflare-fullstack-starter
npm --workspace @cf-starter/web run dev
```

浏览器访问：

```text
http://localhost:5173
```

---

## 21. GitHub Actions 自动部署

创建 `.github/workflows/deploy.yml`：

```bash
cat > .github/workflows/deploy.yml <<'EOF'
name: Deploy Cloudflare Fullstack Starter

on:
  push:
    branches:
      - main
      - dev

jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      deployments: write

    env:
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      PAGES_PROJECT_NAME: cloudflare-fullstack-starter

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Install dependencies
        run: npm ci

      - name: Typecheck
        run: npm run typecheck

      - name: Build web
        run: npm --workspace @cf-starter/web run build

      - name: Select environment
        id: env
        shell: bash
        run: |
          if [ "${GITHUB_REF_NAME}" = "main" ]; then
            echo "cf_env=production" >> "$GITHUB_OUTPUT"
            echo "pages_branch=main" >> "$GITHUB_OUTPUT"
            echo "d1_db=cf_fullstack_db" >> "$GITHUB_OUTPUT"
          else
            echo "cf_env=preview" >> "$GITHUB_OUTPUT"
            echo "pages_branch=dev" >> "$GITHUB_OUTPUT"
            echo "d1_db=cf_fullstack_db_preview" >> "$GITHUB_OUTPUT"
          fi

      - name: Apply D1 migrations
        working-directory: apps/api
        run: |
          npx wrangler d1 migrations apply ${{ steps.env.outputs.d1_db }} \
            --remote \
            --env ${{ steps.env.outputs.cf_env }}

      - name: Deploy Worker API
        working-directory: apps/api
        run: |
          npx wrangler deploy --env ${{ steps.env.outputs.cf_env }}

      - name: Deploy Pages
        run: |
          npx wrangler pages deploy apps/web/dist \
            --project-name $PAGES_PROJECT_NAME \
            --branch ${{ steps.env.outputs.pages_branch }}
EOF
```

GitHub Secrets：

```text
CLOUDFLARE_ACCOUNT_ID=<CLOUDFLARE_ACCOUNT_ID>
CLOUDFLARE_API_TOKEN=<CLOUDFLARE_API_TOKEN>
```

Token 不要写：

```text
Bearer xxxxx
```

只填纯 token 字符串。

---

## 22. 常见踩坑点

### 22.1 `SyntaxError: Unexpected token '<', "<!doctype "... is not valid JSON`

含义：

```text
前端请求 /api/health，但返回了 Pages 的 index.html
```

原因：

```text
/api/* 没有命中 Worker，而是落到 Pages fallback
```

检查：

```bash
curl -i https://example.com/api/health
```

如果返回 HTML，处理：

```text
1. 删除旧 Worker 占用的 example.com/api/*
2. 新 Worker 绑定 example.com/api/*
3. npx wrangler deploy --env production
```

---

### 22.2 `TypeError: Failed to fetch`

常见原因：

```text
生产包里残留 http://localhost:8787
```

检查：

```bash
grep -R "localhost:8787" apps/web/dist/assets/*.js
```

修复：

```bash
mv apps/web/.env.local apps/web/.env.development.local 2>/dev/null || true

cat > apps/web/.env.production <<'EOF'
VITE_API_BASE=
VITE_TURNSTILE_SITE_KEY=<TURNSTILE_SITE_KEY>
EOF

npm --workspace @cf-starter/web run build
```

---

### 22.3 `HTTP/2 522`

含义：

```text
Cloudflare 到 origin 超时
```

在本项目中通常是：

```text
example.com 没正确绑定到 Pages
DNS 中 @ 还指向旧服务器或 dummy IP
```

处理：

```text
1. Cloudflare DNS 删除 @ 的旧 A / AAAA / 错误 CNAME
2. Pages Custom domains 添加 example.com
3. 等待 Active
4. curl -I https://example.com 必须返回 200
```

---

### 22.4 `Authentication error [code: 10000]`

如果日志显示：

```text
The API Token is read from the CLOUDFLARE_API_TOKEN environment variable
```

说明权限不足的 API token 覆盖了浏览器登录态。

处理：

```bash
unset CLOUDFLARE_API_TOKEN
unset CLOUDFLARE_ACCOUNT_ID

npx wrangler logout
npx wrangler login
```

---

### 22.5 `wrangler: command not found`

脚本中不要直接调用：

```bash
wrangler
```

应该使用：

```bash
npx wrangler
```

---

### 22.6 `localhost:8787 Couldn't connect`

原因：

```text
Worker 本地服务没有启动
```

先运行：

```bash
bash scripts/dev-api.sh
```

再运行：

```bash
bash scripts/smoke-test.sh http://127.0.0.1:8787
```

---

### 22.7 `Missing entry-point to Worker script`

原因：

```text
在项目根目录直接执行 npx wrangler dev，Wrangler 找不到 apps/api/wrangler.jsonc
```

正确：

```bash
cd apps/api
npx wrangler dev --ip 0.0.0.0 --port 8787
```

或者：

```bash
bash scripts/dev-api.sh
```

---

### 22.8 R2 上传提示 local

```text
Resource location: local
Use --remote if you want to access the remote instance.
```

远端上传要加：

```bash
--remote
```

---

### 22.9 Access 成功返回 302

`/admin/*` 未登录时返回：

```text
HTTP/2 302
location: https://<team>.cloudflareaccess.com/...
```

这是成功，不是错误。

---

### 22.10 WSL IPv6 导致 cloudflared 失败

错误：

```text
dial tcp [2606:4700:...]:443: connect: network is unreachable
```

说明 WSL 无 IPv6 出口。推荐用 Dashboard 创建 Tunnel，然后 WSL 运行 token：

```bash
cloudflared tunnel --edge-ip-version 4 --protocol http2 run --token '<TUNNEL_TOKEN>'
```

---

## 23. 最终验收命令

```bash
curl -I https://example.com

curl -i https://example.com/api/health

curl -i https://example.com/api/articles

curl -i https://example.com/api/config

curl -i https://example.com/api/files/demo/hello.txt

curl -I https://example.com/admin/
```

成功标准：

| 项目 | 成功结果 |
|---|---|
| `https://example.com` | `HTTP/2 200` |
| `/api/health` | JSON，`ok:true`，`env:production` |
| `/api/articles` | 返回 D1 文章 |
| `/api/config` | 返回 KV 配置 |
| `/api/files/demo/hello.txt` | 返回 `hello from r2` |
| `/admin/` | `302` 到 `cloudflareaccess.com` |

---

## 24. 项目最终判定标准

项目算完成时应满足：

```text
1. Pages 首页可访问
2. Worker /api/health 返回 JSON
3. /api/articles 从 D1 读取
4. /api/config 从 KV 读取
5. /api/files/:key 从 R2 读取
6. /api/contact 不带 Turnstile token 会被拒绝
7. 前端 contact 表单通过 Turnstile 后能写入 D1
8. /admin/* 未登录时跳转 Cloudflare Access
9. Access 登录后能看到 Admin Dashboard
10. /api/contact 配置 Rate Limit 或至少有 Turnstile 兜底
11. dev.example.com 能通过 Tunnel 访问本地 dashboard
12. GitHub Actions 可自动部署 Pages 和 Worker
```

核心主链路：

```bash
curl -i https://example.com/api/health
```

必须返回：

```json
{
  "ok": true,
  "data": {
    "service": "cf-fullstack-api",
    "env": "production",
    "ts": "..."
  }
}
```

如果返回 HTML、522、500 或旧 Worker 内容，说明 Pages / Worker route / DNS 仍未完全正确。

---

## 25. 重点难点总结

| 难点 | 说明 |
|---|---|
| Worker binding | `binding` 必须和代码里的 `env.DB`、`env.APP_CONFIG`、`env.ASSETS_BUCKET` 一致 |
| Wrangler env | D1/KV/R2 不会从 top-level 自动继承到 `env.production` |
| Pages custom domain | 必须从 Pages 项目的 Custom domains 添加，不能只手动加 DNS |
| Worker route | `/api/*` 必须由 Worker 接管，否则前端会拿到 HTML |
| Vite env | `.env.local` 会污染生产构建，导致生产包残留 `localhost:8787` |
| Access | `/admin/*` 未登录返回 302 是成功 |
| R2 | 远端上传必须加 `--remote` |
| Free WAF | 免费版 Rate Limit 不建议用 `http.request.method`，容易触发 add-on |
| WSL IPv6 | cloudflared 可能选 IPv6 导致 `network is unreachable` |
| API Token | `CLOUDFLARE_API_TOKEN` 权限不足会覆盖 `wrangler login` |

---

## 26. 最小成功路径

如果只验证主链路，至少通过：

```bash
curl -I https://example.com
curl -i https://example.com/api/health
curl -i https://example.com/api/articles
curl -i https://example.com/api/config
curl -i https://example.com/api/files/demo/hello.txt
curl -I https://example.com/admin/
```

预期：

```text
首页 200
health JSON
articles JSON
config JSON
R2 文件内容
admin 302 到 Access
```
