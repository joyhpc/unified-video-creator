# Agent 扩展方案

## 1. 横向扩展 Worker

### 1.1 扩展策略

```yaml
# docker-compose.scale.yml
version: '3.8'

services:
  # Image Worker - 10 实例
  worker-image:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      REDIS_URL: redis://redis:6379
      CHANNEL: image
      GEMINI_API_KEY: ${GEMINI_API_KEY}
      WANX_API_KEY: ${WANX_API_KEY}
    volumes:
      - ./output:/app/output
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    deploy:
      replicas: 10  # 从 2 扩到 10
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

  # Video Worker - 10 实例 (GPU 加速)
  worker-video:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker.gpu
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      REDIS_URL: redis://redis:6379
      CHANNEL: video
      SEEDANCE_API_KEY: ${SEEDANCE_API_KEY}
      KLING_API_KEY: ${KLING_API_KEY}
      USE_GPU: "true"
    volumes:
      - ./output:/app/output
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    deploy:
      replicas: 10
      resources:
        limits:
          cpus: '4'
          memory: 16G
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  # Audio Worker - 5 实例
  worker-audio:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres/video_creator
      REDIS_URL: redis://redis:6379
      CHANNEL: audio
      VOLC_API_KEY: ${VOLC_API_KEY}
    volumes:
      - ./output:/app/output
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    deploy:
      replicas: 5
```

### 1.2 动态扩缩容

```python
# backend/lib/autoscaler.py
import asyncio
from datetime import datetime, timedelta
from sqlalchemy import select, func
from lib.db.models import Task

class WorkerAutoscaler:
    """Worker 自动扩缩容"""
    
    def __init__(
        self,
        db,
        redis,
        min_workers: int = 2,
        max_workers: int = 20,
        scale_up_threshold: int = 10,  # 队列长度超过 10 扩容
        scale_down_threshold: int = 2,  # 队列长度低于 2 缩容
        check_interval: int = 60  # 检查间隔 60 秒
    ):
        self.db = db
        self.redis = redis
        self.min_workers = min_workers
        self.max_workers = max_workers
        self.scale_up_threshold = scale_up_threshold
        self.scale_down_threshold = scale_down_threshold
        self.check_interval = check_interval
    
    async def start(self):
        """启动自动扩缩容"""
        while True:
            await self._check_and_scale()
            await asyncio.sleep(self.check_interval)
    
    async def _check_and_scale(self):
        """检查并执行扩缩容"""
        for channel in ['image', 'video', 'audio']:
            # 获取队列长度
            queue_length = await self._get_queue_length(channel)
            
            # 获取当前 Worker 数量
            current_workers = await self._get_worker_count(channel)
            
            # 决策
            if queue_length > self.scale_up_threshold and current_workers < self.max_workers:
                # 扩容
                target_workers = min(
                    current_workers + 2,
                    self.max_workers
                )
                await self._scale_workers(channel, target_workers)
                print(f"[Autoscaler] {channel} 扩容: {current_workers} -> {target_workers}")
            
            elif queue_length < self.scale_down_threshold and current_workers > self.min_workers:
                # 缩容
                target_workers = max(
                    current_workers - 1,
                    self.min_workers
                )
                await self._scale_workers(channel, target_workers)
                print(f"[Autoscaler] {channel} 缩容: {current_workers} -> {target_workers}")
    
    async def _get_queue_length(self, channel: str) -> int:
        """获取队列长度"""
        result = await self.db.execute(
            select(func.count(Task.id))
            .where(
                Task.resource_type == channel,
                Task.status.in_(['queued', 'running'])
            )
        )
        return result.scalar()
    
    async def _get_worker_count(self, channel: str) -> int:
        """获取当前 Worker 数量"""
        # 从 Redis 获取活跃 Worker 列表
        workers = await self.redis.smembers(f"workers:{channel}")
        return len(workers)
    
    async def _scale_workers(self, channel: str, target_count: int):
        """执行扩缩容"""
        # 调用 Docker API 或 Kubernetes API
        import docker
        client = docker.from_env()
        
        service_name = f"video-creator_worker-{channel}"
        service = client.services.get(service_name)
        
        # 更新副本数
        service.update(
            mode={'Replicated': {'Replicas': target_count}}
        )
```

### 1.3 负载均衡策略

