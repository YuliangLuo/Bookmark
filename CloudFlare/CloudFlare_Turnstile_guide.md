# Cloudflare Project 4：安全加固完整学习教程

> 脱敏说明：本文中的域名、项目名、数据库名、Bucket 名、用户名均已替换为示例占位符，例如 `dev.example.com`、`example.com`、`cloudflare-security-demo`、`app_db`、`app-assets`。

> 目标：把一个 Cloudflare Workers + D1 + KV + R2 项目，从“接口能跑”升级到“具备基础生产防护能力”。  
> 本教程基于一次脱敏后的真实调试过程整理，重点覆盖 `/api/contact` 的 Turnstile 服务端校验、PowerShell 调试、Wrangler 本地/线上差异、Route 绑定、D1 migration、错误定位方法。

---

## 0. 最终目标

本阶段要实现：

| 路径 | 防护策略 | 位置 |
|---|---|---|
| `/api/contact` | Rate Limit + Turnstile 服务端校验 + D1 写入 | Cloudflare Edge + Worker |
| `/api/login` | Rate Limit + WAF | Cloudflare Edge |
| `/admin/*` | Cloudflare Access | Cloudflare Zero Trust |
| `/api/files/*` | Bearer Token + 文件类型检查 | Worker |
| `/wp-login.php` | 直接 Block | WAF Custom Rules |
| 异常 User-Agent | Managed Challenge | WAF Custom Rules |
| 非业务地区访问后台 | Block 或 Challenge | WAF / Access |

请求链路：

```text
Browser / API Client
        |
        v
Cloudflare Edge
  ├─ WAF Custom Rules
  ├─ Rate Limiting Rules
  ├─ Cloudflare Access
        |
        v
Cloudflare Worker
  ├─ /api/contact
  │    ├─ JSON body 解析
  │    ├─ Turnstile token 检查
  │    ├─ Siteverify 服务端校验
  │    ├─ 输入参数校验
  │    └─ D1 messages 写入
  |
  ├─ /api/files/*
  │    ├─ Bearer Token
  │    ├─ 文件扩展名检查
  │    ├─ Content-Type 检查
  │    └─ R2 写入
        |
        v
D1 / KV / R2
```

---

## 1. 关键概念

### 1.1 本地 Worker、线上 Worker、域名 Route

必须区分：

```text
npx wrangler dev
  本地 Worker runtime，默认 http://localhost:8787

npx wrangler deploy
  部署 Worker 脚本到 Cloudflare

workers.dev
  Cloudflare 自动提供的 Worker 域名

dev.example.com
  你的自定义域名，需要 route 或 custom domain 绑定到 Worker

wrangler.jsonc routes
  决定 dev.example.com/api/* 是否真的进入这个 Worker
```

调试时必须用 `/api/health` 的 `build_id` 判断线上是否运行当前代码。

---

### 1.2 Turnstile 的 4 个对象

| 对象 | 位置 | 是否公开 | 说明 |
|---|---|---:|---|
| Sitekey | 前端 HTML | 公开 | 渲染 Turnstile widget |
| Secret key | Worker 后端 | 私密 | 调用 Siteverify |
| token | 前端提交给后端 | 一次性 | 用户通过 Turnstile 后生成 |
| Siteverify API | Worker 调用 | 后端调用 | 验证 token 是否真实有效 |

正确链路：

```text
前端使用 sitekey 渲染 widget
  -> 用户完成 Turnstile
  -> 前端拿到 token
  -> POST /api/contact 携带 token
  -> Worker 用 secret 调用 Siteverify
  -> success=true 才写入 D1
```

只检查 `token` 是否存在是不够的。必须调用：

```ts
fetch("https://challenges.cloudflare.com/turnstile/v0/siteverify")
```

---

### 1.3 `.dev.vars` 与 `wrangler secret put`

| 位置 | 作用 |
|---|---|
| `.dev.vars` | 本地 `npx wrangler dev` 使用 |
| `wrangler secret put` | 线上 Worker 使用 |
| `wrangler.jsonc vars` | 不建议放敏感值 |
| 前端代码 | 只能放 Sitekey，不能放 Secret key |

