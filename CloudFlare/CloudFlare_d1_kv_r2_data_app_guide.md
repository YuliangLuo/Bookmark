# 6. Project 3：D1 + KV + R2 数据型应用完整操作指南

> 目标：通过一个文章 / 留言 / 配置 / 文件上传系统，完整理解 Cloudflare 的边缘应用模型：`Worker + D1 + KV + R2 + Secret + Wrangler`。

---

## 6.0 项目总览

本项目是 Cloudflare 学习路径中最关键的后端数据型项目。

完成后应理解：

```text
Cloudflare Worker = 边缘计算入口 / API 层
D1                = SQL 数据库 / 结构化数据
KV                = 配置 / Feature Flag / 低频缓存
R2                = 文件 / 图片 / 日志 / 对象存储
Secret            = 敏感配置 / 管理接口鉴权
Wrangler          = 本地开发 / 资源创建 / 部署工具
```

最终架构：

```text
Client
  ↓ HTTP
Cloudflare Edge
  ↓
Worker
  ├── D1：articles / messages
  ├── KV：site config / feature flag / cache
  ├── R2：files / images / logs
  └── Secret：ADMIN_TOKEN / IP_HASH_SALT
```

最终 API：

| API | Method | Storage | 说明 |
|---|---:|---|---|
| `/api/health` | GET | - | 健康检查 |
| `/api/articles` | GET | D1 | 查询文章列表 |
| `/api/articles` | POST | D1 | 新增文章，需要 Admin Token |
| `/api/articles/:slug` | GET | D1 | 查询单篇文章 |
| `/api/messages` | POST | D1 | 提交留言 |
| `/api/config` | GET | KV | 查询站点配置 |
| `/api/config/:key` | PUT | KV | 更新配置，需要 Admin Token |
| `/api/files/:key` | PUT | R2 | 上传文件，需要 Admin Token |
| `/api/files/:key` | GET | R2 | 下载文件 |
| `/api/files/:key` | DELETE | R2 | 删除文件，需要 Admin Token |

---

## 6.0.1 前置条件

需要安装：

```powershell
node -v
npm -v
npx wrangler --version
```

建议版本：

```text
Node.js >= 20
Wrangler >= 4.x
```

登录 Cloudflare：

```powershell
npx wrangler login
```

确认登录账号：

```powershell
npx wrangler whoami
```

---

## 6.0.2 项目目录

如果已经创建过 Worker 项目，直接进入：

```powershell
cd cf-lab-data-app
```

如果还没有创建：

```powershell
npm create cloudflare@latest -- cf-lab-data-app
cd cf-lab-data-app
```

建议选择：

```text
Worker only
TypeScript
Deploy now? No
```

目标目录结构：

```text
cf-lab-data-app/
├── package.json
├── wrangler.jsonc
├── .dev.vars
├── src/
│   └── index.ts
└── migrations/
    └── 0001_init_schema.sql
```

---

# 6.1 D1：SQL 数据库

## 6.1.1 目标

做一个简单的文章 / 留言系统。

D1 负责存储结构化数据：

```text
articles  → 文章
messages  → 留言
```

适合放 D1 的数据：

```text
- 文章
- 留言
- 用户信息
- 测试记录
- 文件元数据
- 设备信息
- 业务状态
```

不适合放 D1 的数据：

```text
- 大型图片
- PDF 文件
- 大型 log
- zip 包
- 高频时序采样数据
```

---

## 6.1.2 创建 D1 数据库

在项目根目录执行：

```powershell
npx wrangler d1 create cf_lab_db
```

成功后会输出类似：

```jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "cf_lab_db",
      "database_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  ]
}
```

需要记录：

```text
binding       = DB
database_name = cf_lab_db
database_id   = 输出的 UUID
```

后面要写入 `wrangler.jsonc`。

---

## 6.1.3 创建 D1 migration

执行：

```powershell
npx wrangler d1 migrations create cf_lab_db init_schema
```

会生成类似文件：

```text
migrations/0001_init_schema.sql
```

将下面 SQL 写入该文件：

```sql
CREATE TABLE IF NOT EXISTS articles (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  slug TEXT NOT NULL UNIQUE,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_articles_created_at
ON articles(created_at);

CREATE TABLE IF NOT EXISTS messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT,
  message TEXT NOT NULL,
  ip_hash TEXT,
  created_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_messages_created_at
ON messages(created_at);
```

---

## 6.1.4 应用 D1 migration：本地

```powershell
npx wrangler d1 migrations apply cf_lab_db --local
```

检查本地表：

```powershell
npx wrangler d1 execute cf_lab_db --local --command="SELECT name FROM sqlite_master WHERE type='table';"
```

预期能看到：

```text
articles
messages
d1_migrations
```

