# 统一视频创作平台 - 实现细节

## 1. 智能提示词编译器

### 核心算法

```python
class PromptCompiler:
    """智能提示词编译器 - 3层融合 + 时间轴 + Seedance约束"""
    
    def __init__(self):
        self.ref_collector = RefCollector()
        self.seedance_validator = SeedanceValidator()
    
    def compile(
        self,
        scene: Scene,
        characters: List[Character],
        props: List[Prop],
        camera_motion: str,
        dialogue: Optional[str] = None
    ) -> CompiledPrompt:
        """
        编译分镜提示词
        
        Returns:
            CompiledPrompt: 包含 prompt, images, videos, audios
        """
        # 第1层: 收集所有参考资料
        refs = self.ref_collector.collect_all_refs(
            scene=scene,
            characters=characters,
            props=props
        )
        
        # 第2层: 构建时间轴提示词
        timeline_prompt = self._build_timeline_prompt(
            scene=scene,
            camera_motion=camera_motion,
            dialogue=dialogue
        )
        
        # 第3层: 融合角色/场景描述
        character_desc = self._build_character_desc(characters, refs)
        scene_desc = self._build_scene_desc(scene, refs)
        
        # 组装最终提示词
        final_prompt = f"""
{timeline_prompt}

角色: {character_desc}
场景: {scene_desc}
镜头运动: {camera_motion}
"""
        
        if dialogue:
            final_prompt += f"\n对白: {dialogue} (唇形同步)"
        
        # Seedance 2.0 约束校验
        compiled = CompiledPrompt(
            prompt=final_prompt,
            images=refs.images,
            videos=refs.videos,
            audios=refs.audios
        )
        
        self.seedance_validator.validate(compiled)
        
        return compiled
    
    def _build_timeline_prompt(
        self,
        scene: Scene,
        camera_motion: str,
        dialogue: Optional[str]
    ) -> str:
        """构建时间轴分段提示词"""
        duration = scene.duration  # 4-15s
        
        # 根据时长分段
        if duration <= 5:
            # 短片段: 单一动作
            return f"0-{duration}s: {scene.action}"
        elif duration <= 10:
            # 中片段: 起承转
            mid = duration / 2
            return f"""
0-{mid:.1f}s: {scene.action_start}
{mid:.1f}s-{duration}s: {scene.action_end}
"""
        else:
            # 长片段: 起承转合
            t1 = duration / 3
            t2 = duration * 2 / 3
            return f"""
0-{t1:.1f}s: {scene.action_start}
{t1:.1f}s-{t2:.1f}s: {scene.action_middle}
{t2:.1f}s-{duration}s: {scene.action_end}
"""
    
    def _build_character_desc(
        self,
        characters: List[Character],
        refs: CollectedRefs
    ) -> str:
        """构建角色描述 + @image 标记"""
        descs = []
        for char in characters:
            # 获取角色参考图
            char_images = [
                img for img in refs.images 
                if img.resource_id == char.id
            ]
            
            if char_images:
                # 使用参考图
                img_tags = " ".join([
                    f"@{img.material_name}" 
                    for img in char_images[:3]  # 最多3张
                ])
                descs.append(f"{char.name} ({img_tags})")
            else:
                # 纯文本描述
                descs.append(f"{char.name} ({char.description})")
        
        return ", ".join(descs)
    
    def _build_scene_desc(
        self,
        scene: Scene,
        refs: CollectedRefs
    ) -> str:
        """构建场景描述 + @image 标记"""
        # 获取场景参考图
        scene_images = [
            img for img in refs.images 
            if img.resource_id == scene.id
        ]
        
        if scene_images:
            img_tags = " ".join([
                f"@{img.material_name}" 
                for img in scene_images[:3]
            ])
            return f"{scene.description} ({img_tags})"
        else:
            return scene.description


class SeedanceValidator:
    """Seedance 2.0 参数约束校验器"""
    
    MAX_IMAGES = 9
    MAX_VIDEOS = 3
    MAX_AUDIOS = 3
    MAX_PROMPT_LENGTH = 5000
    MAX_IMAGE_SIZE_MB = 30
    MAX_VIDEO_SIZE_MB = 50
    MAX_AUDIO_SIZE_MB = 15
    
    def validate(self, compiled: CompiledPrompt):
        """校验 Seedance 2.0 约束"""
        # 数量约束
        if len(compiled.images) > self.MAX_IMAGES:
            raise ValidationError(
                f"图像数量超限: {len(compiled.images)} > {self.MAX_IMAGES}"
            )
        
        if len(compiled.videos) > self.MAX_VIDEOS:
            raise ValidationError(
                f"视频数量超限: {len(compiled.videos)} > {self.MAX_VIDEOS}"
            )
        
        if len(compiled.audios) > self.MAX_AUDIOS:
            raise ValidationError(
                f"音频数量超限: {len(compiled.audios)} > {self.MAX_AUDIOS}"
            )
        
        # 总数约束
        total = len(compiled.images) + len(compiled.videos) + len(compiled.audios)
        if total > 12:
            raise ValidationError(
                f"总材料数超限: {total} > 12"
            )
        
        # 提示词长度约束
        if len(compiled.prompt) > self.MAX_PROMPT_LENGTH:
            raise ValidationError(
                f"提示词长度超限: {len(compiled.prompt)} > {self.MAX_PROMPT_LENGTH}"
            )
        
        # 文件大小约束
        for img in compiled.images:
            if img.size_mb > self.MAX_IMAGE_SIZE_MB:
                raise ValidationError(
                    f"图像 {img.material_name} 大小超限: {img.size_mb}MB > {self.MAX_IMAGE_SIZE_MB}MB"
                )
        
        for vid in compiled.videos:
            if vid.size_mb > self.MAX_VIDEO_SIZE_MB:
                raise ValidationError(
                    f"视频 {vid.material_name} 大小超限: {vid.size_mb}MB > {self.MAX_VIDEO_SIZE_MB}MB"
                )
        
        for aud in compiled.audios:
            if aud.size_mb > self.MAX_AUDIO_SIZE_MB:
                raise ValidationError(
                    f"音频 {aud.material_name} 大小超限: {aud.size_mb}MB > {self.MAX_AUDIO_SIZE_MB}MB"
                )


class RefCollector:
    """参考资料收集器"""
    
    def collect_all_refs(
        self,
        scene: Scene,
        characters: List[Character],
        props: List[Prop]
    ) -> CollectedRefs:
        """自动收集所有相关参考资料"""
        refs = CollectedRefs()
        
        # 收集角色参考图
        for char in characters:
            # 获取角色的最佳变体（用户选择或评分最高）
            best_variant = self._get_best_variant(char.image_variants)
            if best_variant:
                refs.add_image(
                    resource_id=char.id,
                    material_name=f"character_{char.name}",
                    path=best_variant.path,
                    size_mb=best_variant.size_mb
                )
        
        # 收集场景参考图
        if scene.image_variants:
            best_variant = self._get_best_variant(scene.image_variants)
            if best_variant:
                refs.add_image(
                    resource_id=scene.id,
                    material_name=f"scene_{scene.name}",
                    path=best_variant.path,
                    size_mb=best_variant.size_mb
                )
        
        # 收集道具参考图
        for prop in props:
            best_variant = self._get_best_variant(prop.image_variants)
            if best_variant:
                refs.add_image(
                    resource_id=prop.id,
                    material_name=f"prop_{prop.name}",
                    path=best_variant.path,
                    size_mb=best_variant.size_mb
                )
        
        # 收集首帧图（如果有前一个分镜）
        if scene.previous_scene and scene.previous_scene.video_output:
            refs.add_video(
                resource_id=scene.previous_scene.id,
                material_name=f"previous_scene",
                path=scene.previous_scene.video_output.path,
                size_mb=scene.previous_scene.video_output.size_mb
            )
        
        return refs
    
    def _get_best_variant(self, variants: List[ImageVariant]) -> Optional[ImageVariant]:
        """获取最佳变体（用户选择 > 评分最高 > 第一个）"""
        if not variants:
            return None
        
        # 优先返回用户选择的
        selected = [v for v in variants if v.is_selected]
        if selected:
            return selected[0]
        
        # 其次返回评分最高的
        if any(v.score for v in variants):
            return max(variants, key=lambda v: v.score or 0)
        
        # 最后返回第一个
        return variants[0]
```