本地 `.dev.vars` 配好了，不代表线上 `dev.example.com` 有同样的 secret。

---

## 2. 项目目录建议

```text
cloudflare-security-demo/
├─ src/
│  └─ index.ts
├─ migrations/
├─ docs/
├─ .dev.vars              # 本地 secret，不提交 Git
├─ wrangler.jsonc
├─ contact-no-token.json  # 本地调试用
└─ contact-fake-token.json
```

`.gitignore` 必须包含：

```gitignore
.dev.vars
.dev.vars.*
.env
.env.*
```

---

## 3. PowerShell 创建 `.dev.vars`

### 3.1 生成随机密钥函数

```powershell
function New-HexSecret {
    param(
        [int]$Bytes = 32
    )

    $buffer = [byte[]]::new($Bytes)
    $rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()

    try {
        $rng.GetBytes($buffer)
    }
    finally {
        $rng.Dispose()
    }

    return -join ($buffer | ForEach-Object { $_.ToString("x2") })
}
```

---

### 3.2 本地开发版 `.dev.vars`

本地可以先使用 Cloudflare Turnstile dummy secret：

```powershell
function New-HexSecret {
    param(
        [int]$Bytes = 32
    )

    $buffer = [byte[]]::new($Bytes)
    $rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()

    try {
        $rng.GetBytes($buffer)
    }
    finally {
        $rng.Dispose()
    }

    return -join ($buffer | ForEach-Object { $_.ToString("x2") })
}

$ADMIN_TOKEN = New-HexSecret -Bytes 32
$TURNSTILE_SECRET_KEY = "1x0000000000000000000000000000000AA"
$IP_HASH_SALT = New-HexSecret -Bytes 32
$FILES_API_TOKEN = New-HexSecret -Bytes 48
$ALLOWED_TURNSTILE_HOSTNAMES = "localhost,127.0.0.1,dev.example.com,example.com"

$DevVars = @"
ADMIN_TOKEN="$ADMIN_TOKEN"
TURNSTILE_SECRET_KEY="$TURNSTILE_SECRET_KEY"
IP_HASH_SALT="$IP_HASH_SALT"
FILES_API_TOKEN="$FILES_API_TOKEN"
ALLOWED_TURNSTILE_HOSTNAMES="$ALLOWED_TURNSTILE_HOSTNAMES"
"@

$Utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText((Join-Path (Get-Location) ".dev.vars"), $DevVars, $Utf8NoBom)

Get-Content .dev.vars
```

生产环境必须使用 Cloudflare Turnstile Dashboard 中真实的 **Secret key**，不要误用前端 Sitekey。

---

## 4. `wrangler.jsonc` 关键配置

示例配置：

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "cloudflare-security-demo",
  "main": "src/index.ts",
  "compatibility_date": "2026-05-18",

  "workers_dev": true,
  "preview_urls": true,

  "routes": [
    {
      "pattern": "dev.example.com/api/*",
      "zone_name": "example.com"
    }
  ],

  "secrets": {
    "required": [
      "ADMIN_TOKEN",
      "TURNSTILE_SECRET_KEY",
      "ALLOWED_TURNSTILE_HOSTNAMES",
      "IP_HASH_SALT",
      "FILES_API_TOKEN"
    ]
  },

  "observability": {
    "enabled": true
  },

  "upload_source_maps": true,

  "compatibility_flags": [
    "nodejs_compat"
  ],

  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "app_db",
      "database_id": "replace-with-your-d1-database-id",
      "migrations_dir": "migrations"
    }
  ],

  "kv_namespaces": [
    {
      "binding": "APP_CONFIG",
      "id": "replace-with-your-kv-namespace-id"
    }
  ],

  "r2_buckets": [
    {
      "binding": "ASSETS_BUCKET",
      "bucket_name": "app-assets"
    }
  ]
}
```

检查配置是否能构建：

```powershell
npx wrangler deploy --dry-run
```

---

## 5. `index.ts` 关键代码

下面不是完整业务代码，而是安全加固所需的关键代码块。

---

### 5.1 `BUILD_ID` 与 `Env`

`BUILD_ID` 用于判断线上域名是否运行当前代码。

```ts
const BUILD_ID = "security-contact-v2-20260522";

