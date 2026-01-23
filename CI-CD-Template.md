# 通用 CI/CD 自动化部署模板

> 适用于任何 Go/Node.js/Python 项目的完整自动化部署方案

---

## 一、架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        自动化部署流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   本地开发                                                       │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │ 1. 代码修改                                               │  │
│   │ 2. git commit && git push origin master                  │  │
│   └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│   GitHub Actions CI/CD                                           │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │         ┌─────────┐ ┌─────────┐ ┌─────────┐              │  │
│   │         │  Build  │ │  Test   │ │  Lint   │              │  │
│   │         └────┬────┘ └────┬────┘ └────┬────┘              │  │
│   │              └───────────┼───────────┘                    │  │
│   │                          ▼                                │  │
│   │                   ┌─────────────┐                         │  │
│   │                   │Docker Build │                         │  │
│   │                   └──────┬──────┘                         │  │
│   │                          │                                │  │
│   │           ┌──────────────┼──────────────┐                 │  │
│   │           ▼                             ▼                 │  │
│   │   ┌─────────────┐               ┌─────────────┐          │  │
│   │   │   Deploy    │               │   Release   │          │  │
│   │   │ (master)    │               │  (v* tag)   │          │  │
│   │   └──────┬──────┘               └──────┬──────┘          │  │
│   └──────────┼──────────────────────────────┼────────────────┘  │
│              │                              │                    │
│              ▼                              ▼                    │
│   ┌─────────────────┐           ┌─────────────────┐             │
│   │   生产服务器     │           │ GitHub Releases │             │
│   │   Docker 部署   │           │   安装包下载     │             │
│   └─────────────────┘           └─────────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、文件结构

```
your-project/
├── .github/
│   └── workflows/
│       └── ci.yml              # CI/CD 主配置
├── docker/
│   └── Dockerfile.server       # 服务端 Dockerfile
├── docker-compose.yml          # Docker Compose 配置
├── .gitignore                  # Git 忽略规则
├── env.example                 # 环境变量模板
└── scripts/
    ├── deploy-local.ps1        # Windows 本地部署脚本
    └── server-init.sh          # 服务器初始化脚本
```

---

## 三、核心配置文件

### 3.1 GitHub Actions 工作流 (`.github/workflows/ci.yml`)

```yaml
name: CI/CD

on:
  push:
    branches: [main, master]
    tags: [v*]
  pull_request:
    branches: [main, master]

env:
  # 根据项目修改
  GO_VERSION: "1.24"        # 或 NODE_VERSION: "18" / PYTHON_VERSION: "3.11"
  REGISTRY: ghcr.io         # 可选：使用 GitHub Container Registry
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ==================== 构建测试 ====================
  build:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Go 项目
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Build
        run: go build -v ./...

      - name: Test
        run: go test -v ./... || true

      # Node.js 项目（替换上面的 Go 步骤）
      # - name: Set up Node.js
      #   uses: actions/setup-node@v4
      #   with:
      #     node-version: ${{ env.NODE_VERSION }}
      #     cache: npm
      #
      # - name: Install & Build
      #   run: npm ci && npm run build
      #
      # - name: Test
      #   run: npm test || true

  # ==================== Docker 构建 ====================
  docker:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: [build]
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/Dockerfile.server
          push: false
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ==================== 自动部署 ====================
  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    needs: [docker]
    if: github.ref == 'refs/heads/master' && github.event_name == 'push'
    environment: production
    steps:
      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SERVER_SSH_KEY }}" | base64 -d > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -p ${{ secrets.SERVER_PORT || 22 }} ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy via SSH
        run: |
          ssh -i ~/.ssh/deploy_key -p ${{ secrets.SERVER_PORT || 22 }} ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} << 'ENDSSH'
          set -e

          # 修改为你的项目路径
          APP_DIR="/opt/your-project"

          cd "$APP_DIR"

          echo "📥 Pulling latest code..."
          git fetch origin master
          git reset --hard origin/master

          echo "🔨 Building Docker images..."
          docker compose build --no-cache

          echo "🚀 Starting services..."
          docker compose up -d

          echo "🧹 Cleaning up..."
          docker image prune -f

          echo "✅ Health check..."
          sleep 10
          curl -f http://localhost:8080/health || exit 1

          echo "✅ Deployment completed!"
          ENDSSH

  # ==================== 发布 Release ====================
  release:
    name: Create Release
    runs-on: ubuntu-latest
    needs: [docker]
    if: startsWith(github.ref, 'refs/tags/v')
    permissions:
      contents: write  # 重要：允许创建 Release
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build release artifacts
        run: |
          # 根据项目修改构建命令
          mkdir -p dist
          # go build -o dist/app-linux-amd64 .
          # GOOS=windows go build -o dist/app-windows-amd64.exe .

      - name: Upload Release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*
          draft: false
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 3.2 Docker Compose (`docker-compose.yml`)

```yaml
services:
  # 数据库
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${DB_NAME:-app}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  # 缓存
  redis:
    image: redis:7
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  # 应用服务
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile.server
    ports:
      - "8080:8080"
    environment:
      # 数据库
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: ${DB_NAME:-app}
      DB_USER: postgres
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      # 缓存
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      # 安全
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  # 前端（可选）
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "80:80"
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 3.3 Dockerfile (`docker/Dockerfile.server`)