## 2. 异步任务队列 + Lease 调度

### 核心实现

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update
from datetime import datetime, timedelta
import asyncio

class GenerationQueue:
    """异步任务队列 - Lease 调度 + 去重"""
    
    LEASE_TTL_SECONDS = 10
    HEARTBEAT_INTERVAL_SECONDS = 3
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def enqueue_task(
        self,
        resource_type: str,  # "image" | "video" | "audio"
        resource_id: str,
        provider: str,
        params: dict
    ) -> str:
        """
        入队任务（带去重）
        
        Returns:
            task_id: 任务ID（新建或已存在）
        """
        # 去重检查
        existing = await self.db.execute(
            select(Task).where(
                Task.resource_type == resource_type,
                Task.resource_id == resource_id,
                Task.status.in_(["queued", "running", "succeeded"])
            )
        )
        existing_task = existing.scalar_one_or_none()
        
        if existing_task:
            return existing_task.id
        
        # 创建新任务
        task = Task(
            resource_type=resource_type,
            resource_id=resource_id,
            provider=provider,
            params=params,
            status="queued",
            created_at=datetime.utcnow()
        )
        self.db.add(task)
        await self.db.commit()
        
        return task.id
    
    async def claim_next_task(
        self,
        worker_id: str,
        channel: str  # "image" | "video" | "audio"
    ) -> Optional[Task]:
        """
        Worker 认领下一个任务（Lease 机制）
        
        Returns:
            Task: 认领成功的任务，或 None
        """
        now = datetime.utcnow()
        lease_expires_at = now + timedelta(seconds=self.LEASE_TTL_SECONDS)
        
        # 查找可认领的任务
        # 1. 状态为 queued
        # 2. 或状态为 running 但 lease 已过期（Worker 崩溃）
        result = await self.db.execute(
            select(Task)
            .where(
                Task.resource_type == channel,
                (
                    (Task.status == "queued") |
                    (
                        (Task.status == "running") &
                        (Task.lease_expires_at < now)
                    )
                )
            )
            .order_by(Task.created_at)
            .limit(1)
            .with_for_update(skip_locked=True)  # 行锁 + 跳过已锁定
        )
        task = result.scalar_one_or_none()
        
        if not task:
            return None
        
        # 更新任务状态
        task.status = "running"
        task.worker_id = worker_id
        task.lease_expires_at = lease_expires_at
        task.started_at = now
        
        await self.db.commit()
        
        return task
    
    async def heartbeat(self, task_id: str, worker_id: str):
        """Worker 心跳续期"""
        now = datetime.utcnow()
        lease_expires_at = now + timedelta(seconds=self.LEASE_TTL_SECONDS)
        
        await self.db.execute(
            update(Task)
            .where(
                Task.id == task_id,
                Task.worker_id == worker_id,
                Task.status == "running"
            )
            .values(lease_expires_at=lease_expires_at)
        )
        await self.db.commit()
    
    async def complete_task(
        self,
        task_id: str,
        worker_id: str,
        status: str,  # "succeeded" | "failed"
        result: Optional[dict] = None,
        error: Optional[str] = None
    ):
        """完成任务"""
        now = datetime.utcnow()
        
        await self.db.execute(
            update(Task)
            .where(
                Task.id == task_id,
                Task.worker_id == worker_id
            )
            .values(
                status=status,
                result=result,
                error=error,
                completed_at=now
            )
        )
        await self.db.commit()