interface Env {
  DB: D1Database;
  APP_CONFIG: KVNamespace;
  ASSETS_BUCKET: R2Bucket;

  ADMIN_TOKEN: string;

  TURNSTILE_SECRET_KEY: string;
  ALLOWED_TURNSTILE_HOSTNAMES: string;

  IP_HASH_SALT: string;
  FILES_API_TOKEN: string;
}
```

---

### 5.2 统一错误类型

```ts
class HttpError extends Error {
  status: number;

  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}
```

---

### 5.3 JSON 响应封装

```ts
const CORS_HEADERS: Record<string, string> = {
  "Access-Control-Allow-Origin": "*",
  "access-control-allow-headers": "content-type,authorization",
  "access-control-allow-methods": "GET,POST,PUT,DELETE,OPTIONS",
};

function json(
  data: unknown,
  init?: ResponseInit,
): Response {
  return new Response(JSON.stringify(data), {
    ...init,
    headers: {
      "Content-Type": "application/json; charset=utf-8",
      ...CORS_HEADERS,
      ...(init?.headers ?? {}),
    },
  });
}
```

---

### 5.4 JSON body 读取

```ts
async function readJson<T>(request: Request): Promise<T> {
  const raw = await request.text();

  if (!raw) {
    throw new HttpError(400, "Invalid JSON body");
  }

  if (raw.length > 8192) {
    throw new HttpError(413, "Payload too large");
  }

  try {
    return JSON.parse(raw) as T;
  } catch {
    throw new HttpError(400, "Invalid JSON body");
  }
}
```

PowerShell 下如果直接传 `$Body` 容易破坏 JSON，引发：

```json
{"ok":false,"error":"Invalid JSON body"}
```

推荐用 JSON 文件 + `--data-binary @file` 测试。

---

### 5.5 IP hash

```ts
function getClientIp(request: Request): string | null {
  return (
    request.headers.get("CF-Connecting-IP") ||
    request.headers.get("X-Forwarded-For") ||
    null
  );
}