```dockerfile
# ============ Go 项目 ============
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache git ca-certificates tzdata

ENV GOTOOLCHAIN=local

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" \
    -o server .

# 运行镜像
FROM alpine:latest

RUN apk add --no-cache ca-certificates tzdata curl

ENV TZ=Asia/Shanghai

RUN adduser -D -H -s /sbin/nologin appuser

WORKDIR /app

COPY --from=builder /app/server /usr/local/bin/

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["server"]
```

### 3.4 环境变量模板 (`env.example`)

```bash
# 数据库
POSTGRES_PASSWORD=your_secure_password_here
DB_NAME=app

# 缓存
REDIS_PASSWORD=your_redis_password_here

# 安全
JWT_SECRET=your_jwt_secret_key_here

# 服务器（用于 TURN 等需要公网 IP 的服务）
PUBLIC_IP=your.server.ip.here
```

### 3.5 .gitignore

```gitignore
# 环境变量
.env
*.pem

# 编译产物
dist/
build/
*.exe
!frontend/public/*.exe  # 例外：允许特定目录

# 依赖
node_modules/
vendor/

# IDE
.vscode/
.idea/

# 日志
*.log

# 临时文件
*.tmp
*.bak
```

---

## 四、GitHub Secrets 配置

在 GitHub 仓库 → Settings → Secrets and variables → Actions 中添加：

| Secret 名称 | 说明 | 示例 |
|------------|------|------|
| `SERVER_HOST` | 服务器 IP 或域名 | `38.54.1.124` |
| `SERVER_USER` | SSH 用户名 | `root` |
| `SERVER_PORT` | SSH 端口（可选） | `22` |
| `SERVER_SSH_KEY` | SSH 私钥（Base64 编码） | 见下方 |

### 生成 SSH 密钥并添加到 Secrets

```bash
# 1. 生成 SSH 密钥对
ssh-keygen -t ed25519 -C "github-actions-deploy" -f deploy_key -N ""

# 2. 将公钥添加到服务器
cat deploy_key.pub >> ~/.ssh/authorized_keys

# 3. 将私钥 Base64 编码后添加到 GitHub Secrets
cat deploy_key | base64 -w 0
# 复制输出内容到 SERVER_SSH_KEY
```

---

## 五、服务器初始化

### 5.1 一键初始化脚本 (`scripts/server-init.sh`)

```bash
#!/bin/bash
# 服务器初始化脚本
# 用法: curl -sSL https://raw.githubusercontent.com/you/repo/master/scripts/server-init.sh | bash

set -e

APP_DIR="/opt/your-project"
REPO_URL="https://github.com/you/your-project.git"

echo "=========================================="
echo "  项目服务器初始化"
echo "=========================================="

# 1. 安装 Docker
if ! command -v docker &> /dev/null; then
    echo "[1/4] 安装 Docker..."
    curl -fsSL https://get.docker.com | sh
    systemctl enable docker
    systemctl start docker
else
    echo "[1/4] Docker 已安装"
fi

# 2. 安装 Docker Compose
if ! docker compose version &> /dev/null; then
    echo "[2/4] 安装 Docker Compose..."
    apt-get update && apt-get install -y docker-compose-plugin
else
    echo "[2/4] Docker Compose 已安装"
fi

# 3. 克隆代码
echo "[3/4] 克隆代码..."
if [ -d "$APP_DIR" ]; then
    cd "$APP_DIR"
    git fetch origin
    git reset --hard origin/master
else
    git clone "$REPO_URL" "$APP_DIR"
    cd "$APP_DIR"
fi

# 4. 配置环境变量
echo "[4/4] 配置环境变量..."
if [ ! -f ".env" ]; then
    cp env.example .env
    echo ""
    echo "⚠️  请编辑 .env 文件配置必要参数："
    echo "    nano $APP_DIR/.env"
    echo ""
    exit 0
fi

# 5. 启动服务
echo "[5/5] 启动服务..."
docker compose up -d --build

# 健康检查
echo ""
echo "等待服务启动..."
sleep 15

if curl -sf http://localhost:8080/health > /dev/null; then
    echo "=========================================="
    echo "  ✅ 初始化完成！"
    echo "=========================================="
else
    echo "⚠️  服务启动异常，请检查日志："
    echo "    docker compose logs"
fi
```