class GenerationWorker:
    """后台 Worker - 独立通道执行"""
    
    def __init__(
        self,
        worker_id: str,
        channel: str,  # "image" | "video" | "audio"
        queue: GenerationQueue,
        provider_pool: ProviderPool
    ):
        self.worker_id = worker_id
        self.channel = channel
        self.queue = queue
        self.provider_pool = provider_pool
        self._running = False
    
    async def start(self):
        """启动 Worker"""
        self._running = True
        
        while self._running:
            # 认领任务
            task = await self.queue.claim_next_task(
                worker_id=self.worker_id,
                channel=self.channel
            )
            
            if not task:
                # 无任务，等待 1 秒
                await asyncio.sleep(1)
                continue
            
            # 执行任务（带心跳）
            try:
                result = await self._execute_with_heartbeat(task)
                
                # 标记成功
                await self.queue.complete_task(
                    task_id=task.id,
                    worker_id=self.worker_id,
                    status="succeeded",
                    result=result
                )
            except Exception as e:
                # 标记失败
                await self.queue.complete_task(
                    task_id=task.id,
                    worker_id=self.worker_id,
                    status="failed",
                    error=str(e)
                )
    
    async def _execute_with_heartbeat(self, task: Task) -> dict:
        """执行任务（带心跳续期）"""
        # 启动心跳协程
        heartbeat_task = asyncio.create_task(
            self._heartbeat_loop(task.id)
        )
        
        try:
            # 执行生成
            backend = self.provider_pool.get_backend(
                channel=self.channel,
                provider=task.provider
            )
            
            result = await backend.generate(**task.params)
            
            return result
        finally:
            # 停止心跳
            heartbeat_task.cancel()
            try:
                await heartbeat_task
            except asyncio.CancelledError:
                pass
    
    async def _heartbeat_loop(self, task_id: str):
        """心跳循环"""
        while True:
            await asyncio.sleep(
                GenerationQueue.HEARTBEAT_INTERVAL_SECONDS
            )
            await self.queue.heartbeat(
                task_id=task_id,
                worker_id=self.worker_id
            )
    
    def stop(self):
        """停止 Worker"""
        self._running = False