async function sha256Hex(input: string): Promise<string> {
  const data = new TextEncoder().encode(input);
  const digest = await crypto.subtle.digest("SHA-256", data);

  return Array.from(new Uint8Array(digest))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

async function getIpHash(request: Request, env: Env): Promise<string | null> {
  const ip = getClientIp(request);
  if (!ip || !env.IP_HASH_SALT) return null;
  return sha256Hex(`${env.IP_HASH_SALT}:${ip}`);
}
```

---

### 5.6 Turnstile 类型

```ts
type MessageInput = {
  name?: string;
  email?: string;
  message?: string;
};

type ContactInput = MessageInput & {
  turnstileToken?: string;
  turnstile_token?: string;
  "cf-turnstile-response"?: string;
};

type TurnstileResult = {
  success: boolean;
  challenge_ts?: string;
  hostname?: string;
  action?: string;
  cdata?: string;
  "error-codes"?: string[];
};
```

---

### 5.7 Turnstile Siteverify 校验

最终推荐版本：

```ts
async function verifyTurnstile(
  request: Request,
  env: Env,
  token: string,
): Promise<void> {
  if (!token) {
    throw new HttpError(403, "TURNSTILE_REQUIRED");
  }

  if (!env.TURNSTILE_SECRET_KEY) {
    throw new HttpError(500, "TURNSTILE_SECRET_KEY is not configured");
  }

  const formData = new FormData();
  formData.append("secret", env.TURNSTILE_SECRET_KEY);
  formData.append("response", token);

  const ip = getClientIp(request);
  if (ip) {
    formData.append("remoteip", ip);
  }

  const res = await fetch(
    "https://challenges.cloudflare.com/turnstile/v0/siteverify",
    {
      method: "POST",
      body: formData,
    },
  );

  const raw = await res.text();

  let result: TurnstileResult;
  try {
    result = JSON.parse(raw) as TurnstileResult;
  } catch {
    console.log(
      JSON.stringify({
        tag: "turnstile_non_json_response",
        status: res.status,
        statusText: res.statusText,
        body: raw.slice(0, 500),
      }),
    );

    throw new HttpError(502, "TURNSTILE_VERIFY_NON_JSON_RESPONSE");
  }

  console.log(
    JSON.stringify({
      tag: "turnstile_siteverify_response",
      http_status: res.status,
      success: result.success,
      hostname: result.hostname,
      action: result.action,
      errors: result["error-codes"] ?? [],
    }),
  );

  if (!res.ok) {
    const errors = result["error-codes"] ?? [];

    if (
      errors.includes("missing-input-secret") ||
      errors.includes("invalid-input-secret")
    ) {
      throw new HttpError(500, "TURNSTILE_SECRET_INVALID");
    }

    throw new HttpError(403, "TURNSTILE_FAILED");
  }

  if (!result.success) {
    throw new HttpError(403, "TURNSTILE_FAILED");
  }

  const allowedHostnames = (env.ALLOWED_TURNSTILE_HOSTNAMES ?? "")
    .split(",")
    .map((s) => s.trim())
    .filter(Boolean);

  if (
    result.hostname &&
    allowedHostnames.length > 0 &&
    !allowedHostnames.includes(result.hostname)
  ) {
    console.log(
      JSON.stringify({
        tag: "turnstile_hostname_mismatch",
        hostname: result.hostname,
        allowedHostnames,
      }),
    );

    throw new HttpError(403, "TURNSTILE_HOST_MISMATCH");
  }
}
```

注意：不要在同一个函数里同时保留下面两种声明，否则会报重复声明：

```ts
let result: TurnstileResult;
const result = (await res.json()) as TurnstileResult;
```

---

### 5.8 `/api/contact` 正式处理函数

```ts
async function createContactMessage(
  request: Request,
  env: Env,
): Promise<Response> {
  const body = await readJson<ContactInput>(request);

  const name = body.name?.trim() ?? "";
  const email = body.email?.trim() || null;
  const message = body.message?.trim() ?? "";

  const token =
    body.turnstileToken?.trim() ||
    body.turnstile_token?.trim() ||
    body["cf-turnstile-response"]?.trim() ||
    "";

  if (!name || name.length > 80) {
    throw new HttpError(400, "Name is required and must be <= 80 chars");
  }

  if (email && email.length > 160) {
    throw new HttpError(400, "Email must be <= 160 chars");
  }

  if (!message || message.length > 2000) {
    throw new HttpError(400, "Message is required and must be <= 2000 chars");
  }

  if (!token) {
    throw new HttpError(403, "TURNSTILE_REQUIRED");
  }

  await verifyTurnstile(request, env, token);

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
      accepted: true,
      created_at: now,
    },
    { status: 201 },
  );
}
```

---

### 5.9 `fetch()` 路由关键点

异步 handler 建议使用 `return await`，便于外层 `try/catch` 收敛错误。

```ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    if (request.method === "OPTIONS") {
      return new Response(null, {
        status: 204,
        headers: CORS_HEADERS,
      });
    }

    try {
      if (request.method === "GET" && path === "/api/health") {
        return json({
          ok: true,
          service: "cloudflare-security-demo",
          build_id: BUILD_ID,
          time: new Date().toISOString(),
        });
      }

      if (request.method === "POST" && path === "/api/contact") {
        return await createContactMessage(request, env);
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
      const e = err as { status?: unknown; message?: unknown; stack?: unknown };

      console.error(
        JSON.stringify({
          tag: "fetch_error",
          status: e.status,
          message: e.message,
          stack: e.stack,
        }),
      );

      if (
        typeof e.status === "number" &&
        e.status >= 400 &&
        e.status <= 599 &&
        typeof e.message === "string"
      ) {
        return json(
          {
            ok: false,
            error: e.message,
          },
          { status: e.status },
        );
      }

      return json(
        {
          ok: false,
          error: "Internal Server Error",
        },
        { status: 500 },
      );
    }
  },
};
```

---

## 6. PowerShell 本地测试方法

### 6.1 启动本地 Worker

```powershell
npx wrangler dev
```

另开一个 PowerShell 窗口执行测试。

---

### 6.2 创建无 token JSON 文件

```powershell
$Json = '{"name":"TestUser","email":"test@example.com","message":"hello"}'

$Utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText(
    (Join-Path (Get-Location) "contact-no-token.json"),
    $Json,
    $Utf8NoBom
)

Get-Content .\contact-no-token.json -Raw
```

发送请求：

```powershell
curl.exe -i -X POST "http://localhost:8787/api/contact" `
  -H "Content-Type: application/json" `
  --data-binary "@contact-no-token.json"
```

正确结果：

```http
HTTP/1.1 403 Forbidden
```

```json
{"ok":false,"error":"TURNSTILE_REQUIRED"}
```

---

### 6.3 创建 fake token JSON 文件

```powershell
$Json = '{"name":"TestUser","email":"test@example.com","message":"hello","turnstileToken":"fake-token"}'

$Utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText(
    (Join-Path (Get-Location) "contact-fake-token.json"),
    $Json,
    $Utf8NoBom
)

Get-Content .\contact-fake-token.json -Raw
```

发送请求：

```powershell
curl.exe -i -X POST "http://localhost:8787/api/contact" `
  -H "Content-Type: application/json" `
  --data-binary "@contact-fake-token.json"
```

预期结果之一：

```json
{"ok":false,"error":"TURNSTILE_FAILED"}
```

如果返回：

```json
{"ok":false,"error":"TURNSTILE_SECRET_INVALID"}
```

说明 `TURNSTILE_SECRET_KEY` 填错，常见原因是误填了前端 Sitekey。

---

## 7. 直接测试 Cloudflare Siteverify

用于确认 secret 是否有效。

```powershell
$SecretLine = Get-Content .\.dev.vars | Where-Object { $_ -match '^TURNSTILE_SECRET_KEY=' }
$Secret = $SecretLine -replace '^TURNSTILE_SECRET_KEY="?','' -replace '"$',''

Write-Host "Secret length:" $Secret.Length

curl.exe -i -X POST "https://challenges.cloudflare.com/turnstile/v0/siteverify" `
  -F "secret=$Secret" `
  -F "response=fake-token"
```

常见返回：

```json
{
  "success": false,
  "error-codes": ["invalid-input-response"]
}
```

说明 secret 基本可用，只是 token 是假的。

如果是：

```json
{
  "success": false,
  "error-codes": ["invalid-input-secret"]
}
```

说明 `TURNSTILE_SECRET_KEY` 错了。

---

## 8. D1 migration

如果 Turnstile 校验通过后出现：

```text
no such table: messages
```

说明 D1 migration 没应用到当前环境。

本地：

```powershell
npx wrangler d1 migrations apply app_db --local
```

线上：

```powershell
npx wrangler d1 migrations apply app_db --remote
```

检查表：

```powershell
npx wrangler d1 execute app_db --local --command "SELECT name FROM sqlite_master WHERE type='table';"
```

查看最新 messages：

```powershell
npx wrangler d1 execute app_db --local --command "SELECT id, name, email, message, created_at FROM messages ORDER BY id DESC LIMIT 5;"
```

---

## 9. 线上 Secret 与部署

`.dev.vars` 不会自动进入线上。线上必须执行：

```powershell
npx wrangler secret put ADMIN_TOKEN
npx wrangler secret put TURNSTILE_SECRET_KEY
npx wrangler secret put ALLOWED_TURNSTILE_HOSTNAMES
npx wrangler secret put IP_HASH_SALT
npx wrangler secret put FILES_API_TOKEN
```

建议：

```text
TURNSTILE_SECRET_KEY
  使用 Cloudflare Turnstile Dashboard 的真实 Secret key

ALLOWED_TURNSTILE_HOSTNAMES
  dev.example.com,example.com

IP_HASH_SALT
  PowerShell 随机生成

FILES_API_TOKEN
  PowerShell 随机生成

ADMIN_TOKEN
  PowerShell 随机生成
