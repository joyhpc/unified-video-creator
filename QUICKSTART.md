# 快速开始指南

## 5 分钟快速体验

### 前置要求

```bash
# 检查环境
node --version    # v20+
python --version  # Python 3.12+
docker --version  # Docker 24+
```

### 方式 1: Docker Compose（推荐）

```bash
# 1. 创建项目目录
mkdir unified-video-creator && cd unified-video-creator

# 2. 下载配置文件
curl -O https://raw.githubusercontent.com/your-org/unified-video-creator/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/your-org/unified-video-creator/main/.env.example

# 3. 配置 API Keys
cp .env.example .env
nano .env  # 填入你的 API Keys

# 4. 启动服务
docker compose up -d

# 5. 等待服务就绪（约 30 秒）
docker compose logs -f api

# 6. 访问应用
# 前端: http://localhost:3000
# API 文档: http://localhost:8000/docs
```

### 方式 2: 本地开发

```bash
# 1. 克隆项目
git clone https://github.com/your-org/unified-video-creator.git
cd unified-video-creator

# 2. 启动后端
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
nano .env  # 填入 API Keys
alembic upgrade head
uvicorn main:app --reload &

# 3. 启动 Workers（新终端）
python -m lib.generation_worker --channel image --worker-id img-1 &
python -m lib.generation_worker --channel video --worker-id vid-1 &
python -m lib.generation_worker --channel audio --worker-id aud-1 &

# 4. 启动前端（新终端）
cd ../frontend
npm install
npm run dev

# 5. 访问 http://localhost:5173
```

## 第一个视频项目

### 1. 创建项目

```bash
# 使用 API
curl -X POST http://localhost:8000/api/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "我的第一个视频",
    "description": "测试项目"
  }'

# 响应
{
  "id": "proj_123456",
  "name": "我的第一个视频",
  "stage": "script"
}
```

### 2. 分析剧本

```bash
curl -X POST http://localhost:8000/api/projects/proj_123456/script/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "text": "在一个阳光明媚的早晨，小明走进了咖啡店。他点了一杯拿铁，坐在窗边开始工作。"
  }'

# 响应
{
  "entities": {
    "characters": [
      {"id": "char_001", "name": "小明", "description": "年轻男性"}
    ],
    "scenes": [
      {"id": "scene_001", "name": "咖啡店", "description": "温馨的咖啡店"}
    ],
    "props": [
      {"id": "prop_001", "name": "拿铁", "description": "咖啡"}
    ]
  }
}
```

### 3. 生成角色资产

```bash
curl -X POST http://localhost:8000/api/generate/asset \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "proj_123456",
    "entity_id": "char_001",
    "entity_type": "character",
    "num_variants": 4,
    "provider": "gemini"
  }'

# 响应
{
  "task_id": "task_789",
  "status": "queued"
}
```

### 4. 查看任务进度

```bash
# 轮询任务状态
curl http://localhost:8000/api/tasks/task_789

# 或使用 SSE 实时推送
curl -N http://localhost:8000/api/projects/proj_123456/events
```

### 5. 生成分镜

```bash
curl -X POST http://localhost:8000/api/projects/proj_123456/storyboard/generate \
  -H "Content-Type: application/json" \
  -d '{
    "style": "cinematic",
    "num_frames": 5
  }'
```

### 6. 生成视频

```bash
# 批量生成所有分镜视频
curl -X POST http://localhost:8000/api/generate/motion/batch \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "proj_123456",
    "frame_ids": ["frame_001", "frame_002", "frame_003"],
    "provider": "seedance"
  }'
```

### 7. 合成最终视频

```bash
curl -X POST http://localhost:8000/api/projects/proj_123456/assemble \
  -H "Content-Type: application/json" \
  -d '{
    "transitions": [{"type": "fade", "duration": 0.5}],
    "audio": {"bgm": "/path/to/bgm.mp3", "volume": 0.3}
  }'
```

### 8. 下载视频

```bash
# 获取最终视频 URL
curl http://localhost:8000/api/projects/proj_123456/final-video

# 下载
wget http://localhost:8000/output/proj_123456/final_video.mp4
```

## 使用 Python SDK