---

## 6.1.5 应用 D1 migration：远程

```powershell
npx wrangler d1 migrations apply cf_lab_db --remote
```

检查远程表：

```powershell
npx wrangler d1 execute cf_lab_db --remote --command="SELECT name FROM sqlite_master WHERE type='table';"
```

注意：

```text
--local  操作本地开发数据库
--remote 操作 Cloudflare 远程数据库
```

这两套数据互不相同。

---

## 6.1.6 Worker 访问 D1 的基本写法

```ts
interface Env {
  DB: D1Database;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/api/articles") {
      const result = await env.DB
        .prepare("SELECT id, slug, title, created_at FROM articles ORDER BY id DESC")
        .all();

      return Response.json(result.results);
    }

    return new Response("Not Found", { status: 404 });
  },
};
```

关键点：

```text
D1 通过 env.DB 访问
DB 是 wrangler.jsonc 里的 binding 名称
SQL 应使用 prepare + bind，避免 SQL 注入
```

---

# 6.2 KV：配置与缓存

## 6.2.1 目标

用 KV 保存：

```text
site:title
site:maintenance_mode
feature:enable_feedback
cache:home_page
```

KV 适合：

```text
- feature flag
- 站点配置
- 低频更新缓存
- 小型 key-value 数据
```

KV 不适合：

```text
- 强一致事务数据
- 高频写入日志
- 大型关系查询
- 库存扣减
- 订单状态
- 用户余额
```

---

## 6.2.2 创建 KV namespace

```powershell
npx wrangler kv namespace create APP_CONFIG
```

成功后输出类似：

```jsonc
{
  "kv_namespaces": [
    {
      "binding": "APP_CONFIG",
      "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  ]
}
```

需要记录：

```text
binding = APP_CONFIG
id      = KV namespace ID
```

---

## 6.2.3 初始化本地 KV

```powershell
npx wrangler kv key put site:title "CF Lab" --binding=APP_CONFIG --local
npx wrangler kv key put site:maintenance_mode "false" --binding=APP_CONFIG --local
npx wrangler kv key put feature:enable_feedback "true" --binding=APP_CONFIG --local
npx wrangler kv key put cache:home_page "Welcome to CF Lab" --binding=APP_CONFIG --local
```

---

## 6.2.4 初始化远程 KV

```powershell
npx wrangler kv key put site:title "CF Lab" --binding=APP_CONFIG --remote
npx wrangler kv key put site:maintenance_mode "false" --binding=APP_CONFIG --remote
npx wrangler kv key put feature:enable_feedback "true" --binding=APP_CONFIG --remote
npx wrangler kv key put cache:home_page "Welcome to CF Lab" --binding=APP_CONFIG --remote
```

---

## 6.2.5 Worker 访问 KV 的基本写法

```ts
interface Env {
  APP_CONFIG: KVNamespace;
}

async function getConfig(env: Env) {
  const maintenance = await env.APP_CONFIG.get("site:maintenance_mode");

  return {
    maintenance_mode: maintenance === "true",
  };
}
```

关键点：

```text
KV 通过 env.APP_CONFIG 访问
APP_CONFIG 是 wrangler.jsonc 里的 binding 名称
KV 适合读多写少
KV 不适合强一致业务状态
```

---

# 6.3 R2：对象存储

## 6.3.1 目标

实现图片 / 文件上传和下载。

R2 适合：

```text
- 图片
- PDF
- CSV
- log 文件
- zip 包
- 测试报告
- 示波器波形
- 固件包
- 大型 JSON dump
```

R2 不适合：

```text
- 复杂条件查询
- 表关系查询
- 事务数据
- 频繁修改对象局部内容
```

---

## 6.3.2 创建 R2 bucket

```powershell
npx wrangler r2 bucket create cf-lab-assets
```

查看 bucket：

```powershell
npx wrangler r2 bucket list
```

如果报错：

```text
Please enable R2 through the Cloudflare Dashboard. [code: 10042]
```

说明账号还未开通 R2。

处理路径：

```text
Cloudflare Dashboard
→ Storage & databases
→ R2
→ Overview
→ Enable R2 / Add R2 subscription
```

完成后重新执行：

```powershell
npx wrangler r2 bucket create cf-lab-assets
```

---

## 6.3.3 R2 binding 配置示例

```jsonc
{
  "r2_buckets": [
    {
      "binding": "ASSETS_BUCKET",
      "bucket_name": "cf-lab-assets"
    }
  ]
}
```

---

## 6.3.4 Worker 上传 R2 示例

