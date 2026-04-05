# 剧本处理流程示例 - 古代青楼舞女

## 原始剧本输入

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

---

## 第1步: LLM 剧本分析

### 输入
```json
{
  "text": "夜幕降临，华灯初上。醉仙楼内，丝竹声声...",
  "analysis_prompt": "分析以下剧本，提取角色、场景、道具、情节节点"
}
```

### LLM 分析结果

```json
{
  "entities": {
    "characters": [
      {
        "id": "char_001",
        "name": "红袖",
        "description": "醉仙楼头牌舞姬，身着轻纱罗裙，腰系金丝软带，舞姿优美如仙子",
        "appearance": "年轻女子，容貌姣好，身材婀娜，穿着华丽舞衣",
        "personality": "外表风光，内心孤独",
        "role": "主角"
      },
      {
        "id": "char_002",
        "name": "宾客",
        "description": "醉仙楼的观众，富贵人家",
        "appearance": "身着华服的男子们",
        "role": "群众"
      }
    ],
    "scenes": [
      {
        "id": "scene_001",
        "name": "醉仙楼大堂",
        "description": "古代青楼大堂，华灯璀璨，丝竹悠扬，装饰奢华",
        "atmosphere": "热闹繁华，灯火通明",
        "time": "夜晚",
        "details": "红色灯笼、雕花屏风、圆桌、酒杯、乐器"
      },
      {
        "id": "scene_002",
        "name": "舞台中央",
        "description": "大堂中央的表演区域，四周围坐宾客",
        "atmosphere": "聚焦中心，众人瞩目",
        "lighting": "聚光效果"
      },
      {
        "id": "scene_003",
        "name": "空荡大堂",
        "description": "曲终人散后的大堂，只剩红袖一人",
        "atmosphere": "寂静冷清，形成对比",
        "time": "深夜"
      }
    ],
    "props": [
      {
        "id": "prop_001",
        "name": "轻纱罗裙",
        "description": "红袖的舞衣，轻盈飘逸，色彩艳丽",
        "importance": "核心道具"
      },
      {
        "id": "prop_002",
        "name": "金丝软带",
        "description": "腰间系带，金色丝线编织",
        "importance": "服饰细节"
      },
      {
        "id": "prop_003",
        "name": "琵琶",
        "description": "伴奏乐器，传统四弦琵琶",
        "importance": "音乐道具"
      },
      {
        "id": "prop_004",
        "name": "红灯笼",
        "description": "醉仙楼的标志性装饰",
        "importance": "场景氛围"
      }
    ]
  },
  "plot_nodes": [
    {
      "id": "node_001",
      "description": "夜幕降临，醉仙楼华灯初上",
      "type": "环境建立",
      "duration": "3-5秒"
    },
    {
      "id": "node_002",
      "description": "红袖步入大堂中央，准备表演",
      "type": "角色登场",
      "duration": "5-8秒"
    },
    {
      "id": "node_003",
      "description": "红袖开始舞蹈，舞姿优美",
      "type": "表演开始",
      "duration": "10-15秒"
    },
    {
      "id": "node_004",
      "description": "舞至高潮，旋转跃起",
      "type": "高潮时刻",
      "duration": "5-8秒"
    },
    {
      "id": "node_005",
      "description": "掌声雷动，喝彩不断",
      "type": "观众反应",
      "duration": "3-5秒"
    },
    {
      "id": "node_006",
      "description": "曲终人散，红袖独自望月",
      "type": "情感转折",
      "duration": "5-8秒"
    }
  ]
}
```

---

## 第2步: 资产生成请求

### 2.1 角色资产 - 红袖

```json
{
  "entity_id": "char_001",
  "entity_type": "character",
  "generation_params": {
    "provider": "gemini",
    "prompt": "古代中国青楼舞姬，名叫红袖，年轻美丽，身着华丽的轻纱罗裙，腰系金丝软带，长发盘起插着珠钗，容貌姣好，气质优雅，全身照，正面视角，电影级画质，柔和光线",
    "negative_prompt": "现代服饰，西方面孔，低质量，模糊",
    "num_variants": 4,
    "style": "cinematic, ancient chinese, elegant"
  }
}
```

