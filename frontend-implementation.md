# 前端实现细节

## 1. Zustand 状态管理

### 项目状态 Store

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface Project {
  id: string;
  name: string;
  script?: Script;
  assets: Asset[];
  storyboard: StoryboardFrame[];
  motionClips: MotionClip[];
  finalVideo?: string;
  stage: 'script' | 'asset' | 'story' | 'motion' | 'assembly';
}

interface ProjectStore {
  projects: Project[];
  currentProject: Project | null;
  
  // Actions
  createProject: (name: string) => Promise<void>;
  loadProject: (id: string) => Promise<void>;
  updateScript: (script: Script) => Promise<void>;
  addAsset: (asset: Asset) => Promise<void>;
  updateAsset: (id: string, updates: Partial<Asset>) => Promise<void>;
  addStoryboardFrame: (frame: StoryboardFrame) => Promise<void>;
  updateStoryboardFrame: (id: string, updates: Partial<StoryboardFrame>) => Promise<void>;
  addMotionClip: (clip: MotionClip) => Promise<void>;
  setFinalVideo: (videoPath: string) => Promise<void>;
}

export const useProjectStore = create<ProjectStore>()(
  persist(
    (set, get) => ({
      projects: [],
      currentProject: null,
      
      createProject: async (name: string) => {
        const response = await fetch('/api/projects', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ name })
        });
        const project = await response.json();
        
        set(state => ({
          projects: [...state.projects, project],
          currentProject: project
        }));
      },
      
      loadProject: async (id: string) => {
        const response = await fetch(`/api/projects/${id}`);
        const project = await response.json();
        
        set({ currentProject: project });
      },
      
      updateScript: async (script: Script) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        const response = await fetch(`/api/projects/${currentProject.id}/script`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(script)
        });
        const updated = await response.json();
        
        set(state => ({
          currentProject: state.currentProject 
            ? { ...state.currentProject, script: updated, stage: 'asset' }
            : null
        }));
      },
      
      addAsset: async (asset: Asset) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        const response = await fetch(`/api/projects/${currentProject.id}/assets`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(asset)
        });
        const created = await response.json();
        
        set(state => ({
          currentProject: state.currentProject
            ? {
                ...state.currentProject,
                assets: [...state.currentProject.assets, created]
              }
            : null
        }));
      },
      
      updateAsset: async (id: string, updates: Partial<Asset>) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        const response = await fetch(`/api/projects/${currentProject.id}/assets/${id}`, {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(updates)
        });
        const updated = await response.json();
        
        set(state => ({
          currentProject: state.currentProject
            ? {
                ...state.currentProject,
                assets: state.currentProject.assets.map(a => 
                  a.id === id ? updated : a
                )
              }
            : null
        }));
      },
      
      addStoryboardFrame: async (frame: StoryboardFrame) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        const response = await fetch(`/api/projects/${currentProject.id}/storyboard`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(frame)
        });
        const created = await response.json();
        
        set(state => ({
          currentProject: state.currentProject
            ? {
                ...state.currentProject,
                storyboard: [...state.currentProject.storyboard, created],
                stage: 'motion'
              }
            : null
        }));
      },
      
      updateStoryboardFrame: async (id: string, updates: Partial<StoryboardFrame>) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        const response = await fetch(
          `/api/projects/${currentProject.id}/storyboard/${id}`,
          {
            method: 'PATCH',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(updates)
          }
        );
        const updated = await response.json();
        
        set(state => ({
          currentProject: state.currentProject
            ? {
                ...state.currentProject,
                storyboard: state.currentProject.storyboard.map(f =>
                  f.id === id ? updated : f
                )
              }
            : null
        }));
      },
      
      addMotionClip: async (clip: MotionClip) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        set(state => ({
          currentProject: state.currentProject
            ? {
                ...state.currentProject,
                motionClips: [...state.currentProject.motionClips, clip],
                stage: 'assembly'
              }
            : null
        }));
      },
      
      setFinalVideo: async (videoPath: string) => {
        const { currentProject } = get();
        if (!currentProject) return;
        
        set(state => ({
          currentProject: state.currentProject
            ? { ...state.currentProject, finalVideo: videoPath }
            : null
        }));
      }
    }),
    {
      name: 'project-storage',
      partialize: (state) => ({ projects: state.projects })
    }
  )
);
```

### 任务队列状态 Store

```typescript
interface Task {
  id: string;
  resourceType: 'image' | 'video' | 'audio';
  resourceId: string;
  status: 'queued' | 'running' | 'succeeded' | 'failed';
  progress?: number;
  result?: any;
  error?: string;
  createdAt: string;
  startedAt?: string;
  completedAt?: string;
}

