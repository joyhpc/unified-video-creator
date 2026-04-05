# 统一视频创作平台 - 架构图集

## 主架构图

### 整体系统架构

```mermaid
graph TB
    subgraph Frontend["🎨 前端层"]
        UI[Web UI<br/>React 19]
        Desktop[Desktop App<br/>Electron]
    end
    
    subgraph State["📊 状态层"]
        Store[Zustand Store<br/>Project/Task/Asset]
    end
    
    subgraph API["🚪 API 层"]
        REST[REST API<br/>FastAPI]
        SSE[SSE Stream<br/>实时推送]
    end
    
    subgraph Agent["🤖 智能体层"]
        MainAgent[Main Agent<br/>阶段检测]
        SubAgents[Sub Agents<br/>Script/Asset/Story/Motion]
    end
    
    subgraph Engine["⚙️ 引擎层"]
        Pipeline[Pipeline Manager<br/>工作流编排]
        Compiler[Prompt Compiler<br/>3层融合]
        Collector[Ref Collector<br/>参考收集]
    end
    
    subgraph Queue["📋 队列层"]
        TaskQueue[Generation Queue<br/>Lease调度]
        ImageWorker[Image Workers<br/>10实例]
        VideoWorker[Video Workers<br/>5实例]
        AudioWorker[Audio Workers<br/>3实例]
    end
    
    subgraph Provider["🔌 供应商层"]
        Registry[Provider Registry<br/>统一注册]
        Backends[AI Backends<br/>Gemini/Kling/Seedance/Wanx]
    end
    
    subgraph Storage["💾 存储层"]
        DB[(PostgreSQL<br/>任务/配置)]
        Cache[(Redis<br/>缓存/协调)]
        FS[File System<br/>本地/OSS]
    end
    
    UI --> Store
    Desktop --> Store
    Store --> REST
    Store --> SSE
    REST --> MainAgent
    MainAgent --> SubAgents
    SubAgents --> Pipeline
    Pipeline --> Compiler
    Pipeline --> Collector
    Compiler --> TaskQueue
    TaskQueue --> ImageWorker
    TaskQueue --> VideoWorker
    TaskQueue --> AudioWorker
    ImageWorker --> Registry
    VideoWorker --> Registry
    AudioWorker --> Registry
    Registry --> Backends
    TaskQueue --> DB
    TaskQueue --> Cache
    Backends --> FS
    
    style Frontend fill:#e1f5ff
    style State fill:#f3e5f5
    style API fill:#fff3e0
    style Agent fill:#e8f5e9
    style Engine fill:#fce4ec
    style Queue fill:#f1f8e9
    style Provider fill:#fff9c4
    style Storage fill:#e0f2f1
```

## 子架构图

### 1. 数据流架构

```mermaid
graph LR
    A[📝 剧本输入] -->|LLM分析<br/>3s| B[实体提取<br/>角色/场景/道具]
    B -->|资产生成<br/>30s| C[资产图库<br/>多变体]
    C -->|参考收集<br/>1s| D[智能组装<br/>collectAllRefs]
    D -->|提示词编译<br/>2s| E[3层融合<br/>时间轴+约束]
    E -->|参数校验<br/>1s| F[Seedance验证<br/>≤9图+≤3视频]
    F -->|任务入队<br/>1s| G[异步队列<br/>Lease调度]
    G -->|并行生成<br/>120s| H[分镜视频<br/>4-15s片段]
    H -->|FFmpeg合成<br/>30s| I[🎬 最终视频]
    
    style A fill:#e1f5ff
    style B fill:#dbeafe
    style C fill:#fef3c7
    style D fill:#d1fae5
    style E fill:#fed7aa
    style F fill:#fecaca
    style G fill:#e0e7ff
    style H fill:#f3e5f5
    style I fill:#86efac
```

### 2. 智能体协作架构