**生成的4个变体**:
- variant_001: 红色罗裙，正面站姿
- variant_002: 粉色罗裙，侧面站姿
- variant_003: 白色罗裙，舞蹈姿态
- variant_004: 紫色罗裙，回眸姿态

**用户选择**: variant_003 (白色罗裙，舞蹈姿态)

### 2.2 场景资产 - 醉仙楼大堂

```json
{
  "entity_id": "scene_001",
  "entity_type": "scene",
  "generation_params": {
    "provider": "gemini",
    "prompt": "古代中国青楼大堂内景，华灯璀璨，红色灯笼高悬，雕花屏风，圆桌摆放整齐，丝竹乐器，装饰奢华，夜晚氛围，暖色调光线，电影级画质，广角镜头",
    "negative_prompt": "现代建筑，西方风格，白天，冷色调",
    "num_variants": 4,
    "style": "cinematic, ancient chinese architecture, luxurious"
  }
}
```

**生成的4个变体**:
- variant_001: 正面视角，灯笼为主
- variant_002: 侧面视角，屏风为主
- variant_003: 俯视视角，全景
- variant_004: 仰视视角，华丽天花板

**用户选择**: variant_001 (正面视角，灯笼为主)

### 2.3 道具资产 - 轻纱罗裙

```json
{
  "entity_id": "prop_001",
  "entity_type": "prop",
  "generation_params": {
    "provider": "gemini",
    "prompt": "古代中国舞姬的轻纱罗裙，白色半透明，金丝刺绣，飘逸柔美，特写镜头，柔和光线，电影级画质",
    "num_variants": 4
  }
}
```

**用户选择**: variant_002

---

## 第3步: 参考收集 (collectAllRefs)

### RefCollector 输出

```json
{
  "scene_001_refs": {
    "images": [
      "/output/proj_001/assets/scene_001_variant_001.png"
    ],
    "videos": [],
    "audios": []
  },
  "char_001_refs": {
    "images": [
      "/output/proj_001/assets/char_001_variant_003.png"
    ],
    "videos": [],
    "audios": []
  },
  "prop_001_refs": {
    "images": [
      "/output/proj_001/assets/prop_001_variant_002.png"
    ],
    "videos": [],
    "audios": []
  }
}
```

---

## 第4步: 分镜脚本生成

### LLM 分镜规划

```json
{
  "storyboard": [
    {
      "id": "frame_001",
      "order": 1,
      "scene": "醉仙楼外景",
      "description": "夜幕降临，醉仙楼外景，红灯笼高悬，华灯初上",
      "duration": 4,
      "camera_motion": "推进镜头",
      "characters": [],
      "scene_id": "scene_001",
      "props": ["prop_004"]
    },
    {
      "id": "frame_002",
      "order": 2,
      "scene": "大堂全景",
      "description": "醉仙楼大堂内景，宾客满座，丝竹声声",
      "duration": 5,
      "camera_motion": "横摇镜头",
      "characters": ["char_002"],
      "scene_id": "scene_001",
      "props": ["prop_003", "prop_004"]
    },
    {
      "id": "frame_003",
      "order": 3,
      "scene": "红袖登场",
      "description": "红袖身着白色轻纱罗裙，腰系金丝软带，步入大堂中央",
      "duration": 6,
      "camera_motion": "跟随镜头",
      "characters": ["char_001"],
      "scene_id": "scene_002",
      "props": ["prop_001", "prop_002"]
    },
    {
      "id": "frame_004",
      "order": 4,
      "scene": "舞蹈开始",
      "description": "红袖随琵琶声起舞，舞姿如行云流水",
      "duration": 10,
      "camera_motion": "环绕镜头",
      "characters": ["char_001"],
      "scene_id": "scene_002",
      "props": ["prop_001", "prop_003"]
    },
    {
      "id": "frame_005",
      "order": 5,
      "scene": "舞蹈高潮",
      "description": "红袖旋转跃起，长袖飘飘，裙摆飞扬",
      "duration": 8,
      "camera_motion": "升降镜头 + 慢动作",
      "characters": ["char_001"],
      "scene_id": "scene_002",
      "props": ["prop_001"]
    },
    {
      "id": "frame_006",
      "order": 6,
      "scene": "观众反应",
      "description": "宾客掌声雷动，喝彩不断",
      "duration": 4,
      "camera_motion": "快速切换",
      "characters": ["char_002"],
      "scene_id": "scene_001",
      "props": []
    },
    {
      "id": "frame_007",
      "order": 7,
      "scene": "曲终人散",
      "description": "大堂空荡，红袖独自站立，望向窗外明月",
      "duration": 7,
      "camera_motion": "拉远镜头",
      "characters": ["char_001"],
      "scene_id": "scene_003",
      "props": []
    }
  ]
}
```