interface TaskStore {
  tasks: Task[];
  
  // Actions
  addTask: (task: Task) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  removeTask: (id: string) => void;
  clearCompleted: () => void;
  
  // Selectors
  getTasksByResourceId: (resourceId: string) => Task[];
  getRunningTasks: () => Task[];
  getQueuedTasks: () => Task[];
}

export const useTaskStore = create<TaskStore>((set, get) => ({
  tasks: [],
  
  addTask: (task: Task) => {
    set(state => ({
      tasks: [...state.tasks, task]
    }));
  },
  
  updateTask: (id: string, updates: Partial<Task>) => {
    set(state => ({
      tasks: state.tasks.map(t => 
        t.id === id ? { ...t, ...updates } : t
      )
    }));
  },
  
  removeTask: (id: string) => {
    set(state => ({
      tasks: state.tasks.filter(t => t.id !== id)
    }));
  },
  
  clearCompleted: () => {
    set(state => ({
      tasks: state.tasks.filter(t => 
        t.status !== 'succeeded' && t.status !== 'failed'
      )
    }));
  },
  
  getTasksByResourceId: (resourceId: string) => {
    return get().tasks.filter(t => t.resourceId === resourceId);
  },
  
  getRunningTasks: () => {
    return get().tasks.filter(t => t.status === 'running');
  },
  
  getQueuedTasks: () => {
    return get().tasks.filter(t => t.status === 'queued');
  }
}));
```

## 2. SSE 实时推送

### SSE 客户端

```typescript
class SSEClient {
  private eventSource: EventSource | null = null;
  private listeners: Map<string, Set<(data: any) => void>> = new Map();
  
  connect(projectId: string) {
    if (this.eventSource) {
      this.eventSource.close();
    }
    
    this.eventSource = new EventSource(`/api/projects/${projectId}/events`);
    
    this.eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.emit(data.type, data);
    };
    
    this.eventSource.onerror = (error) => {
      console.error('SSE error:', error);
      // 自动重连
      setTimeout(() => this.connect(projectId), 5000);
    };
  }
  
  on(eventType: string, callback: (data: any) => void) {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, new Set());
    }
    this.listeners.get(eventType)!.add(callback);
  }
  
  off(eventType: string, callback: (data: any) => void) {
    const listeners = this.listeners.get(eventType);
    if (listeners) {
      listeners.delete(callback);
    }
  }
  
  private emit(eventType: string, data: any) {
    const listeners = this.listeners.get(eventType);
    if (listeners) {
      listeners.forEach(callback => callback(data));
    }
  }
  
  disconnect() {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }
  }
}

export const sseClient = new SSEClient();
```

### React Hook

```typescript
import { useEffect } from 'react';
import { sseClient } from './sse-client';
import { useTaskStore } from './stores/task-store';

export function useSSE(projectId: string) {
  const updateTask = useTaskStore(state => state.updateTask);
  const addTask = useTaskStore(state => state.addTask);
  
  useEffect(() => {
    // 连接 SSE
    sseClient.connect(projectId);
    
    // 监听任务更新
    const handleTaskUpdate = (data: any) => {
      updateTask(data.taskId, {
        status: data.status,
        progress: data.progress,
        result: data.result,
        error: data.error
      });
    };
    
    // 监听新任务
    const handleTaskCreated = (data: any) => {
      addTask(data.task);
    };
    
    // 监听阶段完成
    const handleStageComplete = (data: any) => {
      // 显示确认对话框
      showConfirmDialog(data);
    };
    
    sseClient.on('task_update', handleTaskUpdate);
    sseClient.on('task_created', handleTaskCreated);
    sseClient.on('stage_complete', handleStageComplete);
    
    return () => {
      sseClient.off('task_update', handleTaskUpdate);
      sseClient.off('task_created', handleTaskCreated);
      sseClient.off('stage_complete', handleStageComplete);
      sseClient.disconnect();
    };
  }, [projectId, updateTask, addTask]);
}
```

## 3. 五大核心面板

### 剧本编辑器

```typescript
import { useState } from 'react';
import { useProjectStore } from '../stores/project-store';

