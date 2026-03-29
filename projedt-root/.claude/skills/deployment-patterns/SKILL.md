---
name: deployment-patterns
description: Webアプリケーションのためのデプロイメントワークフロー、CI/CDパイプラインパターン、Dockerコンテナ化、ヘルスチェック、ロールバック戦略、プロダクションレディネスチェックリスト。
origin: ECC
---

# デプロイメントパターン

プロダクションデプロイメントワークフローとCI/CDのベストプラクティス。

## 発動条件

- CI/CDパイプラインをセットアップするとき
- アプリケーションをDocker化するとき
- デプロイメント戦略を計画するとき（Blue-Green、カナリア、ローリング）
- ヘルスチェックとレディネスプローブを実装するとき
- プロダクションリリースを準備するとき
- 環境固有の設定を構成するとき

## デプロイメント戦略

### ローリングデプロイメント（デフォルト）

インスタンスを段階的に置換 — ロールアウト中に旧バージョンと新バージョンが同時に実行。

```
Instance 1: v1 → v2  (update first)
Instance 2: v1        (still running v1)
Instance 3: v1        (still running v1)

Instance 1: v2
Instance 2: v1 → v2  (update second)
Instance 3: v1

Instance 1: v2
Instance 2: v2
Instance 3: v1 → v2  (update last)
```

**メリット:** ゼロダウンタイム、段階的ロールアウト
**デメリット:** 2つのバージョンが同時に実行 — 後方互換性のある変更が必要
**使うとき:** 標準的なデプロイメント、後方互換性のある変更

### Blue-Greenデプロイメント

2つの同一環境を実行。トラフィックをアトミックに切り替え。

```
Blue  (v1) ← traffic
Green (v2)   idle, running new version

# After verification:
Blue  (v1)   idle (becomes standby)
Green (v2) ← traffic
```

**メリット:** 即座のロールバック（Blueに切り戻し）、クリーンな切り替え
**デメリット:** デプロイ中に2倍のインフラストラクチャが必要
**使うとき:** クリティカルなサービス、問題に対するゼロトレランス

### カナリアデプロイメント

まず新バージョンに少量のトラフィックをルーティング。

```
v1: 95% of traffic
v2:  5% of traffic  (canary)

# If metrics look good:
v1: 50% of traffic
v2: 50% of traffic

# Final:
v2: 100% of traffic
```

**メリット:** フルロールアウト前にリアルトラフィックで問題を検出
**デメリット:** トラフィック分割インフラ、モニタリングが必要
**使うとき:** 高トラフィックサービス、リスクの高い変更、フィーチャーフラグ

## Docker

### マルチステージDockerfile (Node.js)

```dockerfile
# Stage 1: Install dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production=false

# Stage 2: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
RUN npm prune --production

# Stage 3: Production image
FROM node:22-alpine AS runner
WORKDIR /app

RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001
USER appuser

COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

ENV NODE_ENV=production
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### マルチステージDockerfile (Go)

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM alpine:3.19 AS runner
RUN apk --no-cache add ca-certificates
RUN adduser -D -u 1001 appuser
USER appuser

COPY --from=builder /server /server

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["/server"]
```

### マルチステージDockerfile (Python/Django)

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

FROM python:3.12-slim AS runner
WORKDIR /app

RUN useradd -r -u 1001 appuser
USER appuser

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .

ENV PYTHONUNBUFFERED=1
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')" || exit 1
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### Dockerベストプラクティス

```
# GOOD practices
- Use specific version tags (node:22-alpine, not node:latest)
- Multi-stage builds to minimize image size
- Run as non-root user
- Copy dependency files first (layer caching)
- Use .dockerignore to exclude node_modules, .git, tests
- Add HEALTHCHECK instruction
- Set resource limits in docker-compose or k8s

# BAD practices
- Running as root
- Using :latest tags
- Copying entire repo in one COPY layer
- Installing dev dependencies in production image
- Storing secrets in image (use env vars or secrets manager)
```

## CI/CDパイプライン

### GitHub Actions（標準パイプライン）

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: coverage/

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Deploy to production
        run: |
          # Platform-specific deployment command
          # Railway: railway up
          # Vercel: vercel --prod
          # K8s: kubectl set image deployment/app app=ghcr.io/${{ github.repository }}:${{ github.sha }}
          echo "Deploying ${{ github.sha }}"