```ts
interface Env {
  ASSETS_BUCKET: R2Bucket;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (request.method === "PUT" && url.pathname.startsWith("/api/files/")) {
      const key = url.pathname.replace("/api/files/", "");

      await env.ASSETS_BUCKET.put(key, request.body, {
        httpMetadata: {
          contentType: request.headers.get("content-type") ?? "application/octet-stream",
        },
      });

      return Response.json({ ok: true, key });
    }

    return Response.json({ error: "Not Found" }, { status: 404 });
  },
};
```

关键点：

```text
R2 通过 env.ASSETS_BUCKET 访问
ASSETS_BUCKET 是 wrangler.jsonc 里的 binding 名称
R2 object key 看起来像路径，但本质只是字符串
```

---

# 6.4 Wrangler 配置

## 6.4.1 完整 `wrangler.jsonc`

修改项目根目录的 `wrangler.jsonc`：

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "cf-lab-data-app",
  "main": "src/index.ts",
  "compatibility_date": "2026-05-18",

  "workers_dev": true,
  "preview_urls": true,

  "observability": {
    "enabled": true
  },

  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "cf_lab_db",
      "database_id": "<替换为 d1 create 输出的 database_id>",
      "migrations_dir": "migrations"
    }
  ],

  "kv_namespaces": [
    {
      "binding": "APP_CONFIG",
      "id": "<替换为 kv namespace create 输出的 id>"
    }
  ],

  "r2_buckets": [
    {
      "binding": "ASSETS_BUCKET",
      "bucket_name": "cf-lab-assets"
    }
  ]
}
```

---

## 6.4.2 Route 配置注意事项

学习阶段建议先使用：

```text
https://<worker-name>.<subdomain>.workers.dev
```

不要一开始绑定自定义 route，避免与旧 Worker 冲突。

如果要使用自定义域名，可加：

```jsonc
"routes": [
  {
    "pattern": "dev.example.com/api/*",
    "zone_name": "example.com"
  }
]
```

如果部署时报错：

```text
A route with the same pattern already exists
```

说明该 route 已经被其他 Worker 占用。

解决方式：

```text
方案 A：删除当前 wrangler.jsonc 中的 routes，继续用 workers.dev
方案 B：到 Dashboard 删除旧 Worker 的 route
方案 C：换一个新的 route pattern
```

---

# 6.5 Secret 管理

## 6.5.1 本地 `.dev.vars`

创建 `.dev.vars`：

```powershell
New-Item -ItemType File .dev.vars
```

写入：

```env
ADMIN_TOKEN="dev-admin-token"
IP_HASH_SALT="dev-ip-hash-salt"
```

作用：

```text
ADMIN_TOKEN  → 管理接口鉴权
IP_HASH_SALT → IP hash 盐值，避免明文保存 IP
```

---

## 6.5.2 `.gitignore`

确认 `.gitignore` 包含：

```gitignore
.dev.vars*
.env*
.wrangler/
node_modules/
```

不要提交：

```text
.dev.vars
.env
ADMIN_TOKEN
IP_HASH_SALT
Cloudflare Account ID
D1 database_id
KV namespace id
```

---

## 6.5.3 设置远程 ADMIN_TOKEN

```powershell
npx wrangler secret put ADMIN_TOKEN
```

提示：

```text
Enter a secret value:
```

输入你的 token，例如：

```text
dev-admin-token
```

注意：

```text
这里输入的是 token 的值
不是输入 npx wrangler secret put IP_HASH_SALT
```

---

## 6.5.4 设置远程 IP_HASH_SALT

```powershell
npx wrangler secret put IP_HASH_SALT
```

提示后输入 salt，例如：

```text
cf-lab-ip-hash-salt
```

---

## 6.5.5 Secret 填错怎么办

重新执行同名命令即可覆盖：

```powershell
npx wrangler secret put ADMIN_TOKEN
```

或：

```powershell
npx wrangler secret put IP_HASH_SALT
```

不需要先删除。

---

# 6.6 Worker API 完整实现

将下面内容保存为：

```text
src/index.ts
```

```ts
interface Env {
  DB: D1Database;
  APP_CONFIG: KVNamespace;
  ASSETS_BUCKET: R2Bucket;

  ADMIN_TOKEN: string;
  IP_HASH_SALT?: string;
}

type ArticleListRow = {
  id: number;
  slug: string;
  title: string;
  created_at: string;
};

type ArticleRow = ArticleListRow & {
  content: string;
};

type ArticleInput = {
  slug?: string;
  title?: string;
  content?: string;
};

type MessageInput = {
  name?: string;
  email?: string;
  message?: string;
};

type ConfigInput = {
  value?: string | number | boolean | null;
};

const CONFIG_KEYS = [
  "site:title",
  "site:maintenance_mode",
  "feature:enable_feedback",
  "cache:home_page",
] as const;

const CONFIG_KEY_SET = new Set<string>(CONFIG_KEYS);

