# Part 05: Docker Compose

## สารบัญ
- [Docker Compose คืออะไร](#docker-compose-คืออะไร)
- [docker-compose.yml Syntax](#docker-composeyml-syntax)
- [Services, Networks, Volumes](#services-networks-volumes)
- [Docker Compose Commands](#docker-compose-commands)
- [Workshop: Deploy Web App + Database + Redis ด้วย Compose](#workshop)

---

## Docker Compose คืออะไร

**Docker Compose** คือ Tool สำหรับ Define และ Run Multi-container Docker Applications

แทนที่จะรัน `docker run` หลายครั้ง เราสามารถ Define ทุก Service ใน `docker-compose.yml` แล้วรันด้วยคำสั่งเดียว

### ปัญหาที่ Docker Compose แก้ไข

**ไม่มี Docker Compose:**
```bash
# สร้าง Network
docker network create myapp-network

# สร้าง Volumes
docker volume create mysql-data
docker volume create redis-data

# รัน MySQL
docker run -d \
  --name mysql \
  --network myapp-network \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -p 3306:3306 \
  mysql:8.0

# รัน Redis
docker run -d \
  --name redis \
  --network myapp-network \
  -v redis-data:/data \
  -p 6379:6379 \
  redis:7-alpine

# รัน Backend
docker run -d \
  --name backend \
  --network myapp-network \
  -e DATABASE_URL=mysql://root:secret@mysql:3306/myapp \
  -e REDIS_URL=redis://redis:6379 \
  -p 8080:8080 \
  myapp-backend:latest

# รัน Frontend
docker run -d \
  --name frontend \
  --network myapp-network \
  -e API_URL=http://backend:8080 \
  -p 80:80 \
  myapp-frontend:latest

# รัน Nginx (Load Balancer)
docker run -d \
  --name nginx \
  --network myapp-network \
  -v ./nginx.conf:/etc/nginx/conf.d/default.conf \
  -p 443:443 \
  nginx:alpine

# ทั้งหมดนี้ต้องพิมพ์ทุกครั้งที่ Start!
```

**มี Docker Compose:**
```bash
# แค่คำสั่งเดียว!
docker compose up -d

# ทุกอย่างถูก Define ใน docker-compose.yml
```

### Docker Compose v1 vs v2

```bash
# Docker Compose v1 (deprecated)
docker-compose up  # ใช้ hyphen

# Docker Compose v2 (current)
docker compose up  # ไม่มี hyphen

# ติดตั้ง Docker Compose v2
# มากับ Docker Desktop แล้ว
# หรือติดตั้งแยก:
sudo apt-get install docker-compose-plugin

# ตรวจสอบ version
docker compose version
```

---

## docker-compose.yml Syntax

### โครงสร้างพื้นฐาน

```yaml
# docker-compose.yml

# Version ของ Compose File (ปัจจุบัน 3.x)
version: '3.9'

# Define Services (Containers)
services:
  service-name:
    # Service configuration...

# Define Networks
networks:
  network-name:
    # Network configuration...

# Define Volumes
volumes:
  volume-name:
    # Volume configuration...
```

### Service Configuration Options

```yaml
services:
  myapp:
    # ============================
    # Image หรือ Build
    # ============================
    
    # Option 1: ใช้ Image จาก Registry
    image: nginx:1.25-alpine
    
    # Option 2: Build จาก Dockerfile
    build:
      context: .              # Directory ที่มี Dockerfile
      dockerfile: Dockerfile  # ชื่อ Dockerfile (default: Dockerfile)
      target: production      # Stage ที่ต้องการ (Multi-stage)
      args:                   # Build Arguments
        BUILD_ENV: production
      cache_from:             # Cache จาก Images
        - myapp:latest
    
    # ============================
    # Container Name
    # ============================
    container_name: my-container-name
    
    # ============================
    # Port Mapping
    # ============================
    ports:
      - "8080:80"           # host:container
      - "127.0.0.1:8081:80" # bind to specific IP
      - "80"                 # Random host port
    
    # ============================
    # Environment Variables
    # ============================
    environment:
      - NODE_ENV=production
      - DB_HOST=mysql
      - DB_PORT=3306
    
    # หรือแบบ map
    environment:
      NODE_ENV: production
      DB_HOST: mysql
    
    # โหลดจากไฟล์
    env_file:
      - .env
      - .env.production
    
    # ============================
    # Volumes
    # ============================
    volumes:
      - ./data:/app/data              # Bind Mount
      - myvolume:/app/uploads         # Named Volume
      - /tmp/logs:/app/logs           # Absolute path
      - type: bind                    # Long syntax
        source: ./config
        target: /app/config
        read_only: true
    
    # ============================
    # Networks
    # ============================
    networks:
      - frontend
      - backend
    
    # ============================
    # Dependencies
    # ============================
    depends_on:
      mysql:
        condition: service_healthy   # รอจนกว่า MySQL healthy
      redis:
        condition: service_started   # รอแค่ started
    
    # ============================
    # Health Check
    # ============================
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # ============================
    # Restart Policy
    # ============================
    restart: unless-stopped
    # Options: no, always, on-failure, unless-stopped
    
    # ============================
    # Resource Limits
    # ============================
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
    
    # ============================
    # Logging
    # ============================
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    
    # ============================
    # Command Override
    # ============================
    command: npm start
    # หรือ
    command: ["npm", "start"]
    
    entrypoint: /docker-entrypoint.sh
    
    # ============================
    # Working Directory
    # ============================
    working_dir: /app
    
    # ============================
    # User
    # ============================
    user: "1001:1001"
    
    # ============================
    # Profiles (ใช้ features ตามต้องการ)
    # ============================
    profiles:
      - monitoring
      - debug
```

---

## Services, Networks, Volumes

### Services ในรายละเอียด

```yaml
version: '3.9'

services:
  # Web Application
  webapp:
    build: ./webapp
    ports:
      - "80:3000"
    networks:
      - frontend
      - backend
    depends_on:
      api:
        condition: service_healthy
    environment:
      - API_URL=http://api:8080
    restart: unless-stopped
  
  # API Service
  api:
    build: ./api
    ports:
      - "8080:8080"
    networks:
      - backend
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - DATABASE_URL=mysql://user:pass@mysql:3306/db
      - REDIS_URL=redis://redis:6379
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped
  
  # MySQL Database
  mysql:
    image: mysql:8.0
    volumes:
      - mysql_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      - MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_root_password
      - MYSQL_DATABASE=mydb
      - MYSQL_USER=appuser
      - MYSQL_PASSWORD_FILE=/run/secrets/mysql_password
    secrets:
      - mysql_root_password
      - mysql_password
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
  
  # Redis Cache
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_password:
    file: ./secrets/mysql_password.txt
```

### Networks ในรายละเอียด

```yaml
networks:
  # Frontend Network (สำหรับ User-facing services)
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
  
  # Backend Network (สำหรับ Internal services)
  backend:
    driver: bridge
    internal: true  # ← ไม่สามารถ Access จากภายนอกได้
  
  # External Network (Shared กับ Compose Projects อื่น)
  monitoring:
    external: true  # ← Network ถูกสร้างไว้แล้วภายนอก
    name: monitoring_network
```

**ประโยชน์ของการแยก Networks:**
```
frontend network:
  - webapp (port 80 exposed)
  
backend network:
  - webapp, api, mysql, redis
  - mysql และ redis ไม่ expose port ออกไป
  - ปลอดภัยกว่า!
```

### Volumes ในรายละเอียด

```yaml
volumes:
  # Named Volume (managed by Docker)
  mysql_data:
    driver: local
  
  redis_data:
    driver: local
  
  # Volume with options
  app_logs:
    driver: local
    driver_opts:
      type: none
      device: /host/path/to/logs
      o: bind
  
  # External Volume (สร้างไว้แล้วก่อน)
  backup_volume:
    external: true
    name: my-backup-volume
```

---

## Docker Compose Commands

### Commands หลัก

```bash
# เริ่ม Services ทั้งหมด (Background)
docker compose up -d

# เริ่ม Service เฉพาะ
docker compose up -d webapp mysql

# Build Images และ เริ่ม Services
docker compose up -d --build

# หยุด Services ทั้งหมด
docker compose down

# หยุดและลบ Volumes ด้วย
docker compose down -v

# หยุดและลบ Images ด้วย
docker compose down --rmi all

# ดู Services ที่กำลัง Running
docker compose ps

# ดู Logs
docker compose logs
docker compose logs -f               # Follow
docker compose logs -f webapp        # เฉพาะ Service

# Restart Service
docker compose restart webapp

# Scale Service
docker compose scale webapp=3

# Execute Command ใน Container
docker compose exec webapp sh
docker compose exec mysql mysql -uroot -p

# Pull Latest Images
docker compose pull

# ดู Running Processes
docker compose top

# ดู Configuration ที่ Merge แล้ว
docker compose config

# Pause/Unpause
docker compose pause
docker compose unpause
```

### Environment Files

```bash
# .env file (default)
MYSQL_PASSWORD=secret123
REDIS_PASSWORD=redis_secret
API_PORT=8080

# docker-compose.yml ใช้ variables จาก .env อัตโนมัติ
services:
  mysql:
    environment:
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
  
  api:
    ports:
      - "${API_PORT}:8080"
```

### Override Files

```yaml
# docker-compose.yml (Base)
services:
  webapp:
    image: webapp:latest
    environment:
      - NODE_ENV=production

# docker-compose.override.yml (Development overrides)
# ถูก Apply อัตโนมัติถ้ามีไฟล์นี้
services:
  webapp:
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
    ports:
      - "3000:3000"
```

```bash
# สำหรับ Production (ไม่ใช้ override)
docker compose -f docker-compose.yml up -d

# สำหรับ Development (ใช้ override)
docker compose up -d

# สำหรับ Testing
docker compose -f docker-compose.yml -f docker-compose.test.yml up
```

---

## Workshop: Deploy Web App + Database + Redis ด้วย Compose

### เป้าหมาย

Deploy Stack ที่ประกอบด้วย:
- **Frontend**: React App (Nginx)
- **Backend**: Node.js API (Express)
- **Database**: MySQL 8.0
- **Cache**: Redis 7
- **Reverse Proxy**: Nginx

### โครงสร้าง Project

```
fullstack-app/
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── index.html
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── server.js
├── nginx/
│   └── default.conf
└── db/
    └── init.sql
```

### ขั้นตอน 1: สร้าง Backend API

**`backend/package.json`:**
```json
{
  "name": "backend-api",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.5",
    "redis": "^4.6.12",
    "cors": "^2.8.5"
  }
}
```

**`backend/src/server.js`:**
```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const { createClient } = require('redis');
const cors = require('cors');
const os = require('os');

const app = express();
const PORT = process.env.PORT || 8080;

app.use(cors());
app.use(express.json());

// MySQL Connection
let db;
async function connectDB() {
  try {
    db = await mysql.createConnection({
      host: process.env.DB_HOST || 'mysql',
      user: process.env.DB_USER || 'appuser',
      password: process.env.DB_PASSWORD || 'apppassword',
      database: process.env.DB_NAME || 'myapp'
    });
    console.log('Connected to MySQL');
  } catch (err) {
    console.error('MySQL connection failed:', err.message);
    // Retry after 5 seconds
    setTimeout(connectDB, 5000);
  }
}

// Redis Connection
let redisClient;
async function connectRedis() {
  try {
    redisClient = createClient({
      url: process.env.REDIS_URL || 'redis://redis:6379'
    });
    await redisClient.connect();
    console.log('Connected to Redis');
  } catch (err) {
    console.error('Redis connection failed:', err.message);
  }
}

// Initialize connections
connectDB();
connectRedis();

// Routes
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy',
    container: os.hostname(),
    db: db ? 'connected' : 'disconnected',
    redis: redisClient?.isOpen ? 'connected' : 'disconnected'
  });
});

app.get('/api/items', async (req, res) => {
  try {
    // Try cache first
    if (redisClient?.isOpen) {
      const cached = await redisClient.get('items');
      if (cached) {
        return res.json({ source: 'cache', data: JSON.parse(cached) });
      }
    }
    
    // Query database
    const [rows] = await db.execute('SELECT * FROM items LIMIT 100');
    
    // Cache for 60 seconds
    if (redisClient?.isOpen) {
      await redisClient.setEx('items', 60, JSON.stringify(rows));
    }
    
    res.json({ source: 'database', data: rows });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.post('/api/items', async (req, res) => {
  try {
    const { name, description } = req.body;
    const [result] = await db.execute(
      'INSERT INTO items (name, description) VALUES (?, ?)',
      [name, description]
    );
    
    // Invalidate cache
    if (redisClient?.isOpen) {
      await redisClient.del('items');
    }
    
    res.status(201).json({ 
      id: result.insertId, 
      name, 
      description 
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get('/api/stats', async (req, res) => {
  try {
    const [countRows] = await db.execute('SELECT COUNT(*) as count FROM items');
    const cacheKeys = redisClient?.isOpen ? await redisClient.keys('*') : [];
    
    res.json({
      items_count: countRows[0].count,
      cache_keys: cacheKeys.length,
      container: os.hostname()
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(PORT, () => {
  console.log(`Backend API running on port ${PORT}`);
});
```

**`backend/Dockerfile`:**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY src/ ./src/
COPY package.json ./

ENV NODE_ENV=production PORT=8080

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeapp -u 1001
USER nodeapp

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["node", "src/server.js"]
```

### ขั้นตอน 2: สร้าง Frontend

**`frontend/index.html`:**
```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Docker Compose Workshop</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: Arial, sans-serif; background: #f5f5f5; }
    .container { max-width: 800px; margin: 40px auto; padding: 20px; }
    .header { background: #2c3e50; color: white; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
    .card { background: white; padding: 20px; border-radius: 8px; margin-bottom: 20px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
    .form-group { margin-bottom: 15px; }
    input, textarea { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
    button { background: #3498db; color: white; border: none; padding: 10px 20px; border-radius: 4px; cursor: pointer; }
    button:hover { background: #2980b9; }
    .items-list { list-style: none; }
    .item { padding: 10px; border-bottom: 1px solid #eee; }
    .badge { display: inline-block; padding: 3px 8px; border-radius: 12px; font-size: 12px; }
    .badge-cache { background: #27ae60; color: white; }
    .badge-db { background: #e74c3c; color: white; }
    .stats { display: grid; grid-template-columns: repeat(3, 1fr); gap: 15px; }
    .stat-card { text-align: center; padding: 15px; background: #ecf0f1; border-radius: 8px; }
    .stat-number { font-size: 2em; font-weight: bold; color: #2c3e50; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>🐳 Docker Compose Workshop</h1>
      <p>Web App + MySQL + Redis</p>
    </div>
    
    <div class="card">
      <h2>System Status</h2>
      <div class="stats" id="stats">Loading...</div>
    </div>
    
    <div class="card">
      <h2>Add Item</h2>
      <div class="form-group">
        <input type="text" id="name" placeholder="Item Name" />
      </div>
      <div class="form-group">
        <textarea id="description" placeholder="Description" rows="3"></textarea>
      </div>
      <button onclick="addItem()">Add Item</button>
    </div>
    
    <div class="card">
      <h2>Items <span id="source-badge"></span></h2>
      <ul class="items-list" id="items-list">Loading...</ul>
      <button onclick="loadItems()" style="margin-top: 15px;">Refresh</button>
    </div>
  </div>
  
  <script>
    const API_URL = '/api';
    
    async function loadStats() {
      try {
        const res = await fetch(`${API_URL}/stats`);
        const data = await res.json();
        document.getElementById('stats').innerHTML = `
          <div class="stat-card">
            <div class="stat-number">${data.items_count}</div>
            <div>Total Items</div>
          </div>
          <div class="stat-card">
            <div class="stat-number">${data.cache_keys}</div>
            <div>Cache Keys</div>
          </div>
          <div class="stat-card">
            <div class="stat-number" style="font-size:1em">${data.container}</div>
            <div>Container ID</div>
          </div>
        `;
      } catch (err) {
        document.getElementById('stats').innerHTML = `<p>Error: ${err.message}</p>`;
      }
    }
    
    async function loadItems() {
      try {
        const res = await fetch(`${API_URL}/items`);
        const data = await res.json();
        
        const badge = data.source === 'cache' 
          ? '<span class="badge badge-cache">From Cache</span>' 
          : '<span class="badge badge-db">From Database</span>';
        document.getElementById('source-badge').innerHTML = badge;
        
        if (data.data.length === 0) {
          document.getElementById('items-list').innerHTML = '<li class="item">No items yet</li>';
          return;
        }
        
        document.getElementById('items-list').innerHTML = data.data
          .map(item => `<li class="item"><strong>${item.name}</strong>: ${item.description}</li>`)
          .join('');
      } catch (err) {
        document.getElementById('items-list').innerHTML = `<li>Error: ${err.message}</li>`;
      }
    }
    
    async function addItem() {
      const name = document.getElementById('name').value;
      const description = document.getElementById('description').value;
      
      if (!name) return alert('กรุณาใส่ชื่อ Item');
      
      try {
        await fetch(`${API_URL}/items`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ name, description })
        });
        
        document.getElementById('name').value = '';
        document.getElementById('description').value = '';
        
        loadItems();
        loadStats();
      } catch (err) {
        alert(`Error: ${err.message}`);
      }
    }
    
    // Load on start
    loadStats();
    loadItems();
    
    // Auto-refresh stats every 10 seconds
    setInterval(loadStats, 10000);
  </script>
</body>
</html>
```

**`frontend/nginx.conf`:**
```nginx
server {
    listen 80;
    server_name _;
    
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # Cache static files
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, no-transform";
    }
}
```

**`frontend/Dockerfile`:**
```dockerfile
FROM nginx:1.25-alpine
COPY index.html /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -qO- http://localhost/ || exit 1
```

### ขั้นตอน 3: สร้าง Nginx Reverse Proxy Config

**`nginx/default.conf`:**
```nginx
upstream backend {
    server backend:8080;
    keepalive 32;
}

server {
    listen 80;
    server_name _;
    
    # Frontend
    location / {
        proxy_pass http://frontend:80;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # API
    location /api/ {
        proxy_pass http://backend/api/;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # Timeout settings
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
    
    # Health check endpoint
    location /health {
        proxy_pass http://backend/health;
    }
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript;
}
```

### ขั้นตอน 4: สร้าง Database Init Script

**`db/init.sql`:**
```sql
-- สร้าง Database (ถ้ายังไม่มี)
CREATE DATABASE IF NOT EXISTS myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE myapp;

-- สร้าง User
CREATE USER IF NOT EXISTS 'appuser'@'%' IDENTIFIED BY 'apppassword';
GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'%';
FLUSH PRIVILEGES;

-- สร้าง Table
CREATE TABLE IF NOT EXISTS items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ใส่ Sample Data
INSERT INTO items (name, description) VALUES
    ('Docker', 'Container Platform ที่ยอดเยี่ยม'),
    ('Kubernetes', 'Container Orchestration System'),
    ('Redis', 'In-memory Data Store สำหรับ Cache'),
    ('MySQL', 'Relational Database Management System'),
    ('Nginx', 'High-performance Web Server');
```

### ขั้นตอน 5: สร้าง docker-compose.yml

**`docker-compose.yml`:**
```yaml
version: '3.9'

services:
  # ==================
  # Reverse Proxy
  # ==================
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      frontend:
        condition: service_healthy
      backend:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - frontend
      - backend
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # ==================
  # Frontend
  # ==================
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    expose:
      - "80"
    restart: unless-stopped
    networks:
      - frontend
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ==================
  # Backend API
  # ==================
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    expose:
      - "8080"
    environment:
      - NODE_ENV=production
      - PORT=8080
      - DB_HOST=mysql
      - DB_USER=appuser
      - DB_PASSWORD=${MYSQL_APP_PASSWORD:-apppassword}
      - DB_NAME=myapp
      - REDIS_URL=redis://:${REDIS_PASSWORD:-redispassword}@redis:6379
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - backend
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # ==================
  # MySQL Database
  # ==================
  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD:-rootpassword}
      - MYSQL_DATABASE=myapp
      - MYSQL_USER=appuser
      - MYSQL_PASSWORD=${MYSQL_APP_PASSWORD:-apppassword}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-rootpassword}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"

  # ==================
  # Redis Cache
  # ==================
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redispassword}
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redispassword}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M

volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: false
```

### ขั้นตอน 6: สร้าง .env

**`.env`:**
```bash
# Database
MYSQL_ROOT_PASSWORD=super_secret_root_password
MYSQL_APP_PASSWORD=app_password_123

# Redis
REDIS_PASSWORD=redis_password_123

# App
NODE_ENV=production
```

### ขั้นตอน 7: Deploy และทดสอบ

```bash
# Build และ Start ทุก Services
docker compose up -d --build

# ดู Status
docker compose ps

# ตัวอย่าง Output:
# NAME              IMAGE         STATUS          PORTS
# backend-1         backend       healthy         8080/tcp
# frontend-1        frontend      healthy         80/tcp
# fullstack-mysql-1 mysql:8.0     healthy         3306/tcp
# fullstack-nginx-1 nginx:alpine  Up              0.0.0.0:80->80/tcp
# fullstack-redis-1 redis:alpine  healthy         6379/tcp

# ดู Logs
docker compose logs -f

# ดู Logs ของ Service เฉพาะ
docker compose logs -f backend

# ทดสอบ
curl http://localhost/health
curl http://localhost/api/items
curl http://localhost/api/stats

# เปิด Browser ไปที่ http://localhost
```

### ขั้นตอน 8: ทดสอบ Redis Cache

```bash
# ดู Cache ทำงาน
# ครั้งแรก - จาก Database
curl http://localhost/api/items
# {"source":"database","data":[...]}

# ครั้งที่ 2 - จาก Cache
curl http://localhost/api/items
# {"source":"cache","data":[...]}

# เข้าไปดูใน Redis
docker compose exec redis redis-cli -a ${REDIS_PASSWORD:-redispassword}
# > KEYS *
# 1) "items"
# > TTL items
# (integer) 45   ← เหลือ 45 วินาที
# > GET items
# "[{\"id\":1,\"name\":\"Docker\",...}]"
```

### ขั้นตอน 9: ทดสอบ Scale

```bash
# Scale Backend เป็น 3 instances
docker compose up -d --scale backend=3

# ดู Containers
docker compose ps

# ทดสอบ Load Balancing (Container ID จะเปลี่ยน)
for i in {1..6}; do
  curl -s http://localhost/health | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Container: {d[\"container\"]}')"
done

# Output จะเห็น Container ID สลับกัน:
# Container: abc123def456
# Container: def456ghi789
# Container: ghi789jkl012
# Container: abc123def456
# ...
```

### ขั้นตอน 10: Development Override

**`docker-compose.dev.yml`:**
```yaml
version: '3.9'

services:
  backend:
    build:
      target: deps
    volumes:
      - ./backend/src:/app/src  # Hot reload
    environment:
      - NODE_ENV=development
    command: npm run dev
  
  frontend:
    volumes:
      - ./frontend:/usr/share/nginx/html  # Hot reload
  
  # เพิ่ม Adminer สำหรับ Database Management
  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    networks:
      - backend
    environment:
      - ADMINER_DEFAULT_SERVER=mysql
```

```bash
# Run Development Mode
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# เข้า Adminer: http://localhost:8081
# System: MySQL
# Server: mysql
# Username: appuser
# Password: apppassword
# Database: myapp
```

### ทำความสะอาด

```bash
# หยุด Services
docker compose down

# หยุดและลบ Volumes (ข้อมูลหาย!)
docker compose down -v

# หยุดและลบทุกอย่าง
docker compose down -v --rmi all --remove-orphans
```

---

## สรุป

Docker Compose ทำให้การ Manage Multi-container Applications ง่ายขึ้นมาก:

```
จาก: docker run (หลายคำสั่ง, จดจำยาก)
เป็น: docker compose up -d (คำสั่งเดียว!)

ข้อดีหลัก:
- Single file สำหรับ define ทั้ง Stack
- Networking อัตโนมัติ
- Volume Management ง่าย
- Health Checks
- Dependency Management
- Easy Scaling
```

Docker Compose เหมาะสำหรับ:
- **Development Environment** - รัน Services ทั้งหมดบน Local
- **Testing** - Run Integration Tests
- **Small Production** - สำหรับ Simple Deployments
- **CI/CD** - ทดสอบใน Pipeline

แต่สำหรับ Production ขนาดใหญ่ → ใช้ **Kubernetes** แทน

---

## แบบฝึกหัด

1. เพิ่ม Service อื่น เช่น Elasticsearch หรือ RabbitMQ
2. เพิ่ม Redis Sentinel สำหรับ High Availability
3. สร้าง Script Backup MySQL Database จาก Volume
4. ทดลอง Docker Compose Profiles

## คำถามทบทวน

1. Docker Compose ต่างจาก Kubernetes อย่างไร?
2. `depends_on` กับ `condition: service_healthy` ต่างกันอย่างไร?
3. Network Internal ใน Docker Compose หมายความว่าอะไร?
4. ทำไมต้องแยก Networks (frontend/backend)?

---

*ต่อไป: [Part 06: Kubernetes Architecture](./part-06-k8s-architecture.md)*