```

### パイプラインステージ

```
PR opened:
  lint → typecheck → unit tests → integration tests → preview deploy

Merged to main:
  lint → typecheck → unit tests → integration tests → build image → deploy staging → smoke tests → deploy production
```

## ヘルスチェック

### ヘルスチェックエンドポイント

```typescript
// Simple health check
app.get("/health", (req, res) => {
  res.status(200).json({ status: "ok" });
});

// Detailed health check (for internal monitoring)
app.get("/health/detailed", async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    externalApi: await checkExternalApi(),
  };

  const allHealthy = Object.values(checks).every(c => c.status === "ok");

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? "ok" : "degraded",
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || "unknown",
    uptime: process.uptime(),
    checks,
  });
});

async function checkDatabase(): Promise<HealthCheck> {
  try {
    await db.query("SELECT 1");
    return { status: "ok", latency_ms: 2 };
  } catch (err) {
    return { status: "error", message: "Database unreachable" };
  }
}
```

### Kubernetesプローブ

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 2

startupProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 30    # 30 * 5s = 150s max startup time
```

## 環境設定

### Twelve-Factor Appパターン

```bash
# All config via environment variables — never in code
DATABASE_URL=postgres://user:pass@host:5432/db
REDIS_URL=redis://host:6379/0
API_KEY=${API_KEY}           # injected by secrets manager
LOG_LEVEL=info
PORT=3000

# Environment-specific behavior
NODE_ENV=production          # or staging, development
APP_ENV=production           # explicit app environment
```

### 設定バリデーション

```typescript
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "staging", "production"]),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

// Validate at startup — fail fast if config is wrong
export const env = envSchema.parse(process.env);
```

## ロールバック戦略

### 即座のロールバック

```bash
# Docker/Kubernetes: point to previous image
kubectl rollout undo deployment/app

# Vercel: promote previous deployment
vercel rollback

# Railway: redeploy previous commit
railway up --commit <previous-sha>

# Database: rollback migration (if reversible)
npx prisma migrate resolve --rolled-back <migration-name>
```

### ロールバックチェックリスト

- [ ] 以前のイメージ/アーティファクトが利用可能でタグ付けされている
- [ ] データベースマイグレーションが後方互換性がある（破壊的変更なし）
- [ ] フィーチャーフラグでデプロイなしに新機能を無効化可能
- [ ] エラーレートスパイクのモニタリングアラートが設定されている
- [ ] プロダクションリリース前にステージングでロールバックをテスト済み

## プロダクションレディネスチェックリスト

プロダクションデプロイメント前に:

### アプリケーション
- [ ] すべてのテストが合格（ユニット、統合、E2E）
- [ ] コードや設定ファイルにハードコードされたシークレットがない
- [ ] エラーハンドリングがすべてのエッジケースをカバー
- [ ] ログが構造化（JSON）でPIIを含まない
- [ ] ヘルスチェックエンドポイントが意味のあるステータスを返す

### インフラストラクチャ
- [ ] Dockerイメージが再現可能にビルドされる（ピン留めされたバージョン）
- [ ] 環境変数が文書化され、起動時にバリデーションされる
- [ ] リソースリミットが設定されている（CPU、メモリ）
- [ ] 水平スケーリングが設定されている（最小/最大インスタンス）
- [ ] すべてのエンドポイントでSSL/TLSが有効

### モニタリング
- [ ] アプリケーションメトリクスがエクスポートされている（リクエストレート、レイテンシー、エラー）
- [ ] エラーレート > しきい値のアラートが設定されている
- [ ] ログ集約がセットアップされている（構造化ログ、検索可能）
- [ ] ヘルスエンドポイントの稼働監視

### セキュリティ
- [ ] 依存関係のCVEスキャン済み
- [ ] CORSが許可されたオリジンのみに設定
- [ ] パブリックエンドポイントにレートリミットが有効
- [ ] 認証と認可が検証済み
- [ ] セキュリティヘッダーが設定されている（CSP、HSTS、X-Frame-Options）

### オペレーション
- [ ] ロールバック計画が文書化されテスト済み
- [ ] データベースマイグレーションがプロダクションサイズのデータでテスト済み
- [ ] 一般的な障害シナリオのランブック
- [ ] オンコールローテーションとエスカレーションパスが定義されている