const CORS_HEADERS: Record<string, string> = {
  "access-control-allow-origin": "*",
  "access-control-allow-methods": "GET,POST,PUT,DELETE,OPTIONS",
  "access-control-allow-headers": "content-type,authorization",
};

class HttpError extends Error {
  constructor(
    public status: number,
    message: string,
  ) {
    super(message);
  }
}

function normalizePath(pathname: string): string {
  if (pathname.length <= 1) return pathname;
  return pathname.replace(/\/+$/, "");
}

function addCors(headers: Headers): Headers {
  for (const [k, v] of Object.entries(CORS_HEADERS)) {
    headers.set(k, v);
  }
  return headers;
}

function json(data: unknown, init: ResponseInit = {}): Response {
  const headers = new Headers(init.headers);
  headers.set("content-type", "application/json; charset=utf-8");
  addCors(headers);

  return new Response(JSON.stringify(data), {
    ...init,
    headers,
  });
}

async function readJson<T>(request: Request): Promise<T> {
  const contentType = request.headers.get("content-type") ?? "";

  if (!contentType.includes("application/json")) {
    throw new HttpError(415, "Content-Type must be application/json");
  }

  try {
    return (await request.json()) as T;
  } catch {
    throw new HttpError(400, "Invalid JSON body");
  }
}

function requireAdmin(request: Request, env: Env): void {
  if (!env.ADMIN_TOKEN) {
    throw new HttpError(500, "ADMIN_TOKEN is not configured");
  }

  const auth = request.headers.get("authorization") ?? "";

  if (auth !== `Bearer ${env.ADMIN_TOKEN}`) {
    throw new HttpError(401, "Unauthorized");
  }
}

function validateSlug(slug: string): boolean {
  return /^[a-z0-9][a-z0-9-]{1,80}$/.test(slug);
}

function validateR2Key(key: string): boolean {
  if (!key) return false;
  if (key.length > 512) return false;
  if (key.startsWith("/")) return false;
  if (key.includes("\0")) return false;
  if (key.includes("..")) return false;
  return true;
}

function getClientIp(request: Request): string | null {
  return (
    request.headers.get("cf-connecting-ip") ??
    request.headers.get("x-forwarded-for") ??
    null
  );
}

async function sha256Hex(input: string): Promise<string> {
  const bytes = new TextEncoder().encode(input);
  const digest = await crypto.subtle.digest("SHA-256", bytes);
  return Array.from(new Uint8Array(digest))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

async function getIpHash(request: Request, env: Env): Promise<string | null> {
  const ip = getClientIp(request);
  if (!ip || !env.IP_HASH_SALT) return null;
  return sha256Hex(`${env.IP_HASH_SALT}:${ip}`);
}

async function listArticles(env: Env): Promise<Response> {
  const result = await env.DB.prepare(
    `
    SELECT id, slug, title, created_at
    FROM articles
    ORDER BY id DESC
    LIMIT 100
    `,
  ).all<ArticleListRow>();

  return json({
    ok: true,
    articles: result.results ?? [],
  });
}

async function getArticle(path: string, env: Env): Promise<Response> {
  const slug = decodeURIComponent(path.slice("/api/articles/".length));

  if (!validateSlug(slug)) {
    throw new HttpError(400, "Invalid article slug");
  }

  const article = await env.DB.prepare(
    `
    SELECT id, slug, title, content, created_at
    FROM articles
    WHERE slug = ?
    `,
  )
    .bind(slug)
    .first<ArticleRow>();

  if (!article) {
    throw new HttpError(404, "Article not found");
  }

  return json({
    ok: true,
    article,
  });
}

async function createArticle(
  request: Request,
  env: Env,
): Promise<Response> {
  requireAdmin(request, env);

  const body = await readJson<ArticleInput>(request);

  const slug = body.slug?.trim().toLowerCase() ?? "";
  const title = body.title?.trim() ?? "";
  const content = body.content?.trim() ?? "";

  if (!validateSlug(slug)) {
    throw new HttpError(
      400,
      "Invalid slug. Use lowercase letters, numbers and hyphen only.",
    );
  }

  if (!title || title.length > 160) {
    throw new HttpError(400, "Title is required and must be <= 160 chars");
  }

  if (!content || content.length > 20000) {
    throw new HttpError(400, "Content is required and must be <= 20000 chars");
  }

  const now = new Date().toISOString();

  try {
    await env.DB.prepare(
      `
      INSERT INTO articles (slug, title, content, created_at)
      VALUES (?, ?, ?, ?)
      `,
    )
      .bind(slug, title, content, now)
      .run();
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err);
    if (msg.toLowerCase().includes("unique")) {
      throw new HttpError(409, "Article slug already exists");
    }
    throw err;
  }

  return json(
    {
      ok: true,
      article: {
        slug,
        title,
        created_at: now,
      },
    },
    { status: 201 },
  );
}

