# API 文档

## 1. 认证

所有 API 请求需要在 Header 中携带 API Key：

```http
Authorization: Bearer YOUR_API_KEY
```

## 2. 项目管理

### 创建项目

```http
POST /api/projects
Content-Type: application/json

{
  "name": "我的视频项目",
  "description": "项目描述"
}
```

响应：

```json
{
  "id": "proj_123456",
  "name": "我的视频项目",
  "description": "项目描述",
  "stage": "script",
  "created_at": "2026-04-05T10:00:00Z"
}
```

### 获取项目列表

```http
GET /api/projects?page=1&limit=20
```

响应：

```json
{
  "items": [
    {
      "id": "proj_123456",
      "name": "我的视频项目",
      "stage": "motion",
      "created_at": "2026-04-05T10:00:00Z",
      "updated_at": "2026-04-05T12:00:00Z"
    }
  ],
  "total": 50,
  "page": 1,
  "limit": 20
}
```

### 获取项目详情

```http
GET /api/projects/{project_id}
```

响应：

```json
{
  "id": "proj_123456",
  "name": "我的视频项目",
  "stage": "motion",
  "script": {
    "text": "剧本内容...",
    "entities": {
      "characters": [...],
      "scenes": [...],
      "props": [...]
    }
  },
  "assets": [...],
  "storyboard": [...],
  "motion_clips": [...],
  "final_video": null
}
```

### 删除项目

```http
DELETE /api/projects/{project_id}
```

响应：

```json
{
  "success": true,
  "message": "项目已删除"
}
```

## 3. 剧本分析

### 分析剧本

```http
POST /api/projects/{project_id}/script/analyze
Content-Type: application/json

{
  "text": "在一个阳光明媚的早晨，小明走进了咖啡店..."
}
```

响应：

```json
{
  "entities": {
    "characters": [
      {
        "id": "char_001",
        "name": "小明",
        "description": "主角，年轻男性，穿着休闲"
      }
    ],
    "scenes": [
      {
        "id": "scene_001",
        "name": "咖啡店",
        "description": "温馨的咖啡店，木质装修，阳光透过窗户"
      }
    ],
    "props": [
      {
        "id": "prop_001",
        "name": "咖啡杯",
        "description": "白色陶瓷咖啡杯"
      }
    ]
  },
  "scenes": [
    {
      "id": "scene_001",
      "order": 1,
      "description": "小明走进咖啡店",
      "characters": ["char_001"],
      "scene": "scene_001",
      "props": [],
      "duration": 5
    }
  ]
}
```

### 更新剧本

```http
PUT /api/projects/{project_id}/script
Content-Type: application/json

{
  "text": "更新后的剧本内容...",
  "entities": {...}
}
```

## 4. 资产生成

### 生成资产

```http
POST /api/generate/asset
Content-Type: application/json

{
  "project_id": "proj_123456",
  "entity_id": "char_001",
  "entity_type": "character",
  "description": "年轻男性，穿着休闲，友好的笑容",
  "num_variants": 4,
  "provider": "gemini"
}
```

响应：

```json
{
  "task_id": "task_789",
  "status": "queued",
  "message": "任务已入队"
}
```

### 获取资产列表

```http
GET /api/projects/{project_id}/assets
```

响应：

```json
{
  "items": [
    {
      "id": "asset_001",
      "entity_id": "char_001",
      "entity_type": "character",
      "variants": [
        {
          "id": "var_001",
          "url": "/output/proj_123456/assets/char_001_var_001.png",
          "is_selected": true,
          "score": 0.95
        },
        {
          "id": "var_002",
          "url": "/output/proj_123456/assets/char_001_var_002.png",
          "is_selected": false,
          "score": 0.87
        }
      ],
      "created_at": "2026-04-05T10:30:00Z"
    }
  ]
}
```

### 选择资产变体

```http
PATCH /api/projects/{project_id}/assets/{asset_id}
Content-Type: application/json

{
  "selected_variant_id": "var_002"
}
```

## 5. 分镜生成

### 生成分镜脚本

```http
POST /api/projects/{project_id}/storyboard/generate
Content-Type: application/json

{
  "style": "cinematic",
  "num_frames": 10
}
```

响应：

```json
{
  "frames": [
    {
      "id": "frame_001",
      "order": 1,
      "scene": "咖啡店外景",
      "prompt": "阳光明媚的早晨，咖啡店外景，温暖的色调",
      "camera_motion": "推进镜头",
      "duration": 5,
      "characters": ["char_001"],
      "scene_id": "scene_001"
    }
  ]
}
```

### 更新分镜帧

```http
PATCH /api/projects/{project_id}/storyboard/{frame_id}
Content-Type: application/json

{
  "prompt": "更新后的提示词",
  "camera_motion": "摇镜头",
  "duration": 6
}
```

### 生成分镜图片

```http
POST /api/generate/storyboard-image
Content-Type: application/json

{
  "project_id": "proj_123456",
  "frame_id": "frame_001",
  "provider": "gemini"
}
```

响应：

