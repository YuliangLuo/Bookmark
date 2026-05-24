# CloudFlare_CICD_Github_actions_guide.md 自动部署实战教程

## 目标

从零搭建一个具备工程化部署能力的 Cloudflare Worker 项目，包含：

- Worker API
- D1 数据库
- KV 配置存储
- preview / production 双环境
- GitHub Actions 自动部署
- D1 migration
- Worker runtime secrets
- GitHub repository secrets
- DNS / Route / Git 常见故障排查

最终环境映射：

| 环境 | 分支 | 域名 | Worker |
|---|---|---|---|
| local | 本地开发 | `http://localhost:8787` | Wrangler dev |
| preview | `dev` | `https://dev.quantguard.org/api/*` | `cloudflare-ci-demo-preview` |
| production | `main` | `https://quantguard.org/api/*` | `cloudflare-ci-demo-production` |

---

## 目录

```text
cloudflare-ci-demo/
├── src/
│   └── index.ts
├── migrations/
│   └── 0001_init_schema.sql
├── .github/
│   └── workflows/
│       └── deploy-worker.yml
├── .dev.vars
├── .gitignore
├── package.json
├── tsconfig.json
└── wrangler.jsonc
```

---

# 1. 操作总流程

```text
1. 创建项目目录
2. 初始化 npm / TypeScript / Wrangler
3. 编写 Worker 源代码
4. 编写 D1 migration
5. 创建 Cloudflare D1 preview / production
6. 创建 Cloudflare KV preview / production
7. 配置 wrangler.jsonc
8. 配置本地 .dev.vars
9. 设置 Cloudflare Worker runtime secrets
10. 本地运行和测试
11. 部署 preview
12. 部署 production
13. 处理 DNS / Route 冲突
14. 初始化 Git 仓库
15. 推送到 GitHub
16. 设置 GitHub Actions Secrets
17. 添加 GitHub Actions workflow
18. 验证 dev → preview 自动部署
19. 验证 main → production 自动部署
20. 回滚与故障排查
```

---

# 2. 核心概念

## 2.1 Wrangler 环境模型

`wrangler.jsonc` 中的：

```jsonc
"env": {
  "preview": {},
  "production": {}
}
```

分别对应：

```bash
npx wrangler deploy --env preview
npx wrangler deploy --env production
```

注意：

```text
vars / kv_namespaces / d1_databases / r2_buckets / secrets 等 bindings 不会自动继承顶层配置。
preview 和 production 需要分别显式配置。
```

推荐模型：

```text
代码固定使用 env.DB
preview 的 env.DB 指向 cf_ci_demo_preview
production 的 env.DB 指向 cf_ci_demo_prod
```

代码固定使用：

```ts
env.DB.prepare(...)
env.APP_CONFIG.get(...)
env.APP_CONFIG.put(...)
```

因此配置必须是：

```jsonc
"d1_databases": [
  {
    "binding": "DB"
  }
],
"kv_namespaces": [
  {
    "binding": "APP_CONFIG"
  }
]
```

不要把 binding 写成数据库名，例如：

```jsonc
"binding": "cf_ci_demo_preview"
```

否则代码就必须改成：

```ts
env.cf_ci_demo_preview.prepare(...)
```

这不推荐。

---

## 2.2 Worker runtime secrets 与 GitHub Secrets 的区别

### Worker runtime secrets

用于 Worker 运行时：

```text
ADMIN_TOKEN
IP_HASH_SALT
```

写入方式：

```bash
npx wrangler secret put ADMIN_TOKEN --env preview
npx wrangler secret put IP_HASH_SALT --env preview
```

### GitHub Actions Secrets

用于 GitHub Actions 调用 Wrangler 部署：

```text
CLOUDFLARE_ACCOUNT_ID
CLOUDFLARE_API_TOKEN
```

在 workflow 中引用：

```yaml
env:
  CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
  CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

---

## 2.3 D1 ID 与 KV ID 不要混淆

D1 database ID 通常是 UUID 形式：

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

KV namespace ID 通常是连续 hex 字符串：

```text
0c339c1ec1214d1caab08c1994928d3c
```

如果把 D1 UUID 填到 KV `id`，部署会报错：

```text
KV namespace 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' is not valid.
Please verify the namespace_id in your configuration. [code: 10042]
```

---

# 3. 共用源代码

Linux/WSL 和 PowerShell 使用相同源代码。

---

## 3.1 `src/index.ts`

```ts
export interface Env {
  APP_ENV: string;
  PUBLIC_API_BASE: string;