```

部署：

```powershell
npx wrangler deploy
```

打开实时日志：

```powershell
npx wrangler tail
```

---

## 10. 线上验收

### 10.1 确认线上运行新代码

```powershell
$BASE_URL = "https://dev.example.com"

curl.exe -i "$BASE_URL/api/health"
```

必须看到：

```json
"build_id": "security-contact-v2-20260522"
```

如果没有看到，说明 `dev.example.com/api/*` 没有绑定到当前 Worker，或者部署到了错误环境。

---

### 10.2 无 token 测试

```powershell
curl.exe -i -X POST "$BASE_URL/api/contact" `
  -H "Content-Type: application/json" `
  --data-binary "@contact-no-token.json"
```

正确结果：

```http
HTTP/2 403
```

```json
{"ok":false,"error":"TURNSTILE_REQUIRED"}
```

---

### 10.3 fake token 测试

```powershell
curl.exe -i -X POST "$BASE_URL/api/contact" `
  -H "Content-Type: application/json" `
  --data-binary "@contact-fake-token.json"
```

正确结果：

```http
HTTP/2 403
```

```json
{"ok":false,"error":"TURNSTILE_FAILED"}
```

---

### 10.4 真实前端 token 测试

前端：

```html
<script
  src="https://challenges.cloudflare.com/turnstile/v0/api.js"
  async
  defer>
</script>

<form id="contact-form">
  <input id="name" name="name" required />
  <input id="email" name="email" />
  <textarea id="message" name="message" required></textarea>

  <div
    class="cf-turnstile"
    data-sitekey="你的真实Sitekey">
  </div>

  <button type="submit">Submit</button>
</form>
```

JS：

```js
document.getElementById("contact-form").addEventListener("submit", async (e) => {
  e.preventDefault();

  const token = document.querySelector('[name="cf-turnstile-response"]')?.value;

  const res = await fetch("/api/contact", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      name: document.getElementById("name").value,
      email: document.getElementById("email").value,
      message: document.getElementById("message").value,
      turnstileToken: token,
    }),
  });

  const data = await res.json();
  console.log(data);
});
```

成功结果：

```http
HTTP/2 201
```

```json
{
  "ok": true,
  "accepted": true,
  "created_at": "..."
}
```

---

## 11. WAF / Rate Limit / Access 配置摘要

### 11.1 WAF：Block 扫描路径

表达式：

```cf
(
  http.request.uri.path eq "/wp-login.php" or
  http.request.uri.path eq "/xmlrpc.php" or
  http.request.uri.path eq "/.env" or
  http.request.uri.path wildcard "/phpmyadmin*" or
  http.request.uri.path wildcard "/vendor/phpunit/*" or
  http.request.uri.path wildcard "/cgi-bin/*"
)
```

动作：

```text
Block
```

---

### 11.2 WAF：异常 User-Agent Challenge

```cf
(
  lower(http.user_agent) contains "sqlmap" or
  lower(http.user_agent) contains "nikto" or
  lower(http.user_agent) contains "nmap" or
  lower(http.user_agent) contains "masscan" or
  lower(http.user_agent) contains "acunetix" or
  lower(http.user_agent) contains "nessus" or
  lower(http.user_agent) contains "wpscan"
)
```

动作：

```text
Managed Challenge
```

---

### 11.3 Rate Limit：`/api/contact`

```cf
(
  http.request.uri.path eq "/api/contact" and
  http.request.method eq "POST"
)
```

建议：

```text
5 requests / 60 seconds / IP
mitigation timeout: 10 minutes
action: Block
custom response: 429
```

---

### 11.4 Access：`/admin/*`

路径：

```text
dev.example.com/admin*
```

Policy：

```text
Allow:
  Emails ending in @your-company.com

Require:
  trusted country or corporate IP range
```

禁止使用：

```text
Include Everyone
Include all valid emails
```

---

## 12. 调试案例总结

| 现象 | 根因 | 修复 |
|---|---|---|
| `/api/contact` 返回 404 | 路由未注册或不是当前 Worker | 检查 `src/index.ts` 和 `wrangler.jsonc main` |
| 线上返回 `Contact request received` | 线上跑旧代码或域名没绑定当前 Worker | 加 `build_id`，配置 route |
| 无 token 返回 `ok:true` | 只做了占位逻辑，没强制 Turnstile | 加 `TURNSTILE_REQUIRED` |
| fake token 返回 `ok:true` | 没调用 Siteverify | 加 `verifyTurnstile()` |
| `Invalid JSON body` | PowerShell/curl 传坏 JSON | 用 `--data-binary "@file.json"` |
| 500 Internal Server Error | 异步异常未被 catch 或 helper 漏复制 | `return await handler()`，增强 catch |
| stack trace 暴露给客户端 | Worker 错误没有统一收敛 | catch 中只返回 code，日志写 console |
| `The symbol result has already been declared` | 新旧代码同时存在 | 整函数替换，删除旧 `const result` |
| `TURNSTILE_VERIFY_HTTP_ERROR` | Siteverify 返回非 2xx，但未读取 body | 先读 `res.text()`，打印 error-codes |
| `TURNSTILE_SECRET_INVALID` | 误填 Sitekey 或 Secret 错 | 重新复制 Turnstile Secret key |
| `no such table: messages` | D1 migration 没应用 | 执行 `wrangler d1 migrations apply` |
| 本地正常，线上不正常 | `.dev.vars` 与线上 secret 不一致 | `wrangler secret put` |
| 部署后线上无 `build_id` | 自定义域名未绑定此 Worker | 检查 Dashboard Domains & Routes |

---

## 13. 最终验收标准

| 编号 | 验收项 | 正确结果 |
|---|---|---|
| AC-1 | `GET /api/health` | 返回当前 `build_id` |
| AC-2 | `POST /api/contact` 无 token | `403 TURNSTILE_REQUIRED` |
| AC-3 | `POST /api/contact` fake token | `403 TURNSTILE_FAILED` |
| AC-4 | `POST /api/contact` 真实 token | `201 ok:true` |
| AC-5 | `/api/contact` 高频请求 | Cloudflare Rate Limit |
| AC-6 | `/admin/*` 未登录访问 | Cloudflare Access 登录 |
| AC-7 | `/wp-login.php` | WAF Block |
| AC-8 | Worker 异常 | 不泄露 stack trace |
| AC-9 | D1 写入 | `messages` 表有新增记录 |

---

## 14. 推荐学习顺序

这次调试暴露的知识点，建议按这个顺序补：

```text
1. HTTP Request / Response / Status Code
2. JSON body、Content-Type、curl/PowerShell 请求构造
3. TypeScript async/await、Promise 错误传播
4. Cloudflare Workers fetch handler
5. wrangler dev / deploy / route / custom domain
6. .dev.vars / wrangler secret put / env binding
7. Turnstile sitekey / secret / token / Siteverify
8. D1 binding / migration / local / remote
9. WAF / Rate Limit / Access 分层防护
10. 生产 API 错误收敛和日志观测
```

---

## 15. 最小调试闭环

以后排错按这个顺序：

```text
1. GET /api/health 是否有 build_id？
   没有：查部署和 route。

2. POST /api/contact 是否 404？
   404：查路由。

3. 无 token 是否 403 TURNSTILE_REQUIRED？
   不是：查 /api/contact 代码。

4. fake token 是否 403 TURNSTILE_FAILED？
   不是：查 verifyTurnstile 和 secret。

5. 真实 token 是否 201？
   不是：查 Turnstile hostname、D1 migration、Siteverify 日志。

6. 本地正常、线上不正常？
   查 wrangler secret put、route、自定义域名、D1 remote migration。
```

---

## 16. 建议提交记录

```text
feat(security): enforce Turnstile server-side validation for contact API

- Add TURNSTILE_SECRET_KEY and ALLOWED_TURNSTILE_HOSTNAMES env bindings
- Add Siteverify integration for /api/contact
- Reject missing or invalid Turnstile tokens
- Add build_id to /api/health for deployment verification
- Improve Worker error handling to avoid stack trace leakage
- Add PowerShell-friendly JSON test workflow
```