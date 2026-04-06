# 手动操作工作流指南 - 从需求到成片

这是一个完整的、可手动操作的工作流演示，帮助你理解从需求到最终视频的全过程。

---

## 工作流总览

```
需求输入 → 剧本创作 → 剧本分析 → 资产生成 → 分镜规划 → 视频生成 → 视频合成 → 最终成片
   ↓          ↓          ↓          ↓          ↓          ↓          ↓          ↓
  1分钟      5分钟      30秒       2分钟      1分钟      2分钟      30秒       完成
```

**总耗时**: 约 12 分钟

---

## 第1步: 需求输入（1分钟）

### 你要做什么

明确你想要创作的视频主题、风格、时长。

### 操作步骤

**打开前端界面**:
```
http://localhost:3000
```

**点击"创建新项目"按钮**

**填写项目信息**:
```
项目名称: 古代青楼舞女
描述: 展现古代舞姬红袖在醉仙楼表演《霓裳羽衣舞》的故事
目标时长: 60秒
风格: 古装剧，电影级画质，唯美浪漫
```

**点击"创建"**

### 你会看到

```
✅ 项目创建成功
项目ID: proj_001
当前阶段: script（剧本阶段）
```

### 后台发生了什么

```http
POST /api/projects
Content-Type: application/json

{
  "name": "古代青楼舞女",
  "description": "展现古代舞姬红袖在醉仙楼表演《霓裳羽衣舞》的故事",
  "target_duration": 60,
  "style": "古装剧，电影级画质，唯美浪漫"
}
```

**数据库记录**:
```sql
INSERT INTO projects (id, name, stage, created_at)
VALUES ('proj_001', '古代青楼舞女', 'script', NOW());
```

---

## 第2步: 剧本创作（5分钟）

### 你要做什么

编写或输入视频剧本，描述故事情节。

### 操作步骤

**在"剧本编辑器"面板中输入**:

```
夜幕降临，华灯初上。醉仙楼内，丝竹声声。

红袖是醉仙楼的头牌舞姬，今夜她将为贵客表演《霓裳羽衣舞》。
她身着轻纱罗裙，腰系金丝软带，步入大堂中央。

众宾客屏息凝神，只见红袖轻启朱唇，随着琵琶声起，开始翩翩起舞。
她的舞姿如行云流水，裙摆飞扬间，恍若仙子下凡。

舞至高潮，红袖旋转跃起，长袖飘飘，宛如彩蝶飞舞。
台下掌声雷动，喝彩声不绝于耳。

曲终人散，红袖独自站在空荡的大堂，望向窗外的明月，眼中闪过一丝落寞。
```

**点击"保存剧本"**

**点击"分析剧本"按钮**

### 你会看到

```
🔄 正在分析剧本...
⏱️ 预计耗时: 30秒
```

**30秒后**:
```
✅ 剧本分析完成

提取到:
• 3 个角色（红袖、宾客、琵琶师）
• 3 个场景（醉仙楼外景、大堂、舞台中央）
• 4 个道具（轻纱罗裙、金丝软带、琵琶、红灯笼）
• 6 个情节节点
```

### 后台发生了什么

**API 调用**:
```http
POST /api/projects/proj_001/script/analyze
Content-Type: application/json

{
  "text": "夜幕降临，华灯初上..."
}
```

**LLM 分析**（调用 Claude/GPT）:
```python
# 后端代码
async def analyze_script(project_id: str, text: str):
    # 调用 LLM
    prompt = f"""
分析以下剧本，提取角色、场景、道具、情节节点：

{text}

请以 JSON 格式返回。
"""
    
    result = await llm.generate(prompt)
    
    # 解析结果
    entities = parse_llm_response(result)
    
    # 保存到数据库
    await db.save_entities(project_id, entities)
    
    return entities
```

**数据库记录**:
```sql
-- 角色表
INSERT INTO characters (id, project_id, name, description)
VALUES 
  ('char_001', 'proj_001', '红袖', '醉仙楼头牌舞姬'),
  ('char_002', 'proj_001', '宾客', '观众');

-- 场景表
INSERT INTO scenes (id, project_id, name, description)
VALUES 
  ('scene_001', 'proj_001', '醉仙楼大堂', '华灯璀璨的青楼大堂');

-- 道具表
INSERT INTO props (id, project_id, name, description)
VALUES 
  ('prop_001', 'proj_001', '轻纱罗裙', '白色半透明舞衣');
```