```python
# backend/lib/load_balancer.py
from typing import List, Optional
import random

class LoadBalancer:
    """Worker 负载均衡器"""
    
    def __init__(self, redis):
        self.redis = redis
    
    async def select_worker(
        self,
        channel: str,
        strategy: str = "least_loaded"
    ) -> Optional[str]:
        """
        选择 Worker
        
        策略:
        - least_loaded: 选择负载最低的 Worker
        - round_robin: 轮询
        - random: 随机
        """
        workers = await self._get_active_workers(channel)
        
        if not workers:
            return None
        
        if strategy == "least_loaded":
            return await self._select_least_loaded(workers)
        elif strategy == "round_robin":
            return await self._select_round_robin(channel, workers)
        elif strategy == "random":
            return random.choice(workers)
        else:
            return workers[0]
    
    async def _get_active_workers(self, channel: str) -> List[str]:
        """获取活跃 Worker 列表"""
        workers = await self.redis.smembers(f"workers:{channel}")
        
        # 过滤掉心跳超时的 Worker
        active_workers = []
        for worker_id in workers:
            last_heartbeat = await self.redis.get(f"worker:{worker_id}:heartbeat")
            if last_heartbeat:
                from datetime import datetime, timedelta
                last_time = datetime.fromisoformat(last_heartbeat)
                if datetime.utcnow() - last_time < timedelta(seconds=30):
                    active_workers.append(worker_id)
        
        return active_workers
    
    async def _select_least_loaded(self, workers: List[str]) -> str:
        """选择负载最低的 Worker"""
        loads = {}
        for worker_id in workers:
            load = await self.redis.get(f"worker:{worker_id}:load")
            loads[worker_id] = int(load) if load else 0
        
        return min(loads, key=loads.get)
    
    async def _select_round_robin(self, channel: str, workers: List[str]) -> str:
        """轮询选择 Worker"""
        counter = await self.redis.incr(f"lb:{channel}:counter")
        index = counter % len(workers)
        return workers[index]
```

## 2. 专用 Agent 扩展

### 2.1 提示词优化 Agent

```python
# backend/agents/prompt_optimizer_agent.py
from claude_agent_sdk import Agent, Skill

class PromptOptimizerAgent(Agent):
    """提示词优化 Agent"""
    
    def __init__(self):
        super().__init__(name="prompt-optimizer")
    
    @Skill(name="optimize-prompt")
    async def optimize_prompt(
        self,
        original_prompt: str,
        target_style: str,
        reference_images: List[str]
    ) -> dict:
        """
        优化提示词
        
        策略:
        1. 分析原始提示词的结构
        2. 提取关键元素（主体、动作、场景、风格）
        3. 根据目标风格调整
        4. 添加 Seedance 2.0 关键词
        5. 验证约束条件
        """
        # 分析原始提示词
        analysis = await self._analyze_prompt(original_prompt)
        
        # 提取关键元素
        elements = {
            "subject": analysis.get("subject"),
            "action": analysis.get("action"),
            "scene": analysis.get("scene"),
            "style": analysis.get("style"),
            "camera": analysis.get("camera")
        }
        
        # 根据目标风格优化
        optimized_elements = await self._optimize_by_style(
            elements,
            target_style
        )
        
        # 添加 Seedance 关键词
        enhanced_prompt = await self._add_seedance_keywords(
            optimized_elements,
            reference_images
        )
        
        # 验证约束
        validation = await self._validate_constraints(enhanced_prompt)
        
        return {
            "original": original_prompt,
            "optimized": enhanced_prompt,
            "improvements": [
                "添加了时间轴分段",
                "增强了镜头语言",
                "优化了角色描述"
            ],
            "validation": validation
        }
    
    async def _analyze_prompt(self, prompt: str) -> dict:
        """分析提示词结构"""
        # 使用 LLM 分析
        analysis_prompt = f"""
分析以下提示词，提取关键元素：

提示词: {prompt}

请提取：
1. 主体（人物/物体）
2. 动作
3. 场景
4. 风格
5. 镜头运动
"""
        result = await self.llm.generate(analysis_prompt)
        return self._parse_analysis(result)
    
    async def _optimize_by_style(self, elements: dict, style: str) -> dict:
        """根据风格优化"""
        style_templates = {
            "cinematic": {
                "camera": "电影级镜头，浅景深，专业打光",
                "color": "电影色调，高对比度",
                "motion": "平滑推进，稳定器拍摄"
            },
            "anime": {
                "camera": "动漫风格，夸张透视",
                "color": "鲜艳色彩，高饱和度",
                "motion": "快速切换，动态线条"
            },
            "documentary": {
                "camera": "纪实风格，手持拍摄",
                "color": "自然色调，真实光线",
                "motion": "跟随拍摄，自然晃动"
            }
        }
        
        template = style_templates.get(style, {})
        
        # 合并风格模板
        optimized = elements.copy()
        optimized.update(template)
        
        return optimized
    
    async def _add_seedance_keywords(
        self,
        elements: dict,
        reference_images: List[str]
    ) -> str:
        """添加 Seedance 2.0 关键词"""
        # 构建时间轴
        timeline = f"0-3s: {elements['action']}, 3-5s: {elements['action']} 结束"
        
        # 添加参考图标记
        ref_tags = " ".join([f"@image{i+1}" for i in range(len(reference_images))])
        
        # 组装最终提示词
        prompt = f"""
{timeline}

主体: {elements['subject']} ({ref_tags})
场景: {elements['scene']}
镜头: {elements['camera']}
风格: {elements['style']}
色调: {elements.get('color', '自然色调')}
运动: {elements.get('motion', '平滑运动')}
"""
        return prompt.strip()
    
    async def _validate_constraints(self, prompt: str) -> dict:
        """验证 Seedance 约束"""
        issues = []
        
        # 检查长度
        if len(prompt) > 5000:
            issues.append("提示词长度超过 5000 字符")
        
        # 检查图像数量
        image_count = prompt.count("@image")
        if image_count > 9:
            issues.append(f"图像数量超限: {image_count} > 9")
        
        return {
            "valid": len(issues) == 0,
            "issues": issues
        }
```