async function createMessage(
  request: Request,
  env: Env,
): Promise<Response> {
  const body = await readJson<MessageInput>(request);

  const name = body.name?.trim() ?? "";
  const email = body.email?.trim() || null;
  const message = body.message?.trim() ?? "";

  if (!name || name.length > 80) {
    throw new HttpError(400, "Name is required and must be <= 80 chars");
  }

  if (email && email.length > 160) {
    throw new HttpError(400, "Email must be <= 160 chars");
  }

  if (!message || message.length > 2000) {
    throw new HttpError(400, "Message is required and must be <= 2000 chars");
  }

  const now = new Date().toISOString();
  const ipHash = await getIpHash(request, env);

  await env.DB.prepare(
    `
    INSERT INTO messages (name, email, message, ip_hash, created_at)
    VALUES (?, ?, ?, ?, ?)
    `,
  )
    .bind(name, email, message, ipHash, now)
    .run();

  return json(
    {
      ok: true,
      created_at: now,
    },
    { status: 201 },
  );
}

async function getConfig(env: Env): Promise<Response> {
  const [
    siteTitle,
    maintenanceMode,
    enableFeedback,
    homePageCache,
  ] = await Promise.all([
    env.APP_CONFIG.get("site:title"),
    env.APP_CONFIG.get("site:maintenance_mode"),
    env.APP_CONFIG.get("feature:enable_feedback"),
    env.APP_CONFIG.get("cache:home_page"),
  ]);

  return json({
    ok: true,
    config: {
      site_title: siteTitle ?? "CF Lab",
      maintenance_mode: maintenanceMode === "true",
      enable_feedback: enableFeedback === "true",
      home_page_cache: homePageCache ?? null,
    },
  });
}

async function updateConfig(
  path: string,
  request: Request,
  env: Env,
): Promise<Response> {
  requireAdmin(request, env);

  const key = decodeURIComponent(path.slice("/api/config/".length));

  if (!CONFIG_KEY_SET.has(key)) {
    throw new HttpError(400, `Unsupported config key: ${key}`);
  }

  const body = await readJson<ConfigInput>(request);

  if (body.value === undefined) {
    throw new HttpError(400, "Missing field: value");
  }

  await env.APP_CONFIG.put(key, String(body.value));

  return json({
    ok: true,
    key,
    value: String(body.value),
  });
}

async function putFile(
  path: string,
  request: Request,
  env: Env,
): Promise<Response> {
  requireAdmin(request, env);

  const key = decodeURIComponent(path.slice("/api/files/".length));

  if (!validateR2Key(key)) {
    throw new HttpError(400, "Invalid file key");
  }

  if (!request.body) {
    throw new HttpError(400, "Empty request body");
  }

  const contentType =
    request.headers.get("content-type") ?? "application/octet-stream";

  await env.ASSETS_BUCKET.put(key, request.body, {
    httpMetadata: {
      contentType,
    },
    customMetadata: {
      uploadedAt: new Date().toISOString(),
    },
  });

  return json(
    {
      ok: true,
      key,
      content_type: contentType,
    },
    { status: 201 },
  );
}

async function getFile(path: string, env: Env): Promise<Response> {
  const key = decodeURIComponent(path.slice("/api/files/".length));

  if (!validateR2Key(key)) {
    throw new HttpError(400, "Invalid file key");
  }

  const object = await env.ASSETS_BUCKET.get(key);

  if (!object) {
    throw new HttpError(404, "File not found");
  }

  const headers = new Headers();
  object.writeHttpMetadata(headers);
  headers.set("etag", object.httpEtag);
  headers.set("cache-control", "public, max-age=3600");
  addCors(headers);

  return new Response(object.body, {
    headers,
  });
}