```

## 3. 多供应商抽象层

### 统一协议

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class ImageBackend(ABC):
    """图像生成后端协议"""
    
    @abstractmethod
    async def generate(
        self,
        prompt: str,
        negative_prompt: Optional[str] = None,
        width: int = 1024,
        height: int = 1024,
        num_variants: int = 1,
        **kwargs
    ) -> List[str]:
        """
        生成图像
        
        Returns:
            List[str]: 生成的图像路径列表
        """
        pass


class VideoBackend(ABC):
    """视频生成后端协议"""
    
    @abstractmethod
    async def generate(
        self,
        prompt: str,
        images: Optional[List[str]] = None,
        videos: Optional[List[str]] = None,
        audios: Optional[List[str]] = None,
        duration: int = 5,
        **kwargs
    ) -> str:
        """
        生成视频
        
        Returns:
            str: 生成的视频路径
        """
        pass


class AudioBackend(ABC):
    """音频生成后端协议"""
    
    @abstractmethod
    async def generate(
        self,
        text: str,
        voice: str = "default",
        **kwargs
    ) -> str:
        """
        生成音频
        
        Returns:
            str: 生成的音频路径
        """
        pass


class LLMBackend(ABC):
    """LLM 后端协议"""
    
    @abstractmethod
    async def generate(
        self,
        prompt: str,
        system: Optional[str] = None,
        temperature: float = 0.7,
        max_tokens: int = 2000,
        **kwargs
    ) -> str:
        """
        生成文本
        
        Returns:
            str: 生成的文本
        """
        pass
```

### Provider Registry