---

## 第5步: 提示词编译 (PromptCompiler)

### Frame 1: 醉仙楼外景

#### 第1层: 参考收集
```json
{
  "scene_refs": ["/output/proj_001/assets/scene_001_variant_001.png"],
  "character_refs": [],
  "prop_refs": ["/output/proj_001/assets/prop_004_variant_001.png"],
  "prev_frame_ref": null
}
```

#### 第2层: 时间轴构建
```
0-2s: 镜头从远处推进，醉仙楼外景逐渐清晰
2-4s: 红灯笼特写，华灯初上的氛围
```

#### 第3层: 描述融合
```
场景: 古代中国青楼醉仙楼外景 @image1
道具: 红色灯笼高悬 @image2
时间: 夜晚，华灯初上
氛围: 繁华热闹，暖色调
镜头: 推进镜头，从远到近
光线: 暖黄色灯光，柔和光晕
风格: 电影级画质，古装剧风格
```

#### Seedance 约束验证
```json
{
  "image_count": 2,  // ✓ ≤9
  "video_count": 0,  // ✓ ≤3
  "audio_count": 0,  // ✓ ≤3
  "prompt_length": 156,  // ✓ ≤5000
  "duration": 4,  // ✓ 4-15秒
  "valid": true
}
```

#### 最终编译结果
```json
{
  "prompt": "0-2s: 镜头从远处推进，醉仙楼外景逐渐清晰，2-4s: 红灯笼特写，华灯初上的氛围\n\n场景: 古代中国青楼醉仙楼外景 @image1\n道具: 红色灯笼高悬 @image2\n时间: 夜晚，华灯初上\n氛围: 繁华热闹，暖色调\n镜头: 推进镜头，从远到近\n光线: 暖黄色灯光，柔和光晕\n风格: 电影级画质，古装剧风格，4K分辨率",
  "images": [
    "/output/proj_001/assets/scene_001_variant_001.png",
    "/output/proj_001/assets/prop_004_variant_001.png"
  ],
  "videos": [],
  "audios": [],
  "duration": 4,
  "camera_motion": "推进镜头"
}
```

---

### Frame 3: 红袖登场

#### 第1层: 参考收集
```json
{
  "scene_refs": ["/output/proj_001/assets/scene_001_variant_001.png"],
  "character_refs": ["/output/proj_001/assets/char_001_variant_003.png"],
  "prop_refs": [
    "/output/proj_001/assets/prop_001_variant_002.png",
    "/output/proj_001/assets/prop_002_variant_001.png"
  ],
  "prev_frame_ref": "/output/proj_001/videos/frame_002_first_frame.png"
}
```

#### 第2层: 时间轴构建
```
0-2s: 红袖从大堂侧门走出，步态优雅
2-4s: 红袖走向舞台中央，镜头跟随
4-6s: 红袖站定，准备开始表演
```

#### 第3层: 描述融合
```
角色: 红袖，古代舞姬，身着白色轻纱罗裙 @image1 @image2
服饰: 腰系金丝软带 @image3，长发盘起插珠钗
场景: 醉仙楼大堂 @image4
动作: 从侧门优雅走出，步向舞台中央，步态轻盈婀娜
表情: 专注而自信，眼神坚定
镜头: 跟随镜头，中景到近景
光线: 聚光灯效果，突出主角
氛围: 众人瞩目，万众期待
风格: 电影级画质，古装剧风格，柔和色调
```