---

## 第3步: 资产生成（2分钟）

### 你要做什么

为每个角色、场景、道具生成参考图片。

### 操作步骤

**切换到"资产工作台"面板**

**你会看到**:
```
角色列表:
  □ 红袖 - 未生成
  □ 宾客 - 未生成

场景列表:
  □ 醉仙楼大堂 - 未生成

道具列表:
  □ 轻纱罗裙 - 未生成
  □ 琵琶 - 未生成
```

**点击"红袖"旁边的"生成"按钮**

**系统自动填充提示词**:
```
古代中国青楼舞姬，名叫红袖，年轻美丽，身着华丽的轻纱罗裙，
腰系金丝软带，长发盘起插着珠钗，容貌姣好，气质优雅，
全身照，正面视角，电影级画质，柔和光线
```

**你可以编辑提示词，然后点击"确认生成"**

**选择生成数量**: 4个变体

**点击"开始生成"**

### 你会看到

```
🔄 正在生成资产...

红袖:
  ⏱️ 排队中... → 🔄 生成中... → ✅ 完成 (30秒)
  
  生成了4个变体:
  [图1] [图2] [图3] [图4]
  
  请选择最佳变体: □ □ ☑ □
```

**选择第3个变体（白色罗裙，舞蹈姿态）**

**点击"确认选择"**

**重复以上步骤**，为其他资产生成图片:
- 醉仙楼大堂（选择正面视角）
- 轻纱罗裙（选择飘逸款）

### 后台发生了什么

**API 调用**:
```http
POST /api/generate/asset
Content-Type: application/json

{
  "project_id": "proj_001",
  "entity_id": "char_001",
  "entity_type": "character",
  "num_variants": 4,
  "provider": "gemini",
  "prompt": "古代中国青楼舞姬..."
}
```

**任务入队**:
```python
# 后端代码
async def generate_asset(request: AssetGenerateRequest):
    # 创建任务
    task = Task(
        id="task_001",
        project_id=request.project_id,
        resource_type="image",
        resource_id=request.entity_id,
        status="queued",
        params={
            "provider": request.provider,
            "prompt": request.prompt,
            "num_variants": request.num_variants
        }
    )
    
    # 入队
    await queue.enqueue(task)
    
    return {"task_id": task.id}
```

**Worker 执行**:
```python
# Worker 进程
async def process_image_task(task: Task):
    # 调用 Gemini API
    results = []
    for i in range(task.params["num_variants"]):
        image = await gemini.generate_image(
            prompt=task.params["prompt"]
        )
        
        # 保存图片
        path = f"/output/{task.project_id}/assets/{task.resource_id}_variant_{i+1}.png"
        save_image(image, path)
        
        results.append(path)
    
    # 更新任务状态
    task.status = "completed"
    task.result = {"images": results}
    await db.update_task(task)
```

**SSE 推送**:
```
event: task_update
data: {"task_id": "task_001", "status": "completed", "progress": 1.0}
```

**前端接收**:
```typescript
// React 组件
const eventSource = new EventSource('/api/projects/proj_001/events');

eventSource.addEventListener('task_update', (event) => {
  const data = JSON.parse(event.data);
  
  if (data.status === 'completed') {
    // 更新 UI，显示生成的图片
    updateAssetGallery(data.task_id, data.result.images);
  }
});
```

---

## 第4步: 分镜规划（1分钟）

### 你要做什么

将剧本拆分成多个 8 秒的分镜片段。

### 操作步骤

**切换到"分镜导演台"面板**

**点击"自动生成分镜"按钮**

**系统提示**:
```
根据剧本和目标时长(60秒)，建议生成 7 个分镜
每个分镜 8 秒

是否继续？ [确定] [取消]
```

**点击"确定"**

### 你会看到

```
🔄 正在生成分镜脚本...
⏱️ 预计耗时: 1分钟
```