```python
class ProviderRegistry:
    """多供应商注册中心"""
    
    def __init__(self):
        self._image_backends: Dict[str, Type[ImageBackend]] = {}
        self._video_backends: Dict[str, Type[VideoBackend]] = {}
        self._audio_backends: Dict[str, Type[AudioBackend]] = {}
        self._llm_backends: Dict[str, Type[LLMBackend]] = {}
    
    def register_image_backend(self, name: str, backend_class: Type[ImageBackend]):
        """注册图像后端"""
        self._image_backends[name] = backend_class
    
    def register_video_backend(self, name: str, backend_class: Type[VideoBackend]):
        """注册视频后端"""
        self._video_backends[name] = backend_class
    
    def register_audio_backend(self, name: str, backend_class: Type[AudioBackend]):
        """注册音频后端"""
        self._audio_backends[name] = backend_class
    
    def register_llm_backend(self, name: str, backend_class: Type[LLMBackend]):
        """注册 LLM 后端"""
        self._llm_backends[name] = backend_class
    
    def create_image_backend(self, name: str, config: dict) -> ImageBackend:
        """创建图像后端实例"""
        if name not in self._image_backends:
            raise ValueError(f"Unknown image backend: {name}")
        return self._image_backends[name](**config)
    
    def create_video_backend(self, name: str, config: dict) -> VideoBackend:
        """创建视频后端实例"""
        if name not in self._video_backends:
            raise ValueError(f"Unknown video backend: {name}")
        return self._video_backends[name](**config)
    
    def create_audio_backend(self, name: str, config: dict) -> AudioBackend:
        """创建音频后端实例"""
        if name not in self._audio_backends:
            raise ValueError(f"Unknown audio backend: {name}")
        return self._audio_backends[name](**config)
    
    def create_llm_backend(self, name: str, config: dict) -> LLMBackend:
        """创建 LLM 后端实例"""
        if name not in self._llm_backends:
            raise ValueError(f"Unknown LLM backend: {name}")
        return self._llm_backends[name](**config)


# 全局注册中心
registry = ProviderRegistry()

# 注册内置后端
registry.register_image_backend("gemini", GeminiImageBackend)
registry.register_image_backend("wanx", WanxImageBackend)
registry.register_video_backend("seedance", SeedanceVideoBackend)
registry.register_video_backend("kling", KlingVideoBackend)
registry.register_video_backend("vidu", ViduVideoBackend)
registry.register_audio_backend("volc", VolcAudioBackend)
registry.register_audio_backend("openai", OpenAIAudioBackend)
registry.register_llm_backend("qwen", QwenLLMBackend)
registry.register_llm_backend("claude", ClaudeLLMBackend)
registry.register_llm_backend("gpt", GPTLLMBackend)
```

### Seedance 后端实现

```python
class SeedanceVideoBackend(VideoBackend):
    """Seedance 2.0 视频生成后端"""
    
    def __init__(self, api_key: str, base_url: str = "https://api.seedance.ai"):
        self.api_key = api_key
        self.base_url = base_url
        self.client = httpx.AsyncClient()
    
    async def generate(
        self,
        prompt: str,
        images: Optional[List[str]] = None,
        videos: Optional[List[str]] = None,
        audios: Optional[List[str]] = None,
        duration: int = 5,
        **kwargs
    ) -> str:
        """生成视频"""
        # 构建请求
        files = {}
        
        # 上传图像
        if images:
            for i, img_path in enumerate(images[:9]):  # 最多9张
                with open(img_path, "rb") as f:
                    files[f"image{i+1}"] = f.read()
        
        # 上传视频
        if videos:
            for i, vid_path in enumerate(videos[:3]):  # 最多3个
                with open(vid_path, "rb") as f:
                    files[f"video{i+1}"] = f.read()
        
        # 上传音频
        if audios:
            for i, aud_path in enumerate(audios[:3]):  # 最多3个
                with open(aud_path, "rb") as f:
                    files[f"audio{i+1}"] = f.read()
        
        # 发送请求
        response = await self.client.post(
            f"{self.base_url}/v1/video/generate",
            headers={"Authorization": f"Bearer {self.api_key}"},
            data={
                "prompt": prompt,
                "duration": duration,
                **kwargs
            },
            files=files
        )
        response.raise_for_status()
        
        # 轮询结果
        task_id = response.json()["task_id"]
        
        while True:
            status_response = await self.client.get(
                f"{self.base_url}/v1/video/status/{task_id}",
                headers={"Authorization": f"Bearer {self.api_key}"}
            )
            status_response.raise_for_status()
            
            data = status_response.json()
            
            if data["status"] == "succeeded":
                # 下载视频
                video_url = data["result"]["video_url"]
                local_path = await self._download_video(video_url)
                return local_path
            elif data["status"] == "failed":
                raise Exception(f"Generation failed: {data['error']}")
            
            # 等待 5 秒后重试
            await asyncio.sleep(5)
    
    async def _download_video(self, url: str) -> str:
        """下载视频到本地"""
        response = await self.client.get(url)
        response.raise_for_status()
        
        # 保存到临时文件
        import tempfile
        fd, path = tempfile.mkstemp(suffix=".mp4")
        with os.fdopen(fd, "wb") as f:
            f.write(response.content)
        
        return path
```