#### Seedance 约束验证
```json
{
  "image_count": 4,  // ✓ ≤9
  "video_count": 1,  // ✓ ≤3 (前一帧首帧图)
  "audio_count": 0,  // ✓ ≤3
  "prompt_length": 243,  // ✓ ≤5000
  "duration": 6,  // ✓ 4-15秒
  "valid": true
}
```

#### 最终编译结果
```json
{
  "prompt": "0-2s: 红袖从大堂侧门走出，步态优雅，2-4s: 红袖走向舞台中央，镜头跟随，4-6s: 红袖站定，准备开始表演\n\n角色: 红袖，古代舞姬，身着白色轻纱罗裙 @image1 @image2\n服饰: 腰系金丝软带 @image3，长发盘起插珠钗\n场景: 醉仙楼大堂 @image4\n前一镜头: @video1\n动作: 从侧门优雅走出，步向舞台中央，步态轻盈婀娜\n表情: 专注而自信，眼神坚定\n镜头: 跟随镜头，中景到近景\n光线: 聚光灯效果，突出主角\n氛围: 众人瞩目，万众期待\n风格: 电影级画质，古装剧风格，柔和色调，4K分辨率",
  "images": [
    "/output/proj_001/assets/char_001_variant_003.png",
    "/output/proj_001/assets/prop_001_variant_002.png",
    "/output/proj_001/assets/prop_002_variant_001.png",
    "/output/proj_001/assets/scene_001_variant_001.png"
  ],
  "videos": [
    "/output/proj_001/videos/frame_002_first_frame.png"
  ],
  "audios": [],
  "duration": 6,
  "camera_motion": "跟随镜头"
}
```

---

### Frame 5: 舞蹈高潮

#### 第1层: 参考收集
```json
{
  "scene_refs": ["/output/proj_001/assets/scene_001_variant_001.png"],
  "character_refs": ["/output/proj_001/assets/char_001_variant_003.png"],
  "prop_refs": ["/output/proj_001/assets/prop_001_variant_002.png"],
  "prev_frame_ref": "/output/proj_001/videos/frame_004_first_frame.png"
}
```

#### 第2层: 时间轴构建
```
0-3s: 红袖加速旋转，裙摆飞扬
3-5s: 红袖跃起，慢动作展现
5-8s: 红袖落地，长袖飘飘
```

#### 第3层: 描述融合
```
角色: 红袖 @image1，白色轻纱罗裙 @image2
场景: 醉仙楼舞台中央 @image3
前一镜头: @video1
动作: 快速旋转，裙摆如花绽放，跃起腾空，长袖飘飘如彩蝶飞舞
表情: 沉醉于舞蹈，忘我投入
镜头: 升降镜头 + 慢动作，从低角度仰拍到平视
速度: 0-3s正常速度，3-5s慢动作(0.5x)，5-8s正常速度
光线: 聚光灯追随，背景虚化
特效: 裙摆飞扬的飘逸感，慢动作的唯美感
氛围: 高潮时刻，震撼人心
风格: 电影级画质，古装剧风格，唯美浪漫，4K分辨率
```

#### Seedance 约束验证
```json
{
  "image_count": 3,  // ✓ ≤9
  "video_count": 1,  // ✓ ≤3
  "audio_count": 0,  // ✓ ≤3
  "prompt_length": 312,  // ✓ ≤5000
  "duration": 8,  // ✓ 4-15秒
  "valid": true
}
```