export function ScriptPanel() {
  const currentProject = useProjectStore(state => state.currentProject);
  const updateScript = useProjectStore(state => state.updateScript);
  const [scriptText, setScriptText] = useState(currentProject?.script?.text || '');
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  
  const handleAnalyze = async () => {
    setIsAnalyzing(true);
    
    try {
      // 调用 AI 分析剧本
      const response = await fetch('/api/assistant/analyze-script', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          projectId: currentProject?.id,
          scriptText
        })
      });
      
      const result = await response.json();
      
      // 更新剧本
      await updateScript({
        text: scriptText,
        entities: result.entities,
        scenes: result.scenes
      });
    } finally {
      setIsAnalyzing(false);
    }
  };
  
  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center justify-between p-4 border-b">
        <h2 className="text-lg font-semibold">剧本编辑器</h2>
        <button
          onClick={handleAnalyze}
          disabled={isAnalyzing || !scriptText}
          className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
        >
          {isAnalyzing ? '分析中...' : 'AI 分析'}
        </button>
      </div>
      
      <textarea
        value={scriptText}
        onChange={(e) => setScriptText(e.target.value)}
        placeholder="输入剧本内容..."
        className="flex-1 p-4 resize-none focus:outline-none"
      />
      
      {currentProject?.script?.entities && (
        <div className="p-4 border-t bg-gray-50">
          <h3 className="font-semibold mb-2">提取的实体</h3>
          <div className="space-y-2">
            <div>
              <span className="text-sm text-gray-600">角色:</span>
              <div className="flex flex-wrap gap-2 mt-1">
                {currentProject.script.entities.characters.map(char => (
                  <span key={char.id} className="px-2 py-1 bg-blue-100 text-blue-800 rounded text-sm">
                    {char.name}
                  </span>
                ))}
              </div>
            </div>
            <div>
              <span className="text-sm text-gray-600">场景:</span>
              <div className="flex flex-wrap gap-2 mt-1">
                {currentProject.script.entities.scenes.map(scene => (
                  <span key={scene.id} className="px-2 py-1 bg-green-100 text-green-800 rounded text-sm">
                    {scene.name}
                  </span>
                ))}
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
```

### 资产工作台

```typescript
import { useState } from 'react';
import { useProjectStore } from '../stores/project-store';
import { useTaskStore } from '../stores/task-store';

export function AssetPanel() {
  const currentProject = useProjectStore(state => state.currentProject);
  const addAsset = useProjectStore(state => state.addAsset);
  const updateAsset = useProjectStore(state => state.updateAsset);
  const tasks = useTaskStore(state => state.tasks);
  
  const [selectedEntity, setSelectedEntity] = useState<string | null>(null);
  const [isGenerating, setIsGenerating] = useState(false);
  
  const handleGenerate = async (entityId: string, entityType: string) => {
    setIsGenerating(true);
    
    try {
      // 入队生成任务
      const response = await fetch('/api/generate/asset', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          projectId: currentProject?.id,
          entityId,
          entityType,
          numVariants: 4  // 生成 4 个变体
        })
      });
      
      const result = await response.json();
      
      // 任务会通过 SSE 推送更新
    } finally {
      setIsGenerating(false);
    }
  };
  
  const handleSelectVariant = async (assetId: string, variantId: string) => {
    await updateAsset(assetId, {
      selectedVariantId: variantId
    });
  };
  
  const entities = [
    ...(currentProject?.script?.entities.characters || []).map(c => ({ ...c, type: 'character' })),
    ...(currentProject?.script?.entities.scenes || []).map(s => ({ ...s, type: 'scene' })),
    ...(currentProject?.script?.entities.props || []).map(p => ({ ...p, type: 'prop' }))
  ];
  
  return (
    <div className="flex h-full">
      {/* 左侧实体列表 */}
      <div className="w-64 border-r overflow-y-auto">
        <div className="p-4 border-b">
          <h2 className="text-lg font-semibold">实体列表</h2>
        </div>
        <div className="p-2">
          {entities.map(entity => {
            const asset = currentProject?.assets.find(a => a.entityId === entity.id);
            const entityTasks = tasks.filter(t => t.resourceId === entity.id);
            const isGenerating = entityTasks.some(t => t.status === 'running' || t.status === 'queued');
            
            return (
              <button
                key={entity.id}
                onClick={() => setSelectedEntity(entity.id)}
                className={`w-full p-3 text-left rounded mb-2 ${
                  selectedEntity === entity.id ? 'bg-blue-100' : 'hover:bg-gray-100'
                }`}
              >
                <div className="flex items-center justify-between">
                  <span className="font-medium">{entity.name}</span>
                  {isGenerating && (
                    <span className="text-xs text-blue-600">生成中...</span>
                  )}
                  {asset && !isGenerating && (
                    <span className="text-xs text-green-600">✓</span>
                  )}
                </div>
                <span className="text-xs text-gray-500">{entity.type}</span>
              </button>
            );
          })}
        </div>
      </div>
      
      {/* 右侧资产详情 */}
      <div className="flex-1 overflow-y-auto">
        {selectedEntity ? (
          <AssetDetail
            entity={entities.find(e => e.id === selectedEntity)!}
            asset={currentProject?.assets.find(a => a.entityId === selectedEntity)}
            onGenerate={handleGenerate}
            onSelectVariant={handleSelectVariant}
            isGenerating={isGenerating}
          />
        ) : (
          <div className="flex items-center justify-center h-full text-gray-400">
            选择一个实体查看详情
          </div>
        )}
      </div>
    </div>
  );
}