```mermaid
graph TD
    Start[用户输入] --> Detect{Main Agent<br/>检测阶段}
    
    Detect -->|阶段1| Script[Script Agent<br/>剧本分析]
    Script --> Confirm1{用户确认}
    Confirm1 -->|通过| Detect
    Confirm1 -->|重做| Script
    
    Detect -->|阶段2| Asset[Asset Agent<br/>资产生成]
    Asset --> Confirm2{用户确认}
    Confirm2 -->|通过| Detect
    Confirm2 -->|重做| Asset
    
    Detect -->|阶段3| Story[Story Agent<br/>分镜规划]
    Story --> Confirm3{用户确认}
    Confirm3 -->|通过| Detect
    Confirm3 -->|重做| Story
    
    Detect -->|阶段4| Motion[Motion Agent<br/>动态生成]
    Motion --> Confirm4{用户确认}
    Confirm4 -->|通过| Detect
    Confirm4 -->|重做| Motion
    
    Detect -->|阶段5| Assembly[Assembly Agent<br/>视频合成]
    Assembly --> End[✅ 完成]
    
    style Start fill:#e1f5ff
    style Detect fill:#7c3aed,color:#fff
    style Script fill:#06b6d4,color:#fff
    style Asset fill:#06b6d4,color:#fff
    style Story fill:#06b6d4,color:#fff
    style Motion fill:#06b6d4,color:#fff
    style Assembly fill:#06b6d4,color:#fff
    style End fill:#10b981,color:#fff
```

### 3. 任务队列架构

```mermaid
graph TB
    subgraph Input["任务输入"]
        API[API 请求]
        Agent[Agent 调度]
    end
    
    subgraph Queue["队列管理"]
        Enqueue[入队<br/>去重检查]
        QueueDB[(任务队列<br/>PostgreSQL)]
    end
    
    subgraph Scheduler["Lease 调度器"]
        Claim[认领任务<br/>with_for_update]
        Lease[Lease TTL<br/>10秒]
        Heartbeat[心跳续期<br/>3秒间隔]
    end
    
    subgraph Workers["Worker 池"]
        ImageCh[Image 通道<br/>10 Workers]
        VideoCh[Video 通道<br/>5 Workers]
        AudioCh[Audio 通道<br/>3 Workers]
    end
    
    subgraph Execution["任务执行"]
        Execute[执行生成<br/>调用 AI API]
        Monitor[GPU 监控<br/>内存/利用率]
    end
    
    subgraph Result["结果处理"]
        Success[成功<br/>保存结果]
        Failed[失败<br/>记录错误]
        Notify[SSE 推送<br/>通知前端]
    end
    
    API --> Enqueue
    Agent --> Enqueue
    Enqueue --> QueueDB
    QueueDB --> Claim
    Claim --> Lease
    Lease --> ImageCh
    Lease --> VideoCh
    Lease --> AudioCh
    ImageCh --> Heartbeat
    VideoCh --> Heartbeat
    AudioCh --> Heartbeat
    Heartbeat --> Execute
    Execute --> Monitor
    Monitor --> Success
    Monitor --> Failed
    Success --> Notify
    Failed --> Notify
    
    style Input fill:#e1f5ff
    style Queue fill:#f3e5f5
    style Scheduler fill:#fff3e0
    style Workers fill:#e8f5e9
    style Execution fill:#fce4ec
    style Result fill:#f1f8e9
```

### 4. 多供应商架构

```mermaid
graph TB
    subgraph Client["客户端"]
        Request[生成请求<br/>prompt + params]
    end
    
    subgraph Registry["Provider Registry"]
        Router[智能路由<br/>负载均衡]
        Factory[Backend Factory<br/>创建实例]
    end
    
    subgraph Protocols["统一协议"]
        ImageProto[ImageBackend<br/>Protocol]
        VideoProto[VideoBackend<br/>Protocol]
        AudioProto[AudioBackend<br/>Protocol]
        LLMProto[LLMBackend<br/>Protocol]
    end
    
    subgraph Backends["AI 服务商"]
        Gemini[Gemini<br/>Image]
        Wanx[Wanx<br/>Image/Video]
        Kling[Kling<br/>Video]
        Seedance[Seedance 2.0<br/>Video]
        Vidu[Vidu<br/>Video]
        Volc[Volc<br/>Audio]
        OpenAI[OpenAI<br/>Audio]
        Qwen[Qwen<br/>LLM]
    end
    
    subgraph Wrapper["增强功能"]
        Cost[费用追踪<br/>Usage Tracking]
        Cache[结果缓存<br/>Redis]
        Retry[自动重试<br/>指数退避]
    end
    
    Request --> Router
    Router --> Factory
    Factory --> ImageProto
    Factory --> VideoProto
    Factory --> AudioProto
    Factory --> LLMProto
    
    ImageProto --> Gemini
    ImageProto --> Wanx
    VideoProto --> Wanx
    VideoProto --> Kling
    VideoProto --> Seedance
    VideoProto --> Vidu
    AudioProto --> Volc
    AudioProto --> OpenAI
    LLMProto --> Qwen
    
    Gemini --> Cost
    Wanx --> Cost
    Kling --> Cost
    Seedance --> Cost
    Cost --> Cache
    Cache --> Retry
    
    style Client fill:#e1f5ff
    style Registry fill:#f3e5f5
    style Protocols fill:#fff3e0
    style Backends fill:#e8f5e9
    style Wrapper fill:#fce4ec
```