### 5.2 私有仓库配置 Deploy Key

如果仓库是私有的，需要配置 Deploy Key：

```bash
# 在服务器上执行
ssh-keygen -t ed25519 -C "deploy-key" -f ~/.ssh/github_deploy_key -N ""

# 显示公钥，添加到 GitHub 仓库的 Deploy Keys
cat ~/.ssh/github_deploy_key.pub

# 配置 SSH 使用此密钥
cat >> ~/.ssh/config << 'EOF'
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_deploy_key
    IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/config
```

---

## 六、本地开发脚本

### Windows (`scripts/deploy-local.ps1`)

```powershell
# 本地部署脚本
param(
    [switch]$Push,   # 是否推送
    [switch]$Build   # 是否本地构建测试
)

$ErrorActionPreference = "Stop"

Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  本地部署脚本" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan

# 1. 清理编译产物
Write-Host "`n[1/3] 清理编译产物..." -ForegroundColor Yellow
Get-ChildItem -Recurse -Include *.exe -Path . |
    Where-Object { $_.FullName -notmatch 'node_modules|\.git' } |
    Remove-Item -Force -ErrorAction SilentlyContinue

# 2. 本地构建测试
if ($Build) {
    Write-Host "`n[2/3] 本地构建测试..." -ForegroundColor Yellow
    go build -v . 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        Write-Host "  [ERROR] 构建失败" -ForegroundColor Red
        exit 1
    }
    Write-Host "  [OK] 构建成功" -ForegroundColor Green
    Remove-Item *.exe -Force -ErrorAction SilentlyContinue
}

# 3. 推送
if ($Push) {
    Write-Host "`n[3/3] 推送到远程..." -ForegroundColor Yellow
    git add -A
    $status = git status --porcelain
    if ($status) {
        git commit -m "deploy: $(Get-Date -Format 'yyyy-MM-dd HH:mm')"
    }
    git push origin master
    Write-Host "  [OK] 已推送" -ForegroundColor Green
}

Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "  完成！" -ForegroundColor Green
Write-Host "========================================" -ForegroundColor Cyan
```

---

## 七、常用命令

### 本地开发
```bash
# 推送代码（触发自动部署）
git add -A && git commit -m "feat: xxx" && git push origin master

# 创建 Release（触发发布工作流）
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

### 服务器运维
```bash
# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f app

# 重启服务
docker compose restart app

# 手动部署
cd /opt/your-project
git pull origin master
docker compose up -d --build
```

### GitHub CLI
```bash
# 查看 CI 状态
gh run list --limit 5

# 查看失败日志
gh run view <run-id> --log-failed

# 监控运行中的工作流
gh run watch <run-id>
```

---

## 八、故障排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| SSH 连接失败 | 密钥格式错误 | 确保私钥是 Base64 编码 |
| Release 403 错误 | 权限不足 | 添加 `permissions: contents: write` |
| Docker 构建失败 | 文件未提交 | 检查 `.gitignore` 是否忽略了必要文件 |
| 健康检查失败 | 端口未监听 | 检查服务启动日志 |
| 部署后无变化 | 缓存问题 | 使用 `docker compose build --no-cache` |

---

## 九、安全建议

1. **永远不要**将 `.env` 文件提交到 Git
2. **使用 Secrets** 存储敏感信息，不要硬编码
3. **限制 SSH 密钥权限**，只用于部署
4. **启用 Branch Protection**，保护 master 分支
5. **定期轮换**密钥和密码

---

*模板版本: 1.0.0*
*最后更新: 2026-01-23*