**1分钟后**:
```
✅ 分镜脚本生成完成

Frame 1: 醉仙楼外景 (8秒)
  场景: 醉仙楼外景
  镜头: 推进镜头
  描述: 夜幕降临，红灯笼高悬
  
Frame 2: 大堂全景 (8秒)
  场景: 大堂内景
  镜头: 横摇镜头
  描述: 宾客满座，丝竹声声
  
Frame 3: 红袖登场 (8秒)
  角色: 红袖
  镜头: 跟随镜头
  描述: 红袖步入舞台中央
  
Frame 4: 舞蹈开始 (8秒)
  角色: 红袖
  镜头: 环绕镜头
  描述: 红袖随琵琶声起舞
  
Frame 5: 舞蹈高潮 (8秒)
  角色: 红袖
  镜头: 升降镜头 + 慢动作
  描述: 红袖旋转跃起
  
Frame 6: 观众反应 (8秒)
  角色: 宾客
  镜头: 快速切换
  描述: 掌声雷动
  
Frame 7: 曲终人散 (8秒)
  角色: 红袖
  镜头: 拉远镜头
  描述: 红袖独自望月

总时长: 56秒
```

**你可以**:
- 点击每个分镜查看详情
- 编辑分镜描述
- 调整分镜顺序（拖拽）
- 删除或添加分镜

**确认无误后，点击"确认分镜"**

### 后台发生了什么

**API 调用**:
```http
POST /api/projects/proj_001/storyboard/generate
Content-Type: application/json

{
  "style": "cinematic",
  "target_duration": 60
}
```

**LLM 生成分镜**:
```python
async def generate_storyboard(project_id: str, target_duration: int):
    # 获取剧本和实体
    script = await db.get_script(project_id)
    entities = await db.get_entities(project_id)
    
    # 计算分镜数
    num_frames = target_duration // 8
    
    # 调用 LLM
    prompt = f"""
根据以下剧本，生成 {num_frames} 个分镜，每个 8 秒：

剧本: {script.text}

角色: {entities.characters}
场景: {entities.scenes}
道具: {entities.props}

请为每个分镜指定：
1. 场景
2. 角色
3. 道具
4. 镜头运动
5. 描述
"""
    
    result = await llm.generate(prompt)
    
    # 解析并保存
    frames = parse_storyboard(result)
    await db.save_storyboard(project_id, frames)
    
    return frames
```

---

## 第5步: 视频生成（2分钟）

### 你要做什么

为每个分镜生成 8 秒视频片段。

### 操作步骤

**在"分镜导演台"面板，点击"生成所有视频"按钮**

**系统提示**:
```
将生成 7 个视频片段
预计耗时: 2分钟（并行生成）
预计成本: $14.00

是否继续？ [确定] [取消]
```

**点击"确定"**

### 你会看到

```
🔄 正在生成视频...

Frame 1: ⏱️ 排队中...
Frame 2: ⏱️ 排队中...
Frame 3: ⏱️ 排队中...
Frame 4: ⏱️ 排队中...
Frame 5: ⏱️ 排队中...
Frame 6: ⏱️ 排队中...
Frame 7: ⏱️ 排队中...
```

**10秒后**（Worker 开始认领任务）:
```
Frame 1: 🔄 生成中... 15%
Frame 2: 🔄 生成中... 10%
Frame 3: ⏱️ 排队中...
Frame 4: ⏱️ 排队中...
Frame 5: ⏱️ 排队中...
Frame 6: ⏱️ 排队中...
Frame 7: ⏱️ 排队中...
```

**2分钟后**（所有任务完成）:
```
Frame 1: ✅ 完成 [预览]
Frame 2: ✅ 完成 [预览]
Frame 3: ✅ 完成 [预览]
Frame 4: ✅ 完成 [预览]
Frame 5: ✅ 完成 [预览]
Frame 6: ✅ 完成 [预览]
Frame 7: ✅ 完成 [预览]

所有视频生成完成！
```

**点击"预览"可以查看每个片段**

### 后台发生了什么

