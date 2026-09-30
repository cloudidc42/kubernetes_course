# Part 03: Docker Fundamentals - ตอนที่ 1

## สารบัญ
- [Docker คืออะไร](#docker-คืออะไร)
- [Docker Architecture](#docker-architecture)
- [Docker Image คืออะไร](#docker-image-คืออะไร)
- [Docker Container คืออะไร](#docker-container-คืออะไร)
- [Docker Registry](#docker-registry)
- [Dockerfile - การสร้าง Image](#dockerfile---การสร้าง-image)
- [Docker Commands พื้นฐาน](#docker-commands-พื้นฐาน)
- [Workshop: สร้าง Node.js App และ Dockerize](#workshop-สร้าง-nodejs-app-และ-dockerize)

---

## Docker คืออะไร

**Docker** คือ Platform ที่ช่วยให้นักพัฒนาสามารถ Build, Ship, และ Run Applications ในรูปแบบ Container

ก่อนมี Docker:
```
นักพัฒนา: "โปรแกรมทำงานบน Local ของฉันนะ ทำไมบน Production ไม่ทำงาน?"
Ops Team: "เราก็ไม่รู้ Environment ไม่เหมือนกันเลย..."

ปัญหา:
- Development: macOS, Python 3.9, Node 18
- Production: CentOS 7, Python 3.6, Node 14
```

หลังมี Docker:
```
นักพัฒนา: "ทุกอย่างอยู่ใน Docker Image แล้ว Environment เดียวกัน 100%"

ทุก Environment ใช้ Image เดียวกัน!
```

### เปรียบเทียบ Container กับ VM

```
Virtual Machines:
┌─────────────────────────────────────────────────────────────┐
│                      Physical Hardware                       │
├─────────────────────────────────────────────────────────────┤
│                      Host OS (Linux)                         │
├─────────────────────────────────────────────────────────────┤
│                       Hypervisor                             │
├──────────────────────┬──────────────────────────────────────┤
│       VM 1           │           VM 2                        │
│  ┌─────────────┐     │     ┌─────────────┐                  │
│  │  Guest OS   │     │     │  Guest OS   │                  │
│  │  (1-2GB)    │     │     │  (1-2GB)    │                  │
│  ├─────────────┤     │     ├─────────────┤                  │
│  │    App A    │     │     │    App B    │                  │
│  └─────────────┘     │     └─────────────┘                  │
└──────────────────────┴──────────────────────────────────────┘

Containers:
┌─────────────────────────────────────────────────────────────┐
│                      Physical Hardware                       │
├─────────────────────────────────────────────────────────────┤
│                      Host OS (Linux)                         │
├─────────────────────────────────────────────────────────────┤
│                    Docker Engine                             │
├──────────────────────┬──────────────────────────────────────┤
│   Container 1        │        Container 2                   │
│  ┌─────────────┐     │     ┌─────────────┐                  │
│  │App A + Deps │     │     │App B + Deps │                  │
│  │  (10-50MB)  │     │     │  (10-50MB)  │                  │
│  └─────────────┘     │     └─────────────┘                  │
└──────────────────────┴──────────────────────────────────────┘

เปรียบเทียบ:
VM:        ใหญ่ (GB), เริ่มช้า (นาที), Isolated มาก
Container: เล็ก (MB), เริ่มเร็ว (วินาที), Isolated ดี
```

---

## Docker Architecture

```
Docker Architecture:
┌────────────────────────────────────────────────────────────────┐
│                         Docker Client                           │
│                                                                  │
│  docker build   docker pull   docker run   docker push         │
└──────────────────────────────┬─────────────────────────────────┘
                               │ REST API
                               ▼
┌────────────────────────────────────────────────────────────────┐
│                     Docker Host (dockerd)                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │                   Docker Daemon                         │    │
│  │                                                         │    │
│  │  ┌─────────────┐    ┌──────────────┐    ┌──────────┐  │    │
│  │  │   Images    │    │  Containers  │    │ Networks │  │    │
│  │  └─────────────┘    └──────────────┘    └──────────┘  │    │
│  │                                                         │    │
│  │  ┌─────────────────────────────────────────────────┐   │    │
│  │  │                  Volumes                         │   │    │
│  │  └─────────────────────────────────────────────────┘   │    │
│  └────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────┐
│                      Docker Registry                            │
│                                                                  │
│  Docker Hub    GitHub Container Registry    AWS ECR             │
│  GCR (Google)  Azure ACR    Private Registry                   │
└────────────────────────────────────────────────────────────────┘
```

### Components หลัก

**1. Docker Client (docker CLI)**
- Tool ที่ผู้ใช้ใช้งาน
- ส่ง Command ไปยัง Docker Daemon ผ่าน REST API

**2. Docker Daemon (dockerd)**
- Background Service ที่ทำงานบน Host
- จัดการ Images, Containers, Networks, Volumes
- รับคำสั่งจาก Docker Client

**3. Docker Registry**
- ที่จัดเก็บ Docker Images
- Docker Hub เป็น Public Registry
- สามารถสร้าง Private Registry ได้

---

## Docker Image คืออะไร

**Docker Image** คือ Template แบบ Read-only สำหรับสร้าง Container

ลองนึกภาพว่า Image เหมือนกับ "Class" ใน Object-oriented Programming
และ Container เหมือนกับ "Instance" ของ Class นั้น

### Docker Image Layers

Docker Image ประกอบด้วย Layers หลายชั้น:

```
┌──────────────────────────────────────────┐
│            My Node.js App Image          │
│                                          │
│  Layer 5: COPY app.js (100KB)  ← READ/WRITE (Container Layer)
│  ─────────────────────────────           │
│  Layer 4: RUN npm install (50MB)         │
│  ─────────────────────────────           │  READ-ONLY
│  Layer 3: COPY package.json (1KB)        │  (Image Layers)
│  ─────────────────────────────           │
│  Layer 2: RUN apt-get install (200MB)    │
│  ─────────────────────────────           │
│  Layer 1: node:18 Base Image (300MB)     │
└──────────────────────────────────────────┘

ข้อดีของ Layers:
- แชร์ Layers ระหว่าง Images ได้
- ถ้า Layer ไม่เปลี่ยน ไม่ต้อง Download ใหม่
- ประหยัด Disk Space และ Bandwidth
```

### Image Tags

```bash
# Format: [registry/][namespace/]image[:tag]

# Docker Hub Public Image
docker pull nginx              # = nginx:latest
docker pull nginx:1.25         # Specific version
docker pull nginx:alpine       # Alpine-based (เล็กกว่า)

# Docker Hub Private Image
docker pull myusername/myapp:v1.0

# GitHub Container Registry
docker pull ghcr.io/myorg/myapp:latest

# AWS ECR
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### Image Size Optimization

```bash
# ดูขนาด Image
docker images

# ตัวอย่าง:
# REPOSITORY    TAG       IMAGE ID       SIZE
# node          18        abc123         934MB   <- ใหญ่มาก
# node          18-slim   def456         245MB   <- เล็กกว่า
# node          18-alpine ghi789         107MB   <- เล็กที่สุด
```

---

## Docker Container คืออะไร

**Docker Container** คือ Instance ที่กำลัง Running ของ Docker Image

### Container Lifecycle

```
Container States:
                   docker create
Created ──────────────────────────► Created
                                         │
                                    docker start
                                         │
                                         ▼
Stopped ◄─────────────────────── Running
   │        docker stop/kill         │
   │                                 │ docker pause
   │         docker start            ▼
   └──────────────────────────── Paused
                                      │
                                 docker unpause
                                      │
                                      ▼
                                   Running
                                      │
                              Process exits/crash
                                      │
                                      ▼
                                   Exited
                                      │
                                 docker rm
                                      │
                                      ▼
                                   Deleted
```

### Container vs Image

```bash
# Image: Template (Read-only)
docker images
# REPOSITORY   TAG      IMAGE ID       SIZE
# nginx        latest   abc123         142MB

# Container: Running Instance (มี Writable Layer เพิ่ม)
docker ps -a
# CONTAINER ID   IMAGE   COMMAND   STATUS    NAMES
# def456         nginx   "..."     Running   my-nginx
# ghi789         nginx   "..."     Exited    old-nginx

# หนึ่ง Image สร้าง Container ได้หลายตัว
docker run -d --name nginx-1 nginx
docker run -d --name nginx-2 nginx
docker run -d --name nginx-3 nginx
```

---

## Docker Registry

**Docker Registry** คือที่เก็บ Docker Images

### ประเภทของ Registry

```
1. Docker Hub (hub.docker.com)
   - Public Registry อย่างเป็นทางการ
   - Free สำหรับ Public Images
   - Paid สำหรับ Private Images

2. GitHub Container Registry (ghcr.io)
   - ฟรีสำหรับ Public
   - Integrated กับ GitHub Actions

3. AWS ECR (Elastic Container Registry)
   - สำหรับ AWS Users
   - Integrated กับ EKS, ECS

4. GCR (Google Container Registry)
   - สำหรับ GCP Users
   - Integrated กับ GKE

5. Private Registry (Self-hosted)
   - ใช้ Harbor, GitLab Registry
   - ควบคุมได้เต็มที่
```

### การใช้งาน Registry

```bash
# Login ไปยัง Docker Hub
docker login

# Login ไปยัง AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Pull Image
docker pull nginx:latest

# Tag Image เพื่อ Push
docker tag myapp:latest myusername/myapp:v1.0

# Push Image
docker push myusername/myapp:v1.0

# ดู Images ที่มี
docker images
```

---

## Dockerfile - การสร้าง Image

**Dockerfile** คือไฟล์ text ที่มีชุดคำสั่งสำหรับสร้าง Docker Image

### Dockerfile Instructions

```dockerfile
# ทุก Dockerfile เริ่มจาก Base Image
FROM ubuntu:22.04

# Metadata
LABEL maintainer="developer@example.com"
LABEL version="1.0"

# Run commands ระหว่าง Build
RUN apt-get update && \
    apt-get install -y nodejs npm && \
    rm -rf /var/lib/apt/lists/*

# สร้าง Directory และตั้ง Working Directory
WORKDIR /app

# Copy files จาก Host ไปยัง Container
COPY package*.json ./
COPY . .

# Environment Variables
ENV NODE_ENV=production
ENV PORT=3000

# Run command ระหว่าง Build
RUN npm install --production

# Expose Port (เป็นแค่ Documentation)
EXPOSE 3000

# Default command เมื่อ Container เริ่ม
CMD ["node", "server.js"]
```

### Instruction แต่ละอย่างอธิบาย

| Instruction | ความหมาย | ตัวอย่าง |
|------------|---------|---------|
| FROM | Base Image | `FROM node:18-alpine` |
| RUN | Execute command ระหว่าง Build | `RUN npm install` |
| COPY | Copy files จาก Host | `COPY . .` |
| ADD | เหมือน COPY + รองรับ URL/tar | `ADD https://... /app` |
| WORKDIR | Set working directory | `WORKDIR /app` |
| ENV | Set environment variable | `ENV PORT=3000` |
| EXPOSE | Document port | `EXPOSE 8080` |
| CMD | Default command | `CMD ["node", "app.js"]` |
| ENTRYPOINT | Main command | `ENTRYPOINT ["node"]` |
| ARG | Build-time variable | `ARG VERSION=latest` |
| VOLUME | Mount point | `VOLUME ["/data"]` |
| USER | Set user | `USER node` |

### ความแตกต่างระหว่าง CMD และ ENTRYPOINT

```dockerfile
# ตัวอย่าง 1: ใช้ CMD อย่างเดียว
CMD ["node", "app.js"]
# docker run myimage              → node app.js
# docker run myimage python app.py → python app.py (Override ได้)

# ตัวอย่าง 2: ใช้ ENTRYPOINT อย่างเดียว
ENTRYPOINT ["node"]
# docker run myimage              → node
# docker run myimage app.js       → node app.js (Arguments เพิ่มต่อท้าย)

# ตัวอย่าง 3: ใช้ทั้งคู่ (แนะนำ)
ENTRYPOINT ["node"]
CMD ["app.js"]
# docker run myimage              → node app.js
# docker run myimage server.js    → node server.js (เฉพาะ CMD Override)
```

### .dockerignore

เหมือน `.gitignore` สำหรับ Docker:

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
*.md
.env
dist
coverage
.DS_Store
```

ทำไมสำคัญ:
```bash
# ไม่มี .dockerignore
docker build -t myapp .
# Sending build context: 500MB  ← ช้ามาก (node_modules ถูกส่งไปด้วย)

# มี .dockerignore
docker build -t myapp .
# Sending build context: 2MB    ← เร็วกว่ามาก
```

---

## Docker Commands พื้นฐาน

### การจัดการ Images

```bash
# Pull Image จาก Registry
docker pull ubuntu:22.04
docker pull nginx:latest

# ดู Images ที่มีอยู่
docker images
docker image ls

# ดูรายละเอียด Image
docker inspect nginx:latest

# ลบ Image
docker rmi nginx:latest
docker image rm nginx:latest

# ลบ Images ที่ไม่ใช้
docker image prune

# ลบ Images ทั้งหมดที่ไม่ได้ใช้
docker image prune -a

# Build Image จาก Dockerfile
docker build -t myapp:v1.0 .
docker build -t myapp:v1.0 -f Dockerfile.prod .

# Tag Image
docker tag myapp:v1.0 myapp:latest
docker tag myapp:v1.0 myregistry.com/myapp:v1.0

# Push Image ไปยัง Registry
docker push myregistry.com/myapp:v1.0
```

### การจัดการ Containers

```bash
# Run Container (Background)
docker run -d nginx

# Run Container พร้อมตั้งชื่อ
docker run -d --name my-nginx nginx

# Run Container พร้อม Port Mapping
docker run -d -p 8080:80 nginx
# Host port 8080 → Container port 80

# Run Container พร้อม Environment Variables
docker run -d \
  -e MYSQL_ROOT_PASSWORD=mypassword \
  -e MYSQL_DATABASE=mydb \
  mysql:8.0

# Run Container พร้อม Volume
docker run -d \
  -v /host/path:/container/path \
  nginx

# Run Container แบบ Interactive
docker run -it ubuntu:22.04 /bin/bash
# -i = Interactive
# -t = Pseudo-TTY

# Run Container แล้วลบเมื่อ Exit
docker run --rm ubuntu:22.04 echo "Hello"

# ดู Containers ที่กำลัง Running
docker ps

# ดู Containers ทั้งหมด (รวม Stopped)
docker ps -a
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"

# หยุด Container
docker stop my-nginx
docker stop $(docker ps -q)  # หยุดทุกตัว

# เริ่ม Container ที่หยุดอยู่
docker start my-nginx

# Restart Container
docker restart my-nginx

# ลบ Container
docker rm my-nginx
docker rm -f my-nginx  # Force (แม้กำลัง Running)

# ลบ Containers ที่หยุดอยู่ทั้งหมด
docker container prune
```

### การ Debug และ Monitor

```bash
# ดู Logs
docker logs my-nginx
docker logs -f my-nginx       # Follow (Real-time)
docker logs --tail 100 my-nginx  # แค่ 100 บรรทัดล่าสุด

# Execute Command ใน Container ที่กำลัง Running
docker exec -it my-nginx /bin/bash
docker exec -it my-nginx sh    # ถ้าไม่มี bash
docker exec my-nginx ls /etc

# Copy Files
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf  # Container → Host
docker cp ./my-config.conf my-nginx:/etc/nginx/        # Host → Container

# ดู Resource Usage
docker stats
docker stats my-nginx

# ดู Processes ใน Container
docker top my-nginx

# ดูรายละเอียด Container
docker inspect my-nginx
```

### Docker System Commands

```bash
# ดู System Info
docker info
docker version

# ดู Disk Usage
docker system df

# ทำความสะอาดทุกอย่าง (ระวัง!)
docker system prune
docker system prune -a  # รวม Images ด้วย

# Network
docker network ls
docker network create mynetwork
docker network inspect mynetwork

# Volume
docker volume ls
docker volume create myvolume
docker volume inspect myvolume
docker volume prune
```

---

## Workshop: สร้าง Node.js App และ Dockerize

ใน Workshop นี้เราจะสร้าง Node.js Web Application และทำให้มัน Run ใน Docker Container

### ขั้นตอน 1: สร้าง Node.js Application

```bash
# สร้าง Directory สำหรับ Project
mkdir docker-node-app
cd docker-node-app
```

สร้างไฟล์ `package.json`:

```json
{
  "name": "docker-node-app",
  "version": "1.0.0",
  "description": "Simple Node.js app for Docker workshop",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

สร้างไฟล์ `server.js`:

```javascript
const express = require('express');
const os = require('os');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());

// Routes
app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker!',
    hostname: os.hostname(),
    platform: os.platform(),
    version: process.env.APP_VERSION || '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', uptime: process.uptime() });
});

app.get('/info', (req, res) => {
  res.json({
    node_version: process.version,
    memory: process.memoryUsage(),
    cpu: os.cpus().length + ' cores'
  });
});

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Internal Server Error' });
});

app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
  console.log(`📦 Container ID: ${os.hostname()}`);
  console.log(`🔧 Node.js: ${process.version}`);
});
```

### ขั้นตอน 2: สร้าง .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
*.md
.env
coverage
.nyc_output
```

### ขั้นตอน 3: สร้าง Dockerfile

```dockerfile
# Dockerfile

# ขั้นตอน 1: เลือก Base Image
# ใช้ Node.js 18 บน Alpine Linux (เล็กและเร็ว)
FROM node:18-alpine

# ขั้นตอน 2: ตั้ง Working Directory
WORKDIR /app

# ขั้นตอน 3: Copy Package files ก่อน
# (ทำแบบนี้เพื่อ Cache Layer ของ npm install)
COPY package*.json ./

# ขั้นตอน 4: ติดตั้ง Dependencies
RUN npm ci --only=production

# ขั้นตอน 5: Copy Source Code
COPY . .

# ขั้นตอน 6: ตั้ง Environment Variables
ENV NODE_ENV=production
ENV PORT=3000

# ขั้นตอน 7: Expose Port
EXPOSE 3000

# ขั้นตอน 8: สร้าง Non-root User (Security)
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodeuser -u 1001
USER nodeuser

# ขั้นตอน 9: ตั้ง Command เริ่มต้น
CMD ["node", "server.js"]
```

### ขั้นตอน 4: Build Docker Image

```bash
# Build Image
docker build -t docker-node-app:v1.0 .

# ดู Output:
# Sending build context to Docker daemon  5.12kB
# Step 1/9 : FROM node:18-alpine
# ---> abc123def456
# Step 2/9 : WORKDIR /app
# ---> Running in 789ghi
# ...
# Successfully built jkl012
# Successfully tagged docker-node-app:v1.0

# ตรวจสอบ Image
docker images docker-node-app
```

### ขั้นตอน 5: Run Container

```bash
# Run Container
docker run -d \
  --name my-node-app \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  docker-node-app:v1.0

# ตรวจสอบ
docker ps
docker logs my-node-app

# ทดสอบ
curl http://localhost:3000/
# หรือเปิด Browser ไปที่ http://localhost:3000

# ตัวอย่าง Response:
# {
#   "message": "Hello from Docker!",
#   "hostname": "a1b2c3d4e5f6",   ← Container ID
#   "platform": "linux",
#   "version": "1.0.0",
#   "timestamp": "2024-01-15T10:30:00.000Z"
# }
```

### ขั้นตอน 6: ทดลอง Container Features

```bash
# ดู Logs แบบ Real-time
docker logs -f my-node-app

# ทดสอบ Endpoints ต่างๆ
curl http://localhost:3000/health
curl http://localhost:3000/info

# เข้าไปใน Container
docker exec -it my-node-app sh

# ใน Container:
ls                   # ดู Files
cat package.json     # ดู Package.json
ps aux               # ดู Processes
exit                 # ออกจาก Container

# ดู Resource Usage
docker stats my-node-app

# ดู Container Details
docker inspect my-node-app
```

### ขั้นตอน 7: Push ไปยัง Docker Hub

```bash
# Login ไปยัง Docker Hub
docker login

# Tag Image
docker tag docker-node-app:v1.0 yourusername/docker-node-app:v1.0
docker tag docker-node-app:v1.0 yourusername/docker-node-app:latest

# Push
docker push yourusername/docker-node-app:v1.0
docker push yourusername/docker-node-app:latest

# ทดสอบ Pull จาก Docker Hub
docker rmi docker-node-app:v1.0
docker pull yourusername/docker-node-app:v1.0
docker run -d -p 3001:3000 yourusername/docker-node-app:v1.0
```

### ขั้นตอน 8: ดู Image Layers

```bash
# ดู History ของ Image (แต่ละ Layer)
docker history docker-node-app:v1.0

# ตัวอย่าง Output:
# IMAGE          CREATED        SIZE       COMMENT
# abc123         2 minutes ago  1.82kB     CMD ["node" "server.js"]
# def456         2 minutes ago  0B         USER nodeuser
# ...
# ghi789         2 minutes ago  9.85MB     RUN npm ci --only=production
# jkl012         2 minutes ago  2.05kB     COPY package*.json ./
# mno345         ...            270MB      node:18-alpine
```

### ขั้นตอน 9: ทำความสะอาด

```bash
# หยุดและลบ Container
docker stop my-node-app
docker rm my-node-app

# ลบ Images
docker rmi docker-node-app:v1.0
docker rmi yourusername/docker-node-app:v1.0

# ลบทุกอย่างที่ไม่ใช้
docker system prune
```

---

## ความเข้าใจ Layer Caching

สิ่งสำคัญที่ต้องเข้าใจเพื่อ Optimize Build Time:

```dockerfile
# ❌ BAD: npm install ทุกครั้งที่ source code เปลี่ยน
FROM node:18-alpine
WORKDIR /app
COPY . .              # ถ้า .js ไฟล์เปลี่ยน → Cache Miss
RUN npm install       # ต้อง npm install ใหม่ทุกครั้ง

# ✅ GOOD: npm install เฉพาะตอน package.json เปลี่ยน
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./  # Copy package.json ก่อน
RUN npm install        # Cache ตรงนี้ถ้า package.json ไม่เปลี่ยน
COPY . .               # Copy source code ทีหลัง
```

**ผลลัพธ์:**
```
การ Build ครั้งแรก:
Step 1: FROM node:18-alpine          → Download (ช้า)
Step 2: COPY package*.json ./        → Done
Step 3: RUN npm install              → Install (ช้า)
Step 4: COPY . .                     → Done
Total: ~3 นาที

การ Build ครั้งที่ 2 (เปลี่ยนแค่ server.js):
Step 1: FROM node:18-alpine          → Using cache ✓
Step 2: COPY package*.json ./        → Using cache ✓
Step 3: RUN npm install              → Using cache ✓ (เร็วมาก!)
Step 4: COPY . .                     → Done (เฉพาะตรงนี้ Rebuild)
Total: ~10 วินาที
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Docker** คือ Platform สำหรับ Container ที่แก้ปัญหา "Works on my machine"
2. **Docker Image** คือ Template แบบ Layered Read-only
3. **Docker Container** คือ Running Instance ของ Image
4. **Docker Registry** คือที่เก็บ Images
5. **Dockerfile** คือชุดคำสั่งสร้าง Image
6. **Layer Caching** ช่วยลด Build Time

ใน Part ต่อไปจะเรียนเรื่อง **Docker Networking, Volumes, Multi-stage Build**

---

## แบบฝึกหัด

1. สร้าง Python Flask Application และ Dockerize
2. ทดลองเปลี่ยน Base Image จาก `node:18` เป็น `node:18-alpine` ดูขนาดที่ต่างกัน
3. เพิ่ม Health Check ใน Dockerfile
4. ลอง Push Image ไปยัง Docker Hub

## คำถามทบทวน

1. Docker กับ Virtual Machine ต่างกันอย่างไร?
2. ทำไม Layer Caching ถึงสำคัญ?
3. .dockerignore ทำงานอย่างไร?
4. CMD กับ ENTRYPOINT ต่างกันอย่างไร?

---

*ต่อไป: [Part 04: Docker Fundamentals 2](./part-04-docker-fundamentals-2.md)*