#### 最终编译结果
```json
{
  "prompt": "0-3s: 红袖加速旋转，裙摆飞扬，3-5s: 红袖跃起，慢动作展现，5-8s: 红袖落地，长袖飘飘\n\n角色: 红袖 @image1，白色轻纱罗裙 @image2\n场景: 醉仙楼舞台中央 @image3\n前一镜头: @video1\n动作: 快速旋转，裙摆如花绽放，跃起腾空，长袖飘飘如彩蝶飞舞\n表情: 沉醉于舞蹈，忘我投入\n镜头: 升降镜头 + 慢动作，从低角度仰拍到平视\n速度: 0-3s正常速度，3-5s慢动作(0.5x)，5-8s正常速度\n光线: 聚光灯追随，背景虚化\n特效: 裙摆飞扬的飘逸感，慢动作的唯美感\n氛围: 高潮时刻，震撼人心\n风格: 电影级画质，古装剧风格，唯美浪漫，4K分辨率，细节丰富",
  "images": [
    "/output/proj_001/assets/char_001_variant_003.png",
    "/output/proj_001/assets/prop_001_variant_002.png",
    "/output/proj_001/assets/scene_001_variant_001.png"
  ],
  "videos": [
    "/output/proj_001/videos/frame_004_first_frame.png"
  ],
  "audios": [],
  "duration": 8,
  "camera_motion": "升降镜头 + 慢动作"
}
```

---

## 第6步: 任务入队

### 7个分镜任务入队

```json
{
  "tasks": [
    {
      "id": "task_001",
      "resource_type": "video",
      "resource_id": "frame_001",
      "status": "queued",
      "params": {
        "provider": "seedance",
        "prompt": "...",
        "images": [...],
        "duration": 4
      }
    },
    {
      "id": "task_002",
      "resource_type": "video",
      "resource_id": "frame_002",
      "status": "queued",
      "params": {
        "provider": "seedance",
        "prompt": "...",
        "images": [...],
        "duration": 5
      }
    },
    // ... frame_003 到 frame_007
  ]
}
```

---

## 完整流程总结

```
原始剧本 (200字)
    ↓ [3秒] LLM分析
实体提取 (3角色 + 3场景 + 4道具)
    ↓ [30秒] 并行生成资产
资产图库 (10张图片，每个4变体)
    ↓ [1秒] 用户选择最佳变体
参考收集 (collectAllRefs)
    ↓ [5秒] LLM生成分镜脚本
分镜脚本 (7个分镜帧)
    ↓ [2秒/帧] 提示词编译
编译后提示词 (7个完整提示词)
    ↓ [1秒] 参数校验
Seedance验证通过
    ↓ [1秒] 任务入队
异步队列 (7个视频生成任务)
    ↓ [120秒/任务] 并行生成
7个分镜视频 (4-8秒/片段)
    ↓ [30秒] FFmpeg合成
最终视频 (44秒完整视频)
```

### 时间消耗
- 剧本分析: 3秒
- 资产生成: 30秒 (并行)
- 用户选择: 10秒 (人工)
- 分镜规划: 5秒
- 提示词编译: 14秒 (7帧 × 2秒)
- 视频生成: 120秒 (并行，最慢的一个)
- 视频合成: 30秒

**总耗时**: 约 3.5 分钟

### 资源消耗
- 图像生成: 10个实体 × 4变体 = 40张图片
- 视频生成: 7个分镜片段
- LLM调用: 3次 (剧本分析 + 分镜规划 + 提示词优化)
- 存储空间: 约 500MB

### 成本估算
- 图像生成: 40张 × $0.04 = $1.60
- 视频生成: 7片段 × $2.00 = $14.00
- LLM调用: 3次 × $0.10 = $0.30
- **总成本**: $15.90

---

## 关键技术点

### 1. 智能参考收集
- 自动选择最佳变体
- 前一帧首帧图作为连续性参考
- 避免角色/场景不一致

### 2. 时间轴分段
- 根据片段时长自动分段
- 短片段(≤5s): 单一动作
- 中片段(5-10s): 起承转
- 长片段(>10s): 起承转合

### 3. Seedance 约束
- 图像 ≤9张
- 视频 ≤3个
- 音频 ≤3个
- 提示词 ≤5000字符
- 时长 4-15秒

### 4. 多模态融合
- @image1 @image2 标记图像
- @video1 标记视频
- @audio1 标记音频
- 自动组装参考资料