## 4. Claude Agent SDK 集成

### 主 Agent 工作流

```python
from claude_agent_sdk import Agent, Skill, Subagent

class VideoCreationAgent(Agent):
    """视频创作主 Agent"""
    
    def __init__(self, project_manager: ProjectManager):
        super().__init__(name="video-creation-agent")
        self.project_manager = project_manager
        
        # 注册 Subagents
        self.register_subagent("script-analyzer", ScriptAnalyzerAgent())
        self.register_subagent("asset-generator", AssetGeneratorAgent())
        self.register_subagent("story-planner", StoryPlannerAgent())
        self.register_subagent("motion-generator", MotionGeneratorAgent())
    
    @Skill(name="video-workflow")
    async def video_workflow(self, project_id: str, user_input: str):
        """视频创作完整工作流"""
        # 检测项目状态
        project = await self.project_manager.get_project(project_id)
        stage = self._detect_stage(project)
        
        if stage == "script":
            # 阶段1: 剧本分析
            result = await self.dispatch_subagent(
                "script-analyzer",
                prompt=f"分析剧本并提取实体: {user_input}"
            )
            await self._confirm_with_user(result)
            
        elif stage == "asset":
            # 阶段2: 资产生成
            result = await self.dispatch_subagent(
                "asset-generator",
                prompt=f"为以下实体生成资产: {project.entities}"
            )
            await self._confirm_with_user(result)
            
        elif stage == "story":
            # 阶段3: 分镜规划
            result = await self.dispatch_subagent(
                "story-planner",
                prompt=f"生成分镜脚本: {project.script}"
            )
            await self._confirm_with_user(result)
            
        elif stage == "motion":
            # 阶段4: 动态生成
            result = await self.dispatch_subagent(
                "motion-generator",
                prompt=f"生成分镜视频: {project.storyboard}"
            )
            await self._confirm_with_user(result)
            
        else:
            # 阶段5: 合成
            await self._assemble_final_video(project)
    
    def _detect_stage(self, project: Project) -> str:
        """检测项目当前阶段"""
        if not project.script:
            return "script"
        elif not project.assets:
            return "asset"
        elif not project.storyboard:
            return "story"
        elif not project.motion_clips:
            return "motion"
        else:
            return "assembly"
    
    async def _confirm_with_user(self, result: dict):
        """与用户确认结果"""
        # 通过 SSE 推送结果给前端
        await self.send_sse_event({
            "type": "stage_complete",
            "result": result,
            "action": "confirm"
        })
        
        # 等待用户确认
        confirmation = await self.wait_for_user_input()
        
        if not confirmation["approved"]:
            # 用户拒绝，重新执行
            raise StageRejectedError(confirmation["feedback"])
```

继续下一部分...