async function deleteFile(
  path: string,
  request: Request,
  env: Env,
): Promise<Response> {
  requireAdmin(request, env);

  const key = decodeURIComponent(path.slice("/api/files/".length));

  if (!validateR2Key(key)) {
    throw new HttpError(400, "Invalid file key");
  }

  await env.ASSETS_BUCKET.delete(key);

  return json({
    ok: true,
    deleted: key,
  });
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === "OPTIONS") {
      return new Response(null, {
        status: 204,
        headers: CORS_HEADERS,
      });
    }

    try {
      const url = new URL(request.url);
      const path = normalizePath(url.pathname);

      if (request.method === "GET" && path === "/api/health") {
        return json({
          ok: true,
          service: "cf-lab-data-app",
          time: new Date().toISOString(),
        });
      }

      if (request.method === "GET" && path === "/api/articles") {
        return listArticles(env);
      }

      if (request.method === "POST" && path === "/api/articles") {
        return createArticle(request, env);
      }

      if (request.method === "GET" && path.startsWith("/api/articles/")) {
        return getArticle(path, env);
      }

      if (request.method === "POST" && path === "/api/messages") {
        return createMessage(request, env);
      }

      if (request.method === "GET" && path === "/api/config") {
        return getConfig(env);
      }

      if (request.method === "PUT" && path.startsWith("/api/config/")) {
        return updateConfig(path, request, env);
      }

      if (request.method === "PUT" && path.startsWith("/api/files/")) {
        return putFile(path, request, env);
      }

      if (request.method === "GET" && path.startsWith("/api/files/")) {
        return getFile(path, env);
      }

      if (request.method === "DELETE" && path.startsWith("/api/files/")) {
        return deleteFile(path, request, env);
      }

      return json(
        {
          ok: false,
          error: "Not Found",
          path,
        },
        { status: 404 },
      );
    } catch (err) {
      if (err instanceof HttpError) {
        return json(
          {
            ok: false,
            error: err.message,
          },
          { status: err.status },
        );
      }

      console.error(err);

      return json(
        {
          ok: false,
          error: "Internal Server Error",
        },
        { status: 500 },
      );
    }
  },
} satisfies ExportedHandler<Env>;
```

---

# 6.7 生成 TypeScript 类型

执行：

```powershell
npx wrangler types
```

作用：

```text
根据 wrangler.jsonc 生成 Worker binding 类型
让 TypeScript 识别 D1Database / KVNamespace / R2Bucket
```

---

# 6.8 本地运行

启动 Worker：

```powershell
npx wrangler dev
```

默认本地地址：

```text
http://127.0.0.1:8787
```

PowerShell 中设置基础变量：

```powershell
$base = "http://127.0.0.1:8787"

$adminHeaders = @{
  Authorization = "Bearer dev-admin-token"
}
```

注意 URI 推荐写法：

```powershell
"$($base)/api/articles"
```

不要写成依赖变量拼接的模糊形式。

---

# 6.9 本地 API 测试

## 6.9.1 健康检查

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/health"
```

预期：

```json
{
  "ok": true,
  "service": "cf-lab-data-app",
  "time": "..."
}
```

---

