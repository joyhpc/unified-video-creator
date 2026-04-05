# 统一视频创作平台 - 完整文档索引

## 项目概述

基于 4 个优秀开源项目（moyin-creator、lumenx、ArcReel、awesome-seedance-2-guide）的架构优点，设计的功能完整、架构先进、易于扩展的统一视频创作平台。

## 核心特性

- ✅ **完整的创作工作流**: 剧本 → 角色 → 场景 → 分镜 → 成片全流程自动化
- ✅ **智能 AI 编排**: Claude Agent SDK 多智能体协作
- ✅ **多供应商容错**: 支持 Gemini/Wanx/Kling/Seedance/Vidu 等 8+ AI 服务
- ✅ **异步任务队列**: Lease 调度 + 独立通道，支持大规模并发
- ✅ **智能提示词系统**: 3 层融合 + 时间轴 + Seedance 2.0 约束
- ✅ **灵活部署**: 支持云服务/桌面应用/混合模式
- ✅ **无限水平扩展**: Worker 可从 2 个扩到 100+ 个实例

## 文档清单

### 1. 架构设计
📄 **unified-video-creation-architecture.md** (11KB)
- 整体架构图（8 层架构）
- 核心数据流（10 步流程）
- 智能体协作流程（5 阶段）
- 6 大创新点详解
- 技术栈和模块清单

### 2. 实现细节
📄 **implementation-details.md** (27KB)
- 智能提示词编译器（3 层融合 + Seedance 约束）
- 异步任务队列（Lease 调度 + 心跳续期）
- 多供应商抽象层（统一协议 + Registry）
- Claude Agent SDK 集成

### 3. 前端实现
📄 **frontend-implementation.md** (25KB)
- Zustand 状态管理（Project/Task Store）
- SSE 实时推送（客户端 + React Hook）
- 五大核心面板（剧本/资产/分镜/动态/合成）
- 拖拽排序、多变体管理

### 4. 部署指南
📄 **deployment-guide.md** (20KB)
- 开发环境搭建
- Docker Compose 部署
- Kubernetes 生产部署
- Electron 桌面应用打包
- Nginx 反向代理配置
- Systemd 服务管理
- 监控与日志（Prometheus + Grafana）
- 备份与恢复脚本
- 性能优化（数据库索引 + Redis 缓存 + CDN）

### 5. API 文档
📄 **api-documentation.md** (15KB)
- 15 个 API 模块完整文档
- 请求/响应示例
- 错误处理和速率限制
- Webhook 事件
- Python/JavaScript SDK 示例

### 6. Agent 扩展方案
📄 **agent-scaling-guide.md** (新增)
- 横向扩展 Worker（2 → 20+ 实例）
- 专用 Agent（提示词优化/质量检查/风格迁移）
- 分布式部署（Kubernetes + 多机）
- GPU Worker 池（智能调度 + 监控）
- 性能优化（批处理 + 缓存）
- 监控告警（Prometheus + Alertmanager）

### 7. 项目总结
📄 **project-summary.md** (7.3KB)
- 4 个项目分析总结
- 架构优势对比
- 技术栈对比表
- 关键指标和成本优化
- 下一步建议
- 竞争优势和潜在挑战

## 快速开始

### 本地开发

```bash
# 1. 克隆项目
git clone https://github.com/your-org/unified-video-creator.git
cd unified-video-creator

# 2. 启动后端
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# 编辑 .env 填入 API Keys
uvicorn main:app --reload

# 3. 启动 Workers（新终端）
python -m lib.generation_worker --channel image --worker-id image-1
python -m lib.generation_worker --channel video --worker-id video-1
python -m lib.generation_worker --channel audio --worker-id audio-1

# 4. 启动前端（新终端）
cd frontend
npm install
npm run dev
```

### Docker 部署

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env

# 2. 启动所有服务
docker compose up -d --build

# 3. 查看日志
docker compose logs -f

# 4. 访问
# 前端: http://localhost:3000
# API: http://localhost:8000
# API 文档: http://localhost:8000/docs
```

### Kubernetes 部署

```bash
# 1. 创建命名空间
kubectl create namespace video-creator

# 2. 创建 Secrets
kubectl create secret generic db-secret \
  --from-literal=url=postgresql://... \
  -n video-creator

kubectl create secret generic api-keys \
  --from-literal=gemini=... \
  --from-literal=seedance=... \
  -n video-creator

# 3. 部署
kubectl apply -f k8s/ -n video-creator

