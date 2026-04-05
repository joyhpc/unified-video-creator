# 部署指南

## 1. 开发环境搭建

### 前置要求

```bash
# Node.js 20+
node --version  # v20.x.x

# Python 3.12+
python --version  # Python 3.12.x

# Docker & Docker Compose
docker --version  # Docker version 24.x.x
docker compose version  # Docker Compose version v2.x.x

# FFmpeg (用于视频合成)
ffmpeg -version  # ffmpeg version 6.x.x
```

### 克隆项目

```bash
git clone https://github.com/your-org/unified-video-creator.git
cd unified-video-creator
```

### 后端设置

```bash
cd backend

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env 填入 API Keys
```

`.env` 配置示例：

```bash
# 数据库
DATABASE_URL=sqlite:///./data/app.db
# 生产环境使用 PostgreSQL:
# DATABASE_URL=postgresql+asyncpg://user:pass@localhost/dbname

# AI 服务商 API Keys
SEEDANCE_API_KEY=your_seedance_key
KLING_API_KEY=your_kling_key
GEMINI_API_KEY=your_gemini_key
QWEN_API_KEY=your_qwen_key
CLAUDE_API_KEY=your_claude_key

# OSS 存储 (可选)
OSS_ENABLED=false
OSS_ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com
OSS_ACCESS_KEY_ID=your_access_key
OSS_ACCESS_KEY_SECRET=your_secret_key
OSS_BUCKET_NAME=your_bucket

# Claude Agent SDK
CLAUDE_AGENT_API_KEY=your_claude_key
CLAUDE_AGENT_MODEL=claude-opus-4-6

# 服务配置
API_HOST=0.0.0.0
API_PORT=8000
CORS_ORIGINS=http://localhost:3000,http://localhost:5173
```

### 前端设置

```bash
cd frontend

# 安装依赖
npm install

# 配置环境变量
cp .env.example .env.local
# 编辑 .env.local
```

`.env.local` 配置示例：

```bash
VITE_API_URL=http://localhost:8000
VITE_SSE_URL=http://localhost:8000
```

### 数据库迁移

```bash
cd backend

# 运行迁移
alembic upgrade head

# 创建新迁移 (修改模型后)
alembic revision --autogenerate -m "description"
```

## 2. 本地开发运行

### 启动后端

```bash
cd backend

# 启动 FastAPI 服务
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# 或使用 Python 直接运行
python main.py
```

### 启动 Worker

```bash
cd backend

# 启动 Image Worker
python -m lib.generation_worker --channel image --worker-id image-worker-1

# 启动 Video Worker
python -m lib.generation_worker --channel video --worker-id video-worker-1

# 启动 Audio Worker
python -m lib.generation_worker --channel audio --worker-id audio-worker-1
```

### 启动前端

```bash
cd frontend

# 开发模式
npm run dev

# 访问 http://localhost:5173
```

## 3. Docker 部署

### Docker Compose 配置

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  # PostgreSQL 数据库
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: video_creator
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis (可选，用于缓存)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # FastAPI 后端
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      REDIS_URL: redis://redis:6379
      SEEDANCE_API_KEY: ${SEEDANCE_API_KEY}
      KLING_API_KEY: ${KLING_API_KEY}
      GEMINI_API_KEY: ${GEMINI_API_KEY}
      QWEN_API_KEY: ${QWEN_API_KEY}
      CLAUDE_API_KEY: ${CLAUDE_API_KEY}
    volumes:
      - ./output:/app/output
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  # Image Worker
  worker-image:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      CHANNEL: image
      WORKER_ID: image-worker-1
      GEMINI_API_KEY: ${GEMINI_API_KEY}
    volumes:
      - ./output:/app/output
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      replicas: 2

  # Video Worker
  worker-video:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      CHANNEL: video
      WORKER_ID: video-worker-1
      SEEDANCE_API_KEY: ${SEEDANCE_API_KEY}
      KLING_API_KEY: ${KLING_API_KEY}
    volumes:
      - ./output:/app/output
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      replicas: 2

  # Audio Worker
  worker-audio:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      CHANNEL: audio
      WORKER_ID: audio-worker-1
    volumes:
      - ./output:/app/output
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  # React 前端
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VITE_API_URL: http://localhost:8000
    ports:
      - "3000:80"
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 后端 Dockerfile

`backend/Dockerfile`:

```dockerfile
FROM python:3.12-slim

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    ffmpeg \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 创建输出目录
RUN mkdir -p /app/output

# 运行数据库迁移
RUN alembic upgrade head

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Worker Dockerfile

`backend/Dockerfile.worker`:

```dockerfile
FROM python:3.12-slim

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    ffmpeg \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 创建输出目录
RUN mkdir -p /app/output