**提示词编译**:
```python
# 对每个分镜编译提示词
async def compile_prompt(frame: Frame):
    # 第1层: 收集参考
    refs = await collect_all_refs(frame)
    
    # 第2层: 构建时间轴
    timeline = build_timeline(frame, duration=8)
    
    # 第3层: 融合描述
    prompt = f"""
{timeline}

角色: {frame.character.description} @image1
场景: {frame.scene.description} @image2
动作: {frame.action}
镜头: {frame.camera_motion}
风格: 电影级画质，古装剧风格
"""
    
    # 验证 Seedance 约束
    validate_seedance_constraints(prompt, refs)
    
    return CompiledPrompt(prompt, refs.images, refs.videos, refs.audios)
```

**任务入队**:
```python
# 为每个分镜创建任务
tasks = []
for frame in frames:
    compiled = await compile_prompt(frame)
    
    task = Task(
        id=f"task_video_{frame.id}",
        project_id=project_id,
        resource_type="video",
        resource_id=frame.id,
        status="queued",
        params={
            "provider": "seedance",
            "prompt": compiled.prompt,
            "images": compiled.images,
            "videos": compiled.videos,
            "audios": compiled.audios,
            "duration": 8
        }
    )
    
    await queue.enqueue(task)
    tasks.append(task)
```

**Worker 并行执行**:
```python
# Video Worker 进程
async def process_video_task(task: Task):
    # 调用 Seedance API
    video = await seedance.generate_video(
        prompt=task.params["prompt"],
        images=task.params["images"],
        duration=task.params["duration"]
    )
    
    # 保存视频
    path = f"/output/{task.project_id}/videos/{task.resource_id}.mp4"
    save_video(video, path)
    
    # 更新任务
    task.status = "completed"
    task.result = {"video_path": path}
    await db.update_task(task)
    
    # SSE 推送
    await sse.push(task.project_id, {
        "event": "task_update",
        "data": {
            "task_id": task.id,
            "status": "completed",
            "result": task.result
        }
    })
```

---

## 第6步: 视频合成（30秒）

### 你要做什么

将 7 个视频片段合成为一个完整视频。

### 操作步骤

**点击"合成最终视频"按钮**

**配置合成选项**:
```
转场效果: 淡入淡出
转场时长: 0.5秒

背景音乐: [选择文件] 或 [使用默认]
音量: 30%

字幕: □ 添加字幕

输出格式: MP4
分辨率: 1920×1080
帧率: 24fps
```

**点击"开始合成"**

### 你会看到

```
🔄 正在合成视频...

进度: ████████░░ 80%
当前: 正在添加转场效果...
预计剩余: 10秒
```

**30秒后**:
```
✅ 视频合成完成！

最终视频:
  文件名: 古代青楼舞女_final.mp4
  时长: 56秒
  大小: 45.2 MB
  分辨率: 1920×1080
  
[下载] [预览] [分享]
```

**点击"预览"观看完整视频**

### 后台发生了什么

**API 调用**:
```http
POST /api/projects/proj_001/assemble
Content-Type: application/json

{
  "transitions": [
    {"type": "fade", "duration": 0.5}
  ],
  "audio": {
    "bgm": "/path/to/bgm.mp3",
    "volume": 0.3
  }
}
```

**FFmpeg 合成**:
```python
async def assemble_video(project_id: str, config: AssembleConfig):
    # 获取所有分镜视频
    frames = await db.get_storyboard_frames(project_id)
    video_paths = [f.video_path for f in frames]
    
    # 创建 FFmpeg 命令
    cmd = build_ffmpeg_command(
        inputs=video_paths,
        transitions=config.transitions,
        bgm=config.audio.bgm,
        bgm_volume=config.audio.volume,
        output=f"/output/{project_id}/final_video.mp4"
    )
    
    # 执行
    await run_ffmpeg(cmd)
    
    return {
        "video_path": f"/output/{project_id}/final_video.mp4",
        "duration": sum(f.duration for f in frames),
        "size": get_file_size(f"/output/{project_id}/final_video.mp4")
    }
```