### 2.2 质量检查 Agent

```python
# backend/agents/quality_checker_agent.py
from claude_agent_sdk import Agent, Skill
import cv2
import numpy as np

class QualityCheckerAgent(Agent):
    """质量检查 Agent"""
    
    def __init__(self):
        super().__init__(name="quality-checker")
    
    @Skill(name="check-quality")
    async def check_quality(
        self,
        video_path: str,
        quality_standards: dict
    ) -> dict:
        """
        检查视频质量
        
        检查项:
        1. 分辨率
        2. 帧率
        3. 码率
        4. 画面稳定性
        5. 色彩一致性
        6. 音频质量
        """
        results = {
            "overall_score": 0.0,
            "checks": []
        }
        
        # 1. 基础参数检查
        basic_check = await self._check_basic_params(video_path, quality_standards)
        results["checks"].append(basic_check)
        
        # 2. 画面质量检查
        visual_check = await self._check_visual_quality(video_path)
        results["checks"].append(visual_check)
        
        # 3. 音频质量检查
        audio_check = await self._check_audio_quality(video_path)
        results["checks"].append(audio_check)
        
        # 4. 角色一致性检查
        consistency_check = await self._check_character_consistency(video_path)
        results["checks"].append(consistency_check)
        
        # 计算总分
        results["overall_score"] = np.mean([
            check["score"] for check in results["checks"]
        ])
        
        # 判定
        results["passed"] = results["overall_score"] >= 0.8
        
        return results
    
    async def _check_basic_params(self, video_path: str, standards: dict) -> dict:
        """检查基础参数"""
        cap = cv2.VideoCapture(video_path)
        
        width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
        fps = cap.get(cv2.CAP_PROP_FPS)
        frame_count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
        
        cap.release()
        
        # 检查分辨率
        expected_resolution = standards.get("resolution", "1920x1080")
        expected_w, expected_h = map(int, expected_resolution.split("x"))
        resolution_match = (width == expected_w and height == expected_h)
        
        # 检查帧率
        expected_fps = standards.get("fps", 24)
        fps_match = abs(fps - expected_fps) < 1
        
        score = (resolution_match + fps_match) / 2
        
        return {
            "name": "基础参数",
            "score": score,
            "details": {
                "resolution": f"{width}x{height}",
                "fps": fps,
                "frame_count": frame_count,
                "resolution_match": resolution_match,
                "fps_match": fps_match
            }
        }
    
    async def _check_visual_quality(self, video_path: str) -> dict:
        """检查画面质量"""
        cap = cv2.VideoCapture(video_path)
        
        # 采样帧
        frame_count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
        sample_indices = np.linspace(0, frame_count - 1, 10, dtype=int)
        
        sharpness_scores = []
        brightness_scores = []
        
        for idx in sample_indices:
            cap.set(cv2.CAP_PROP_POS_FRAMES, idx)
            ret, frame = cap.read()
            if not ret:
                continue
            
            # 检查清晰度（Laplacian 方差）
            gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
            laplacian_var = cv2.Laplacian(gray, cv2.CV_64F).var()
            sharpness_scores.append(laplacian_var)
            
            # 检查亮度
            brightness = np.mean(gray)
            brightness_scores.append(brightness)
        
        cap.release()
        
        # 评分
        avg_sharpness = np.mean(sharpness_scores)
        avg_brightness = np.mean(brightness_scores)
        
        # 清晰度评分（> 100 为清晰）
        sharpness_score = min(avg_sharpness / 100, 1.0)
        
        # 亮度评分（80-180 为正常）
        brightness_score = 1.0 if 80 <= avg_brightness <= 180 else 0.5
        
        score = (sharpness_score + brightness_score) / 2
        
        return {
            "name": "画面质量",
            "score": score,
            "details": {
                "sharpness": avg_sharpness,
                "brightness": avg_brightness,
                "sharpness_score": sharpness_score,
                "brightness_score": brightness_score
            }
        }
    
    async def _check_audio_quality(self, video_path: str) -> dict:
        """检查音频质量"""
        import subprocess
        
        # 使用 ffprobe 获取音频信息
        cmd = [
            'ffprobe',
            '-v', 'error',
            '-select_streams', 'a:0',
            '-show_entries', 'stream=codec_name,sample_rate,channels',
            '-of', 'json',
            video_path
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        if result.returncode != 0:
            return {
                "name": "音频质量",
                "score": 0.0,
                "details": {"error": "无音频流"}
            }
        
        import json
        data = json.loads(result.stdout)
        
        if not data.get("streams"):
            return {
                "name": "音频质量",
                "score": 0.0,
                "details": {"error": "无音频流"}
            }
        
        audio_info = data["streams"][0]
        
        # 检查采样率（44100 或 48000 为标准）
        sample_rate = int(audio_info.get("sample_rate", 0))
        sample_rate_ok = sample_rate in [44100, 48000]
        
        # 检查声道数（1 或 2 为正常）
        channels = int(audio_info.get("channels", 0))
        channels_ok = channels in [1, 2]
        
        score = (sample_rate_ok + channels_ok) / 2
        
        return {
            "name": "音频质量",
            "score": score,
            "details": {
                "codec": audio_info.get("codec_name"),
                "sample_rate": sample_rate,
                "channels": channels,
                "sample_rate_ok": sample_rate_ok,
                "channels_ok": channels_ok
            }
        }
    
    async def _check_character_consistency(self, video_path: str) -> dict:
        """检查角色一致性（使用人脸识别）"""
        # 这里简化实现，实际可以使用 face_recognition 库
        # 检查视频中人脸特征的一致性
        
        return {
            "name": "角色一致性",
            "score": 0.9,  # 占位符
            "details": {
                "faces_detected": 1,
                "consistency_score": 0.9
            }
        }
```

