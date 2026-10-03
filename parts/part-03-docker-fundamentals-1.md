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

---

## Docker Architecture แบบ Deep Dive

### ส่วนประกอบภายใน Docker

```
Docker Architecture (Detailed):

┌─────────────────────────────────────────────────────────────────────┐
│                         Docker Client                                │
│                                                                       │
│   $ docker build    $ docker pull    $ docker run                   │
│   $ docker push     $ docker ps      $ docker exec                  │
└────────────────────────────┬────────────────────────────────────────┘
                              │  REST API / Unix Socket
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Docker Daemon (dockerd)                        │
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │   Image Manager  │  │ Container Manager│  │  Volume Manager  │ │
│  │                  │  │                  │  │                  │ │
│  │  - Pull/Push     │  │  - Create/Start  │  │  - Create/Mount  │ │
│  │  - Build         │  │  - Stop/Delete   │  │  - Backup        │ │
│  │  - Tag           │  │  - Exec          │  │                  │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Network Manager                                   │  │
│  │  - bridge / host / overlay / macvlan / none                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    containerd (Container Runtime)                     │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     containerd-shim                            │  │
│  │                                                                │  │
│  │   Container Process 1     Container Process 2                 │  │
│  │   ┌──────────────────┐   ┌──────────────────┐               │  │
│  │   │  runc (OCI)      │   │  runc (OCI)      │               │  │
│  │   │  - Namespaces    │   │  - Namespaces    │               │  │
│  │   │  - Cgroups       │   │  - Cgroups       │               │  │
│  │   │  - Seccomp       │   │  - Seccomp       │               │  │
│  │   └──────────────────┘   └──────────────────┘               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Linux Kernel Features ที่ Docker ใช้

```
Docker ใช้ Linux Kernel Features:

1. Namespaces (Isolation):
   ┌────────────────────────────────────────────────────────────┐
   │  Namespace Type  │  Isolates                               │
   ├────────────────────────────────────────────────────────────┤
   │  pid             │  Process IDs (Container เห็น PID 1)   │
   │  net             │  Network interfaces, IP, ports          │
   │  ipc             │  IPC, message queues, semaphores        │
   │  mnt             │  Mount points, filesystems              │
   │  uts             │  Hostname, domain name                  │
   │  user            │  UIDs, GIDs (User namespace)           │
   │  cgroup          │  Control group root                     │
   └────────────────────────────────────────────────────────────┘

2. Control Groups (cgroups) - Resource Limiting:
   ┌────────────────────────────────────────────────────────────┐
   │  Resource        │  What It Controls                       │
   ├────────────────────────────────────────────────────────────┤
   │  cpu             │  CPU usage, scheduling                  │
   │  cpuset          │  Which CPU cores to use                 │
   │  memory          │  RAM + Swap limits                      │
   │  blkio           │  Block device I/O                       │
   │  net_cls         │  Network packet classification          │
   │  devices         │  Device access                          │
   └────────────────────────────────────────────────────────────┘

3. Union File System (OverlayFS):
   ┌────────────────────────────────────────────────────────────┐
   │           Container Writable Layer                          │
   ├────────────────────────────────────────────────────────────┤
   │         Image Layer 4 (App Code)         ← Read-only      │
   ├────────────────────────────────────────────────────────────┤
   │         Image Layer 3 (npm install)      ← Read-only      │
   ├────────────────────────────────────────────────────────────┤
   │         Image Layer 2 (WORKDIR)          ← Read-only      │
   ├────────────────────────────────────────────────────────────┤
   │         Image Layer 1 (node:18-alpine)   ← Read-only      │
   └────────────────────────────────────────────────────────────┘
```

### Docker Networking Deep Dive

```
Docker Network Types:

1. Bridge Network (Default):
   ┌─────────────────────────────────────────────────────────┐
   │                      Host Machine                        │
   │                                                          │
   │  eth0: 192.168.1.100                                    │
   │                                                          │
   │  ┌──────────────────────────────────────────────────┐  │
   │  │           docker0 (Bridge: 172.17.0.1)           │  │
   │  │                                                    │  │
   │  │  ┌──────────────┐    ┌──────────────┐            │  │
   │  │  │  Container 1 │    │  Container 2 │            │  │
   │  │  │  172.17.0.2  │    │  172.17.0.3  │            │  │
   │  │  └──────────────┘    └──────────────┘            │  │
   │  └──────────────────────────────────────────────────┘  │
   └─────────────────────────────────────────────────────────┘
   
   - Default network สำหรับ Standalone containers
   - Containers คุยกันได้ผ่าน Bridge
   - External access ผ่าน Port Mapping