## 6.9.2 查询配置

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/config" | ConvertTo-Json -Depth 10
```

---

## 6.9.3 更新配置

```powershell
$body = @{
  value = "QuantGuard Lab"
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Method Put `
  -Uri "$($base)/api/config/site:title" `
  -Headers $adminHeaders `
  -ContentType "application/json" `
  -Body $body
```

注意：

```text
PowerShell 中不要优先使用 curl.exe --data '{"value":"xxx"}'
容易因为引号转义导致 Invalid JSON body。
```

---

## 6.9.4 新增文章

```powershell
$body = @{
  slug = "hello-cloudflare"
  title = "Hello Cloudflare"
  content = "D1 + KV + R2 data app."
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Method Post `
  -Uri "$($base)/api/articles" `
  -Headers $adminHeaders `
  -ContentType "application/json" `
  -Body $body
```

---

## 6.9.5 查询文章列表

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/articles" | ConvertTo-Json -Depth 10
```

---

## 6.9.6 查询单篇文章

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/articles/hello-cloudflare" | ConvertTo-Json -Depth 10
```

---

## 6.9.7 提交留言

```powershell
$body = @{
  name = "Tester"
  email = "tester@example.com"
  message = "This is a test message."
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Method Post `
  -Uri "$($base)/api/messages" `
  -ContentType "application/json" `
  -Body $body
```

---

## 6.9.8 上传文件到 R2

创建测试文件：

```powershell
"hello r2" | Out-File -Encoding utf8 hello.txt
```

上传：

```powershell
Invoke-RestMethod `
  -Method Put `
  -Uri "$($base)/api/files/test/hello.txt" `
  -Headers $adminHeaders `
  -ContentType "text/plain" `
  -InFile ".\hello.txt"
```

---

## 6.9.9 下载 R2 文件

```powershell
Invoke-WebRequest `
  -Method Get `
  -Uri "$($base)/api/files/test/hello.txt" `
  -OutFile ".\downloaded-hello.txt"

Get-Content .\downloaded-hello.txt
```

预期：

```text
hello r2
```

---

## 6.9.10 删除 R2 文件

```powershell
Invoke-RestMethod `
  -Method Delete `
  -Uri "$($base)/api/files/test/hello.txt" `
  -Headers $adminHeaders
```

---

# 6.10 远程部署

## 6.10.1 部署前检查

确认已经完成：

```text
1. wrangler.jsonc 已配置 D1 / KV / R2 binding
2. D1 remote migration 已执行
3. KV remote 数据已初始化
4. ADMIN_TOKEN 已通过 wrangler secret put 设置
5. IP_HASH_SALT 已通过 wrangler secret put 设置
```

执行远程 migration：

```powershell
npx wrangler d1 migrations apply cf_lab_db --remote
```

设置远程 secret：

```powershell
npx wrangler secret put ADMIN_TOKEN
npx wrangler secret put IP_HASH_SALT
```

---

## 6.10.2 部署

```powershell
npx wrangler deploy
```

成功日志类似：

```text
Uploaded cf-lab-data-app
Deployed cf-lab-data-app triggers
https://cf-lab-data-app.<subdomain>.workers.dev
Current Version ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

没有 `✘ [ERROR]` 才算部署命令成功。

---

# 6.11 远程 API 测试

设置远程 base：

```powershell
$base = "https://cf-lab-data-app.<subdomain>.workers.dev"

$adminHeaders = @{
  Authorization = "Bearer dev-admin-token"
}
```

测试 health：

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/health"
```

测试 D1：

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/articles" | ConvertTo-Json -Depth 10
```

测试 KV：

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "$($base)/api/config" | ConvertTo-Json -Depth 10
```

测试写 D1：

```powershell
$body = @{
  slug = "remote-test"
  title = "Remote Test"
  content = "This article was created on deployed Worker."
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Method Post `
  -Uri "$($base)/api/articles" `
  -Headers $adminHeaders `
  -ContentType "application/json" `
  -Body $body
```

测试 R2：

```powershell
"hello remote r2" | Out-File -Encoding utf8 remote-hello.txt

Invoke-RestMethod `
  -Method Put `
  -Uri "$($base)/api/files/test/remote-hello.txt" `
  -Headers $adminHeaders `
  -ContentType "text/plain" `
  -InFile ".\remote-hello.txt"
```

下载 R2 文件：

```powershell
Invoke-WebRequest `
  -Method Get `
  -Uri "$($base)/api/files/test/remote-hello.txt" `
  -OutFile ".\downloaded-remote-hello.txt"

Get-Content .\downloaded-remote-hello.txt
```

---

# 6.12 怎样才算部署成功

部署成功分两层：

```text
1. Wrangler deploy 成功
2. 远程 Worker API 实际可用
```

Wrangler 成功标准：

```text
有 Uploaded
有 Deployed triggers
有 workers.dev URL
有 Current Version ID
没有 ✘ [ERROR]
```

应用成功标准：

| 检查项 | 成功标准 |
|---|---|
| `/api/health` | 返回 `ok: true` |
| `/api/articles` | 能返回文章列表 |
| `/api/config` | 能返回 KV 配置 |
| `POST /api/articles` | 带 Authorization 后能新增文章 |
| `PUT /api/files/...` | 能上传 R2 文件 |
| `GET /api/files/...` | 能下载 R2 文件 |
| `DELETE /api/files/...` | 能删除 R2 文件 |

完整判定：

```text
Worker + D1 + KV + R2 + Secret 全部可用，才算 Project 3 真正跑通。
```

---

# 6.13 常见问题排查

## 6.13.1 R2 创建失败

错误：

```text
Please enable R2 through the Cloudflare Dashboard. [code: 10042]
```

原因：

```text
Cloudflare 账号没有开通 R2。
```

修正：

```text
Dashboard → Storage & databases → R2 → Enable R2
```

---

## 6.13.2 Secret 填错

重新执行：

```powershell
npx wrangler secret put ADMIN_TOKEN
```

或：

```powershell
npx wrangler secret put IP_HASH_SALT
```

---

## 6.13.3 PowerShell JSON body 报错

错误：

```text
Invalid JSON body
```

原因：

```text
PowerShell + curl.exe 对 JSON 引号处理不稳定。
```

推荐改用：

```powershell
$body = @{
  value = "QuantGuard Lab"
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Method Put `
  -Uri "$($base)/api/config/site:title" `
  -Headers $adminHeaders `
  -ContentType "application/json" `
  -Body $body
```

---

## 6.13.4 URI hostname could not be parsed

错误：

```text
Invalid URI: The hostname could not be parsed.
```

原因：

```text
$base 没有定义，或者 URI 拼接错误。
```

修正：

```powershell
$base = "http://127.0.0.1:8787"
```

使用：

```powershell
-Uri "$($base)/api/articles"
```

---

## 6.13.5 Route 冲突

错误：

```text
A route with the same pattern already exists
```

原因：

```text
同一个 route pattern 已经被其他 Worker 占用。
```

修正：

```text
1. 删除当前项目 wrangler.jsonc 中的 routes，使用 workers.dev
2. 或到 Dashboard 删除旧 Worker 的 route
3. 或换一个新的 route pattern
```

---

## 6.13.6 D1 表不存在

错误：

```text
no such table: articles
```

本地修正：

```powershell
npx wrangler d1 migrations apply cf_lab_db --local
```

远程修正：

```powershell
npx wrangler d1 migrations apply cf_lab_db --remote
```

---

## 6.13.7 KV 远程为空

原因：

```text
本地 KV 和远程 KV 是两套数据。
```

远程初始化：

```powershell
npx wrangler kv key put site:title "CF Lab" --binding=APP_CONFIG --remote
npx wrangler kv key put site:maintenance_mode "false" --binding=APP_CONFIG --remote
npx wrangler kv key put feature:enable_feedback "true" --binding=APP_CONFIG --remote
npx wrangler kv key put cache:home_page "Welcome to CF Lab" --binding=APP_CONFIG --remote
```

---

## 6.13.8 401 Unauthorized

检查请求头：

```text
Authorization: Bearer dev-admin-token
```

本地 token 来自：

```text
.dev.vars
```

远程 token 来自：

```powershell
npx wrangler secret put ADMIN_TOKEN
```

---

# 6.14 项目知识点总结

## 6.14.1 Edge Runtime

你掌握：

```text
fetch(request, env)
Request
Response
URL
headers
body
stream
Web Crypto
```

能力判定：

```text
能独立写 Worker API
能根据 method + pathname 做路由分发
能返回 JSON / status code
能处理异常
```

---

## 6.14.2 D1

你掌握：

```text
D1 create
migration create
migration apply
SQL schema
prepared statement
bind
all / first / run
local / remote database
```

---

## 6.14.3 KV

你掌握：

```text
namespace create
binding
get
put
本地 KV
远程 KV
配置 / feature flag / cache 使用边界
```

---

## 6.14.4 R2

你掌握：

```text
bucket create
bucket list
binding
object put
object get
object delete
content-type
metadata
stream upload
```

---

## 6.14.5 Secret

你掌握：

```text
.dev.vars
wrangler secret put
ADMIN_TOKEN
IP_HASH_SALT
本地 secret 与远程 secret 分离
```

---

## 6.14.6 工程化

你掌握：

```text
wrangler.jsonc
wrangler dev
wrangler deploy
wrangler types
route 冲突
workers.dev
PowerShell API 调试
```

---

# 6.15 技能迁移

## 6.15.1 迁移到传统后端

Cloudflare 版本：

```text
Worker
  ├── D1
  ├── KV
  └── R2
```

传统后端版本：

```text
FastAPI / Node.js
  ├── PostgreSQL
  ├── Redis
  └── MinIO / S3
```

对应关系：

| Cloudflare | 通用概念 | 传统后端 |
|---|---|---|
| Worker | API Server | FastAPI / Express / Gin |
| D1 | SQL Database | PostgreSQL / MySQL |
| KV | Config / Cache | Redis / etcd / Consul |
| R2 | Object Storage | S3 / MinIO |
| Secret | Secret Management | `.env` / Vault |
| Wrangler | DevOps CLI | Docker / Terraform / Helm |

---

## 6.15.2 迁移到 BMS 测试数据平台

当前项目可以改成：

```text
articles  → test_cases
messages  → test_feedback
R2 files  → logs / waveform / report / screenshots
KV config → platform config / feature flags
```

架构：

```text
Worker API
  ├── D1：测试用例索引 / 执行记录 / 设备信息
  ├── R2：log 文件 / CSV / scope 截图 / report
  ├── KV：平台配置 / feature flag / 当前 release
  └── Secret：AE/Admin 管理接口
```

推荐数据模型：

```text
platforms
firmware_versions
test_cases
test_runs
test_results
test_artifacts
```

R2 key 规范：

```text
artifacts/{platform}/{fw_version}/{run_id}/smbus.log
artifacts/{platform}/{fw_version}/{run_id}/scope.csv
artifacts/{platform}/{fw_version}/{run_id}/report.html
artifacts/{platform}/{fw_version}/{run_id}/screenshots/fuseout.png
```

---

# 6.16 最终记忆模型

```text
Worker API
  ├── /api/health
  ├── /api/articles      → D1
  ├── /api/messages      → D1
  ├── /api/config        → KV
  └── /api/files/*       → R2
```

存储选择规则：

```text
能用 SQL 查询和约束的数据 → D1 / 数据库
只需要 key-value 读取的数据 → KV
文件和大对象 → R2 / 对象存储
敏感配置 → Secret
```

项目核心价值：

```text
不是单独学 D1、KV、R2，
而是建立“边缘计算入口 + SQL 数据库 + 分布式 KV + 对象存储 + Secret + 部署”的完整后端工程模型。
```