### 5. 提示词编译架构

```mermaid
graph TB
    subgraph Input["输入"]
        Scene[Scene<br/>场景信息]
        Chars[Characters<br/>角色列表]
        Props[Props<br/>道具列表]
    end
    
    subgraph Layer1["第1层: 参考收集"]
        Collect[collectAllRefs<br/>智能收集]
        CharImg[角色参考图<br/>最佳变体]
        SceneImg[场景参考图<br/>最佳变体]
        PropImg[道具参考图<br/>最佳变体]
        PrevVideo[前一分镜<br/>首帧图]
    end
    
    subgraph Layer2["第2层: 时间轴构建"]
        Timeline[时间轴分段<br/>根据时长]
        Short["短片段 ≤5s<br/>单一动作"]
        Medium["中片段 5-10s<br/>起承转"]
        Long["长片段 >10s<br/>起承转合"]
    end
    
    subgraph Layer3["第3层: 描述融合"]
        Fusion[融合描述<br/>+ @image标记]
        CharDesc[角色描述<br/>@image1 @image2]
        SceneDesc[场景描述<br/>@image3]
        Camera[镜头运动<br/>推进/摇镜头]
    end
    
    subgraph Validation["Seedance 约束验证"]
        Check[参数校验]
        ImgCheck["图像 ≤9张<br/>≤30MB"]
        VidCheck["视频 ≤3个<br/>≤50MB"]
        AudCheck["音频 ≤3个<br/>≤15MB"]
        LenCheck["提示词 ≤5000字符"]
    end
    
    subgraph Output["输出"]
        Result[CompiledPrompt<br/>prompt + refs]
    end
    
    Scene --> Collect
    Chars --> Collect
    Props --> Collect
    Collect --> CharImg
    Collect --> SceneImg
    Collect --> PropImg
    Collect --> PrevVideo
    
    CharImg --> Timeline
    SceneImg --> Timeline
    PropImg --> Timeline
    PrevVideo --> Timeline
    
    Timeline --> Short
    Timeline --> Medium
    Timeline --> Long
    
    Short --> Fusion
    Medium --> Fusion
    Long --> Fusion
    
    Fusion --> CharDesc
    Fusion --> SceneDesc
    Fusion --> Camera
    
    CharDesc --> Check
    SceneDesc --> Check
    Camera --> Check
    
    Check --> ImgCheck
    Check --> VidCheck
    Check --> AudCheck
    Check --> LenCheck
    
    ImgCheck --> Result
    VidCheck --> Result
    AudCheck --> Result
    LenCheck --> Result
    
    style Input fill:#e1f5ff
    style Layer1 fill:#dbeafe
    style Layer2 fill:#fef3c7
    style Layer3 fill:#d1fae5
    style Validation fill:#fed7aa
    style Output fill:#86efac
```

### 6. 扩展架构