### 2.3 风格迁移 Agent

```python
# backend/agents/style_transfer_agent.py
from claude_agent_sdk import Agent, Skill

class StyleTransferAgent(Agent):
    """风格迁移 Agent"""
    
    def __init__(self):
        super().__init__(name="style-transfer")
    
    @Skill(name="transfer-style")
    async def transfer_style(
        self,
        source_video: str,
        target_style: str,
        reference_images: List[str] = None
    ) -> dict:
        """
        风格迁移
        
        支持的风格:
        - anime: 动漫风格
        - oil_painting: 油画风格
        - sketch: 素描风格
        - cyberpunk: 赛博朋克风格
        - watercolor: 水彩风格
        """
        # 1. 提取源视频的关键帧
        keyframes = await self._extract_keyframes(source_video)
        
        # 2. 对每个关键帧应用风格迁移
        styled_frames = []
        for frame in keyframes:
            styled_frame = await self._apply_style(
                frame,
                target_style,
                reference_images
            )
            styled_frames.append(styled_frame)
        
        # 3. 重建视频
        output_video = await self._reconstruct_video(
            source_video,
            styled_frames
        )
        
        return {
            "output_video": output_video,
            "style": target_style,
            "keyframes_processed": len(keyframes)
        }
    
    async def _extract_keyframes(self, video_path: str) -> List[str]:
        """提取关键帧"""
        import subprocess
        import tempfile
        import os
        
        temp_dir = tempfile.mkdtemp()
        
        # 使用 ffmpeg 提取关键帧
        cmd = [
            'ffmpeg',
            '-i', video_path,
            '-vf', 'select=eq(pict_type\\,I)',
            '-vsync', 'vfr',
            f'{temp_dir}/keyframe_%04d.png'
        ]
        
        subprocess.run(cmd, capture_output=True)
        
        # 获取所有关键帧路径
        keyframes = sorted([
            os.path.join(temp_dir, f)
            for f in os.listdir(temp_dir)
            if f.endswith('.png')
        ])
        
        return keyframes
    
    async def _apply_style(
        self,
        frame_path: str,
        style: str,
        reference_images: List[str]
    ) -> str:
        """应用风格迁移"""
        # 调用图像生成 API（如 Stable Diffusion img2img）
        # 或使用专门的风格迁移模型
        
        style_prompts = {
            "anime": "anime style, vibrant colors, cel shading",
            "oil_painting": "oil painting style, impressionist, brush strokes",
            "sketch": "pencil sketch, black and white, detailed lines",
            "cyberpunk": "cyberpunk style, neon lights, futuristic",
            "watercolor": "watercolor painting, soft colors, paper texture"
        }
        
        prompt = style_prompts.get(style, "artistic style")
        
        # 调用 img2img API
        result = await self.image_backend.img2img(
            image=frame_path,
            prompt=prompt,
            strength=0.7  # 保留 30% 原图结构
        )
        
        return result["output_path"]
    
    async def _reconstruct_video(
        self,
        source_video: str,
        styled_frames: List[str]
    ) -> str:
        """重建视频"""
        import subprocess
        import tempfile
        
        # 创建临时输出文件
        output_path = tempfile.mktemp(suffix='.mp4')
        
        # 使用 ffmpeg 重建视频
        # 1. 从风格化的关键帧插值生成中间帧
        # 2. 合并音频
        
        cmd = [
            'ffmpeg',
            '-i', source_video,
            '-i', f'{styled_frames[0]}',  # 简化：这里应该是帧序列
            '-c:v', 'libx264',
            '-c:a', 'copy',
            output_path
        ]
        
        subprocess.run(cmd, capture_output=True)
        
        return output_path
```