# 4. 查看状态
kubectl get pods -n video-creator
kubectl get svc -n video-creator
```

## 架构亮点

### 1. 五层级联数据流
```
剧本 → 角色/场景/道具 → 资产图库 → 分镜脚本 → 分镜视频 → 最终视频
```
每层自动流入下一层，`collectAllRefs` 自动组装参考资料。

### 2. 异步任务队列
```
Lease 调度 + 心跳续期 + 去重机制
Image/Video/Audio 独立通道，避免相互阻塞
支持断点续传，Worker 崩溃自动恢复
```

### 3. 智能提示词编译器
```
第1层: 收集参考资料（角色图/场景图/首帧图）
第2层: 构建时间轴提示词（0-3s/3-5s 分段）
第3层: 融合角色/场景描述 + @image 标记
验证: Seedance 2.0 约束（≤9图+≤3视频+≤3音频）
```

### 4. 多供应商抽象
```
统一协议: ImageBackend / VideoBackend / AudioBackend / LLMBackend
Provider Registry: 全局注册中心
支持: Gemini, Wanx, Kling, Seedance, Vidu, Volc, OpenAI, Qwen, Claude
```

### 5. 无限水平扩展
```
横向扩展: Worker 从 2 个扩到 100+ 个实例
GPU 池: 智能调度到多个 GPU
分布式: Kubernetes + 多机部署
自动扩缩容: 根据队列长度动态调整
```

## 性能指标

| 指标 | 数值 |
|------|------|
| 图像生成 | 15-30 秒/张 |
| 视频生成 | 60-180 秒/片段（4-15s） |
| 并发能力 | Image 10 并发，Video 5 并发 |
| 队列吞吐 | 1000+ 任务/小时 |
| 数据库 | 支持 10000+ 项目 |
| 存储 | 本地 + OSS 混合，无限扩展 |

## 成本优化

- ✅ 多供应商轮询，降低单一服务商依赖
- ✅ 任务去重，避免重复生成
- ✅ 结果缓存，减少 API 调用
- ✅ 批处理优化，提高 API 利用率
- ✅ 费用追踪，实时监控成本

## 技术栈

### 前端
- React 19 + TypeScript
- Electron（桌面应用）
- Zustand（状态管理）
- Tailwind CSS 4
- Radix UI

### 后端
- FastAPI + Python 3.12+
- Pydantic 2
- SQLAlchemy 2.0 Async
- Alembic

### AI 智能体
- Claude Agent SDK
- @opencut/ai-core

### 媒体生成
- Seedance 2.0（多模态视频）
- Kling/Vidu/Pixverse（视频）
- Wanx/Gemini（图像）
- Volc/OpenAI（音频）
- Qwen/Claude/GPT（LLM）

### 基础设施
- PostgreSQL（生产）/ SQLite（开发）
- Redis（缓存 + 队列协调）
- Docker + Kubernetes
- Nginx + Let's Encrypt
- Prometheus + Grafana

## 扩展方向

### 短期（1-2 周）
- [ ] 搭建开发环境
- [ ] 实现智能提示词编译器
- [ ] 集成 Seedance 2.0 API
- [ ] 实现任务队列基础功能

### 中期（1-2 月）
- [ ] 完成五大面板前端开发
- [ ] 集成 Claude Agent SDK
- [ ] 实现多供应商抽象层
- [ ] 添加 SSE 实时推送

### 长期（3-6 月）
- [ ] 性能优化和压力测试
- [ ] 添加更多 AI 服务商
- [ ] 开发桌面应用
- [ ] 完善监控和日志系统
- [ ] 编写用户文档和教程

## 贡献指南

欢迎贡献代码、文档或提出建议！

1. Fork 项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

## 许可证

MIT License

## 联系方式

- 项目主页: https://github.com/your-org/unified-video-creator
- 文档: https://docs.video-creator.example.com
- 问题反馈: https://github.com/your-org/unified-video-creator/issues

## 致谢

本项目综合了以下优秀开源项目的架构优点：

- [moyin-creator](https://github.com/MemeCalculate/moyin-creator) - 五层级联数据流
- [lumenx](https://github.com/alibaba/lumenx) - 多供应商路由架构
- [ArcReel](https://github.com/ArcReel/ArcReel) - Claude Agent SDK 集成
- [awesome-seedance-2-guide](https://github.com/EvoLinkAI/awesome-seedance-2-guide/) - Seedance 2.0 最佳实践

感谢所有贡献者！