**FFmpeg 命令示例**:
```bash
ffmpeg \
  -i frame_001.mp4 \
  -i frame_002.mp4 \
  -i frame_003.mp4 \
  -i frame_004.mp4 \
  -i frame_005.mp4 \
  -i frame_006.mp4 \
  -i frame_007.mp4 \
  -i bgm.mp3 \
  -filter_complex "\
    [0:v]fade=t=out:st=7.5:d=0.5[v0]; \
    [1:v]fade=t=in:st=0:d=0.5,fade=t=out:st=7.5:d=0.5[v1]; \
    [2:v]fade=t=in:st=0:d=0.5,fade=t=out:st=7.5:d=0.5[v2]; \
    [3:v]fade=t=in:st=0:d=0.5,fade=t=out:st=7.5:d=0.5[v3]; \
    [4:v]fade=t=in:st=0:d=0.5,fade=t=out:st=7.5:d=0.5[v4]; \
    [5:v]fade=t=in:st=0:d=0.5,fade=t=out:st=7.5:d=0.5[v5]; \
    [6:v]fade=t=in:st=0:d=0.5[v6]; \
    [v0][v1][v2][v3][v4][v5][v6]concat=n=7:v=1:a=0[outv]; \
    [7:a]volume=0.3[outa]" \
  -map "[outv]" -map "[outa]" \
  -c:v libx264 -preset medium -crf 23 \
  -c:a aac -b:a 192k \
  final_video.mp4
```

---

## 完整流程总结

### 时间线

```
0:00 - 需求输入（创建项目）
1:00 - 剧本创作（编写剧本）
6:00 - 剧本分析（LLM提取实体）
6:30 - 资产生成（生成参考图）
8:30 - 分镜规划（生成分镜脚本）
9:30 - 视频生成（并行生成7个片段）
11:30 - 视频合成（FFmpeg合成）
12:00 - 完成！
```

### 数据流

```
用户输入
  ↓
剧本文本 (200字)
  ↓ [LLM分析]
实体列表 (3角色 + 3场景 + 4道具)
  ↓ [图像生成]
资产图库 (40张图片，选择10张)
  ↓ [参考收集]
参考资料包 (10张图片)
  ↓ [LLM规划]
分镜脚本 (7个分镜)
  ↓ [提示词编译]
编译后提示词 (7个完整提示词)
  ↓ [视频生成]
分镜视频 (7个8秒片段)
  ↓ [FFmpeg合成]
最终视频 (56秒完整视频)
```

### 成本明细

```
图像生成: 40张 × $0.04 = $1.60
视频生成: 7片段 × $2.00 = $14.00
LLM调用: 3次 × $0.10 = $0.30
────────────────────────────
总成本: $15.90
```

### 存储占用

```
资产图片: 40张 × 2MB = 80MB
分镜视频: 7个 × 50MB = 350MB
最终视频: 1个 × 45MB = 45MB
────────────────────────────
总存储: 475MB
```

---

## 常见问题

### Q1: 如果资产生成失败怎么办？

**A**: 点击"重新生成"按钮，系统会自动重试。如果多次失败，检查：
- API Key 是否有效
- 是否超过速率限制
- 提示词是否符合规范

### Q2: 可以修改已生成的分镜吗？

**A**: 可以。在"分镜导演台"面板：
- 点击分镜卡片进入编辑模式
- 修改描述、镜头运动等
- 点击"重新生成此分镜"

### Q3: 如何调整分镜顺序？

**A**: 在"分镜导演台"面板，直接拖拽分镜卡片即可调整顺序。

### Q4: 视频生成很慢怎么办？

**A**: 
- 检查 Worker 数量（可以增加 Worker 实例）
- 检查 AI 服务商状态
- 使用多个 API Key 轮询

### Q5: 如何导出项目？

**A**: 点击"导出项目"按钮，会生成一个包含所有资产和配置的 ZIP 文件。

---

## 下一步

现在你已经了解了完整的工作流程，可以：

1. **尝试不同的剧本** - 创建不同主题的视频
2. **调整参数** - 尝试不同的风格、时长、分镜数
3. **优化提示词** - 学习如何编写更好的提示词
4. **探索高级功能** - 使用专用 Agent、批量生成等

**开始创作你的第一个视频吧！** 🎬
