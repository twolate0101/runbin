# RunBin

RunBin 是一个在线代码执行和分享平台，支持多种编程语言的代码运行、编译和结果展示。

## ✨ 功能特性

- 🚀 在线代码执行 - 支持多种编程语言的代码运行
- 📝 代码分享 - 通过唯一 ID 分享代码片段
- 🐳 Docker 隔离 - 使用 Docker 容器保证代码执行安全
- 💾 灵活存储 - 支持内存和数据库两种存储方式
- 🔄 异步处理 - Worker 进程异步处理代码执行任务
- 📊 性能统计 - 记录执行时间和内存使用情况
- 🌐 Web 界面 - 现代化的前端界面，支持代码编辑器

## 🏗️ 架构

项目采用前后端分离架构：

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│             │      │             │      │             │
│  Web 前端   │─────▶│  API 服务   │─────▶│  PostgreSQL │
│   (Vue 3)   │      │   (Gin)     │      │   数据库    │
│             │      │             │      │             │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            │
                            ▼
                     ┌─────────────┐      ┌─────────────┐
                     │             │      │             │
                     │   Worker    │─────▶│   Docker    │
                     │   服务      │      │   运行环境   │
                     │             │      │             │
                     └─────────────┘      └─────────────┘
```

### 组件说明

- **Web 前端**: 基于 Vue 3 + Vite 的现代化前端应用
- **API 服务**: 使用 Gin 框架构建的 RESTful API 服务
- **Worker 服务**: 后台任务处理服务，负责执行用户提交的代码
- **PostgreSQL**: 持久化存储代码和执行结果
- **Docker**: 提供安全的代码执行环境

## 📋 前置要求

- Go 1.24.2 或更高版本
- Docker 和 Docker daemon
- PostgreSQL 数据库（可选，用于生产环境）
- Node.js 和 npm（用于前端开发）

## 🚀 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/dreamstarsky/runbin.git
cd runbin
```

### 2. 安装依赖

```bash
# 安装 Go 依赖
go mod download

# 安装前端依赖
cd web
npm install
cd ..
```

### 3. 配置数据库（可选）

如果使用数据库存储，需要先创建数据库并运行迁移：

```bash
# 创建数据库
createdb runbin

# 运行迁移
psql -d runbin -f migrations/0001_init_pastes.sql
psql -d runbin -f migrations/0002_create_queue_table.sql
psql -d runbin -f migrations/0003_add_compilelog_column.sql
```

### 4. 配置服务

编辑配置文件：

#### API 服务配置 (`config/api.yaml`)

```yaml
app:
  env: "release" # release or debug
  port: 8080

storage:
  type: "database"  # memory or database
  database:
    dsn: "host=localhost port=5432 user=postgres password=password dbname=runbin sslmode=disable"
```

#### Worker 服务配置 (`config/worker.yaml`)

```yaml
storage:
  type: "database" 
  database:
    dsn: "host=localhost port=5432 user=postgres password=password dbname=runbin sslmode=disable"

limit:
  time: 10.0      # 最大执行时间（秒）
  cpu: 1.0        # CPU 限制
  memory: 512     # 内存限制（MB）
  size: 1024000   # 输出大小限制（字节）

process: 1        # Worker 进程数

name: "default name"
  
compilerimage: "cpp_gcc-latest:latest"  # 编译器镜像
```

### 5. 启动服务

#### 启动 API 服务

```bash
go run cmd/api/main.go
```

API 服务将在 `http://localhost:8080` 上运行。

#### 启动 Worker 服务

```bash
go run cmd/worker/main.go
```

#### 启动前端开发服务器

```bash
cd web
npm run dev
```

前端将在 `http://localhost:5173` 上运行。

## 🐳 Docker 部署

### 构建镜像

```bash
# 构建 API 和 Worker 镜像
docker build -t runbin-api -f Dockerfile.api .
docker build -t runbin-worker -f Dockerfile.worker .

# 构建前端镜像
cd web
docker build -t runbin-web .
```

### 使用 Docker Compose

创建 `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: runbin
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  api:
    image: runbin-api
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    environment:
      DATABASE_DSN: "host=postgres port=5432 user=postgres password=password dbname=runbin sslmode=disable"

  worker:
    image: runbin-worker
    depends_on:
      - postgres
      - api
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      DATABASE_DSN: "host=postgres port=5432 user=postgres password=password dbname=runbin sslmode=disable"

  web:
    image: runbin-web
    ports:
      - "80:80"
    environment:
      BACKEND_URL: "http://localhost:8080"

volumes:
  postgres_data:
```

启动所有服务：

```bash
docker-compose up -d
```

## 📡 API 文档

### 提交代码

```http
POST /api/pastes
Content-Type: application/json

{
  "code": "your code here",
  "language": "c++20",
  "stdin": "input data",
  "run": true
}
```

响应：

```json
{
  "message": "Created",
  "paste_id": "uuid-string",
  "url": "/api/pastes/uuid-string"
}
```

### 获取代码结果

```http
GET /api/pastes/:id
```

响应：

```json
{
  "ID": "uuid-string",
  "code": "your code here",
  "language": "c++20",
  "stdin": "input data",
  "stdout": "program output",
  "stderr": "error output",
  "status": "completed",
  "compile_log": "compilation output",
  "execution_time_ms": 100,
  "memory_usage_kb": 1024,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:01Z",
  "backend": "worker-name"
}
```

### 获取支持的语言列表

```http
GET /api/languages
```

响应：

```json
{
  "languages": ["c++20"]
}
```

## 🗂️ 项目结构

```
.
├── cmd/
│   ├── api/          # API 服务入口
│   └── worker/       # Worker 服务入口
├── config/           # 配置文件
│   ├── api.yaml
│   └── worker.yaml
├── internal/
│   ├── config/       # 配置加载
│   ├── controller/   # 控制器层
│   ├── model/        # 数据模型
│   ├── repository/   # 数据访问层
│   ├── router/       # 路由配置
│   └── worker/       # Worker 任务处理
├── migrations/       # 数据库迁移文件
├── web/              # 前端应用
│   ├── src/          # 源代码
│   ├── public/       # 静态资源
│   └── dist/         # 构建输出
├── workerEnv/        # Worker 环境配置
├── go.mod            # Go 模块定义
└── LICENSE           # Apache 2.0 许可证
```

## 🔧 开发指南

### 添加新的编程语言支持

1. 在 Worker 中添加新的任务处理器（参考 `internal/worker/RunCppTask.go`）
2. 准备相应的 Docker 镜像
3. 在 `worker.yaml` 中配置编译器镜像
4. 在控制器中更新支持的语言列表

### 本地开发

```bash
# 启动开发模式 API
go run cmd/api/main.go

# 启动开发模式 Worker
go run cmd/worker/main.go

# 启动前端热重载
cd web && npm run dev
```

### 构建生产版本

```bash
# 构建 API
go build -o bin/api cmd/api/main.go

# 构建 Worker
go build -o bin/worker cmd/worker/main.go

# 构建前端
cd web && npm run build
```

## 🔒 安全性

- 所有代码在 Docker 容器中执行，与主机隔离
- 配置了资源限制（CPU、内存、执行时间）
- 支持输出大小限制，防止恶意代码
- CORS 配置保护 API 访问

## 🤝 贡献

欢迎贡献代码！请遵循以下步骤：

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

## 📄 许可证

本项目采用 Apache License 2.0 许可证。详见 [LICENSE](LICENSE) 文件。

## 📞 联系方式

如有问题或建议，请通过 GitHub Issues 联系我们。

---

Made with ❤️ by dreamstarsky