## 3. 分布式部署

### 3.1 多机部署架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Load Balancer (Nginx)                   │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐   ┌────────▼────────┐   ┌──────▼──────────┐
│  API Server 1  │   │  API Server 2   │   │  API Server 3   │
│  (FastAPI)     │   │  (FastAPI)      │   │  (FastAPI)      │
└────────────────┘   └─────────────────┘   └─────────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐   ┌────────▼────────┐   ┌──────▼──────────┐
│  PostgreSQL    │   │     Redis       │   │   Shared NFS    │
│  (Primary)     │   │   (Cluster)     │   │   /output       │
└────────────────┘   └─────────────────┘   └─────────────────┘
        │
        │ Replication
        │
┌───────▼────────┐
│  PostgreSQL    │
│  (Replica)     │
└────────────────┘

Worker Cluster:
┌─────────────────────────────────────────────────────────────┐
│  Machine 1 (Image Workers)                                   │
│  ├─ worker-image-1  ├─ worker-image-2  ├─ worker-image-3   │
│  ├─ worker-image-4  ├─ worker-image-5                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Machine 2 (Video Workers - GPU)                             │
│  ├─ worker-video-1  ├─ worker-video-2  ├─ worker-video-3   │
│  └─ 4x NVIDIA A100 GPUs                                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Machine 3 (Audio Workers)                                   │
│  ├─ worker-audio-1  ├─ worker-audio-2  ├─ worker-audio-3   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Machine 4 (Specialized Agents)                              │
│  ├─ prompt-optimizer  ├─ quality-checker                    │
│  ├─ style-transfer    ├─ video-assembler                    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Kubernetes 部署配置

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: video-creator-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: video-creator-api
  template:
    metadata:
      labels:
        app: video-creator-api
    spec:
      containers:
      - name: api
        image: video-creator/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: REDIS_URL
          value: redis://redis-service:6379
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-image
spec:
  replicas: 10
  selector:
    matchLabels:
      app: worker-image
  template:
    metadata:
      labels:
        app: worker-image
    spec:
      containers:
      - name: worker
        image: video-creator/worker:latest
        env:
        - name: CHANNEL
          value: "image"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: REDIS_URL
          value: redis://redis-service:6379
        - name: GEMINI_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: gemini
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        volumeMounts:
        - name: output
          mountPath: /app/output
      volumes:
      - name: output
        persistentVolumeClaim:
          claimName: output-pvc

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-video
spec:
  replicas: 5
  selector:
    matchLabels:
      app: worker-video
  template:
    metadata:
      labels:
        app: worker-video
    spec:
      nodeSelector:
        gpu: "true"  # 调度到 GPU 节点
      containers:
      - name: worker
        image: video-creator/worker-gpu:latest
        env:
        - name: CHANNEL
          value: "video"
        - name: USE_GPU
          value: "true"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: REDIS_URL
          value: redis://redis-service:6379
        - name: SEEDANCE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: seedance
        resources:
          requests:
            memory: "8Gi"
            cpu: "2000m"
            nvidia.com/gpu: 1
          limits:
            memory: "16Gi"
            cpu: "4000m"
            nvidia.com/gpu: 1
        volumeMounts:
        - name: output
          mountPath: /app/output
      volumes:
      - name: output
        persistentVolumeClaim:
          claimName: output-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: video-creator-api
spec:
  selector:
    app: video-creator-api
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: LoadBalancer

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-image-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker-image
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "5"
```

### 3.3 共享存储配置

```yaml
# k8s/storage.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: output-pv
spec:
  capacity:
    storage: 1Ti
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /exports/video-creator/output
  mountOptions:
    - hard
    - nfsvers=4.1

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: output-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Ti
  storageClassName: ""
  volumeName: output-pv
```

### 3.4 服务发现与注册

```python
# backend/lib/service_discovery.py
import asyncio
from typing import Dict, List
import consul