2. Host Network:
   Container ใช้ Network ของ Host โดยตรง
   - ไม่มี Network Isolation
   - Performance ดีที่สุด (ไม่มี NAT overhead)
   - ใช้สำหรับ Network-intensive apps

3. Overlay Network (Docker Swarm/Multi-host):
   - Containers ข้าม Hosts คุยกันได้
   - ใช้ VXLAN Tunneling
   - Kubernetes ใช้ CNI Plugins แทน

4. None Network:
   - ไม่มี Network เลย
   - ใช้สำหรับ Batch Jobs ที่ไม่ต้องการ Network
```

---

## Dockerfile Instructions ทุก Instruction

### FROM - Base Image

```dockerfile
# Syntax ต่างๆ:
FROM <image>
FROM <image>:<tag>
FROM <image>@<digest>
FROM <image> AS <name>  # Multi-stage build

# ตัวอย่าง:
FROM ubuntu:22.04
FROM node:18-alpine
FROM python:3.12-slim
FROM scratch  # Empty base image สำหรับ Static binaries

# Multi-stage build:
FROM node:18-alpine AS builder
# Build stage...

FROM node:18-alpine AS runner
# Runtime stage...
```

### RUN - Execute Commands

```dockerfile
# Syntax:
RUN <command>              # Shell form (sh -c)
RUN ["executable", "arg1", "arg2"]  # Exec form

