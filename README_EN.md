# RunBin

[中文](README.md) | English

RunBin is an online code execution and sharing platform that supports running, compiling, and displaying results for multiple programming languages.

## ✨ Features

- 🚀 Online Code Execution - Run code in multiple programming languages
- 📝 Code Sharing - Share code snippets via unique IDs
- 🐳 Docker Isolation - Secure code execution using Docker containers
- 💾 Flexible Storage - Supports both in-memory and database storage
- 🔄 Async Processing - Worker processes handle code execution tasks asynchronously
- 📊 Performance Stats - Track execution time and memory usage
- 🌐 Web Interface - Modern frontend with code editor support

## 🏗️ Architecture

The project uses a frontend-backend separation architecture:

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│             │      │             │      │             │
│  Web Client │─────▶│  API Server │─────▶│  PostgreSQL │
│   (Vue 3)   │      │   (Gin)     │      │   Database  │
│             │      │             │      │             │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            │
                            ▼
                     ┌─────────────┐      ┌─────────────┐
                     │             │      │             │
                     │   Worker    │─────▶│   Docker    │
                     │   Service   │      │  Environment │
                     │             │      │             │
                     └─────────────┘      └─────────────┘
```

### Components

- **Web Client**: Modern frontend application built with Vue 3 + Vite
- **API Server**: RESTful API service built with Gin framework
- **Worker Service**: Background task processing service for executing user-submitted code
- **PostgreSQL**: Persistent storage for code and execution results
- **Docker**: Provides secure code execution environment

## 📋 Prerequisites

- Go 1.24.2 or higher
- Docker and Docker daemon
- PostgreSQL database (optional, for production)
- Node.js and npm (for frontend development)

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/dreamstarsky/runbin.git
cd runbin
```

### 2. Install Dependencies

```bash
# Install Go dependencies
go mod download

# Install frontend dependencies
cd web
npm install
cd ..
```

### 3. Setup Database (Optional)

If using database storage, create the database and run migrations:

```bash
# Create database
createdb runbin

# Run migrations
psql -d runbin -f migrations/0001_init_pastes.sql
psql -d runbin -f migrations/0002_create_queue_table.sql
psql -d runbin -f migrations/0003_add_compilelog_column.sql
```

### 4. Configure Services

Edit the configuration files:

#### API Service Configuration (`config/api.yaml`)

```yaml
app:
  env: "release" # release or debug
  port: 8080

storage:
  type: "database"  # memory or database
  database:
    dsn: "host=localhost port=5432 user=postgres password=password dbname=runbin sslmode=disable"
```

#### Worker Service Configuration (`config/worker.yaml`)

```yaml
storage:
  type: "database" 
  database:
    dsn: "host=localhost port=5432 user=postgres password=password dbname=runbin sslmode=disable"

limit:
  time: 10.0      # Max execution time (seconds)
  cpu: 1.0        # CPU limit
  memory: 512     # Memory limit (MB)
  size: 1024000   # Output size limit (bytes)

process: 1        # Number of worker processes

name: "default name"
  
compilerimage: "cpp_gcc-latest:latest"  # Compiler image
```

### 5. Start Services

#### Start API Server

```bash
go run cmd/api/main.go
```

The API server will run at `http://localhost:8080`.

#### Start Worker Service

```bash
go run cmd/worker/main.go
```

#### Start Frontend Development Server

```bash
cd web
npm run dev
```

The frontend will run at `http://localhost:5173`.

## 🐳 Docker Deployment

### Build Images

```bash
# Build API and Worker images
docker build -t runbin-api -f Dockerfile.api .
docker build -t runbin-worker -f Dockerfile.worker .

# Build frontend image
cd web
docker build -t runbin-web .
```

### Using Docker Compose

Create `docker-compose.yml`:

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

Start all services:

```bash
docker-compose up -d
```

## 📡 API Documentation

### Submit Code

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

Response:

```json
{
  "message": "Created",
  "paste_id": "uuid-string",
  "url": "/api/pastes/uuid-string"
}
```

### Get Code Result

```http
GET /api/pastes/:id
```

Response:

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

### Get Supported Languages

```http
GET /api/languages
```

Response:

```json
{
  "languages": ["c++20"]
}
```

## 🗂️ Project Structure

```
.
├── cmd/
│   ├── api/          # API service entry point
│   └── worker/       # Worker service entry point
├── config/           # Configuration files
│   ├── api.yaml
│   └── worker.yaml
├── internal/
│   ├── config/       # Configuration loading
│   ├── controller/   # Controller layer
│   ├── model/        # Data models
│   ├── repository/   # Data access layer
│   ├── router/       # Route configuration
│   └── worker/       # Worker task processing
├── migrations/       # Database migration files
├── web/              # Frontend application
│   ├── src/          # Source code
│   ├── public/       # Static assets
│   └── dist/         # Build output
├── workerEnv/        # Worker environment configuration
├── go.mod            # Go module definition
└── LICENSE           # Apache 2.0 License
```

## 🔧 Development Guide

### Adding New Language Support

1. Add a new task processor in Worker (refer to `internal/worker/RunCppTask.go`)
2. Prepare the corresponding Docker image
3. Configure the compiler image in `worker.yaml`
4. Update the supported language list in the controller

### Local Development

```bash
# Start API in development mode
go run cmd/api/main.go

# Start Worker in development mode
go run cmd/worker/main.go

# Start frontend with hot reload
cd web && npm run dev
```

### Build for Production

```bash
# Build API
go build -o bin/api cmd/api/main.go

# Build Worker
go build -o bin/worker cmd/worker/main.go

# Build frontend
cd web && npm run build
```

## 🔒 Security

- All code executes in Docker containers, isolated from the host
- Configured resource limits (CPU, memory, execution time)
- Output size limits to prevent malicious code
- CORS configuration to protect API access

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the Apache License 2.0. See the [LICENSE](LICENSE) file for details.

## 📞 Contact

For questions or suggestions, please contact us via GitHub Issues.

---

Made with ❤️ by dreamstarsky