```python
from video_creator import VideoCreatorClient

# 初始化
client = VideoCreatorClient(api_key="your_api_key")

# 创建项目
project = client.projects.create(name="我的第一个视频")

# 分析剧本
script = """
在一个阳光明媚的早晨，小明走进了咖啡店。
他点了一杯拿铁，坐在窗边开始工作。
"""
result = client.scripts.analyze(project.id, text=script)

# 生成资产
for character in result.entities.characters:
    client.assets.generate(
        project_id=project.id,
        entity_id=character.id,
        entity_type="character",
        num_variants=4
    )

# 等待完成
client.tasks.wait_for_completion(project.id)

# 生成分镜
storyboard = client.storyboard.generate(project.id)

# 生成视频
for frame in storyboard.frames:
    client.motion.generate(
        project_id=project.id,
        frame_id=frame.id,
        provider="seedance"
    )

# 等待完成
client.tasks.wait_for_completion(project.id)

# 合成
final_video = client.projects.assemble(project.id)

print(f"视频生成完成: {final_video.url}")
```

## 使用 JavaScript SDK

```javascript
import { VideoCreatorClient } from '@video-creator/sdk';

const client = new VideoCreatorClient({
  apiKey: 'your_api_key'
});

// 创建项目
const project = await client.projects.create({
  name: '我的第一个视频'
});

// 分析剧本
const script = `
在一个阳光明媚的早晨，小明走进了咖啡店。
他点了一杯拿铁，坐在窗边开始工作。
`;
const result = await client.scripts.analyze(project.id, { text: script });

// 生成资产
await Promise.all(
  result.entities.characters.map(character =>
    client.assets.generate({
      projectId: project.id,
      entityId: character.id,
      entityType: 'character',
      numVariants: 4
    })
  )
);

// 监听进度
client.tasks.on('progress', (task) => {
  console.log(`进度: ${task.progress * 100}%`);
});

// 等待完成
await client.tasks.waitForCompletion(project.id);

// 生成分镜
const storyboard = await client.storyboard.generate(project.id);

// 生成视频
await Promise.all(
  storyboard.frames.map(frame =>
    client.motion.generate({
      projectId: project.id,
      frameId: frame.id,
      provider: 'seedance'
    })
  )
);

// 等待完成
await client.tasks.waitForCompletion(project.id);

// 合成
const finalVideo = await client.projects.assemble(project.id);

console.log(`视频生成完成: ${finalVideo.url}`);
```

## 常见问题

### Q: 如何配置 API Keys?

A: 编辑 `.env` 文件：

```bash
SEEDANCE_API_KEY=your_seedance_key
KLING_API_KEY=your_kling_key
GEMINI_API_KEY=your_gemini_key
QWEN_API_KEY=your_qwen_key
```

### Q: 视频生成失败怎么办?

A: 检查日志：

```bash
# Docker
docker compose logs worker-video

# 本地
tail -f logs/worker-video.log
```

常见原因：
- API Key 无效
- 超过速率限制
- 提示词不符合 Seedance 约束

### Q: 如何扩展 Worker 数量?

A: 修改 `docker-compose.yml`：

```yaml
worker-video:
  deploy:
    replicas: 10  # 从 2 改为 10
```

然后重启：

```bash
docker compose up -d --scale worker-video=10
```

### Q: 如何切换 AI 供应商?

A: 在生成请求中指定 `provider` 参数：

```bash
curl -X POST http://localhost:8000/api/generate/motion \
  -d '{"provider": "kling"}'  # 或 "seedance", "vidu"
```

### Q: 如何监控系统状态?

A: 访问监控端点：

```bash
# 队列状态
curl http://localhost:8000/api/tasks?status=running

# GPU 状态（如果有）
curl http://localhost:8000/api/gpu/stats

# Prometheus 指标
curl http://localhost:9090/metrics
```

## 下一步

- 📖 阅读 [完整文档](README.md)
- 🏗️ 查看 [架构设计](unified-video-creation-architecture.md)
- 🚀 学习 [部署指南](deployment-guide.md)
- 📚 浏览 [API 文档](api-documentation.md)
- 🔧 了解 [扩展方案](agent-scaling-guide.md)

## 获取帮助

- GitHub Issues: https://github.com/your-org/unified-video-creator/issues
- 文档: https://docs.video-creator.example.com
- Discord: https://discord.gg/video-creator