# Best Practices:
# ✓ รวม commands ด้วย && เพื่อลด Layers
RUN apt-get update && \
    apt-get install -y \
        curl \
        git \
        vim \
    && rm -rf /var/lib/apt/lists/*

# ✗ หลายบรรทัดแยกกัน = หลาย Layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# Cache Busting:
RUN apt-get update && apt-get install -y \
    curl=7.81.0-* \  # ← Pin version เพื่อ Reproducibility
    git=1:2.34.1-*

# Run as non-root:
RUN groupadd -r appuser && useradd -r -g appuser appuser
```

### COPY - Copy Files

```dockerfile
# Syntax:
COPY <src> <dest>
COPY ["<src>", "<dest>"]  # สำหรับ path ที่มี spaces

# Options:
COPY --chown=<user>:<group> <src> <dest>
COPY --from=<stage> <src> <dest>  # Multi-stage
COPY --chmod=<permissions> <src> <dest>

# ตัวอย่าง:
COPY package*.json ./         # Copy package files
COPY . .                       # Copy ทุกอย่าง (ระวัง! ใช้ .dockerignore)
COPY --chown=nodeuser:nodeuser . .  # Copy พร้อมกำหนด Owner
COPY --from=builder /app/dist ./dist  # Copy จาก build stage

# ความต่างจาก ADD:
# COPY: Copy local files เท่านั้น (แนะนำ)
# ADD:  Copy local files + Extract tar + Download URLs (หลีกเลี่ยง)
```

### ADD - Add Files (ใช้ COPY แทนถ้าไม่จำเป็น)

```dockerfile
# Syntax:
ADD <src> <dest>

# กรณีที่ควรใช้ ADD:
# 1. Extract tar.gz โดยอัตโนมัติ
ADD app.tar.gz /app/  # Extract อัตโนมัติ

# 2. Download จาก URL (ไม่แนะนำ - ใช้ curl/wget ใน RUN แทน)
ADD https://example.com/file.tar.gz /tmp/

# Best Practice: ใช้ COPY สำหรับ Local files เสมอ
```

### ENV - Environment Variables

```dockerfile
# Syntax:
ENV <key>=<value>
ENV <key> <value>  # Deprecated syntax

# ตัวอย่าง:
ENV NODE_ENV=production
ENV PORT=3000
ENV DB_HOST=localhost \
    DB_PORT=5432 \
    DB_NAME=myapp

# ใช้ใน Dockerfile:
ENV APP_DIR=/app
WORKDIR $APP_DIR
COPY . $APP_DIR

# Override ตอน run:
docker run -e NODE_ENV=development myapp

# ข้อควรระวัง:
# ENV จะ Persist ใน Image ทำให้เห็นใน docker inspect
# ห้ามใส่ Secrets ใน ENV!
```

### ARG - Build Arguments

```dockerfile
# Syntax:
ARG <name>[=<default>]

# ตัวอย่าง:
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

ARG APP_VERSION="1.0.0"
ARG BUILD_DATE
ARG GIT_COMMIT

# ใช้งาน:
ENV APP_VERSION=${APP_VERSION}
LABEL version=${APP_VERSION} \
      build-date=${BUILD_DATE} \
      git-commit=${GIT_COMMIT}

# Build ด้วย --build-arg:
docker build \
  --build-arg NODE_VERSION=20 \
  --build-arg APP_VERSION=2.0.0 \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  -t myapp:2.0.0 .

# ความต่างจาก ENV:
# ARG: มีผลแค่ตอน Build (ไม่ Persist ใน Image)
# ENV: Persist ใน Image และ Runtime
```

### EXPOSE - Document Ports

```dockerfile
# Syntax:
EXPOSE <port>[/<protocol>]

# ตัวอย่าง:
EXPOSE 3000          # TCP (default)
EXPOSE 80/tcp
EXPOSE 53/udp
EXPOSE 8080 8443     # Multiple ports

# ข้อสำคัญ:
# EXPOSE เป็นแค่ Documentation ไม่ได้ Publish Port จริง!
# ต้องใช้ -p ตอน docker run:
docker run -p 3000:3000 myapp     # Publish port
docker run -P myapp               # Publish all exposed ports (random host port)

# ใน Kubernetes: EXPOSE ไม่สำคัญ ใช้ Service แทน
```

### CMD - Default Command

```dockerfile
# Syntax:
CMD ["executable", "arg1", "arg2"]  # Exec form (แนะนำ)
CMD ["arg1", "arg2"]                 # Default args สำหรับ ENTRYPOINT
CMD command arg1 arg2               # Shell form

# ตัวอย่าง:
CMD ["node", "app.js"]
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0"]
CMD ["nginx", "-g", "daemon off;"]

# Override ตอน run:
docker run myapp node other.js  # Override CMD

# ข้อสำคัญ:
# มีแค่ CMD เดียวใน Dockerfile (อันสุดท้ายจะมีผล)
# CMD ถูก Override ได้ง่ายโดย docker run argument
```

### ENTRYPOINT - Container Executable

```dockerfile
# Syntax:
ENTRYPOINT ["executable", "arg1"]  # Exec form (แนะนำ)
ENTRYPOINT command arg              # Shell form

# ตัวอย่าง:
ENTRYPOINT ["node", "app.js"]

# ENTRYPOINT + CMD ร่วมกัน:
ENTRYPOINT ["node"]
CMD ["app.js"]
# docker run myapp                 → node app.js
# docker run myapp other.js        → node other.js (Override CMD เท่านั้น)
# docker run --entrypoint sh myapp → sh (Override ENTRYPOINT)

# Pattern ที่นิยม: entrypoint.sh
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "app.js"]
```

```bash
# entrypoint.sh
#!/bin/sh
set -e

# Run migrations
if [ "$RUN_MIGRATIONS" = "true" ]; then
  echo "Running database migrations..."
  node migrate.js
fi

# Start application
exec "$@"
```

### WORKDIR - Working Directory

```dockerfile
# Syntax:
WORKDIR /path/to/workdir

# ตัวอย่าง:
WORKDIR /app

# สร้าง Directory อัตโนมัติถ้าไม่มี
WORKDIR /usr/src/app

# ใช้ ENV ร่วมกัน:
ENV APP_HOME=/app
WORKDIR $APP_HOME

# Multiple WORKDIR:
WORKDIR /app
WORKDIR src     # สัมพัทธ์กับ /app → /app/src
WORKDIR /other  # Absolute path

# Best Practice:
# ✓ ใช้ WORKDIR แทน RUN cd
# ✓ ใช้ Absolute path
# ✗ อย่าใช้ RUN cd /app && ...
```

### USER - Switch User

```dockerfile
# Syntax:
USER <user>[:<group>]
USER <UID>[:<GID>]

# ตัวอย่าง:
# สร้าง Non-root User
RUN groupadd -r appuser && \
    useradd -r -g appuser -s /bin/false appuser

# สร้าง Directory และกำหนด Permission
RUN mkdir -p /app && chown -R appuser:appuser /app

# Switch to non-root
USER appuser

# Alpine Linux:
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs
USER nodeuser

# ทำไม Non-root สำคัญ:
# Container breaks out vulnerability → Host อาจถูก Compromise
# Production: ห้ามรัน Container เป็น root!
```

### VOLUME - Mount Points

```dockerfile
# Syntax:
VOLUME ["/path/inside/container"]
VOLUME /path/inside/container

# ตัวอย่าง:
VOLUME ["/app/data"]
VOLUME /var/log/app
VOLUME ["/data", "/logs"]  # Multiple volumes

# Database Example:
FROM postgres:15
VOLUME /var/lib/postgresql/data  # ← Data Persist

# ข้อสำคัญ:
# VOLUME สร้าง Anonymous Volume อัตโนมัติ
# เมื่อ Container หยุด Data ยังอยู่ใน Volume
# docker run -v mydata:/data myapp  ← Named volume
# docker run -v /host/path:/data myapp  ← Bind mount
```

### HEALTHCHECK - Container Health

```dockerfile
# Syntax:
HEALTHCHECK [OPTIONS] CMD <command>
HEALTHCHECK NONE  # Disable inherited healthcheck

# Options:
# --interval=30s    (default: 30s) ตรวจสอบทุก N วินาที
# --timeout=30s     (default: 30s) Timeout ของ Health check
# --start-period=5s (default: 0s)  รอ Start ก่อนเริ่มตรวจ
# --retries=3       (default: 3)   ล้มเหลวกี่ครั้งถือว่า Unhealthy

# ตัวอย่าง:
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health \
  || exit 1

# HTTP Health Check:
HEALTHCHECK --interval=15s --timeout=5s \
  CMD curl -f http://localhost:3000/health || exit 1

# Custom Health Script:
COPY healthcheck.sh /
RUN chmod +x /healthcheck.sh
HEALTHCHECK --interval=30s --timeout=10s \
  CMD /healthcheck.sh

# Status Values:
# 0 = healthy
# 1 = unhealthy
# 2 = reserved (ไม่ใช้)
```

### ONBUILD - Deferred Instructions

```dockerfile
# ใช้สำหรับ Base Image ที่ต้องการให้ Child Image ทำอะไรบางอย่าง
# Instruction จะรันเมื่อมี Image อื่น FROM image นี้

# Base Image (node-base):
FROM node:18-alpine
WORKDIR /app
ONBUILD COPY package*.json ./
ONBUILD RUN npm install
ONBUILD COPY . .

# Child Image:
FROM node-base  # ← ONBUILD instructions รันตรงนี้
# ไม่ต้องเขียน COPY/RUN อีก

# ตัวอย่างการใช้จริง:
# - Framework templates
# - Organization base images
# - ลด Boilerplate ใน Project Dockerfiles
```

### STOPSIGNAL - Stop Signal

```dockerfile
# Signal ที่ Docker ส่งเพื่อหยุด Container
# Default: SIGTERM

STOPSIGNAL SIGTERM   # ปกติ
STOPSIGNAL SIGQUIT   # สำหรับ Nginx
STOPSIGNAL SIGINT    # สำหรับ Python processes

# ตัวเลขแทน Signal:
STOPSIGNAL 15  # = SIGTERM
STOPSIGNAL 9   # = SIGKILL (ไม่แนะนำ)

# Graceful Shutdown ที่ถูกต้อง:
# 1. Docker ส่ง STOPSIGNAL (default SIGTERM)
# 2. App รับ Signal และ Cleanup (close connections, etc.)
# 3. ถ้าไม่ตอบสนองใน --stop-timeout (default 10s) → SIGKILL
```

### SHELL - Override Default Shell

```dockerfile
# Default Shell:
# Linux: ["/bin/sh", "-c"]
# Windows: ["cmd", "/S", "/C"]

# Override Shell:
SHELL ["/bin/bash", "-c"]
RUN echo $BASH_VERSION

SHELL ["/bin/sh", "-exo", "pipefail", "-c"]
RUN echo hello | cat  # pipefail catches errors in pipes

# Windows:
SHELL ["powershell", "-command"]
RUN Write-Host "Hello from PowerShell"

# แนะนำสำหรับ Linux:
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
# -o pipefail: ทำให้ pipe fail ถ้า command ใดๆ ใน pipe fail
```

### LABEL - Metadata

```dockerfile
# Syntax:
LABEL <key>=<value>

# ตัวอย่าง:
LABEL maintainer="developer@company.com"
LABEL version="1.0.0"
LABEL description="My Application"

# OCI Standard Labels:
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="My Application" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.authors="developer@company.com" \
      org.opencontainers.image.url="https://github.com/myorg/myapp" \
      org.opencontainers.image.source="https://github.com/myorg/myapp" \
      org.opencontainers.image.revision="abc123" \
      org.opencontainers.image.created="2024-01-15T10:00:00Z"

# ดู Labels:
docker inspect myapp | jq '.[0].Config.Labels'
```

---

## .dockerignore ที่ถูกต้อง

### ทำไม .dockerignore สำคัญ?

```
ถ้าไม่มี .dockerignore:

Build Context ที่ส่งไป Docker Daemon:
project/
├── node_modules/    ← 500MB ไม่จำเป็น!
├── .git/            ← 100MB ไม่จำเป็น!
├── build/           ← 200MB ไม่จำเป็น!
├── logs/            ← ไม่จำเป็น
├── .env             ← อันตราย! ไม่ควร include
├── coverage/        ← ไม่จำเป็น
└── src/             ← ต้องการ

Total Context Size: ~800MB (ช้ามาก!)

หลังมี .dockerignore:
Total Context Size: ~10MB (เร็วมาก!)
```

### .dockerignore สำหรับ Node.js Projects

```gitignore
# .dockerignore สำหรับ Node.js

# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Build outputs
build/
dist/
out/

# Test files
coverage/
.nyc_output/
*.test.js
*.spec.js
__tests__/
jest.config.js
.jest/

# Development files
.env
.env.local
.env.development
.env.test

# Version control
.git/
.gitignore
.gitattributes

# IDE files
.vscode/
.idea/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db
desktop.ini

# Documentation
docs/
*.md
LICENSE

# Docker files (ไม่ต้อง copy Dockerfile เข้าไปใน Image)
Dockerfile*
docker-compose*
.dockerignore

# Logs
logs/
*.log

# Temp files
tmp/
temp/
.tmp/

# CI/CD
.github/
.gitlab-ci.yml
.travis.yml
.circleci/
Jenkinsfile
```

### .dockerignore สำหรับ Python Projects

```gitignore
# .dockerignore สำหรับ Python

# Virtual environments
.venv/
venv/
env/
ENV/
.env/

# Python cache
__pycache__/
*.py[cod]
*$py.class
*.pyc

# Distribution
dist/
build/
*.egg-info/
*.egg

# Testing
.tox/
.pytest_cache/
htmlcov/
.coverage
.coverage.*
coverage.xml

# Type checking
.mypy_cache/
.pytype/

# Development
.env
.env.local
*.env

# IDEs
.vscode/
.idea/

# Git
.git/
.gitignore

# Documentation
docs/
*.md
LICENSE

# Docker
Dockerfile*
docker-compose*
.dockerignore

# OS
.DS_Store
Thumbs.db
```

### .dockerignore สำหรับ Go Projects

```gitignore
# .dockerignore สำหรับ Go

# Binary outputs
*.exe
*.exe~
*.dll
*.so
*.dylib
bin/
dist/

# Test
*_test.go
testdata/

# Go workspace
go.work
go.work.sum

# Development
.env
*.env.local

# IDEs
.vscode/
.idea/
*.swp

# Git
.git/
.gitignore

# Documentation
docs/
*.md
LICENSE

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Profiling
*.prof
*.pprof

# OS
.DS_Store
Thumbs.db
```

---

## Docker Image Layer Caching อย่างละเอียด

### หลักการของ Layer Caching

```
Docker Build ทุกครั้ง:
1. อ่าน Dockerfile ทีละ Instruction
2. ตรวจสอบว่ามี Cache Layer หรือไม่
3. ถ้ามี Cache → ใช้ Cache (เร็วมาก)
4. ถ้าไม่มี Cache → Build ใหม่ทุก Instruction ที่เหลือ

Cache Invalidation Rules:
- Layer ไหน Invalidate → ทุก Layer หลังจากนั้น Invalidate ด้วย

ตัวอย่าง Dockerfile ที่ BAD (Cache ไม่ดี):
FROM node:18-alpine
WORKDIR /app
COPY . .           ← Copy ทุกอย่าง
RUN npm install    ← ทุกครั้งที่ Source code เปลี่ยน npm install ใหม่!
CMD ["node", "app.js"]

ทุกครั้งที่แก้ไข app.js:
Layer 1: FROM → Cache ✓
Layer 2: WORKDIR → Cache ✓
Layer 3: COPY . . → Miss! (app.js เปลี่ยน)
Layer 4: npm install → Miss! (ต้อง install ใหม่ ~60 วินาที!)
Layer 5: CMD → Miss!

ตัวอย่าง Dockerfile ที่ GOOD (Cache ดี):
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./  ← Copy เฉพาะ package files ก่อน
RUN npm install        ← Cache จนกว่า package.json จะเปลี่ยน
COPY . .               ← Copy Source code (ทีหลัง)
CMD ["node", "app.js"]

ทุกครั้งที่แก้ไข app.js:
Layer 1: FROM → Cache ✓
Layer 2: WORKDIR → Cache ✓
Layer 3: COPY package*.json → Cache ✓ (ไม่ได้เปลี่ยน)
Layer 4: npm install → Cache ✓ (~60 วินาทีประหยัดได้!)
Layer 5: COPY . . → Miss! (app.js เปลี่ยน)
Layer 6: CMD → Miss!

ประหยัดเวลา: 60 วินาที ต่อ Build!
```

### Layer Caching กับ Python

```dockerfile
# BAD - Pip install ทุกครั้งที่ Code เปลี่ยน
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# GOOD - Cache pip install
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .      # ← Copy requirements ก่อน
RUN pip install --no-cache-dir -r requirements.txt  # ← Cache!
COPY . .                     # ← Copy source ทีหลัง
CMD ["python", "app.py"]
```

### Multi-stage Build เพื่อ Optimize Image Size

```dockerfile
# Multi-stage Build: Node.js React App

# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# ติดตั้ง dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source และ Build
COPY . .
RUN npm run build

# Stage 2: Production Image
FROM nginx:alpine AS runner

# Copy built files จาก builder stage
COPY --from=builder /app/build /usr/share/nginx/html

# Copy nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost/ || exit 1

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# ผลลัพธ์:
# Single-stage image: ~800MB (node:18-alpine + node_modules + source)
# Multi-stage image: ~25MB (nginx:alpine + built files เท่านั้น!)
```

### เทคนิค Advanced Caching

```dockerfile
# Cache Mount สำหรับ Package Managers (BuildKit)

# Python:
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Node.js:
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Go:
FROM golang:1.22-alpine
WORKDIR /app
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
```

---

## Workshop: Dockerize Python Flask App + Node.js + Go App

### Workshop 1: Python Flask App

```bash
# สร้าง Project
mkdir flask-app && cd flask-app

# สร้าง Requirements
cat > requirements.txt << 'EOF'
flask==3.0.0
gunicorn==21.2.0
flask-sqlalchemy==3.1.1
psycopg2-binary==2.9.9
python-dotenv==1.0.0
EOF

# สร้าง Application
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

APP_VERSION = os.environ.get('APP_VERSION', '1.0.0')
DB_URL = os.environ.get('DATABASE_URL', 'sqlite:///app.db')

@app.route('/')
def index():
    return jsonify({
        'message': 'Hello from Flask!',
        'version': APP_VERSION,
        'hostname': socket.gethostname(),
        'python_version': os.sys.version
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'version': APP_VERSION})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=False)
EOF

# สร้าง Dockerfile
cat > Dockerfile << 'EOF'
# Base Image
FROM python:3.12-slim

# Security: Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install System Dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        gcc \
    && rm -rf /var/lib/apt/lists/*

# Set Working Directory
WORKDIR /app

# Copy requirements first (Layer Caching)
COPY requirements.txt .

# Install Python Dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy Application Code
COPY --chown=appuser:appuser . .

# Switch to non-root
USER appuser

# Environment Variables
ENV FLASK_APP=app.py \
    FLASK_ENV=production \
    PORT=5000

# Expose Port
EXPOSE 5000

# Health Check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Start with Gunicorn (Production WSGI Server)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "--timeout", "120", "app:app"]
EOF

# Build Image
docker build -t flask-app:v1.0 .

# Run Container
docker run -d \
  --name flask-app \
  -p 5000:5000 \
  -e APP_VERSION=1.0.0 \
  flask-app:v1.0

# ทดสอบ
curl http://localhost:5000
curl http://localhost:5000/health

# ดู Logs
docker logs flask-app

# ดู Container Stats
docker stats flask-app --no-stream
```

### Workshop 2: Node.js Express App

```bash
mkdir node-app && cd node-app

# package.json
cat > package.json << 'EOF'
{
  "name": "node-app",
  "version": "1.0.0",
  "description": "Node.js Docker Workshop",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0",
    "compression": "^1.7.4"
  }
}
EOF

# สร้าง Source
mkdir src
cat > src/app.js << 'EOF'
const express = require('express');
const helmet = require('helmet');
const morgan = require('morgan');
const compression = require('compression');
const os = require('os');

const app = express();
const PORT = process.env.PORT || 3000;
const APP_VERSION = process.env.APP_VERSION || '1.0.0';

// Security Middleware
app.use(helmet());
app.use(compression());
app.use(morgan('combined'));
app.use(express.json());

// Routes
app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Node.js!',
    version: APP_VERSION,
    hostname: os.hostname(),
    uptime: process.uptime(),
    nodeVersion: process.version,
    memory: process.memoryUsage()
  });
});

app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    version: APP_VERSION,
    timestamp: new Date().toISOString()
  });
});

app.get('/ready', (req, res) => {
  // Check dependencies here (DB, Cache, etc.)
  res.json({ status: 'ready' });
});

// Graceful Shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down...');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Version: ${APP_VERSION}`);
});
EOF

# Dockerfile Production-ready
cat > Dockerfile << 'EOF'
# Stage 1: Dependencies
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Stage 2: Build (ถ้ามี TypeScript หรือ Build step)
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
# RUN npm run build  # ถ้ามี Build step

# Stage 3: Production
FROM node:18-alpine AS runner
WORKDIR /app

# Security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeapp -u 1001 -G nodejs

# Copy from builder
COPY --from=builder --chown=nodeapp:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeapp:nodejs /app/src ./src
COPY --from=builder --chown=nodeapp:nodejs /app/package.json ./

USER nodeapp

ENV NODE_ENV=production \
    PORT=3000

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "src/app.js"]
EOF

# Build และ Run
docker build -t node-app:v1.0 .
docker run -d --name node-app -p 3000:3000 node-app:v1.0

curl http://localhost:3000
```

### Workshop 3: Go Application

```bash
mkdir go-app && cd go-app

# สร้าง Go Module
go mod init github.com/myorg/go-app

# สร้าง Application
cat > main.go << 'EOF'
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "runtime"
    "time"
)

type Response struct {
    Message    string `json:"message"`
    Version    string `json:"version"`
    Hostname   string `json:"hostname"`
    GoVersion  string `json:"goVersion"`
    GOOS       string `json:"goos"`
    GOARCH     string `json:"goarch"`
    Timestamp  string `json:"timestamp"`
}

type HealthResponse struct {
    Status    string `json:"status"`
    Version   string `json:"version"`
    Timestamp string `json:"timestamp"`
}

var appVersion = os.Getenv("APP_VERSION")

func init() {
    if appVersion == "" {
        appVersion = "1.0.0"
    }
}

func indexHandler(w http.ResponseWriter, r *http.Request) {
    hostname, _ := os.Hostname()
    resp := Response{
        Message:   "Hello from Go!",
        Version:   appVersion,
        Hostname:  hostname,
        GoVersion: runtime.Version(),
        GOOS:      runtime.GOOS,
        GOARCH:    runtime.GOARCH,
        Timestamp: time.Now().UTC().Format(time.RFC3339),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    resp := HealthResponse{
        Status:    "healthy",
        Version:   appVersion,
        Timestamp: time.Now().UTC().Format(time.RFC3339),
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    
    http.HandleFunc("/", indexHandler)
    http.HandleFunc("/health", healthHandler)
    
    fmt.Printf("Server starting on port %s\n", port)
    if err := http.ListenAndServe(":"+port, nil); err != nil {
        fmt.Printf("Server error: %v\n", err)
        os.Exit(1)
    }
}
EOF

# Dockerfile สำหรับ Go (Multi-stage)
cat > Dockerfile << 'EOF'
# Stage 1: Build
FROM golang:1.22-alpine AS builder

# Install git สำหรับ go mod download
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /build

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download && go mod verify

# Copy source
COPY . .

# Build binary
# CGO_ENABLED=0 = ไม่ใช้ C libraries (static binary)
# GOOS=linux = Build สำหรับ Linux
# -ldflags="-w -s" = Strip debug info (ลดขนาด binary)
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags="-w -s -X main.appVersion=1.0.0" \
    -o /app/server \
    .

# Stage 2: Minimal Runtime Image
FROM scratch AS runner
# scratch = Image ว่างเปล่า (ขนาดเล็กที่สุด!)

# Copy CA Certificates (สำหรับ HTTPS)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy timezone data
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Copy binary เท่านั้น
COPY --from=builder /app/server /server

# ไม่มี Shell หรือ OS ใดๆ ในนี้!
EXPOSE 8080

ENV PORT=8080

ENTRYPOINT ["/server"]
EOF

# Build และ Run
docker build -t go-app:v1.0 .

# ดูขนาด Image
docker images go-app
# REPOSITORY   TAG   IMAGE ID   CREATED   SIZE
# go-app       v1.0  xxx        1m        ~8MB  ← เล็กมาก!

docker run -d --name go-app -p 8080:8080 go-app:v1.0

curl http://localhost:8080
```

---

## แบบฝึกหัดพร้อมเฉลย

### ข้อที่ 1: Optimize Dockerfile

**โจทย์**: Dockerfile ต่อไปนี้มีปัญหาอะไรบ้าง? แก้ไขให้ดีขึ้น

```dockerfile
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y nodejs npm
COPY . .
RUN npm install
EXPOSE 3000
CMD node app.js
```

**เฉลย**:

```dockerfile
# ปัญหาที่พบ:
# 1. FROM ubuntu:latest → ควรระบุ version, ควรใช้ Alpine (เล็กกว่า)
# 2. RUN แยก 2 บรรทัด → ควรรวมเป็น 1 Layer
# 3. ไม่ Clean apt cache → Image ใหญ่เกินไป
# 4. COPY . . ก่อน npm install → Cache ไม่ทำงาน
# 5. ไม่มี .dockerignore → copy node_modules ด้วย
# 6. ไม่มี Non-root user → Security risk
# 7. ไม่มี Resource → ไม่รู้ว่าต้องการ Memory/CPU เท่าไร
# 8. CMD node app.js → Shell form, ไม่รับ Signal ได้ดี

# Dockerfile ที่ดีกว่า:
FROM node:18-alpine

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeapp -u 1001 -G nodejs

WORKDIR /app

# Layer Caching: Copy package files ก่อน
COPY package*.json ./

# Install deps
RUN npm ci --only=production && npm cache clean --force

# Copy source
COPY --chown=nodeapp:nodejs . .

USER nodeapp

ENV NODE_ENV=production PORT=3000

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Exec form สำหรับรับ SIGTERM
CMD ["node", "app.js"]
```

### ข้อที่ 2: Multi-stage Build

**โจทย์**: สร้าง Multi-stage Dockerfile สำหรับ Python App ที่:
- Stage 1: Build/Compile
- Stage 2: Production (เล็กที่สุด)

**เฉลย**:

```dockerfile
# Stage 1: Build Dependencies
FROM python:3.12 AS builder

WORKDIR /build

# Install build tools
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# Build wheels สำหรับ Dependencies ทั้งหมด
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /wheels -r requirements.txt

# Stage 2: Production
FROM python:3.12-slim AS production

# Install only runtime dependencies
RUN apt-get update && apt-get install -y \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Security: Non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Install wheels จาก builder
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir /wheels/*.whl && rm -rf /wheels

# Copy source
COPY --chown=appuser:appuser . .

USER appuser

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=5000

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=5s \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

### ข้อที่ 3: Docker Compose สำหรับ Development

**โจทย์**: สร้าง docker-compose.yml สำหรับ Full-stack App (Frontend, Backend, DB, Cache)

**เฉลย**:

```yaml
# docker-compose.yml สำหรับ Development
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
    - "3000:3000"
    volumes:
    - ./frontend/src:/app/src  # Hot reload
    - ./frontend/public:/app/public
    environment:
    - REACT_APP_API_URL=http://localhost:8080
    depends_on:
    - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
    - "8080:8080"
    volumes:
    - ./backend:/app  # Hot reload
    - /app/node_modules  # ไม่ Override node_modules
    environment:
    - NODE_ENV=development
    - PORT=8080
    - DATABASE_URL=postgres://user:password@postgres:5432/myapp
    - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    ports:
    - "5432:5432"
    environment:
    - POSTGRES_DB=myapp
    - POSTGRES_USER=user
    - POSTGRES_PASSWORD=password
    volumes:
    - postgres-data:/var/lib/postgresql/data
    - ./backend/migrations:/docker-entrypoint-initdb.d  # Auto-run SQL
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
    - "6379:6379"
    volumes:
    - redis-data:/data
    command: redis-server --appendonly yes  # Persistence
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  postgres-data:
  redis-data:

networks:
  default:
    name: myapp-network
```

### ข้อที่ 4: HEALTHCHECK ที่ถูกต้อง

**โจทย์**: เพิ่ม HEALTHCHECK ที่เหมาะสมสำหรับ Application แต่ละประเภท

**เฉลย**:

```dockerfile
# 1. Web Application (HTTP)
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# หรือใช้ wget (Alpine ไม่มี curl):
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# 2. Database (PostgreSQL)
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
    CMD pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB} || exit 1

# 3. Redis
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD redis-cli ping | grep PONG || exit 1

# 4. Custom Script Health Check
COPY healthcheck.sh /healthcheck.sh
RUN chmod +x /healthcheck.sh
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD /healthcheck.sh

# healthcheck.sh:
#!/bin/sh
# ตรวจสอบหลายอย่าง
check_http() {
    wget --no-verbose --tries=1 --spider http://localhost:3000/health 2>&1
    return $?
}

check_db() {
    nc -z postgres 5432 2>&1
    return $?
}

check_http && check_db && exit 0 || exit 1
```

---

## สรุป Part 03

ใน Part นี้เราได้เรียนรู้:

1. **Docker Architecture Deep Dive** - containerd, runc, Linux Namespaces, cgroups
2. **Dockerfile Instructions ครบทุก Instruction** - FROM, RUN, COPY, ADD, ENV, ARG, EXPOSE, CMD, ENTRYPOINT, WORKDIR, USER, VOLUME, HEALTHCHECK, ONBUILD, STOPSIGNAL, SHELL, LABEL
3. **.dockerignore ที่ถูกต้อง** - ลด Build Context Size และ Security
4. **Layer Caching อย่างละเอียด** - เทคนิคการ Optimize Build Time
5. **Workshop จริง** - Dockerize Flask, Node.js, Go App

### Checklist ก่อนไปต่อ

- [ ] เข้าใจ Docker Architecture และ Linux Kernel Features
- [ ] รู้จัก Dockerfile Instructions ทุกตัว
- [ ] สร้าง .dockerignore ที่ถูกต้องได้
- [ ] เข้าใจ Layer Caching และ Optimize Dockerfile ได้
- [ ] Dockerize App ได้ทั้ง Flask, Node.js, Go
- [ ] ใช้ Multi-stage Build เพื่อลดขนาด Image ได้
- [ ] ทำแบบฝึกหัดครบ 4 ข้อ

---

*ต่อไป: [Part 04: Docker Fundamentals 2](./part-04-docker-fundamentals-2.md)*