  DB: D1Database;
  APP_CONFIG: KVNamespace;

  ADMIN_TOKEN: string;
  IP_HASH_SALT: string;
}

const corsHeaders: Record<string, string> = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET,POST,PUT,OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type,Authorization",
};

function json(data: unknown, status = 200): Response {
  return new Response(JSON.stringify(data, null, 2), {
    status,
    headers: {
      "Content-Type": "application/json; charset=utf-8",
      ...corsHeaders,
    },
  });
}

function text(data: string, status = 200): Response {
  return new Response(data, {
    status,
    headers: {
      "Content-Type": "text/plain; charset=utf-8",
      ...corsHeaders,
    },
  });
}

function unauthorized(): Response {
  return json({ ok: false, error: "unauthorized" }, 401);
}

function isAdmin(request: Request, env: Env): boolean {
  const auth = request.headers.get("Authorization") ?? "";
  return auth === `Bearer ${env.ADMIN_TOKEN}`;
}

async function sha256Hex(input: string): Promise<string> {
  const data = new TextEncoder().encode(input);
  const digest = await crypto.subtle.digest("SHA-256", data);

  return [...new Uint8Array(digest)]
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

async function readJson<T>(request: Request): Promise<T | null> {
  try {
    return (await request.json()) as T;
  } catch {
    return null;
  }
}

type ContactBody = {
  name?: string;
  email?: string;
  message?: string;
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    if (request.method === "OPTIONS") {
      return new Response(null, {
        status: 204,
        headers: corsHeaders,
      });
    }

    if (path === "/api/health" && request.method === "GET") {
      return json({
        ok: true,
        service: "cloudflare-ci-demo",
        env: env.APP_ENV,
        status: "healthy",
        timestamp: new Date().toISOString(),
      });
    }

    if (path === "/api/version" && request.method === "GET") {
      return json({
        ok: true,
        service: "cloudflare-ci-demo",
        version: "0.1.0",
        env: env.APP_ENV,
        apiBase: env.PUBLIC_API_BASE,
        runtime: "cloudflare-workers",
      });
    }

    if (path === "/api/articles" && request.method === "GET") {
      const result = await env.DB.prepare(
        `
        SELECT id, slug, title, created_at
        FROM articles
        ORDER BY id DESC
        LIMIT 20
        `
      ).all();

      return json({
        ok: true,
        articles: result.results,
      });
    }

    if (path === "/api/contact" && request.method === "POST") {
      const body = await readJson<ContactBody>(request);

      if (!body?.name || !body?.message) {
        return json(
          {
            ok: false,
            error: "name and message are required",
          },
          400
        );
      }

      const ip = request.headers.get("CF-Connecting-IP") ?? "unknown";
      const ipHash = await sha256Hex(`${env.IP_HASH_SALT}:${ip}`);

      await env.DB.prepare(
        `
        INSERT INTO messages(name, email, message, ip_hash, created_at)
        VALUES (?, ?, ?, ?, ?)
        `
      )
        .bind(
          body.name.trim(),
          body.email?.trim() ?? null,
          body.message.trim(),
          ipHash,
          new Date().toISOString()
        )
        .run();

      return json({
        ok: true,
      });
    }

    if (path.startsWith("/api/config/") && request.method === "GET") {
      const key = decodeURIComponent(path.slice("/api/config/".length));
      const value = await env.APP_CONFIG.get(key);

      if (value === null) {
        return json(
          {
            ok: false,
            error: "config not found",
            key,
          },
          404
        );
      }

      return json({
        ok: true,
        key,
        value,
      });
    }

    if (path.startsWith("/api/config/") && request.method === "PUT") {
      if (!isAdmin(request, env)) {
        return unauthorized();
      }

      const key = decodeURIComponent(path.slice("/api/config/".length));
      const body = await readJson<{ value?: string }>(request);

      if (!body || typeof body.value !== "string") {
        return json(
          {
            ok: false,
            error: "value must be string",
          },
          400
        );
      }

      await env.APP_CONFIG.put(key, body.value);

      return json({
        ok: true,
        key,
      });
    }

    return json(
      {
        ok: false,
        error: "not_found",
        path,
      },
      404
    );
  },
};
```

---

## 3.2 `migrations/0001_init_schema.sql`

```sql
CREATE TABLE IF NOT EXISTS articles (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  slug TEXT NOT NULL UNIQUE,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT,
  message TEXT NOT NULL,
  ip_hash TEXT,
  created_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_articles_slug ON articles(slug);
CREATE INDEX IF NOT EXISTS idx_articles_created_at ON articles(created_at);
CREATE INDEX IF NOT EXISTS idx_messages_created_at ON messages(created_at);

INSERT OR IGNORE INTO articles(slug, title, content, created_at)
VALUES
  ('hello-cloudflare', 'Hello Cloudflare', 'This is the first article.', datetime('now'));
```

---

## 3.3 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "WebWorker"],
    "types": ["@cloudflare/workers-types"],
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"]
}
```

---

## 3.4 `package.json`

```json
{
  "name": "cloudflare-ci-demo",
  "version": "1.0.0",
  "type": "module",
  "private": true,
  "scripts": {
    "dev": "wrangler dev --env preview",
    "dev:remote": "wrangler dev --env preview --remote",
    "typecheck": "tsc --noEmit",
    "test": "npm run typecheck",
    "build": "npm run typecheck",
    "migrate:local": "wrangler d1 migrations apply DB --local --env preview",
    "migrate:preview": "wrangler d1 migrations apply DB --remote --env preview",
    "migrate:production": "wrangler d1 migrations apply DB --remote --env production",
    "deploy:preview": "wrangler deploy --env preview",
    "deploy:production": "wrangler deploy --env production",
    "tail:preview": "wrangler tail --env preview",
    "tail:production": "wrangler tail --env production",
    "rollback:preview": "wrangler rollback --env preview",
    "rollback:production": "wrangler rollback --env production"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "latest",
    "typescript": "latest",
    "wrangler": "latest"
  }
}
```

---

## 3.5 `wrangler.jsonc`

替换以下占位符：

```text
<CLOUDFLARE_ACCOUNT_ID>
<PREVIEW_D1_DATABASE_ID>
<PROD_D1_DATABASE_ID>
<PREVIEW_KV_NAMESPACE_ID>
<PROD_KV_NAMESPACE_ID>
```

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",

  "name": "cloudflare-ci-demo",
  "main": "src/index.ts",
  "compatibility_date": "2026-05-24",
  "account_id": "<CLOUDFLARE_ACCOUNT_ID>",
  "preview_urls": false,

  "env": {
    "preview": {
      "name": "cloudflare-ci-demo-preview",
      "workers_dev": true,

      "routes": [
        {
          "pattern": "dev.quantguard.org/api/*",
          "zone_name": "quantguard.org"
        }
      ],

      "vars": {
        "APP_ENV": "preview",
        "PUBLIC_API_BASE": "https://dev.quantguard.org"
      },

      "d1_databases": [
        {
          "binding": "DB",
          "database_name": "cf_ci_demo_preview",
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

      "secrets": {
        "required": [
          "ADMIN_TOKEN",
          "IP_HASH_SALT"
        ]
      }
    },

    "production": {
      "name": "cloudflare-ci-demo-production",
      "workers_dev": false,

      "routes": [
        {
          "pattern": "quantguard.org/api/*",
          "zone_name": "quantguard.org"
        }
      ],

      "vars": {
        "APP_ENV": "production",
        "PUBLIC_API_BASE": "https://quantguard.org"
      },

      "d1_databases": [
        {
          "binding": "DB",
          "database_name": "cf_ci_demo_prod",
          "database_id": "<PROD_D1_DATABASE_ID>",
          "migrations_dir": "migrations"
        }
      ],

      "kv_namespaces": [
        {
          "binding": "APP_CONFIG",
          "id": "<PROD_KV_NAMESPACE_ID>"
        }
      ],

      "secrets": {
        "required": [
          "ADMIN_TOKEN",
          "IP_HASH_SALT"
        ]
      }
    }
  }
}
```

---

## 3.6 `.dev.vars`

```env
ADMIN_TOKEN="local-admin-token"
IP_HASH_SALT="local-ip-hash-salt"
```

---

## 3.7 `.gitignore`

```gitignore
node_modules/
.wrangler/
.dev.vars
.dev.vars.*
.env
.env.*
.secrets/
dist/
```

---

# 4. Linux / WSL 操作步骤

## 4.1 创建项目

```bash
mkdir -p ~/cloudflare-ci-demo
cd ~/cloudflare-ci-demo

git init
npm init -y

npm install -D wrangler typescript @cloudflare/workers-types

mkdir -p src migrations .github/workflows
```

---

## 4.2 创建文件

可以将第 3 章的共用源代码分别保存为：

```text
src/index.ts
migrations/0001_init_schema.sql
tsconfig.json
package.json
wrangler.jsonc
.dev.vars
.gitignore
```

也可以使用 `cat > file <<'EOF'` 的方式创建文件。

示例：

```bash
cat > .gitignore <<'EOF'
node_modules/
.wrangler/
.dev.vars
.dev.vars.*
.env
.env.*
.secrets/
dist/
EOF
```

---

## 4.3 登录 Cloudflare

```bash
npx wrangler login
```

---

## 4.4 创建 D1 数据库

```bash
npx wrangler d1 create cf_ci_demo_preview --location apac
npx wrangler d1 create cf_ci_demo_prod --location apac
```

记录输出里的：

```text
database_id
```

---

## 4.5 创建 KV namespace

```bash
npx wrangler kv namespace create cf_ci_demo_app_config_preview
npx wrangler kv namespace create cf_ci_demo_app_config_prod
```

记录输出里的：

```text
id
```

检查：

```bash
npx wrangler d1 list
npx wrangler kv namespace list
```

---

## 4.6 修改 `wrangler.jsonc`

把真实 ID 写入：

```text
<CLOUDFLARE_ACCOUNT_ID>
<PREVIEW_D1_DATABASE_ID>
<PROD_D1_DATABASE_ID>
<PREVIEW_KV_NAMESPACE_ID>
<PROD_KV_NAMESPACE_ID>
```

---

## 4.7 生成 secrets

```bash
mkdir -p .secrets
chmod 700 .secrets

{
  echo "PREVIEW_ADMIN_TOKEN=$(openssl rand -base64 32)"
  echo "PREVIEW_IP_HASH_SALT=$(openssl rand -base64 32)"
  echo "PROD_ADMIN_TOKEN=$(openssl rand -base64 32)"
  echo "PROD_IP_HASH_SALT=$(openssl rand -base64 32)"
} > .secrets/cloudflare-ci-demo.env

chmod 600 .secrets/cloudflare-ci-demo.env
cat .secrets/cloudflare-ci-demo.env
```

注意：

```text
Base64 末尾的 = 是值的一部分，输入时要保留。
```

---

## 4.8 写入 Cloudflare Worker runtime secrets

preview：

```bash
npx wrangler secret put ADMIN_TOKEN --env preview
npx wrangler secret put IP_HASH_SALT --env preview
```

production：

```bash
npx wrangler secret put ADMIN_TOKEN --env production
npx wrangler secret put IP_HASH_SALT --env production
```

验证：

```bash
npx wrangler secret list --env preview
npx wrangler secret list --env production
```

---

## 4.9 本地测试

```bash
npm install
npm run typecheck
npm run migrate:local
npm run dev
```

另开终端：

```bash
curl -i http://localhost:8787/api/health
curl -i http://localhost:8787/api/version
curl -i http://localhost:8787/api/articles
```

写本地 KV：

```bash
curl -i -X PUT "http://localhost:8787/api/config/site:title" \
  -H "Authorization: Bearer local-admin-token" \
  -H "Content-Type: application/json" \
  --data '{"value":"QuantGuard Local"}'
```

读本地 KV：

```bash
curl -i "http://localhost:8787/api/config/site:title"
```

---

## 4.10 部署 preview

```bash
npm run typecheck
npm run migrate:preview
npm run deploy:preview
```

验证：

```bash
curl -i https://dev.quantguard.org/api/health
curl -i https://dev.quantguard.org/api/version
```

写 preview KV：

```bash
curl -i -X PUT "https://dev.quantguard.org/api/config/site:title" \
  -H "Authorization: Bearer 你的真实_PREVIEW_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"value":"QuantGuard Preview"}'
```

读 preview KV：

```bash
curl -i "https://dev.quantguard.org/api/config/site:title"
```

---

## 4.11 部署 production

```bash
npm run typecheck
npm run migrate:production
npm run deploy:production
```

验证：

```bash
curl -i https://quantguard.org/api/health
curl -i https://quantguard.org/api/version
```

---

## 4.12 GitHub SSH 配置

如果出现：

```text
git@github.com: Permission denied (publickey).
```

执行：

```bash
ls -al ~/.ssh
ssh-keygen -t ed25519 -C "你的GitHub邮箱"

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

cat ~/.ssh/id_ed25519.pub
```

把 `.pub` 公钥添加到 GitHub：

```text
GitHub
→ Settings
→ SSH and GPG keys
→ New SSH key
```

测试：

```bash
ssh -T git@github.com
```

成功示例：

```text
Hi <your-github-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## 4.13 Git 初始化和推送

```bash
git status
git add .
git commit -m "init cloudflare ci demo"

git branch -M dev
git remote add origin git@github.com:YuliangLuo/cloudflare-ci-demo.git
git push -u origin dev
```

创建 main：

```bash
git checkout -b main
git push -u origin main
git checkout dev
```

如果 main 推送出现 non-fast-forward：

```bash
git pull --rebase origin main
git push origin main
```

如果确认远端初始化内容可以丢弃：

```bash
git push --force-with-lease origin main
```

---

# 5. PowerShell 操作步骤

## 5.1 创建项目

```powershell
mkdir C:\Users\Dwayne\cloudflare-ci-demo
cd C:\Users\Dwayne\cloudflare-ci-demo

git init
npm init -y

npm install -D wrangler typescript @cloudflare/workers-types

mkdir src
mkdir migrations
mkdir .github
mkdir .github\workflows
```

---

## 5.2 创建文件

PowerShell 推荐用 here-string：

```powershell
@'
文件内容
'@ | Set-Content -Encoding utf8 文件路径
```

示例创建 `.gitignore`：

```powershell
@'
node_modules/
.wrangler/
.dev.vars
.dev.vars.*
.env
.env.*
.secrets/
dist/
'@ | Set-Content -Encoding utf8 .gitignore
```

其他源代码文件可直接复制第 3 章内容创建。

---

## 5.3 登录 Cloudflare

```powershell
npx wrangler login
```

---

## 5.4 创建 Cloudflare 资源

```powershell
npx wrangler d1 create cf_ci_demo_preview --location apac
npx wrangler d1 create cf_ci_demo_prod --location apac

npx wrangler kv namespace create cf_ci_demo_app_config_preview
npx wrangler kv namespace create cf_ci_demo_app_config_prod

npx wrangler d1 list
npx wrangler kv namespace list
```

---

## 5.5 生成 secrets

```powershell
mkdir .secrets -Force

function New-Base64Secret {
  $bytes = New-Object byte[] 32
  [System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
  [Convert]::ToBase64String($bytes)
}

@"
PREVIEW_ADMIN_TOKEN=$(New-Base64Secret)
PREVIEW_IP_HASH_SALT=$(New-Base64Secret)
PROD_ADMIN_TOKEN=$(New-Base64Secret)
PROD_IP_HASH_SALT=$(New-Base64Secret)
"@ | Set-Content -Encoding utf8 .secrets\cloudflare-ci-demo.env

Get-Content .secrets\cloudflare-ci-demo.env
```

写入 Cloudflare secrets：

```powershell
npx wrangler secret put ADMIN_TOKEN --env preview
npx wrangler secret put IP_HASH_SALT --env preview

npx wrangler secret put ADMIN_TOKEN --env production
npx wrangler secret put IP_HASH_SALT --env production
```

验证：

```powershell
npx wrangler secret list --env preview
npx wrangler secret list --env production
```

---

## 5.6 本地测试

```powershell
npm install
npm run typecheck
npm run migrate:local
npm run dev
```

另开 PowerShell：

```powershell
Invoke-RestMethod -Uri "http://localhost:8787/api/health" -Method GET
Invoke-RestMethod -Uri "http://localhost:8787/api/version" -Method GET
Invoke-RestMethod -Uri "http://localhost:8787/api/articles" -Method GET
```

写本地 KV：

```powershell
$headers = @{
  "Authorization" = "Bearer local-admin-token"
  "Content-Type"  = "application/json"
}

$body = @{
  value = "QuantGuard Local"
} | ConvertTo-Json

Invoke-RestMethod `
  -Uri "http://localhost:8787/api/config/site:title" `
  -Method PUT `
  -Headers $headers `
  -Body $body
```

读本地 KV：

```powershell
Invoke-RestMethod -Uri "http://localhost:8787/api/config/site:title" -Method GET
```

---

## 5.7 部署 preview

```powershell
npm run typecheck
npm run migrate:preview
npm run deploy:preview
```

验证：

```powershell
Invoke-RestMethod -Uri "https://dev.quantguard.org/api/health" -Method GET
Invoke-RestMethod -Uri "https://dev.quantguard.org/api/version" -Method GET
```

写 preview KV：

```powershell
$headers = @{
  "Authorization" = "Bearer 你的真实_PREVIEW_ADMIN_TOKEN"
  "Content-Type"  = "application/json"
}

$body = @{
  value = "QuantGuard Preview"
} | ConvertTo-Json

Invoke-RestMethod `
  -Uri "https://dev.quantguard.org/api/config/site:title" `
  -Method PUT `
  -Headers $headers `
  -Body $body
```

读 preview KV：

```powershell
Invoke-RestMethod -Uri "https://dev.quantguard.org/api/config/site:title" -Method GET
```

也可以使用真实 curl：

```powershell
curl.exe -i -X PUT "https://dev.quantguard.org/api/config/site:title" `
  -H "Authorization: Bearer 你的真实_PREVIEW_ADMIN_TOKEN" `
  -H "Content-Type: application/json" `
  --data "{`"value`":`"QuantGuard Preview`"}"
```

注意：

```text
PowerShell 里 curl 可能是 Invoke-WebRequest alias。
如要使用真正 curl，请写 curl.exe。
```

---

## 5.8 部署 production

```powershell
npm run typecheck
npm run migrate:production
npm run deploy:production
```

验证：

```powershell
Invoke-RestMethod -Uri "https://quantguard.org/api/health" -Method GET
Invoke-RestMethod -Uri "https://quantguard.org/api/version" -Method GET
```

---

## 5.9 GitHub SSH 配置

```powershell
ssh-keygen -t ed25519 -C "你的GitHub邮箱"

Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

复制公钥到：

```text
GitHub
→ Settings
→ SSH and GPG keys
→ New SSH key
```

测试：

```powershell
ssh -T git@github.com
```

---

## 5.10 Git 推送

```powershell
git status
git add .
git commit -m "init cloudflare ci demo"

git branch -M dev
git remote add origin git@github.com:YuliangLuo/cloudflare-ci-demo.git
git push -u origin dev

git checkout -b main
git push -u origin main
git checkout dev
```

如果 main 非快进：

```powershell
git pull --rebase origin main
git push origin main
```

---

# 6. GitHub Actions 配置

## 6.1 GitHub Secrets

GitHub Secrets 不是 JSON，不是 `.env`，不是 `KEY=value` 文件。

它是一条 secret 一条记录：

```text
Name:   CLOUDFLARE_ACCOUNT_ID
Secret: 你的 Cloudflare Account ID
```

```text
Name:   CLOUDFLARE_API_TOKEN
Secret: 你的 Cloudflare API Token 原文
```

路径：

```text
GitHub Repository
→ Settings
→ Secrets and variables
→ Actions
→ Repository secrets
→ New repository secret
```

不要填成：

```text
CLOUDFLARE_ACCOUNT_ID=xxxx
CLOUDFLARE_API_TOKEN=xxxx
```

---

## 6.2 `.github/workflows/deploy-worker.yml`

```yaml
name: Deploy Cloudflare Worker

on:
  push:
    branches:
      - main
      - dev

  pull_request:
    branches:
      - main
      - dev

  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        type: choice
        options:
          - preview
          - production

permissions:
  contents: read

concurrency:
  group: cloudflare-worker-${{ github.ref }}
  cancel-in-progress: false

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Test
        run: npm test

      - name: Build
        run: npm run build

  deploy-preview:
    name: Deploy Preview
    runs-on: ubuntu-latest
    needs: test

    if: |
      github.ref == 'refs/heads/dev' ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'preview')

    environment: preview

    env:
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Apply D1 migrations to preview
        run: npx wrangler d1 migrations apply DB --remote --env preview

      - name: Deploy Worker to preview
        run: |
          npx wrangler deploy \
            --env preview \
            --minify \
            --tag "${GITHUB_SHA}" \
            --message "preview deploy ${GITHUB_SHA}"

      - name: Smoke test preview
        run: |
          curl -fsS https://dev.quantguard.org/api/health
          curl -fsS https://dev.quantguard.org/api/version

  deploy-production:
    name: Deploy Production
    runs-on: ubuntu-latest
    needs: test

    if: |
      github.ref == 'refs/heads/main' ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production')

    environment: production

    env:
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Apply D1 migrations to production
        run: npx wrangler d1 migrations apply DB --remote --env production

      - name: Deploy Worker to production
        run: |
          npx wrangler deploy \
            --env production \
            --minify \
            --tag "${GITHUB_SHA}" \
            --message "production deploy ${GITHUB_SHA}"

      - name: Smoke test production
        run: |
          curl -fsS https://quantguard.org/api/health
          curl -fsS https://quantguard.org/api/version
```

---

# 7. DNS 和 Route 检查

## 7.1 DNS 记录

Cloudflare DNS 至少需要：

```text
A     @      192.0.2.1    Proxied
A     dev    192.0.2.1    Proxied
```

如果使用真实源站 IP，可以替换 `192.0.2.1`。

如果出现：

```text
curl: (6) Could not resolve host: quantguard.org
```

Linux / WSL 检查：

```bash
dig @1.1.1.1 quantguard.org A
dig @1.1.1.1 dev.quantguard.org A
```

PowerShell 检查：

```powershell
nslookup quantguard.org 1.1.1.1
nslookup dev.quantguard.org 1.1.1.1
```

---

## 7.2 Route 冲突

如果出现：

```text
The route with pattern "dev.quantguard.org/api/*" is already associated with another worker called "week2-workers-api".
```

说明旧 Worker 占用了 route。

处理：

```text
Cloudflare Dashboard
→ Workers & Pages
→ week2-workers-api
→ Settings
→ Domains & Routes
→ 删除 dev.quantguard.org/api/*
```

然后重新部署：

```bash
npx wrangler deploy --env preview
```

不要优先删除旧 Worker。只删旧 route 更安全。

同时要去旧项目 `wrangler.jsonc` 或 `wrangler.toml` 删除：

```jsonc
"routes": [
  {
    "pattern": "dev.quantguard.org/api/*",
    "zone_name": "quantguard.org"
  }
]
```

否则以后重新部署旧项目，它会再次抢回 route。

---

# 8. 验收命令

## 8.1 Linux / WSL

```bash
curl -i https://dev.quantguard.org/api/health
curl -i https://dev.quantguard.org/api/version

curl -i -X PUT "https://dev.quantguard.org/api/config/site:title" \
  -H "Authorization: Bearer 你的真实_PREVIEW_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"value":"QuantGuard Preview"}'

curl -i "https://dev.quantguard.org/api/config/site:title"

curl -i https://quantguard.org/api/health
curl -i https://quantguard.org/api/version
```

---

## 8.2 PowerShell

```powershell
Invoke-RestMethod -Uri "https://dev.quantguard.org/api/health" -Method GET
Invoke-RestMethod -Uri "https://dev.quantguard.org/api/version" -Method GET

$headers = @{
  "Authorization" = "Bearer 你的真实_PREVIEW_ADMIN_TOKEN"
  "Content-Type"  = "application/json"
}

$body = @{
  value = "QuantGuard Preview"
} | ConvertTo-Json

Invoke-RestMethod `
  -Uri "https://dev.quantguard.org/api/config/site:title" `
  -Method PUT `
  -Headers $headers `
  -Body $body

Invoke-RestMethod -Uri "https://dev.quantguard.org/api/config/site:title" -Method GET

Invoke-RestMethod -Uri "https://quantguard.org/api/health" -Method GET
Invoke-RestMethod -Uri "https://quantguard.org/api/version" -Method GET
```

---

## 8.3 正确返回示例

`/api/health`：

```json
{
  "ok": true,
  "service": "cloudflare-ci-demo",
  "env": "preview",
  "status": "healthy"
}
```

`/api/config/site:title`：

```json
{
  "ok": true,
  "key": "site:title",
  "value": "QuantGuard Preview"
}
```

---

# 9. 回滚

查看 production 部署状态：

```bash
npx wrangler deployments status --env production
```

查看版本：

```bash
npx wrangler versions list --env production
```

回滚：

```bash
npx wrangler rollback --env production
```

指定版本回滚：

```bash
npx wrangler rollback <VERSION_ID> --env production
```

注意：

```text
Worker rollback 不会回滚 D1/KV 数据。
如果 production migration 做了破坏性 schema 变更，旧 Worker 代码可能不兼容新数据库结构。
```

生产 migration 推荐：

```text
Expand → Deploy → Contract

1. 先新增字段/表/索引
2. 部署兼容新旧 schema 的代码
3. backfill 数据
4. 稳定后删除旧字段/旧逻辑
```

---

# 10. 常见错误表

| 错误 | 根因 | 修复 |
|---|---|---|
| `KV namespace ... is not valid` | 把 D1 UUID 填到了 KV `id` | `npx wrangler kv namespace list`，使用真正 KV ID |
| `service: week2-workers-api` | dev route 命中旧 Worker | 删除旧 Worker route，重新部署 preview |
| `Could not resolve host` | DNS 未生效或 WSL DNS 异常 | 检查 Cloudflare DNS / `dig @1.1.1.1` |
| `Permission denied (publickey)` | GitHub SSH key 未配置 | 生成 ed25519 key，添加到 GitHub |
| `non-fast-forward` | 本地 main 落后远端 main | `git pull --rebase origin main` |
| `Authorization failed` | `ADMIN_TOKEN` 用错环境 | preview 用 preview token，production 用 production token |
| PowerShell curl 参数异常 | `curl` 是 alias | 用 `curl.exe` 或 `Invoke-RestMethod` |
| route already associated | route 被旧 Worker 占用 | 删除旧 route 或换 path |
| GitHub Secrets 不生效 | Secret name 或 value 填错 | 一条 secret 一条记录，只填值，不写 `KEY=value` |

---

# 11. 知识重点难点

## 11.1 Cloudflare Worker 的工程化核心

```text
开发阶段：wrangler dev
测试环境：wrangler deploy --env preview
正式环境：wrangler deploy --env production
自动部署：GitHub Actions 调 wrangler deploy
```

关键是把本地、preview、production 三套资源隔离。

---

## 11.2 资源绑定设计

推荐固定代码接口：

```ts
env.DB
env.APP_CONFIG
```

环境差异放到 `wrangler.jsonc`：

```text
preview.DB     → cf_ci_demo_preview
production.DB  → cf_ci_demo_prod

preview.APP_CONFIG     → cf_ci_demo_app_config_preview
production.APP_CONFIG  → cf_ci_demo_app_config_prod
```

这样代码不需要根据环境写 if/else。

---

## 11.3 D1 migration 风险

D1 migration 是数据库结构变更，不等同于 Worker 代码部署。

风险点：

```text
Worker 可以 rollback
D1 schema 不会自动 rollback
KV/R2 数据也不会自动 rollback
```

因此生产环境要避免直接：

```sql
DROP TABLE ...
DROP COLUMN ...
```

---

## 11.4 Route 与 DNS 是线上链路关键

部署成功不代表域名可访问。

必须同时满足：

```text
DNS record 存在
DNS record 是 Proxied
route 没有被旧 Worker 占用
wrangler.jsonc routes 配置正确
deploy --env preview / production 对应正确
```

验证是否命中新 Worker：

```bash
npx wrangler tail --env preview
curl -i https://dev.quantguard.org/api/health
```

---

## 11.5 CI/CD 的边界

GitHub Actions 做的是：

```text
拉代码
安装依赖
类型检查
运行测试
执行 migration
部署 Worker
执行 smoke test
```

不要把生产 secret 明文写入仓库。

---

## 11.6 PowerShell 与 Bash 差异

Bash 多行续接：

```bash
curl -i -X PUT "https://dev.quantguard.org/api/config/site:title" \
  -H "Content-Type: application/json" \
  --data '{"value":"QuantGuard Preview"}'
```

PowerShell 多行续接：

```powershell
curl.exe -i -X PUT "https://dev.quantguard.org/api/config/site:title" `
  -H "Content-Type: application/json" `
  --data "{`"value`":`"QuantGuard Preview`"}"
```

建议 PowerShell 优先使用：

```powershell
Invoke-RestMethod
```

---

# 12. 最终完成标准

满足以下条件即完成 Project 6：

```text
1. npm run dev 本地可运行
2. /api/health 返回 200
3. /api/articles 能读取 D1
4. /api/config/site:title 能写入和读取 KV
5. npm run deploy:preview 成功
6. dev.quantguard.org/api/health 返回 cloudflare-ci-demo
7. npm run deploy:production 成功
8. quantguard.org/api/health 返回 cloudflare-ci-demo
9. GitHub dev 分支 push 后自动部署 preview
10. GitHub main 分支 push 后自动部署 production
11. GitHub Actions 中没有明文 Cloudflare Token
12. Worker 可以用 wrangler rollback 回滚
```