```mermaid
graph TB
    subgraph Horizontal["横向扩展"]
        Scale[Worker 扩展<br/>2 → 100+ 实例]
        Image[Image Workers<br/>10 → 20]
        Video[Video Workers<br/>5 → 10]
        Audio[Audio Workers<br/>3 → 5]
    end
    
    subgraph Vertical["纵向扩展"]
        GPU[GPU Worker 池<br/>智能调度]
        GPU1[GPU 0<br/>2 Workers]
        GPU2[GPU 1<br/>2 Workers]
        GPU3[GPU 2<br/>2 Workers]
        GPU4[GPU 3<br/>2 Workers]
    end
    
    subgraph Distributed["分布式部署"]
        K8s[Kubernetes<br/>容器编排]
        Node1[Node 1<br/>Image Workers]
        Node2[Node 2<br/>Video Workers]
        Node3[Node 3<br/>Audio Workers]
        Node4[Node 4<br/>Specialized Agents]
    end
    
    subgraph Autoscale["自动扩缩容"]
        Monitor[队列监控<br/>长度/负载]
        HPA[HPA<br/>水平扩展]
        ScaleUp[扩容<br/>+2 实例]
        ScaleDown[缩容<br/>-1 实例]
    end
    
    subgraph Specialized["专用 Agent"]
        Optimizer[Prompt Optimizer<br/>提示词优化]
        QA[Quality Checker<br/>质量检查]
        Style[Style Transfer<br/>风格迁移]
    end
    
    Scale --> Image
    Scale --> Video
    Scale --> Audio
    
    GPU --> GPU1
    GPU --> GPU2
    GPU --> GPU3
    GPU --> GPU4
    
    K8s --> Node1
    K8s --> Node2
    K8s --> Node3
    K8s --> Node4
    
    Monitor --> HPA
    HPA --> ScaleUp
    HPA --> ScaleDown
    
    Node4 --> Optimizer
    Node4 --> QA
    Node4 --> Style
    
    style Horizontal fill:#e1f5ff
    style Vertical fill:#f3e5f5
    style Distributed fill:#fff3e0
    style Autoscale fill:#e8f5e9
    style Specialized fill:#fce4ec
```

### 7. 部署架构

```mermaid
graph TB
    subgraph Docker["Docker Compose 部署"]
        DC[docker-compose.yml]
        API1[API Server<br/>FastAPI]
        Worker1[Workers<br/>Image/Video/Audio]
        PG1[(PostgreSQL)]
        Redis1[(Redis)]
    end
    
    subgraph K8s["Kubernetes 部署"]
        Ingress[Ingress<br/>Nginx]
        Service[Service<br/>LoadBalancer]
        Deploy[Deployment<br/>API + Workers]
        PG2[(PostgreSQL<br/>StatefulSet)]
        Redis2[(Redis<br/>Cluster)]
        PVC[PVC<br/>Shared Storage]
    end
    
    subgraph Electron["Electron 桌面应用"]
        Main[Main Process<br/>启动后端]
        Renderer[Renderer Process<br/>React UI]
        Local[(SQLite<br/>本地数据库)]
        LocalFS[Local FS<br/>output/]
    end
    
    DC --> API1
    DC --> Worker1
    DC --> PG1
    DC --> Redis1
    
    Ingress --> Service
    Service --> Deploy
    Deploy --> PG2
    Deploy --> Redis2
    Deploy --> PVC
    
    Main --> Renderer
    Main --> Local
    Main --> LocalFS
    
    style Docker fill:#e1f5ff
    style K8s fill:#f3e5f5
    style Electron fill:#fff3e0
```

## 架构图使用说明

### 主架构图
- **整体系统架构**: 展示 8 层架构的完整视图
- 适用场景: 项目介绍、技术分享、架构评审

### 子架构图
1. **数据流架构**: 展示从剧本到成片的完整流程
2. **智能体协作架构**: 展示 5 阶段 Agent 协作流程
3. **任务队列架构**: 展示 Lease 调度和 Worker 执行
4. **多供应商架构**: 展示 Provider Registry 和统一协议
5. **提示词编译架构**: 展示 3 层融合和 Seedance 约束
6. **扩展架构**: 展示横向/纵向/分布式扩展方案
7. **部署架构**: 展示 3 种部署方式

### 在线渲染
- GitHub: 自动渲染 Mermaid 图表
- VS Code: 安装 Mermaid Preview 插件
- 在线工具: https://mermaid.live/

### 导出图片
```bash
# 使用 mermaid-cli
npm install -g @mermaid-js/mermaid-cli
mmdc -i ARCHITECTURE_DIAGRAMS.md -o architecture.png
```