function AssetDetail({ entity, asset, onGenerate, onSelectVariant, isGenerating }) {
  return (
    <div className="p-6">
      <div className="flex items-center justify-between mb-6">
        <div>
          <h2 className="text-2xl font-bold">{entity.name}</h2>
          <p className="text-gray-600 mt-1">{entity.description}</p>
        </div>
        <button
          onClick={() => onGenerate(entity.id, entity.type)}
          disabled={isGenerating}
          className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
        >
          {isGenerating ? '生成中...' : asset ? '重新生成' : '生成资产'}
        </button>
      </div>
      
      {asset && (
        <div>
          <h3 className="font-semibold mb-4">变体 ({asset.variants.length})</h3>
          <div className="grid grid-cols-2 gap-4">
            {asset.variants.map(variant => (
              <div
                key={variant.id}
                className={`relative border-2 rounded-lg overflow-hidden cursor-pointer ${
                  variant.id === asset.selectedVariantId
                    ? 'border-blue-600'
                    : 'border-gray-200 hover:border-gray-400'
                }`}
                onClick={() => onSelectVariant(asset.id, variant.id)}
              >
                <img
                  src={variant.url}
                  alt={`${entity.name} - 变体 ${variant.index}`}
                  className="w-full aspect-square object-cover"
                />
                {variant.id === asset.selectedVariantId && (
                  <div className="absolute top-2 right-2 bg-blue-600 text-white px-2 py-1 rounded text-xs">
                    已选择
                  </div>
                )}
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

### 分镜导演台

```typescript
import { useState } from 'react';
import { useProjectStore } from '../stores/project-store';
import { DragDropContext, Droppable, Draggable } from '@hello-pangea/dnd';

export function StoryboardPanel() {
  const currentProject = useProjectStore(state => state.currentProject);
  const addStoryboardFrame = useProjectStore(state => state.addStoryboardFrame);
  const updateStoryboardFrame = useProjectStore(state => state.updateStoryboardFrame);
  
  const [isGenerating, setIsGenerating] = useState(false);
  
  const handleGenerateStoryboard = async () => {
    setIsGenerating(true);
    
    try {
      // 调用 AI 生成分镜脚本
      const response = await fetch('/api/assistant/generate-storyboard', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          projectId: currentProject?.id
        })
      });
      
      const result = await response.json();
      
      // 添加分镜帧
      for (const frame of result.frames) {
        await addStoryboardFrame(frame);
      }
    } finally {
      setIsGenerating(false);
    }
  };
  
  const handleDragEnd = async (result) => {
    if (!result.destination) return;
    
    const frames = Array.from(currentProject?.storyboard || []);
    const [removed] = frames.splice(result.source.index, 1);
    frames.splice(result.destination.index, 0, removed);
    
    // 更新顺序
    for (let i = 0; i < frames.length; i++) {
      await updateStoryboardFrame(frames[i].id, { order: i });
    }
  };
  
  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center justify-between p-4 border-b">
        <h2 className="text-lg font-semibold">分镜导演台</h2>
        <button
          onClick={handleGenerateStoryboard}
          disabled={isGenerating || !currentProject?.assets.length}
          className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
        >
          {isGenerating ? '生成中...' : 'AI 生成分镜'}
        </button>
      </div>
      
      <div className="flex-1 overflow-y-auto p-4">
        {currentProject?.storyboard.length ? (
          <DragDropContext onDragEnd={handleDragEnd}>
            <Droppable droppableId="storyboard">
              {(provided) => (
                <div
                  {...provided.droppableProps}
                  ref={provided.innerRef}
                  className="space-y-4"
                >
                  {currentProject.storyboard.map((frame, index) => (
                    <Draggable key={frame.id} draggableId={frame.id} index={index}>
                      {(provided) => (
                        <div
                          ref={provided.innerRef}
                          {...provided.draggableProps}
                          {...provided.dragHandleProps}
                        >
                          <StoryboardFrameCard frame={frame} index={index} />
                        </div>
                      )}
                    </Draggable>
                  ))}
                  {provided.placeholder}
                </div>
              )}
            </Droppable>
          </DragDropContext>
        ) : (
          <div className="flex items-center justify-center h-full text-gray-400">
            {currentProject?.assets.length
              ? '点击"AI 生成分镜"开始'
              : '请先完成资产生成'}
          </div>
        )}
      </div>
    </div>
  );
}

function StoryboardFrameCard({ frame, index }) {
  const updateStoryboardFrame = useProjectStore(state => state.updateStoryboardFrame);
  const [isEditing, setIsEditing] = useState(false);
  const [prompt, setPrompt] = useState(frame.prompt);
  
  const handleSave = async () => {
    await updateStoryboardFrame(frame.id, { prompt });
    setIsEditing(false);
  };
  
  return (
    <div className="border rounded-lg p-4 bg-white shadow-sm">
      <div className="flex items-start gap-4">
        <div className="flex-shrink-0 w-12 h-12 bg-blue-100 rounded flex items-center justify-center font-bold text-blue-600">
          {index + 1}
        </div>
        
        <div className="flex-1">
          <div className="flex items-center justify-between mb-2">
            <h3 className="font-semibold">{frame.scene}</h3>
            <button
              onClick={() => setIsEditing(!isEditing)}
              className="text-sm text-blue-600 hover:underline"
            >
              {isEditing ? '取消' : '编辑'}
            </button>
          </div>
          
          {isEditing ? (
            <div>
              <textarea
                value={prompt}
                onChange={(e) => setPrompt(e.target.value)}
                className="w-full p-2 border rounded resize-none"
                rows={4}
              />
              <button
                onClick={handleSave}
                className="mt-2 px-3 py-1 bg-blue-600 text-white rounded text-sm hover:bg-blue-700"
              >
                保存
              </button>
            </div>
          ) : (
            <p className="text-gray-600 text-sm">{frame.prompt}</p>
          )}
          
          <div className="mt-3 flex items-center gap-4 text-sm text-gray-500">
            <span>时长: {frame.duration}s</span>
            <span>镜头: {frame.cameraMotion}</span>
          </div>
        </div>
      </div>
    </div>
  );
}
```

继续下一部分...