class ServiceRegistry:
    """服务注册与发现"""
    
    def __init__(self, consul_host: str = "localhost", consul_port: int = 8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
        self.service_id = None
    
    async def register_worker(
        self,
        worker_id: str,
        channel: str,
        host: str,
        port: int,
        tags: List[str] = None
    ):
        """注册 Worker"""
        self.service_id = f"worker-{channel}-{worker_id}"
        
        self.consul.agent.service.register(
            name=f"worker-{channel}",
            service_id=self.service_id,
            address=host,
            port=port,
            tags=tags or [],
            check=consul.Check.ttl("30s")
        )
        
        # 启动心跳
        asyncio.create_task(self._heartbeat_loop())
    
    async def _heartbeat_loop(self):
        """心跳循环"""
        while True:
            try:
                self.consul.agent.check.ttl_pass(f"service:{self.service_id}")
            except Exception as e:
                print(f"Heartbeat failed: {e}")
            
            await asyncio.sleep(10)
    
    async def discover_workers(self, channel: str) -> List[Dict]:
        """发现 Workers"""
        _, services = self.consul.health.service(
            f"worker-{channel}",
            passing=True
        )
        
        workers = []
        for service in services:
            workers.append({
                "id": service["Service"]["ID"],
                "address": service["Service"]["Address"],
                "port": service["Service"]["Port"],
                "tags": service["Service"]["Tags"]
            })
        
        return workers
    
    async def deregister(self):
        """注销服务"""
        if self.service_id:
            self.consul.agent.service.deregister(self.service_id)
```

## 4. GPU Worker 池

### 4.1 GPU Worker 实现

```python
# backend/lib/gpu_worker.py
import torch
from lib.generation_worker import GenerationWorker

class GPUWorker(GenerationWorker):
    """GPU 加速 Worker"""
    
    def __init__(
        self,
        worker_id: str,
        channel: str,
        queue,
        provider_pool,
        gpu_id: int = 0
    ):
        super().__init__(worker_id, channel, queue, provider_pool)
        self.gpu_id = gpu_id
        self.device = f"cuda:{gpu_id}"
        
        # 检查 GPU 可用性
        if not torch.cuda.is_available():
            raise RuntimeError("CUDA not available")
        
        # 设置 GPU
        torch.cuda.set_device(gpu_id)
        
        print(f"[GPUWorker] 初始化 GPU {gpu_id}: {torch.cuda.get_device_name(gpu_id)}")
    
    async def start(self):
        """启动 GPU Worker"""
        # 预热 GPU
        await self._warmup_gpu()
        
        # 启动主循环
        await super().start()
    
    async def _warmup_gpu(self):
        """预热 GPU"""
        print(f"[GPUWorker] 预热 GPU {self.gpu_id}...")
        
        # 运行一个小任务预热
        dummy_tensor = torch.randn(1000, 1000, device=self.device)
        _ = torch.matmul(dummy_tensor, dummy_tensor)
        
        torch.cuda.synchronize()
        print(f"[GPUWorker] GPU {self.gpu_id} 预热完成")
    
    async def _execute_with_heartbeat(self, task):
        """执行任务（GPU 加速）"""
        # 监控 GPU 使用情况
        gpu_monitor = asyncio.create_task(self._monitor_gpu())
        
        try:
            result = await super()._execute_with_heartbeat(task)
            return result
        finally:
            gpu_monitor.cancel()
    
    async def _monitor_gpu(self):
        """监控 GPU 使用情况"""
        while True:
            # 获取 GPU 内存使用
            memory_allocated = torch.cuda.memory_allocated(self.gpu_id) / 1024**3
            memory_reserved = torch.cuda.memory_reserved(self.gpu_id) / 1024**3
            
            # 获取 GPU 利用率
            utilization = torch.cuda.utilization(self.gpu_id)
            
            # 上报到 Redis
            await self.redis.hset(
                f"worker:{self.worker_id}:gpu",
                mapping={
                    "memory_allocated_gb": memory_allocated,
                    "memory_reserved_gb": memory_reserved,
                    "utilization": utilization
                }
            )
            
            await asyncio.sleep(5)
```

### 4.2 GPU 池管理

```python
# backend/lib/gpu_pool.py
import torch
from typing import List, Optional
import asyncio

class GPUPool:
    """GPU 池管理器"""
    
    def __init__(self):
        self.gpu_count = torch.cuda.device_count()
        self.gpu_workers: Dict[int, List[GPUWorker]] = {}
        self.gpu_locks: Dict[int, asyncio.Lock] = {}
        
        for gpu_id in range(self.gpu_count):
            self.gpu_workers[gpu_id] = []
            self.gpu_locks[gpu_id] = asyncio.Lock()
        
        print(f"[GPUPool] 检测到 {self.gpu_count} 个 GPU")
    
    async def allocate_gpu(self, channel: str) -> Optional[int]:
        """分配 GPU"""
        # 选择负载最低的 GPU
        min_load = float('inf')
        selected_gpu = None
        
        for gpu_id in range(self.gpu_count):
            # 获取当前 GPU 的 Worker 数量
            worker_count = len(self.gpu_workers[gpu_id])
            
            # 获取 GPU 内存使用情况
            memory_allocated = torch.cuda.memory_allocated(gpu_id) / 1024**3
            
            # 计算负载（Worker 数量 + 内存使用）
            load = worker_count + memory_allocated / 10
            
            if load < min_load:
                min_load = load
                selected_gpu = gpu_id
        
        return selected_gpu
    
    async def register_worker(self, gpu_id: int, worker: GPUWorker):
        """注册 Worker 到 GPU"""
        async with self.gpu_locks[gpu_id]:
            self.gpu_workers[gpu_id].append(worker)
    
    async def unregister_worker(self, gpu_id: int, worker: GPUWorker):
        """从 GPU 注销 Worker"""
        async with self.gpu_locks[gpu_id]:
            if worker in self.gpu_workers[gpu_id]:
                self.gpu_workers[gpu_id].remove(worker)
    
    async def get_gpu_stats(self) -> List[Dict]:
        """获取所有 GPU 统计信息"""
        stats = []
        
        for gpu_id in range(self.gpu_count):
            memory_allocated = torch.cuda.memory_allocated(gpu_id) / 1024**3
            memory_reserved = torch.cuda.memory_reserved(gpu_id) / 1024**3
            memory_total = torch.cuda.get_device_properties(gpu_id).total_memory / 1024**3
            
            stats.append({
                "gpu_id": gpu_id,
                "name": torch.cuda.get_device_name(gpu_id),
                "worker_count": len(self.gpu_workers[gpu_id]),
                "memory_allocated_gb": memory_allocated,
                "memory_reserved_gb": memory_reserved,
                "memory_total_gb": memory_total,
                "memory_usage_percent": (memory_allocated / memory_total) * 100
            })
        
        return stats
```

### 4.3 GPU Worker 启动脚本

```python
# backend/start_gpu_workers.py
import asyncio
import torch
from lib.gpu_worker import GPUWorker
from lib.gpu_pool import GPUPool
from lib.generation_queue import GenerationQueue
from lib.provider_pool import ProviderPool

async def main():
    # 初始化 GPU 池
    gpu_pool = GPUPool()
    
    # 初始化队列和供应商池
    queue = GenerationQueue(db)
    provider_pool = ProviderPool()
    
    # 为每个 GPU 启动多个 Worker
    workers_per_gpu = 2
    workers = []
    
    for gpu_id in range(gpu_pool.gpu_count):
        for i in range(workers_per_gpu):
            worker_id = f"gpu{gpu_id}-worker{i}"
            
            worker = GPUWorker(
                worker_id=worker_id,
                channel="video",
                queue=queue,
                provider_pool=provider_pool,
                gpu_id=gpu_id
            )
            
            await gpu_pool.register_worker(gpu_id, worker)
            workers.append(worker)
            
            print(f"[Main] 启动 Worker {worker_id} on GPU {gpu_id}")
    
    # 启动所有 Workers
    await asyncio.gather(*[worker.start() for worker in workers])

if __name__ == "__main__":
    asyncio.run(main())
```

### 4.4 GPU 监控面板

```python
# backend/routers/gpu_stats.py
from fastapi import APIRouter, Depends
from lib.gpu_pool import GPUPool

router = APIRouter(prefix="/api/gpu", tags=["GPU"])

@router.get("/stats")
async def get_gpu_stats(gpu_pool: GPUPool = Depends()):
    """获取 GPU 统计信息"""
    stats = await gpu_pool.get_gpu_stats()
    return {"gpus": stats}

@router.get("/workers")
async def get_gpu_workers(gpu_pool: GPUPool = Depends()):
    """获取 GPU Worker 分布"""
    distribution = {}
    
    for gpu_id, workers in gpu_pool.gpu_workers.items():
        distribution[f"gpu_{gpu_id}"] = [
            {
                "worker_id": w.worker_id,
                "channel": w.channel,
                "status": "running"
            }
            for w in workers
        ]
    
    return {"distribution": distribution}
```

## 5. 性能优化

### 5.1 批处理优化

```python
# backend/lib/batch_processor.py
from typing import List
import asyncio

class BatchProcessor:
    """批处理器 - 合并多个小任务"""
    
    def __init__(
        self,
        batch_size: int = 10,
        batch_timeout: float = 5.0
    ):
        self.batch_size = batch_size
        self.batch_timeout = batch_timeout
        self.pending_tasks: List = []
        self.lock = asyncio.Lock()
    
    async def add_task(self, task):
        """添加任务到批处理队列"""
        async with self.lock:
            self.pending_tasks.append(task)
            
            # 如果达到批大小，立即处理
            if len(self.pending_tasks) >= self.batch_size:
                await self._process_batch()
    
    async def _process_batch(self):
        """处理一批任务"""
        if not self.pending_tasks:
            return
        
        batch = self.pending_tasks[:self.batch_size]
        self.pending_tasks = self.pending_tasks[self.batch_size:]
        
        # 批量调用 API
        results = await self._batch_generate(batch)
        
        # 分发结果
        for task, result in zip(batch, results):
            await self._complete_task(task, result)
    
    async def _batch_generate(self, tasks: List) -> List:
        """批量生成（调用支持批处理的 API）"""
        # 例如：Gemini Batch API
        prompts = [task.params["prompt"] for task in tasks]
        
        results = await self.provider.batch_generate(prompts)
        
        return results
    
    async def start_timer(self):
        """启动定时器，定期处理未满批的任务"""
        while True:
            await asyncio.sleep(self.batch_timeout)
            
            async with self.lock:
                if self.pending_tasks:
                    await self._process_batch()
```

### 5.2 缓存优化

```python
# backend/lib/result_cache.py
import hashlib
import json

class ResultCache:
    """结果缓存 - 避免重复生成"""
    
    def __init__(self, redis):
        self.redis = redis
        self.ttl = 86400 * 7  # 7 天
    
    async def get(self, task_params: dict) -> Optional[dict]:
        """获取缓存结果"""
        cache_key = self._generate_key(task_params)
        
        cached = await self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        return None
    
    async def set(self, task_params: dict, result: dict):
        """设置缓存结果"""
        cache_key = self._generate_key(task_params)
        
        await self.redis.setex(
            cache_key,
            self.ttl,
            json.dumps(result)
        )
    
    def _generate_key(self, params: dict) -> str:
        """生成缓存 key"""
        # 排序参数确保一致性
        sorted_params = json.dumps(params, sort_keys=True)
        
        # 生成 hash
        hash_obj = hashlib.sha256(sorted_params.encode())
        
        return f"result_cache:{hash_obj.hexdigest()}"
```

## 6. 监控与告警

### 6.1 Prometheus 指标

```python
# backend/lib/metrics.py
from prometheus_client import Counter, Histogram, Gauge, Info

# Worker 指标
worker_info = Info('worker', 'Worker information')
worker_tasks_total = Counter(
    'worker_tasks_total',
    'Total tasks processed',
    ['worker_id', 'channel', 'status']
)
worker_task_duration = Histogram(
    'worker_task_duration_seconds',
    'Task execution duration',
    ['worker_id', 'channel', 'provider']
)

# GPU 指标
gpu_memory_usage = Gauge(
    'gpu_memory_usage_bytes',
    'GPU memory usage',
    ['gpu_id']
)
gpu_utilization = Gauge(
    'gpu_utilization_percent',
    'GPU utilization',
    ['gpu_id']
)

# 队列指标
queue_length = Gauge(
    'queue_length',
    'Number of tasks in queue',
    ['channel', 'status']
)
queue_wait_time = Histogram(
    'queue_wait_time_seconds',
    'Time tasks spend in queue',
    ['channel']
)
```

### 6.2 告警规则

```yaml
# prometheus/alerts.yml
groups:
- name: video_creator_alerts
  interval: 30s
  rules:
  # 队列积压告警
  - alert: QueueBacklog
    expr: queue_length{status="queued"} > 50
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "队列积压 ({{ $labels.channel }})"
      description: "{{ $labels.channel }} 队列长度 {{ $value }}"
  
  # Worker 失败率告警
  - alert: HighFailureRate
    expr: |
      rate(worker_tasks_total{status="failed"}[5m]) /
      rate(worker_tasks_total[5m]) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Worker 失败率过高"
      description: "失败率 {{ $value | humanizePercentage }}"
  
  # GPU 内存告警
  - alert: GPUMemoryHigh
    expr: gpu_memory_usage_bytes / gpu_memory_total_bytes > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "GPU {{ $labels.gpu_id }} 内存使用过高"
      description: "使用率 {{ $value | humanizePercentage }}"
```

## 总结

通过以上扩展方案，系统可以实现：

1. **横向扩展**: Worker 从 2 个扩到 20+ 个实例
2. **专用 Agent**: 提示词优化、质量检查、风格迁移
3. **分布式部署**: Kubernetes + 多机部署
4. **GPU 加速**: GPU Worker 池 + 智能调度
5. **性能优化**: 批处理 + 缓存 + 监控告警

核心优势：Lease 调度 + 独立通道设计，天然支持无限水平扩展。
