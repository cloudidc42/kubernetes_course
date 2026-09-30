# Part 04: Docker Fundamentals - ตอนที่ 2

## สารบัญ
- [Docker Networking](#docker-networking)
- [Docker Volumes](#docker-volumes)
- [Docker Multi-stage Build](#docker-multi-stage-build)
- [Docker Best Practices](#docker-best-practices)
- [Workshop: Build Production-ready Image](#workshop-build-production-ready-image)

---

## Docker Networking

Docker Networking ช่วยให้ Container สื่อสารกันได้และกับ External World

### Network Drivers

```
Docker Network Types:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. bridge (default)                                        │
│     - สร้าง Virtual Network ใน Host                         │
│     - Container ใน Bridge เดียวกันคุยกันได้                 │
│     - NAT ไปยัง External Network                            │
│                                                             │
│  2. host                                                    │
│     - Container ใช้ Network ของ Host โดยตรง                │
│     - ไม่มี Isolation                                        │
│     - Performance ดีกว่า Bridge                             │
│                                                             │
│  3. none                                                    │
│     - ไม่มี Network                                         │
│     - ใช้สำหรับ Isolated Containers                         │
│                                                             │
│  4. overlay                                                 │
│     - สำหรับ Multi-host Docker Swarm                        │
│                                                             │
│  5. macvlan                                                 │
│     - Container มี MAC Address ของตัวเอง                   │
│     - เหมาะสำหรับ Legacy Applications                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Default Bridge Network

```bash
# ดู Networks ที่มีอยู่
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# abc123         bridge    bridge    local
# def456         host      host      local
# ghi789         none      null      local

# Run Containers แบบ Default (ใช้ bridge network)
docker run -d --name container1 nginx
docker run -d --name container2 nginx

# Container ใน default bridge คุยกันผ่าน IP (ไม่ใช่ชื่อ)
docker inspect container1 | grep IPAddress
# "IPAddress": "172.17.0.2"

# Container2 ต้องรู้ IP ของ Container1
docker exec container2 ping 172.17.0.2
```

**ปัญหา:** IP เปลี่ยนทุกครั้งที่ Restart

### Custom Bridge Network (แนะนำ)

```bash
# สร้าง Custom Network
docker network create myapp-network

# Run Containers ใน Custom Network
docker run -d --name db --network myapp-network mysql:8.0
docker run -d --name app --network myapp-network myapp:latest

# ใน Custom Network: ใช้ชื่อ Container เป็น DNS
docker exec app ping db      # Works!
docker exec app curl http://webserver:80  # Works!
```

### Network Diagram

```
Default Bridge Network:
┌─────────────────────────────────────────────┐
│                   Docker Host               │
│                                             │
│  ┌──────────┐    ┌──────────┐             │
│  │Container1│    │Container2│             │
│  │172.17.0.2│    │172.17.0.3│             │
│  └─────┬────┘    └────┬─────┘             │
│        │              │                    │
│  ┌─────▼──────────────▼─────┐             │
│  │   docker0 bridge (172.17.0.1)           │
│  └─────────────┬────────────┘             │
│                │ NAT                        │
│        External Network                    │
└─────────────────────────────────────────────┘

Custom Bridge Network:
┌─────────────────────────────────────────────┐
│                   Docker Host               │
│                                             │
│  ┌──────────┐    ┌──────────┐             │
│  │  webapp  │    │    db    │             │
│  │ (app1)   │    │ (mysql)  │             │
│  └─────┬────┘    └────┬─────┘             │
│        │              │                    │
│  ┌─────▼──────────────▼─────┐             │
│  │    myapp-network          │             │
│  │  webapp → db (DNS works!) │             │
│  └───────────────────────────┘             │
└─────────────────────────────────────────────┘
```

### Port Mapping

```bash
# -p hostPort:containerPort
docker run -d -p 8080:80 nginx
# Access: http://localhost:8080 → Container port 80

# Bind to specific IP
docker run -d -p 127.0.0.1:8080:80 nginx
# เฉพาะ localhost เท่านั้น

# Publish all exposed ports
docker run -d -P nginx
# Docker จะ Map ทุก EXPOSE port ไปยัง Random host port

# ดู Port Mapping
docker port my-nginx
# 80/tcp -> 0.0.0.0:8080
```

### Workshop: Docker Networking

```bash
# สร้าง Network
docker network create --driver bridge app-network

# รัน Database
docker run -d \
  --name mysql-db \
  --network app-network \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=testdb \
  mysql:8.0

# รัน App ที่เชื่อมต่อกับ DB
docker run -d \
  --name php-app \
  --network app-network \
  -p 8080:80 \
  -e DB_HOST=mysql-db \
  -e DB_PASSWORD=password \
  php:8-apache

# ทดสอบ Connection
docker exec php-app ping mysql-db -c 3

# ดู Network Details
docker network inspect app-network
```

---

## Docker Volumes

Docker Volumes คือกลไกสำหรับจัดเก็บข้อมูลที่ Container ต้องการ Persist

### ทำไมต้องใช้ Volume?

```
ปัญหา: Container เป็น Stateless โดย default
docker run mysql
# ข้อมูลใน MySQL อยู่ใน Container Layer
docker rm mysql
# ข้อมูลหาย!!!

แก้ด้วย Volumes:
docker run -v /data/mysql:/var/lib/mysql mysql
# ข้อมูลเก็บใน Host: /data/mysql
docker rm mysql
docker run -v /data/mysql:/var/lib/mysql mysql
# ข้อมูลยังอยู่!
```

### ประเภทของ Storage

```
1. Volumes (แนะนำ)
   - Managed โดย Docker
   - เก็บใน /var/lib/docker/volumes/
   - Portable และ Backup ง่าย

2. Bind Mounts
   - Map ไปยัง Host Directory โดยตรง
   - ใช้ Path เต็มของ Host
   - เหมาะสำหรับ Development

3. tmpfs Mounts
   - เก็บใน Memory
   - ไม่ Persist
   - ใช้สำหรับข้อมูล Sensitive ชั่วคราว
```

### Docker Volumes

```bash
# สร้าง Volume
docker volume create mydata

# ดู Volumes
docker volume ls

# ดูรายละเอียด
docker volume inspect mydata

# ใช้ Volume
docker run -d \
  --name my-mysql \
  -v mydata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=password \
  mysql:8.0

# ลบ Volume (ระวัง! ข้อมูลหาย)
docker volume rm mydata

# ลบ Volumes ที่ไม่ได้ใช้
docker volume prune
```

### Bind Mounts

```bash
# Bind Mount สำหรับ Development
docker run -d \
  --name dev-app \
  -v $(pwd):/app \        # ← Current directory → /app ใน Container
  -p 3000:3000 \
  node:18 \
  npm start

# เมื่อแก้ไข Code บน Host → เห็นผลใน Container ทันที (Hot reload)
```

### Volume ใน Dockerfile

```dockerfile
# ประกาศ Volume ใน Dockerfile
FROM mysql:8.0
VOLUME ["/var/lib/mysql"]   # บอก Docker ว่า directory นี้ควรใช้ Volume
```

### Backup และ Restore Volume

```bash
# Backup Volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mydata-backup.tar.gz /data

# Restore Volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mydata-backup.tar.gz -C /
```

### เปรียบเทียบ Volumes vs Bind Mounts

| Feature | Volumes | Bind Mounts |
|---------|---------|-------------|
| Management | Docker Managed | User Managed |
| Location | `/var/lib/docker/volumes/` | Any host path |
| Portability | สูง | ต่ำ (path-dependent) |
| Use case | Production | Development |
| Security | ดีกว่า | ต่ำกว่า |
| Performance | ดีกว่า (Linux) | ปานกลาง |

---

## Docker Multi-stage Build

Multi-stage Build ช่วยสร้าง Image ขนาดเล็กสำหรับ Production โดยแยก Build Process ออกจาก Runtime

### ปัญหาของ Single-stage Build

```dockerfile
# ❌ Single-stage Build (Image ใหญ่มาก)
FROM node:18

WORKDIR /app

# ติดตั้ง Build Tools
RUN apt-get install -y build-essential python3

COPY package*.json ./
RUN npm install  # รวม devDependencies (Jest, TypeScript, etc.)

COPY . .
RUN npm run build  # Compile TypeScript

EXPOSE 3000
CMD ["node", "dist/server.js"]

# Image Size: ~1.2GB (มี node_modules dev tools ทั้งหมด)
```

### Multi-stage Build

```dockerfile
# ✅ Multi-stage Build (Image เล็กมาก)

# Stage 1: Builder
FROM node:18 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci              # ติดตั้ง ALL dependencies (รวม dev)

COPY . .
RUN npm run build       # Compile TypeScript → JavaScript
RUN npm prune --production  # ลบ dev dependencies

# Stage 2: Production
FROM node:18-alpine AS production

WORKDIR /app

# Copy เฉพาะ Output จาก Builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

ENV NODE_ENV=production
EXPOSE 3000

CMD ["node", "dist/server.js"]

# Image Size: ~200MB (เฉพาะ Runtime ที่จำเป็น)
```

### Go Multi-stage Build (เล็กมาก!)

```dockerfile
# Go Multi-stage Build
# Stage 1: Build
FROM golang:1.21 AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# Stage 2: Production (minimal!)
FROM scratch AS production
# FROM scratch = Image ว่างเปล่า (0 bytes base!)

COPY --from=builder /app/server /server
EXPOSE 8080

CMD ["/server"]

# Image Size: ~10-20MB เท่านั้น!
```

### Python Multi-stage Build

```dockerfile
# Python Multi-stage Build
# Stage 1: Dependencies
FROM python:3.11 AS deps

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt \
    --target=/app/deps

# Stage 2: Test (Optional)
FROM deps AS test

COPY . .
RUN python -m pytest tests/

# Stage 3: Production
FROM python:3.11-slim AS production

WORKDIR /app

# Copy Dependencies
COPY --from=deps /app/deps /app/deps
ENV PYTHONPATH=/app/deps

# Copy Source
COPY app/ ./app/

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

### เลือก Stage ที่ต้องการ Build

```bash
# Build ทุก Stages จนถึง Production
docker build -t myapp:prod .

# Build แค่ Stage test
docker build --target test -t myapp:test .

# Build แค่ Stage builder
docker build --target builder -t myapp:builder .
```

### ตัวอย่างการลด Image Size

```
Image Size Comparison (Node.js App):
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  node:18             1.13GB  ███████████████████████████  │
│  Single-stage        1.20GB  ████████████████████████████ │
│  Multi-stage         200MB   █████                        │
│  Alpine Multi-stage  100MB   ██                           │
│  Distroless          80MB    ██                           │
│                                                            │
└────────────────────────────────────────────────────────────┘

ทำไม Image Size สำคัญ:
- Pull Time: Image 1GB ใช้เวลา Pull นานกว่า 100MB มาก
- Security: น้อย Layer = น้อย Attack Surface
- Deployment Speed: Push/Pull เร็วกว่า
- Storage Cost: ประหยัด Registry Storage
```

---

## Docker Best Practices

### 1. ใช้ Official Images

```dockerfile
# ✅ ใช้ Official Images
FROM node:18-alpine
FROM python:3.11-slim
FROM nginx:1.25-alpine

# ❌ หลีกเลี่ยง Unknown Images
FROM randomuser/node-app  # ไม่รู้ว่ามีอะไรอยู่บ้าง
```

### 2. Pin Version ของ Image

```dockerfile
# ❌ ใช้ latest (อันตราย!)
FROM node:latest   # ไม่รู้ว่าพรุ่งนี้จะเป็น Version อะไร

# ✅ Pin ไปยัง Specific Version
FROM node:18.19.0-alpine3.18

# หรือ Pin ด้วย Digest (ปลอดภัยที่สุด)
FROM node@sha256:abc123def456...
```

### 3. ลด Layer จำนวนน้อย

```dockerfile
# ❌ หลาย Layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN rm -rf /var/lib/apt/lists/*

# ✅ รวม Layer
RUN apt-get update && \
    apt-get install -y \
      curl \
      wget && \
    rm -rf /var/lib/apt/lists/*
```

### 4. ลบ Cache หลัง Install

```dockerfile
# ❌ ไม่ลบ Cache
RUN apt-get update
RUN apt-get install -y nodejs

# ✅ ลบ Cache ทันที
RUN apt-get update && \
    apt-get install -y nodejs && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*  # ← ลบ Cache ทันที

# สำหรับ npm
RUN npm install && \
    npm cache clean --force      # ← ลบ npm cache
```

### 5. ใช้ Non-root User

```dockerfile
# ❌ รันเป็น root (อันตราย!)
FROM node:18-alpine
WORKDIR /app
COPY . .
CMD ["node", "server.js"]  # ← รันเป็น root

# ✅ สร้างและใช้ Non-root User
FROM node:18-alpine

WORKDIR /app
COPY --chown=node:node . .  # ← Copy ด้วย Permission ของ node user

USER node  # ← Switch ไปใช้ node user (มีอยู่แล้วใน node image)

CMD ["node", "server.js"]
```

### 6. Immutable Image

```dockerfile
# ❌ Mutable Image (แก้ไขได้จากภายนอก)
FROM node:18-alpine
ENV LOG_LEVEL=debug    # ← ค่าแก้ได้จากภายนอก

# ✅ ใช้ ARG สำหรับ Build-time, ENV สำหรับ Runtime
ARG BUILD_VERSION=dev
ENV APP_VERSION=$BUILD_VERSION  # ← กำหนดตอน Build
```

### 7. Health Check

```dockerfile
# เพิ่ม Health Check
FROM node:18-alpine

WORKDIR /app
COPY . .
RUN npm ci --production

EXPOSE 3000

# Health Check
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=10s \
            --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

```bash
# ดู Health Status
docker ps
# CONTAINER ID  IMAGE  STATUS                     NAMES
# abc123        app    Up 2 hours (healthy)        my-app
# def456        app    Up 1 hour (unhealthy)       broken-app
```

### 8. Image Labels

```dockerfile
# เพิ่ม Metadata ด้วย Labels
FROM node:18-alpine

LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="My awesome app" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.source="https://github.com/myorg/myapp" \
      org.opencontainers.image.licenses="MIT" \
      maintainer="dev@example.com"
```

### 9. Secrets Management

```dockerfile
# ❌ อย่าเก็บ Secret ใน Dockerfile!
FROM ubuntu
ENV DB_PASSWORD=mypassword  # ← ใครก็ดูได้จาก docker inspect!
RUN echo $DB_PASSWORD > /app/config

# ✅ ใช้ Docker Secrets (Production)
FROM ubuntu
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password > /app/config

# ✅ ใช้ Environment Variables (ส่งตอน run)
FROM ubuntu
ENV DB_PASSWORD=""  # ← ว่างเปล่า, ส่งตอน run
```

```bash
# ส่ง Secret ตอน run (ไม่ Baked ลง Image)
docker run -e DB_PASSWORD=mypassword myapp
```

### 10. Security Scanning

```bash
# Scan Image ด้วย Docker Scout
docker scout cves myapp:latest

# หรือใช้ Trivy (Open Source)
trivy image myapp:latest

# ตัวอย่าง Output:
# myapp:latest (alpine 3.18.4)
# Total: 3 (CRITICAL: 0, HIGH: 1, MEDIUM: 2)
```

---

## Workshop: Build Production-ready Image

### เป้าหมาย

สร้าง Production-ready Docker Image สำหรับ Node.js API Application ที่:
- ขนาดเล็ก
- ปลอดภัย
- มี Health Check
- Non-root User
- Multi-stage Build

### ขั้นตอน 1: สร้าง TypeScript Node.js App

สร้าง Directory Structure:
```
production-app/
├── src/
│   ├── server.ts
│   └── routes/
│       └── index.ts
├── package.json
├── tsconfig.json
├── Dockerfile
├── .dockerignore
└── docker-compose.yml
```

**`package.json`:**
```json
{
  "name": "production-app",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/server.js",
    "dev": "ts-node src/server.ts",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.2",
    "helmet": "^7.1.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0",
    "typescript": "^5.3.2",
    "jest": "^29.7.0",
    "ts-node": "^10.9.2"
  }
}
```

**`tsconfig.json`:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**`src/server.ts`:**
```typescript
import express, { Request, Response, NextFunction } from 'express';
import helmet from 'helmet';
import os from 'os';

const app = express();
const PORT = parseInt(process.env.PORT || '3000', 10);

// Security Middleware
app.use(helmet());
app.use(express.json({ limit: '10mb' }));

// Request Logging
app.use((req: Request, res: Response, next: NextFunction) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
  next();
});

// Routes
app.get('/', (req: Request, res: Response) => {
  res.json({
    status: 'ok',
    message: 'Production-ready Docker App',
    container_id: os.hostname(),
    node_version: process.version,
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req: Request, res: Response) => {
  const health = {
    status: 'healthy',
    uptime: process.uptime(),
    memory: {
      used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024) + 'MB',
      total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024) + 'MB'
    },
    timestamp: new Date().toISOString()
  };
  res.status(200).json(health);
});

app.get('/metrics', (req: Request, res: Response) => {
  res.json({
    memory_usage_mb: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
    cpu_count: os.cpus().length,
    load_average: os.loadavg(),
    uptime_seconds: process.uptime()
  });
});

// 404 Handler
app.use((req: Request, res: Response) => {
  res.status(404).json({ error: 'Not Found', path: req.path });
});

// Error Handler
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error('Error:', err.message);
  res.status(500).json({ error: 'Internal Server Error' });
});

// Graceful Shutdown
process.on('SIGTERM', () => {
  console.log('Received SIGTERM, shutting down gracefully...');
  process.exit(0);
});

process.on('SIGINT', () => {
  console.log('Received SIGINT, shutting down gracefully...');
  process.exit(0);
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server started on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV}`);
});

export default app;
```

### ขั้นตอน 2: สร้าง Production Dockerfile

**`Dockerfile`:**
```dockerfile
# ========================================
# Stage 1: Development Dependencies
# ========================================
FROM node:20-alpine3.19 AS deps

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including devDependencies)
RUN npm ci && \
    npm cache clean --force

# ========================================
# Stage 2: Build
# ========================================
FROM node:20-alpine3.19 AS builder

WORKDIR /app

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy source
COPY tsconfig.json ./
COPY src/ ./src/

# Build TypeScript → JavaScript
RUN npx tsc --build

# ========================================
# Stage 3: Production Dependencies Only
# ========================================
FROM node:20-alpine3.19 AS prod-deps

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# ========================================
# Stage 4: Production Image (Final)
# ========================================
FROM node:20-alpine3.19 AS production

# Security: Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Metadata
LABEL org.opencontainers.image.title="Production App" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.description="Production-ready Node.js App"

WORKDIR /app

# Copy production dependencies
COPY --from=prod-deps /app/node_modules ./node_modules

# Copy compiled application
COPY --from=builder /app/dist ./dist

# Copy package.json (needed for npm scripts)
COPY package.json ./

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeapp -u 1001

# Change ownership
RUN chown -R nodeapp:nodejs /app

# Switch to non-root user
USER nodeapp

# Environment
ENV NODE_ENV=production \
    PORT=3000

# Expose port
EXPOSE 3000

# Health Check
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=15s \
            --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### ขั้นตอน 3: สร้าง .dockerignore

```
# .dockerignore
node_modules
npm-debug.log*
yarn-debug.log*
yarn-error.log*
dist
.git
.gitignore
*.md
.env
.env.*
coverage
.nyc_output
*.test.ts
*.spec.ts
Dockerfile*
docker-compose*
.dockerignore
```

### ขั้นตอน 4: Build และ Test

```bash
# Build Production Image
docker build \
  --target production \
  -t myapp:prod \
  .

# ดูขนาด Image
docker images myapp

# Build ทุก Stages แยกกัน
docker build --target deps -t myapp:deps .
docker build --target builder -t myapp:builder .
docker build --target prod-deps -t myapp:prod-deps .
docker build --target production -t myapp:production .

# เปรียบเทียบขนาด
docker images | grep myapp
```

### ขั้นตอน 5: Run Production Container

```bash
# Run Container
docker run -d \
  --name production-app \
  -p 3000:3000 \
  --restart unless-stopped \
  --memory="256m" \
  --cpus="0.5" \
  -e NODE_ENV=production \
  myapp:prod

# ตรวจสอบ
docker ps
# STATUS: Up 30 seconds (health: starting)
# รอสักครู่...
# STATUS: Up 2 minutes (healthy)

# ทดสอบ
curl http://localhost:3000/
curl http://localhost:3000/health
curl http://localhost:3000/metrics

# ดู Logs
docker logs production-app

# ดู Stats
docker stats production-app
```

### ขั้นตอน 6: Security Check

```bash
# ตรวจสอบว่ารันเป็น Non-root
docker exec production-app whoami
# nodeapp  ← ✅ ไม่ใช่ root

docker exec production-app id
# uid=1001(nodeapp) gid=1001(nodejs)

# ดู Environment ที่ตั้ง
docker exec production-app env | grep NODE_ENV
# NODE_ENV=production  ← ✅

# ลอง Security Scan ด้วย Trivy
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:prod
```

### ขั้นตอน 7: Optimize Image Size

```bash
# ดูขนาด Layer
docker history myapp:prod

# ใช้ dive เพื่อวิเคราะห์ Image
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:prod

# เปรียบเทียบกับ Alpine version
docker images | grep myapp
# REPOSITORY  TAG         SIZE
# myapp       prod        180MB   ← Multi-stage + Alpine
# myapp       fat         1.2GB   ← Single-stage + Full
```

### ขั้นตอน 8: Docker Compose สำหรับ Local Development

**`docker-compose.yml`:**
```yaml
version: '3.9'

services:
  # Production-like environment
  app:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
    networks:
      - appnet

  # Development environment (with hot reload)
  app-dev:
    build:
      context: .
      target: deps
    volumes:
      - ./src:/app/src    # ← Mount source for hot reload
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=development
    command: npm run dev
    networks:
      - appnet

networks:
  appnet:
    driver: bridge
```

```bash
# Run Production
docker-compose up -d app

# Run Development (with hot reload)
docker-compose up app-dev

# ดู Logs
docker-compose logs -f app

# ทำความสะอาด
docker-compose down
docker-compose down --volumes  # รวม volumes ด้วย
```

### สรุปผล Workshop

```
Image Comparison:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Simple Dockerfile:                                      │
│  - Size: 1.2GB                                           │
│  - Security: root user                                   │
│  - Layers: 10+                                           │
│                                                          │
│  Production Dockerfile (Multi-stage):                    │
│  - Size: ~180MB (85% reduction!)                         │
│  - Security: non-root user (nodeapp)                     │
│  - Health Check: ✅                                       │
│  - Signal Handling: dumb-init ✅                          │
│  - Graceful Shutdown: ✅                                  │
│  - Resource Limits: ✅                                    │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Docker Networking** - Bridge, Custom Networks, Port Mapping
2. **Docker Volumes** - Named Volumes, Bind Mounts, Backup/Restore
3. **Multi-stage Build** - ลด Image Size อย่างมาก
4. **Best Practices** - Security, Optimization, Health Check

---

## แบบฝึกหัด

1. สร้าง Docker Network สำหรับ 3-tier Application (Frontend, Backend, Database)
2. ทดลอง Backup และ Restore MySQL Volume
3. สร้าง Multi-stage Build สำหรับ Go Application
4. เพิ่ม Security Scanning ใน CI/CD Pipeline

## คำถามทบทวน

1. ทำไมถึงแนะนำให้ใช้ Custom Bridge Network แทน Default Bridge?
2. Multi-stage Build ช่วยอะไรบ้าง?
3. ทำไมต้องรัน Container เป็น Non-root User?
4. Health Check ทำงานอย่างไรและสำคัญอย่างไร?

---

*ต่อไป: [Part 05: Docker Compose](./part-05-docker-compose.md)*