```json
{
  "task_id": "task_790",
  "status": "queued"
}
```

## 6. 视频生成

### 生成分镜视频

```http
POST /api/generate/motion
Content-Type: application/json

{
  "project_id": "proj_123456",
  "frame_id": "frame_001",
  "provider": "seedance",
  "duration": 5,
  "params": {
    "quality": "high",
    "fps": 24
  }
}
```

响应：

```json
{
  "task_id": "task_791",
  "status": "queued",
  "estimated_time": 120
}
```

### 批量生成视频

```http
POST /api/generate/motion/batch
Content-Type: application/json

{
  "project_id": "proj_123456",
  "frame_ids": ["frame_001", "frame_002", "frame_003"],
  "provider": "seedance"
}
```

响应：

```json
{
  "tasks": [
    {
      "frame_id": "frame_001",
      "task_id": "task_791",
      "status": "queued"
    },
    {
      "frame_id": "frame_002",
      "task_id": "task_792",
      "status": "queued"
    }
  ]
}
```

## 7. 视频合成

### 合成最终视频

```http
POST /api/projects/{project_id}/assemble
Content-Type: application/json

{
  "transitions": [
    {
      "type": "fade",
      "duration": 0.5
    }
  ],
  "audio": {
    "bgm": "/path/to/bgm.mp3",
    "volume": 0.3
  }
}
```

响应：

```json
{
  "task_id": "task_800",
  "status": "queued",
  "estimated_time": 60
}
```

### 获取最终视频

```http
GET /api/projects/{project_id}/final-video
```

响应：

```json
{
  "url": "/output/proj_123456/final_video.mp4",
  "duration": 120,
  "size_mb": 45.6,
  "resolution": "1920x1080",
  "fps": 24,
  "created_at": "2026-04-05T14:00:00Z"
}
```

## 8. 任务管理

### 获取任务状态

```http
GET /api/tasks/{task_id}
```

响应：

```json
{
  "id": "task_791",
  "resource_type": "video",
  "resource_id": "frame_001",
  "status": "running",
  "progress": 0.65,
  "worker_id": "video-worker-1",
  "created_at": "2026-04-05T12:00:00Z",
  "started_at": "2026-04-05T12:01:00Z",
  "estimated_completion": "2026-04-05T12:03:00Z"
}
```

### 获取任务列表

```http
GET /api/tasks?project_id=proj_123456&status=running&page=1&limit=20
```

响应：

```json
{
  "items": [
    {
      "id": "task_791",
      "resource_type": "video",
      "status": "running",
      "progress": 0.65
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 20
}
```

### 取消任务

```http
POST /api/tasks/{task_id}/cancel
```

响应：

```json
{
  "success": true,
  "message": "任务已取消"
}
```

### 重试失败任务

```http
POST /api/tasks/{task_id}/retry
```

响应：

```json
{
  "task_id": "task_801",
  "status": "queued",
  "message": "任务已重新入队"
}
```

## 9. AI 助手

### 发送消息

```http
POST /api/assistant/chat
Content-Type: application/json

{
  "project_id": "proj_123456",
  "message": "帮我优化第3个分镜的提示词",
  "context": {
    "frame_id": "frame_003"
  }
}
```

响应：

```json
{
  "message": "我建议将提示词修改为...",
  "suggestions": [
    {
      "type": "prompt_update",
      "frame_id": "frame_003",
      "new_prompt": "优化后的提示词..."
    }
  ]
}
```

### SSE 流式响应

```http
GET /api/assistant/stream?project_id=proj_123456&message=帮我分析剧本
```

响应（SSE 流）：

```
event: message
data: {"type": "text", "content": "正在分析剧本..."}

event: message
data: {"type": "text", "content": "发现3个角色、2个场景"}

event: message
data: {"type": "result", "entities": {...}}

event: done
data: {"success": true}
```

## 10. 配置管理

### 获取供应商配置

```http
GET /api/config/providers
```

响应：

```json
{
  "image": [
    {
      "name": "gemini",
      "enabled": true,
      "models": ["imagen-3.0"],
      "rate_limit": 60
    },
    {
      "name": "wanx",
      "enabled": true,
      "models": ["wanx-v1"],
      "rate_limit": 30
    }
  ],
  "video": [
    {
      "name": "seedance",
      "enabled": true,
      "models": ["seedance-2.0"],
      "rate_limit": 10
    }
  ]
}
```

### 更新供应商配置

```http
PUT /api/config/providers/{provider_name}
Content-Type: application/json

{
  "enabled": true,
  "api_key": "new_api_key",
  "rate_limit": 100
}
```

### 获取项目配置

```http
GET /api/projects/{project_id}/config
```

响应：

```json
{
  "default_providers": {
    "image": "gemini",
    "video": "seedance",
    "audio": "volc",
    "llm": "qwen"
  },
  "quality_settings": {
    "image_resolution": "1024x1024",
    "video_resolution": "1920x1080",
    "video_fps": 24
  },
  "storage": {
    "use_oss": false,
    "local_path": "/output/proj_123456"
  }
}
```