# 启动命令
CMD ["python", "-m", "lib.generation_worker"]
```

### 前端 Dockerfile

`frontend/Dockerfile`:

```dockerfile
# 构建阶段
FROM node:20-alpine AS builder

WORKDIR /app

# 安装依赖
COPY package*.json ./
RUN npm ci

# 复制代码
COPY . .

# 构建
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# 生产阶段
FROM nginx:alpine

# 复制构建产物
COPY --from=builder /app/dist /usr/share/nginx/html

# 复制 Nginx 配置
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Nginx 配置

`frontend/nginx.conf`:

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # SPA 路由
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 代理
    location /api {
        proxy_pass http://api:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # SSE 代理
    location /api/projects {
        proxy_pass http://api:8000;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 启动 Docker 服务

```bash
# 构建并启动所有服务
docker compose up -d --build

# 查看日志
docker compose logs -f

# 停止服务
docker compose down

# 停止并删除数据
docker compose down -v
```

## 4. 桌面应用打包

### Electron 配置

`desktop/main.js`:

```javascript
const { app, BrowserWindow } = require('electron');
const path = require('path');
const { spawn } = require('child_process');

let mainWindow;
let backendProcess;

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1400,
    height: 900,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js')
    }
  });

  // 加载前端
  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:5173');
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(path.join(__dirname, '../frontend/dist/index.html'));
  }
}

function startBackend() {
  const backendPath = path.join(__dirname, '../backend/main.py');
  
  backendProcess = spawn('python', [backendPath], {
    env: {
      ...process.env,
      DATABASE_URL: `sqlite:///${app.getPath('userData')}/app.db`,
      OUTPUT_DIR: path.join(app.getPath('userData'), 'output')
    }
  });

  backendProcess.stdout.on('data', (data) => {
    console.log(`Backend: ${data}`);
  });

  backendProcess.stderr.on('data', (data) => {
    console.error(`Backend Error: ${data}`);
  });
}

app.whenReady().then(() => {
  startBackend();
  
  // 等待后端启动
  setTimeout(() => {
    createWindow();
  }, 3000);
});

app.on('window-all-closed', () => {
  if (backendProcess) {
    backendProcess.kill();
  }
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    createWindow();
  }
});
```

### 打包配置

`desktop/package.json`:

```json
{
  "name": "unified-video-creator",
  "version": "1.0.0",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "pack": "electron-builder --dir",
    "dist": "electron-builder"
  },
  "build": {
    "appId": "com.example.video-creator",
    "productName": "Unified Video Creator",
    "directories": {
      "output": "dist"
    },
    "files": [
      "main.js",
      "preload.js",
      "../frontend/dist/**/*",
      "../backend/**/*",
      "!../backend/venv",
      "!../backend/__pycache__"
    ],
    "extraResources": [
      {
        "from": "../backend",
        "to": "backend",
        "filter": ["**/*", "!venv", "!__pycache__"]
      }
    ],
    "mac": {
      "category": "public.app-category.video",
      "target": ["dmg", "zip"]
    },
    "win": {
      "target": ["nsis", "portable"]
    },
    "linux": {
      "target": ["AppImage", "deb"],
      "category": "Video"
    }
  },
  "dependencies": {
    "electron": "^28.0.0"
  },
  "devDependencies": {
    "electron-builder": "^24.9.1"
  }
}
```

### 打包命令

```bash
cd desktop

# 安装依赖
npm install

# 开发模式运行
npm start

# 打包（当前平台）
npm run dist

# 打包所有平台
npm run dist -- --mac --win --linux
```

## 5. 生产环境部署

### 使用 Nginx 反向代理

`/etc/nginx/sites-available/video-creator`:

```nginx
upstream api_backend {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name video-creator.example.com;

    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name video-creator.example.com;

    # SSL 证书
    ssl_certificate /etc/letsencrypt/live/video-creator.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/video-creator.example.com/privkey.pem;

    # SSL 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # 前端静态文件
    root /var/www/video-creator/frontend/dist;
    index index.html;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # API 代理
    location /api {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # 超时设置（视频生成可能很慢）
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }

    # SSE 代理
    location /api/projects {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        proxy_read_timeout 86400s;
    }

    # SPA 路由
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 上传大小限制
    client_max_body_size 100M;
}
```

### Systemd 服务配置

`/etc/systemd/system/video-creator-api.service`:

```ini
[Unit]
Description=Video Creator API
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/video-creator/backend
Environment="PATH=/var/www/video-creator/backend/venv/bin"
ExecStart=/var/www/video-creator/backend/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Worker 服务配置（Image/Video/Audio 各一个）:

`/etc/systemd/system/video-creator-worker-image@.service`:

```ini
[Unit]
Description=Video Creator Image Worker %i
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/video-creator/backend
Environment="PATH=/var/www/video-creator/backend/venv/bin"
Environment="CHANNEL=image"
Environment="WORKER_ID=image-worker-%i"
ExecStart=/var/www/video-creator/backend/venv/bin/python -m lib.generation_worker
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
# 启动 API
sudo systemctl enable video-creator-api
sudo systemctl start video-creator-api

# 启动 Workers (每个通道 2 个实例)
sudo systemctl enable video-creator-worker-image@{1,2}
sudo systemctl start video-creator-worker-image@{1,2}

sudo systemctl enable video-creator-worker-video@{1,2}
sudo systemctl start video-creator-worker-video@{1,2}

sudo systemctl enable video-creator-worker-audio@1
sudo systemctl start video-creator-worker-audio@1

# 查看状态
sudo systemctl status video-creator-api
sudo systemctl status video-creator-worker-image@1

# 查看日志
sudo journalctl -u video-creator-api -f
sudo journalctl -u video-creator-worker-image@1 -f
```

## 6. 监控与日志

### Prometheus 监控

`backend/lib/metrics.py`:

```python
from prometheus_client import Counter, Histogram, Gauge
from prometheus_client import start_http_server

# 任务计数器
task_counter = Counter(
    'generation_tasks_total',
    'Total number of generation tasks',
    ['channel', 'status']
)

# 任务耗时
task_duration = Histogram(
    'generation_task_duration_seconds',
    'Task execution duration',
    ['channel', 'provider']
)

# 队列长度
queue_length = Gauge(
    'generation_queue_length',
    'Number of tasks in queue',
    ['channel', 'status']
)

# 启动 Prometheus HTTP 服务器
start_http_server(9090)
```

### 日志配置

`backend/lib/logging_config.py`:

```python
import logging
from logging.handlers import RotatingFileHandler
import sys

def setup_logging():
    # 创建 logger
    logger = logging.getLogger('video_creator')
    logger.setLevel(logging.INFO)
    
    # 控制台 handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    console_handler.setFormatter(console_formatter)
    
    # 文件 handler (自动轮转)
    file_handler = RotatingFileHandler(
        'logs/app.log',
        maxBytes=10*1024*1024,  # 10MB
        backupCount=5
    )
    file_handler.setLevel(logging.INFO)
    file_formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    file_handler.setFormatter(file_formatter)
    
    # 添加 handlers
    logger.addHandler(console_handler)
    logger.addHandler(file_handler)
    
    return logger
```

## 7. 备份与恢复

### 数据库备份脚本

`scripts/backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/var/backups/video-creator"
DATE=$(date +%Y%m%d_%H%M%S)

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份 PostgreSQL
pg_dump -U postgres video_creator | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# 备份输出文件
tar -czf $BACKUP_DIR/output_$DATE.tar.gz /var/www/video-creator/output

# 删除 30 天前的备份
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

### 恢复脚本

`scripts/restore.sh`:

```bash
#!/bin/bash

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: ./restore.sh <backup_file>"
    exit 1
fi

# 恢复数据库
gunzip -c $BACKUP_FILE | psql -U postgres video_creator

echo "Restore completed"
```

### 定时备份 (Cron)

```bash
# 编辑 crontab
sudo crontab -e

# 每天凌晨 2 点备份
0 2 * * * /var/www/video-creator/scripts/backup.sh
```

## 8. 性能优化

### 数据库索引

```sql
-- 任务查询优化
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_resource_type ON tasks(resource_type);
CREATE INDEX idx_tasks_created_at ON tasks(created_at);
CREATE INDEX idx_tasks_lease_expires_at ON tasks(lease_expires_at);

-- 复合索引
CREATE INDEX idx_tasks_claim ON tasks(resource_type, status, created_at);
```

### Redis 缓存

```python
import redis.asyncio as redis
from functools import wraps

redis_client = redis.from_url("redis://localhost:6379")

def cache(ttl: int = 300):
    """缓存装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 生成缓存 key
            cache_key = f"{func.__name__}:{args}:{kwargs}"
            
            # 尝试从缓存获取
            cached = await redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # 执行函数
            result = await func(*args, **kwargs)
            
            # 写入缓存
            await redis_client.setex(
                cache_key,
                ttl,
                json.dumps(result)
            )
            
            return result
        return wrapper
    return decorator
```

### CDN 配置

使用 Cloudflare 或阿里云 CDN 加速静态资源和生成的媒体文件。

```nginx
# 添加 CDN 缓存头
location ~* \.(mp4|jpg|png|gif)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    add_header X-Content-Type-Options "nosniff";
}
```

---

部署完成后，访问 `https://video-creator.example.com` 即可使用平台。