## 11. 统计与分析

### 获取项目统计

```http
GET /api/projects/{project_id}/stats
```

响应：

```json
{
  "total_assets": 15,
  "total_frames": 20,
  "total_clips": 18,
  "total_duration": 120,
  "generation_time": 3600,
  "cost": {
    "image": 2.5,
    "video": 15.8,
    "audio": 1.2,
    "total": 19.5,
    "currency": "USD"
  },
  "provider_usage": {
    "gemini": 45,
    "seedance": 30,
    "kling": 15
  }
}
```

### 获取使用统计

```http
GET /api/stats/usage?start_date=2026-04-01&end_date=2026-04-05
```

响应：

```json
{
  "total_projects": 10,
  "total_tasks": 250,
  "success_rate": 0.95,
  "avg_generation_time": {
    "image": 15,
    "video": 120,
    "audio": 30
  },
  "total_cost": 195.5,
  "daily_breakdown": [
    {
      "date": "2026-04-01",
      "projects": 2,
      "tasks": 50,
      "cost": 38.5
    }
  ]
}
```

## 12. Webhook

### 配置 Webhook

```http
POST /api/webhooks
Content-Type: application/json

{
  "url": "https://your-domain.com/webhook",
  "events": ["task.completed", "task.failed", "project.completed"],
  "secret": "your_webhook_secret"
}
```

响应：

```json
{
  "id": "webhook_001",
  "url": "https://your-domain.com/webhook",
  "events": ["task.completed", "task.failed", "project.completed"],
  "created_at": "2026-04-05T10:00:00Z"
}
```

### Webhook 事件格式

任务完成事件：

```json
{
  "event": "task.completed",
  "timestamp": "2026-04-05T12:00:00Z",
  "data": {
    "task_id": "task_791",
    "project_id": "proj_123456",
    "resource_type": "video",
    "resource_id": "frame_001",
    "result": {
      "url": "/output/proj_123456/videos/frame_001.mp4",
      "duration": 5,
      "size_mb": 12.3
    }
  }
}
```

任务失败事件：

```json
{
  "event": "task.failed",
  "timestamp": "2026-04-05T12:00:00Z",
  "data": {
    "task_id": "task_792",
    "project_id": "proj_123456",
    "resource_type": "video",
    "error": "API rate limit exceeded"
  }
}
```

## 13. 错误处理

### 错误响应格式

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "duration",
      "issue": "Must be between 4 and 15 seconds"
    }
  }
}
```

### 常见错误码

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| `VALIDATION_ERROR` | 400 | 请求参数验证失败 |
| `UNAUTHORIZED` | 401 | 未授权，API Key 无效 |
| `FORBIDDEN` | 403 | 无权限访问资源 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `RATE_LIMIT_EXCEEDED` | 429 | 超过速率限制 |
| `PROVIDER_ERROR` | 502 | AI 服务商错误 |
| `INTERNAL_ERROR` | 500 | 服务器内部错误 |

## 14. 速率限制

API 速率限制：

- 免费用户：100 请求/小时
- 基础用户：1000 请求/小时
- 专业用户：10000 请求/小时

响应头：

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1712318400
```

超过限制时：

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded",
    "retry_after": 3600
  }
}
```

## 15. SDK 示例

### Python SDK

```python
from video_creator import VideoCreatorClient

# 初始化客户端
client = VideoCreatorClient(api_key="your_api_key")

# 创建项目
project = client.projects.create(name="我的视频项目")

# 分析剧本
script = """
在一个阳光明媚的早晨，小明走进了咖啡店...
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

# 等待资产生成完成
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

# 等待视频生成完成
client.tasks.wait_for_completion(project.id)

# 合成最终视频
final_video = client.projects.assemble(project.id)

print(f"视频生成完成: {final_video.url}")
```

### JavaScript SDK

```javascript
import { VideoCreatorClient } from '@video-creator/sdk';

// 初始化客户端
const client = new VideoCreatorClient({
  apiKey: 'your_api_key'
});

// 创建项目
const project = await client.projects.create({
  name: '我的视频项目'
});

// 分析剧本
const script = `
在一个阳光明媚的早晨，小明走进了咖啡店...
`;
const result = await client.scripts.analyze(project.id, { text: script });

// 生成资产
const assetTasks = await Promise.all(
  result.entities.characters.map(character =>
    client.assets.generate({
      projectId: project.id,
      entityId: character.id,
      entityType: 'character',
      numVariants: 4
    })
  )
);

// 监听任务进度
client.tasks.on('progress', (task) => {
  console.log(`任务 ${task.id} 进度: ${task.progress * 100}%`);
});

// 等待完成
await client.tasks.waitForCompletion(project.id);

// 生成分镜
const storyboard = await client.storyboard.generate(project.id);

// 生成视频
const motionTasks = await Promise.all(
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

// 合成最终视频
const finalVideo = await client.projects.assemble(project.id);

console.log(`视频生成完成: ${finalVideo.url}`);
```

---

完整 API 文档请访问：https://docs.video-creator.example.com
