# 「拾光」项目系统化学习指南

> 一个 Electron 桌面应用的完整技术拆解 —— 从整体架构到每一行真实代码
>
> 适用对象：想通过学习「拾光」源码掌握 Electron + React + TypeScript 全栈开发的同学
> 阅读方式：按顺序从第 1 章读到第 8 章；学有余力再挑战第 9 章
> 项目版本：v3.1.x（含 v3.2 部分 UI 预留）

---

## 目录

1. [项目全景：拾光是什么](#1-项目全景拾光是什么)
2. [整体架构：模块划分与数据流转](#2-整体架构模块划分与数据流转)
3. [技术栈全景：按层分类的完整清单](#3-技术栈全景按层分类的完整清单)
4. [关键模块代码实战（一）：进程通信 IPC](#4-关键模块代码实战一进程通信-ipc)
5. [关键模块代码实战（二）：状态管理与数据建模](#5-关键模块代码实战二状态管理与数据建模)
6. [关键模块代码实战（三）：接口调用与核心服务](#6-关键模块代码实战三接口调用与核心服务)
7. [关键模块代码实战（四）：UI 层与「伪路由」](#7-关键模块代码实战四ui-层与伪路由)
8. [学习路线图：从零基础到读懂本项目](#8-学习路线图从零基础到读懂本项目)
9. [进阶专题（选读）](#9-进阶专题选读)
10. [术语表](#10-术语表)
11. [难点答疑：初学者常见困惑](#11-难点答疑初学者常见困惑)

---

## 1. 项目全景：拾光是什么

### 1.1 一句话介绍

**拾光**（英文名 ScreenAura，开发代号 screen-ai-tracker）是一个**完全本地运行的桌面时间追踪工具**：它每隔一段时间自动截一张屏幕截图，交给本地的 AI 视觉模型（Ollama + qwen3-vl）理解"你正在做什么"，然后整理成活动记录、10 分钟汇总、每日日记，还能回答"我昨天下午在干嘛"这类问题。

### 1.2 核心功能一览

| 功能 | 引入版本 | 一句话说明 |
|------|---------|-----------|
| 定时截屏 | v1 | 每 20 秒（可配置）截取屏幕 |
| AI 活动总结 | v1 | 本地视觉模型把截图翻译成 5 句话中文描述 |
| 小时/月度统计 | v1 | 按小时 JSON 文件组织记录 |
| 系统托盘 | v1 | 托盘图标 + 右键菜单控制采集 |
| 10 分钟汇总 | v2 | 把多条记录合并成一段自然语言工作摘要 |
| 每日日记 | v2 | 基于当天汇总生成结构化日记 |
| 数据看板 | v2 | 饼图/柱状图展示分类、软件、项目专注度 |
| 智能问答 | v2 | 检索式问答（自动搜索记录→生成答案） |
| 截图清理 | v3 | 分级清理旧截图，控制磁盘占用 |
| pHash 去重 | v3 | 画面没变就跳过 AI 分析，省钱省算力 |
| 负载感知 | v3.1 | 电脑高负载时自动降级为"事件驱动"静默记录 |

### 1.3 为什么值得学这个项目

它几乎覆盖了**现代桌面应用开发的全部知识点**，而且每个点都有真实、完整的落地代码：

- **Electron 三进程模型**：主进程 / preload / 渲染进程，一个不少
- **进程间通信（IPC）**：`invoke/handle` 请求-响应 + `send/on` 事件推送，两种模式都有
- **TypeScript 全量类型安全**：主进程和渲染进程共享一套类型定义
- **React 18 + Zustand**：UI 层标准组合
- **本地 AI 集成**：HTTP 调用 Ollama REST API，含重试、降级、自动拉起
- **文件系统存储**：JSON 按小时组织、原子写入、内存缓存
- **系统能力调用**：截屏（screenshot-desktop）、图像处理（sharp）、系统信息（systeminformation）
- **工程化**：Vite 三份构建配置、Vitest 单元测试（366 个）、electron-builder 打包

---

## 2. 整体架构：模块划分与数据流转

### 2.1 Electron 的"三进程"世界观（必须先懂）

传统网页应用只有"浏览器"一个运行环境。**Electron 桌面应用**把 Chrome 拆成了几个独立运行的部分，理解这个模型是看懂一切代码的前提：

```
┌──────────────────────────────────────────────────────────┐
│                   你的 Electron 应用                      │
│                                                          │
│  ┌──────────────────────────────────────────┐            │
│  │  主进程 Main Process（Node.js 环境）      │            │
│  │  · 只有它能用 Node API（文件、网络、系统） │            │
│  │  · 管理窗口生命周期、托盘、定时任务        │            │
│  └───────────────▲──────────────────────────┘            │
│                  │  IPC（进程间通信）                     │
│  ┌───────────────┴──────────────┐                        │
│  │  Preload 预加载脚本           │                        │
│  │  · 唯一桥梁，安全暴露 API     │                        │
│  └───────────────▲──────────────┘                        │
│                  │                                       │
│  ┌───────────────┴──────────────────────────┐            │
│  │  渲染进程 Renderer（Chrome 页面环境）      │            │
│  │  · React UI、浏览器 API（DOM、CSS）       │            │
│  │  · 没有 Node API（被隔离，更安全）        │            │
│  └──────────────────────────────────────────┘            │
└──────────────────────────────────────────────────────────┘
```

**通俗理解**：主进程像"公司总部"（掌握所有资源，但离用户远）；渲染进程像"前台接待"（直接面对用户，但权限很小）；preload 像"内部对讲机"（总部只通过它给前台传递消息）。

本项目还有**第四个角色**——外部进程 **Ollama**：一个独立的本地 AI 服务（跑在 `localhost:11434`），主进程用 HTTP 请求它分析图片。

### 2.2 模块划分图

本项目按"功能职责"把主进程拆成一个个 **Service（服务）**，由入口 `index.ts` 统一"装配"（创建 + 互相连接）。这是非常典型的**依赖注入 / 组合根**模式：

```
src/
├── main/                        # 主进程（Node.js 世界）
│   ├── index.ts                 # 入口：装配所有服务（组合根）
│   ├── windows/MainWindow.ts    # 主窗口创建
│   ├── services/                # ★ 业务核心：14 个服务
│   │   ├── SettingsService      #   设置读写（settings.json）
│   │   ├── ScreenshotService    #   截屏
│   │   ├── OllamaService        #   AI 调用（视觉+文本+批量）
│   │   ├── StorageService       #   记录存储（JSON 原子写）
│   │   ├── CaptureScheduler     #   ★ 采集调度器（核心状态机）
│   │   ├── DedupService         #   pHash 去重
│   │   ├── SummaryService       #   10 分钟汇总
│   │   ├── DiaryService         #   每日日记
│   │   ├── AnalyticsService     #   看板聚合
│   │   ├── QAService            #   智能问答
│   │   ├── CleanupService       #   截图清理
│   │   ├── LoadMonitorService   #   系统负载监测
│   │   ├── ContextAwareCache    #   高负载缓存
│   │   └── IpcService           #   ★ IPC 通道注册中心
│   ├── utils/                   # 工具函数（logger/paths/phash/...）
│   └── types/                   # 主进程内部类型
├── preload/index.ts             # preload 脚本：contextBridge 暴露 window.api
├── renderer/                    # 渲染进程（浏览器世界）
│   ├── main.tsx                 # React 挂载入口
│   ├── App.tsx                  # 根组件
│   ├── store/appStore.ts        # Zustand 全局状态
│   ├── hooks/                   # 数据订阅 hooks
│   └── components/              # 组件（tabs/activity/diary/qa/...）
└── shared/                      # ★ 两进程共享
    ├── types.ts                 #   全部数据类型（契约）
    └── constants.ts             #   IPC 通道名、默认设置、业务常量
```

### 2.3 数据流转主链路（v1 核心循环）

这是全项目最重要的一条链路：**截屏 → AI 分析 → 存储 → 推送 UI**。

```
┌──────────┐  每20秒   ┌────────────────┐
│ 定时器    │──────────▶│ Screenshot     │  截取屏幕
│ (Scheduler)│          │ Service        │  → Buffer
└──────────┘           └───────┬────────┘
                               │ base64 图片
                               ▼
                        ┌──────────────┐   相似则跳过
                        │ DedupService │──▶(省钱)
                        │ (pHash比对)  │
                        └───────┬──────┘
                                │ 不相似
                                ▼
                        ┌──────────────┐  POST /api/chat
                        │ OllamaService│──▶ 本地AI → 5句话总结
                        └───────┬──────┘
                                │ LogRecord
                                ▼
                        ┌──────────────┐  写 JSON 文件
                        │ StorageService│  (原子写入)
                        └───────┬──────┘
                                │ 推送 capture:status
                                ▼
                        ┌─────────────────────────┐
                        │ preload → 渲染进程       │
                        │ → Zustand → React UI    │
                        └─────────────────────────┘
```

### 2.4 数据流转主链路（v3.1 负载感知分支）

电脑高负载时（CPU>70% 等），应用切换到"事件驱动"模式：**不按时截屏，只在高负载进入/退出时抓首尾两帧**，把中间过程压缩成一条"高负载会话"记录。这是本项目最精巧的设计，细节见第 6 章。

### 2.5 存储布局（"数据库"长什么样）

本项目没有数据库，用**文件系统当数据库**：

```
dataRoot/  （默认 ~/Documents/ScreenAITracker）
├── YYYY-MM/                    # 按月份分目录
│   └── YYYYMMDD_HH.json        # 按小时存记录（如 20260811_14.json）
├── summaries/                  # 10 分钟汇总（v2）
├── diaries/                    # 每日日记（v2）
├── screenshots/                # 截图文件（jpg）
├── .pending/                   # 高负载缓存区（v3.1）
│   ├── sessions/               # 会话元数据
│   └── frames/                 # 首尾帧图片
└── cache-manifest.json         # 缓存索引
```

---

## 3. 技术栈全景：按层分类的完整清单

以下清单直接来自项目的 `package.json`（依赖版本为真实值）。

### 3.1 编程语言层

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| TypeScript | ^5.5.4 | 全项目语言。主/渲染/共享代码全部用 TS，靠它保证跨进程数据形状一致 |
| Node.js | ≥18（运行时） | 主进程的运行环境，提供 fs/path/child_process 等系统能力 |

### 3.2 框架层（应用骨架）

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| Electron | ^31.3.0 | 桌面应用容器。提供 main/preload/renderer 三进程结构 |
| React | ^18.3.1 | UI 库。渲染进程所有界面都是 React 组件 |
| react-dom | ^18.3.1 | 把 React 组件挂载到真实 DOM |

### 3.3 核心库层（功能组件）

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| Zustand | ^5.0.0 | 全局状态管理。渲染进程所有共享数据存这里 |
| axios | ^1.7.7 | HTTP 客户端。调 Ollama REST API（聊天/视觉/批量分析） |
| dayjs | ^1.11.13 | 日期时间库。格式化小时键（YYYYMMDD_HH）、统计周期 |
| screenshot-desktop | ^1.15.0 | 截屏。返回屏幕图像 Buffer |
| sharp | ^0.33.5 | 图像处理。pHash 计算前的缩放/灰度化；截图转 JPEG |
| systeminformation | ^5.22.0 | 系统信息。采集 CPU/GPU/显存占用，驱动负载感知 |
| react-window | ^1.8.11 | 虚拟列表。动态时间线大量记录只渲染可见部分 |
| recharts | ^3.10.1 | 图表库。看板页的饼图/柱状图 |
| lucide-react | ^1.30.0 | 图标库。UI 各处线性图标 |

### 3.4 构建与开发工具层

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| Vite | ^5.4.3 | 构建工具。dev 热更新 + 三份生产构建配置 |
| @vitejs/plugin-react | ^4.3.1 | Vite 的 React 插件（JSX 编译、Fast Refresh） |
| vite-plugin-electron | ^0.28.8 | Vite 与 Electron 集成（dev 时自动起主进程） |
| Tailwind CSS | ^3.4.10 | 原子化 CSS 框架。全部 UI 样式 |
| postcss / autoprefixer | ^8.4.45 / ^10.4.20 | Tailwind 的底层处理链（CSS 编译、浏览器前缀） |

### 3.5 测试层

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| Vitest | ^2.1.9 | 单元测试框架。25 个测试文件 / 366 个用例，覆盖核心服务 |

### 3.6 打包部署层

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| electron-builder | ^24.13.3 | 打包成安装包（NSIS exe），处理图标、asar、资源 |
| NSIS（内嵌） | — | Windows 安装程序脚本，含自定义 `installer.nsh`（杀旧进程） |

### 3.7 外部运行时依赖

| 技术 | 版本 | 项目中的实际用途 |
|------|------|----------------|
| Ollama | 本地服务 | AI 推理引擎，监听 localhost:11434 |
| qwen3-vl:4b-instruct-q4_k_m | 模型 | 视觉语言模型：看图说话、汇总、日记、问答 |

### 3.8 依赖全景图

```
                       ┌────────────────────────────┐
                       │   TypeScript（全项目语言）  │
                       └──────┬─────────────┬───────┘
              ┌────────────────┘             └────────────────┐
              ▼                                              ▼
   ┌────────────────────┐                        ┌────────────────────┐
   │  主进程 (Node)      │                        │  渲染进程 (React)   │
   │  electron           │                        │  react/react-dom    │
   │  screenshot-desktop │                        │  zustand            │
   │  sharp              │                        │  tailwindcss        │
   │  systeminformation  │                        │  recharts           │
   │  axios → Ollama     │                        │  react-window       │
   │  dayjs              │                        │  lucide-react       │
   └────────────────────┘                        └────────────────────┘
              │                                            │
              └─────────────── Vite 三份配置 ───────────────┘
                              │
                     ┌────────┴────────┐
                     ▼                 ▼
              Vitest（测试）    electron-builder（打包）
```

---

## 4. 关键模块代码实战（一）：进程通信 IPC

**IPC（Inter-Process Communication，进程间通信）** 是本项目所有"前后端交互"的通道。它有两种模式，本项目两种都用：

| 模式 | 方向 | API | 适用场景 |
|------|------|-----|---------|
| 请求-响应 | 渲染进程 → 主进程 | `ipcRenderer.invoke` / `ipcMain.handle` | "帮我开始采集"、"给我设置" |
| 事件推送 | 主进程 → 渲染进程 | `webContents.send` / `ipcRenderer.on` | "新记录产生了"、"负载变高了" |

### 4.1 第一步：通道名常量（shared/constants.ts）

所有 IPC 通道名集中定义在 `src/shared/constants.ts`，保证三处（主/preload/渲染）引用同一个字符串，不会打错：

```ts
export const IPC_CHANNELS = {
  // 渲染进程 → 主进程（invoke，请求-响应）
  CAPTURE_START: 'capture:start',
  SETTINGS_GET: 'settings:get',
  QA_ASK: 'qa:ask',
  // 主进程 → 渲染进程（push，事件推送）
  CAPTURE_STATUS: 'capture:status',
  ACTIVITY_NEW: 'activity:new',
  OLLAMA_STATUS: 'ollama:status',
  // ...（共 30+ 个通道）
} as const;
```

> **命名约定**：通道名用 `域名:动作` 格式（如 `capture:start`），一眼看出属于哪个模块、做什么。

### 4.2 第二步：preload 安全桥（src/preload/index.ts）

渲染进程**不能直接碰 ipcRenderer**（安全隔离）。preload 脚本用 `contextBridge` 把通道包装成一个个好用的函数，暴露为全局对象 `window.api`：

```ts
import { contextBridge, ipcRenderer, type IpcRendererEvent } from 'electron';
import { IPC_CHANNELS } from '../shared/constants';

const api: ElectronAPI = {
  capture: {
    start: (): Promise<void> => ipcRenderer.invoke(IPC_CHANNELS.CAPTURE_START),
    pause: (): Promise<void> => ipcRenderer.invoke(IPC_CHANNELS.CAPTURE_PAUSE),
    stop: (): Promise<void> => ipcRenderer.invoke(IPC_CHANNELS.CAPTURE_STOP),
  },
  settings: {
    get: (): Promise<AppSettings> => ipcRenderer.invoke(IPC_CHANNELS.SETTINGS_GET),
    set: (settings: Partial<AppSettings>): Promise<AppSettings> =>
      ipcRenderer.invoke(IPC_CHANNELS.SETTINGS_SET, settings),
  },
  on: {
    // 订阅主进程推送 —— 返回"取消订阅函数"（重要模式！）
    captureStatus: (callback: (status: CaptureStatus) => void): (() => void) => {
      const handler = (_e: IpcRendererEvent, status: CaptureStatus): void => callback(status);
      ipcRenderer.on(IPC_CHANNELS.CAPTURE_STATUS, handler);
      return () => {
        ipcRenderer.removeListener(IPC_CHANNELS.CAPTURE_STATUS, handler);  // 清理，防内存泄漏
      };
    },
    // ...
  },
};

contextBridge.exposeInMainWorld('api', api);
```

**这段代码的三个学习要点**：
1. **类型**：`ElectronAPI` 接口定义在 `shared/types.ts`（见第 5 章），渲染进程的 `window.api` 有完整类型提示。
2. **取消订阅**：每个 `on.xxx` 都返回一个清理函数。React 的 `useEffect` 里 `return unsubscribe` 就能在组件卸载时自动取消，**这是防内存泄漏的标准写法**。
3. **安全**：渲染进程只知道 `window.api` 这些函数，永远接触不到原始 ipcRenderer，也无法越权。

### 4.3 第三步：主进程处理（src/main/services/IpcService.ts）

主进程把所有 `ipcMain.handle` 集中注册在 `IpcService.registerAll()` 里。它就像一个"总机"，收到请求后分发给对应的业务 Service：

```ts
export class IpcService {
  // 构造时注入所有服务（依赖注入）
  constructor(deps: {
    scheduler: CaptureScheduler;
    ollamaService: OllamaService;
    storageService: StorageService;
    // ... 十几个服务
  }) { /* 保存引用 */ }

  /** 注册所有 IPC 通道处理器 */
  registerAll(): void {
    // 采集控制
    ipcMain.handle(IPC_CHANNELS.CAPTURE_START, (): void => this.scheduler.start());
    ipcMain.handle(IPC_CHANNELS.CAPTURE_PAUSE, (): void => this.scheduler.pause());
    ipcMain.handle(IPC_CHANNELS.CAPTURE_STOP, (): void => this.scheduler.stop());

    // 设置
    ipcMain.handle(IPC_CHANNELS.SETTINGS_GET, () => this.settingsService.getSettings());
    ipcMain.handle(IPC_CHANNELS.SETTINGS_SET, (_e, partial: Partial<AppSettings>) =>
      this.settingsService.setSettings(partial));

    // 问答（复杂的业务转发给 QAService）
    ipcMain.handle(IPC_CHANNELS.QA_ASK, (_e, question: string) =>
      this.qaService.ask(question));
    // ... 共 30+ 个 handler
  }
}
```

### 4.4 第四步：主进程 → 渲染进程推送

采集完成、状态变化时，主进程**主动**推给 UI。推送入口封装在 IpcService 里：

```ts
/** 推送采集状态到渲染进程 */
pushCaptureStatus(status: CaptureStatus): void {
  this.mainWindow?.webContents.send(IPC_CHANNELS.CAPTURE_STATUS, status);
}
```

`buildAndPushStatus`（在 `index.ts` 中）把分散的数据组装成一次完整状态快照再推送：

```ts
function buildAndPushStatus(state: CaptureState, ollamaStatus: OllamaStatus): void {
  const status: CaptureStatus = {
    state,
    totalCaptured: counters.totalCaptured,
    successCount: counters.successCount,
    ollamaStatus,
    ollamaModel: settings.ollamaModel,
    // v3.1: 并入负载模式与待分析缓存数
    ...(loadMode !== undefined ? { loadMode } : {}),
    ...(pendingCount > 0 ? { pendingCount } : {}),
  };
  ipcService.pushCaptureStatus(status);
  trayService.updateMenu(state);  // 托盘菜单也一起更新
}
```

### 4.5 IPC 全链路小结

```
渲染进程组件                  preload                     主进程
─────────                   ───────                      ──────
button onClick
  → window.api.capture.start()
      → ipcRenderer.invoke('capture:start')
          ──────────────────────────────▶  ipcMain.handle('capture:start')
                                               → scheduler.start()
                                               → 状态变了
                                               → webContents.send('capture:status', s)
          ◀──────────────────────────────
      → 回调触发（on.captureStatus 注册的）
  → setState(Zustand)
  → React 重渲染，UI 更新
```

---

## 5. 关键模块代码实战（二）：状态管理与数据建模

### 5.1 数据建模：shared/types.ts（全项目的"契约"）

`src/shared/types.ts` 定义了所有跨进程传输的数据结构。它的角色类似数据库的"表结构定义"——主进程生成数据、preload 传输、渲染进程消费，**三方都遵守同一份类型**，这就是 TypeScript 在大型项目里最大的价值。

看两个最核心的类型：

```ts
/** 一条活动记录（对应一次"截图+AI分析"的产物） */
export interface LogRecord {
  timestamp: number;              // Unix 时间戳（秒级）
  screenshot_path: string;        // 截图相对路径
  activity_summary: string;       // AI 生成的 5 句话中文总结
  status: RecordStatus;           // success | failed | placeholder
  repeat_count?: number;          // v3: 相同活动合并计数
  dedup?: boolean;                // v3: 是否去重跳过分析
  last_seen?: number;             // v3: 最后看到该活动的时间
  loadContext?: LoadContext;      // v3.1: 负载上下文
}

/** 应用设置（很长，截取一部分） */
export interface AppSettings {
  captureInterval: number;        // 截屏间隔秒数
  autoStart: boolean;             // 开机自启
  ollamaEndpoint: string;         // http://localhost:11434
  ollamaModel: string;            // qwen3-vl:4b-instruct-q4_k_m
  dataRoot: string;               // 数据目录
  // v3: 清理/去重/补处理配置...
  // v3.1: 运行模式/负载感知/缓存/AI批量配置...
}
```

**给渲染进程的全局 API 类型**（`ElectronAPI`）也在同一文件里，最后通过 `declare global` 让 `window.api` 全局可用：

```ts
export interface ElectronAPI {
  capture: { start(): Promise<void>; pause(): Promise<void>; stop(): Promise<void> };
  settings: { get(): Promise<AppSettings>; set(s: Partial<AppSettings>): Promise<AppSettings> };
  on: {
    captureStatus: (cb: (s: CaptureStatus) => void) => () => void;
    activityNew: (cb: (r: LogRecord) => void) => () => void;
    // ...
  };
  summary: { trigger(periodStart: number): Promise<void>; getRange(s: number, e: number): Promise<SummaryRecord[]> };
  diary: { get(date: string): Promise<DiaryRecord | null>; generate(date: string): Promise<DiaryRecord> };
  qa: { ask(question: string): Promise<QAResponse> };
  // ...
}

declare global {
  interface Window { api: ElectronAPI; }
}
```

> **学习提示**：读懂这个文件，你就掌握了"这个应用到底有哪些数据、哪些操作"——它是整个项目的"词汇表"。

### 5.2 状态管理：Zustand（src/renderer/store/appStore.ts）

渲染进程的全局状态全部放在一个 Zustand store 里。Zustand 的设计哲学是"小而美"：一个 `create()` 函数搞定 store + action：

```ts
import { create } from 'zustand';

interface AppState {
  // ── 状态（数据） ──
  captureStatus: CaptureStatus | null;   // 采集状态
  ollamaStatus: OllamaStatus;            // AI 连接状态
  activities: LogRecord[];               // 活动记录（最多 200 条）
  settings: AppSettings | null;          // 应用设置
  activeTab: TabType;                    // 当前激活的标签页
  summaryStatus: SummaryStatus | null;   // 汇总状态
  // ... 还有日记/看板/问答/存储/负载等 20+ 个状态字段

  // ── Actions（改状态的函数） ──
  setCaptureStatus: (status: CaptureStatus) => void;
  addActivity: (record: LogRecord) => void;
  setActiveTab: (tab: TabType) => void;
  // ...
}

export const useAppStore = create<AppState>((set) => ({
  captureStatus: null,
  ollamaStatus: 'disconnected',
  activities: [],
  settings: null,
  activeTab: 'dashboard',

  // action 实现 —— 注意：set 是 Zustand 提供的"改状态"函数
  setCaptureStatus: (status) => set({ captureStatus: status }),
  // 新记录插到最前面，最多保留 200 条（防止内存无限增长）
  addActivity: (record) =>
    set((state) => ({ activities: [record, ...state.activities].slice(0, 200) })),
  setActiveTab: (tab) => set({ activeTab: tab }),
}));
```

**组件里怎么用**：用 selector 精确取状态，只有取到的部分变化时才重渲染：

```tsx
// 只订阅 activeTab 和它的 setter —— 别的状态变了不会触发本组件重渲染
const activeTab = useAppStore((s) => s.activeTab);
const setActiveTab = useAppStore((s) => s.setActiveTab);
```

**hooks 层（src/renderer/hooks/）**：每个数据源一个 hook，负责"订阅 IPC 推送 → 写入 Zustand"。这是本项目的关键模式：

```ts
// useCaptureStatus.ts —— 订阅采集状态推送
export function useCaptureStatus(): void {
  useEffect(() => {
    // 订阅主进程推送，写入 store
    const unsubscribe = window.api.on.captureStatus((status) => {
      useAppStore.getState().setCaptureStatus(status);
    });
    return unsubscribe;  // 卸载时自动取消订阅
  }, []);
}
```

> **架构精髓**：**hooks 负责"取数"，组件负责"展示"**。组件从不直接碰 IPC，只管读 store——数据流单向、清晰、可测试。

### 5.3 数据是怎么"存"的：StorageService 的原子写入

主进程的 StorageService 用 JSON 文件存记录，为了**不丢数据**用了三个技巧：

```ts
writeRecord(record: LogRecord, dataRoot: string): WriteResult {
  if (this.diskErrorMode) {
    this.bufferRecord(record);          // 技巧3：磁盘坏了先缓冲到内存
    return { merged: false, record };
  }
  try {
    const result = this.atomicWriteRecord(record, dataRoot);
    if (this.diskErrorBuffer.length > 0) this.flushBuffer(dataRoot);  // 恢复后补写
    return result;
  } catch (err) {
    this.diskErrorMode = true;          // 技巧2：进入"故障降级"模式
    this.bufferRecord(record);
    return { merged: false, record };
  }
}

private atomicWriteRecord(record: LogRecord, dataRoot: string): WriteResult {
  const jsonPath = getHourJsonPath(dataRoot, record.timestamp);  // 按小时定位文件
  const hourKey = getHourKey(record.timestamp);
  // 技巧1：内存缓存当前小时，跨小时才读写盘
  if (this.currentHourKey !== hourKey) {
    this.currentHourKey = hourKey;
    // ...读入新小时已有记录
  }
  // ...合并（相同总结 repeat_count++）→ 写 .tmp → rename（原子替换）
}
```

**三个保数据技巧（面试常问）**：
1. **内存缓存 + 跨小时落盘**：同一小时内的多条记录合并成一次磁盘写，减少 IO。
2. **故障降级**：磁盘写失败 → 切到内存缓冲模式，不再尝试写盘，恢复后自动补写。
3. **原子写入**：先写 `.tmp` 再 `rename`——rename 是原子操作，要么旧文件要么新文件，永远不会有半个文件。

---

## 6. 关键模块代码实战（三）：接口调用与核心服务

### 6.1 AI 接口调用：OllamaService（axios + 重试 + 自动拉起）

本项目"AI 接口"不是云端 API，而是本地 Ollama 的 REST API。OllamaService 是**最值得精读的一个服务**，它展示了真实项目中 HTTP 客户端该怎么写。

```ts
export class OllamaService {
  private httpClient: AxiosInstance;

  constructor(endpoint: string, model: string) {
    this.httpClient = axios.create({
      baseURL: endpoint,                                  // http://localhost:11434
      timeout: BUSINESS.OLLAMA_TIMEOUT,                   // 30 秒超时
      headers: { 'Content-Type': 'application/json' },
      family: 4,   // v3.1.2: 强制 IPv4（localhost 可能解析成 ::1 连不上）
    });
  }

  /** 连接检测：GET /api/tags，失败自动重连 + 自动拉起 */
  async checkConnection(): Promise<OllamaStatus> {
    this.setStatus('checking');
    try {
      const response = await this.httpClient.get('/api/tags', { timeout: 5000 });
      if (response.status === 200) {
        this.setStatus('connected');
        this.stopReconnectTimer();
        return 'connected';
      }
    } catch (err) {
      logger.warn(TAG, `Ollama 连接失败: ${(err as Error).message}`);
    }
    this.setStatus('disconnected');
    if (this.autoLaunchEnabled) {          // v3.1.2: 自动拉起本地 Ollama
      await this.tryLaunchOllama();
    }
    this.startReconnectTimer();            // 60 秒后自动重试
    return 'disconnected';
  }

  /** 视觉分析：POST /api/chat，图片走 base64 */
  async analyze(base64: string, options: { temperature: number; numPredict: number }): Promise<AnalysisResult> {
    let retryCount = 0;
    while (retryCount <= BUSINESS.OLLAMA_MAX_RETRIES) {   // 最多 3 次重试
      try {
        const response = await this.httpClient.post('/api/chat', {
          model: this.model,
          messages: [{ role: 'user', content: BUSINESS.ACTIVITY_PROMPT, images: [base64] }],
          stream: false,
          options: { temperature: options.temperature, num_predict: options.numPredict },
        });
        // ...解析出 summary
        return { summary, status: 'success', retryCount };
      } catch (err) {
        retryCount++;
        if (retryCount <= BUSINESS.OLLAMA_MAX_RETRIES) {
          // 指数退避：2s → 4s → 8s
          const delay = BUSINESS.OLLAMA_RETRY_BASE_DELAY * 2 ** (retryCount - 1);
          await sleep(delay);
        }
      }
    }
    // 三次都失败 → 降级返回占位文案（不崩溃，记录标记为 placeholder）
    return { summary: PLACEHOLDER_SUMMARY, status: 'placeholder', retryCount };
  }
}
```

**学习要点（都是真实项目必备的健壮性代码）**：
1. **axios 实例化**：统一 baseURL/超时/请求头，而不是每次裸调 `axios.get`。
2. **指数退避重试**：失败后等 `2s→4s→8s` 再试，避免"疯狂重试打爆服务"。
3. **降级而非崩溃**：AI 不可达时返回占位文案，记录标 `placeholder`，应用继续运行。
4. **连接状态机**：connected / disconnected / checking + 定时重连。
5. **自动拉起**：连接失败时自动 spawn 本地 Ollama 进程（`tryLaunchOllama`），轮询等它就绪（最多 30 秒）。

### 6.2 核心调度器：CaptureScheduler（状态机）

采集调度器是一个 **有限状态机**：`idle → running ↔ paused`。这是本项目主进程最核心的类：

```ts
export class CaptureScheduler {
  private state: CaptureState = 'idle';
  private timer: NodeJS.Timeout | null = null;
  private isCapturing = false;   // 防并发标志

  start(): void {
    if (this.state === 'running') return;   // 幂等：已运行就什么都不做
    this.state = 'running';
    this.subscribeLoadMonitor();            // v3.1: 订阅负载事件
    this.scheduleTick(0);                   // 立即执行一次
    this.notifyStatusChange();
  }

  pause(): void {
    if (this.state !== 'running') return;
    this.state = 'paused';
    this.clearTimer();
    this.notifyStatusChange();
  }

  /** 单次采集流程（核心） */
  private async tick(): Promise<void> {
    if (this.isCapturing) return;           // 上一次还没完成，跳过（防重入）
    this.isCapturing = true;
    try {
      // v3.1: 先看负载 → 高负载走事件驱动分支
      if (this.isHighLoad()) { this.handleHighLoadMode(); return; }

      // 1. 截屏
      const shot = await this.deps.screenshotService.capture(...);
      // 2. pHash 去重判断
      const pHash = computePHash(shot.buffer);
      const { skip } = this.deps.dedupService.shouldSkip(pHash, settings);
      let record: LogRecord;
      if (skip) {
        // 画面没变 → 复用上一条总结，dedup 标记
        record = { ..., dedup: true, repeat_count: prev.repeat_count + 1 };
      } else {
        // 3. AI 分析
        const result = await this.deps.ollamaService.analyze(base64, opts);
        record = { ..., activity_summary: result.summary, status: result.status };
        this.deps.dedupService.update(pHash, result.summary, false);
      }
      // 4. 存储 + 推送
      this.deps.storageService.writeRecord(record, dataRoot);
      this.deps.onNewActivity(record);
    } finally {
      this.isCapturing = false;
    }
  }
}
```

**学习要点**：
1. **幂等操作**：`if (this.state === 'running') return` 防止重复启动。
2. **防重入**：`isCapturing` 标志防止上一次采集未完成时又触发一次（异步竞态）。
3. **依赖注入**：所有协作服务（截图/AI/存储/去重）通过构造函数传入，方便测试时替换 mock。

### 6.3 负载感知：LoadMonitorService（v3.1 事件驱动）

这是本项目最有技术含量的设计，值得单独讲讲。

**背景问题**：旧版每 20 秒截屏+AI 分析，如果用户在打游戏/编译大型项目（高负载），截屏分析会抢占 CPU/GPU，雪上加霜。

**解决思路**：监测系统负载，高负载时**彻底安静**（零定时器、零截图、零 AI），只在"进入高负载"和"退出高负载"两个**瞬间**各抓一帧，中间过程压缩为一条高负载会话记录。等负载降下来再批量分析缓存帧。

```
负载低（正常模式）:  每20秒截屏 → AI → 存记录
负载高（事件模式）:  [进入]抓首帧 → 静默（啥也不做）→ [退出]抓尾帧 → 批量AI分析
```

核心代码（LoadMonitorService 状态机）：

```ts
// 状态流转：normal → high（连续5次采样超阈值）→ normal（防抖5秒后退出）
// 采样间隔 1 秒；判定窗口 5 次；退出防抖 5 秒

// 在 CaptureScheduler 中订阅事件：
this.subscribeLoadMonitor = () => {
  this.deps.loadMonitor.onStateChange((change: LoadStateChange) => {
    if (change.type === 'high_load_entered') {
      // 抓首帧 + 建会话 + 取消定时器（静默）
    } else if (change.type === 'high_load_exited') {
      // 抓尾帧 + 结算 + 恢复调度 + 触发批量分析
    }
  });
};
```

而 `index.ts` 里把负载事件联动到了 Ollama 生命周期（v3.1.3）：

```ts
onLoadEvent: (change: LoadStateChange): void => {
  if (change.type === 'high_load_entered') {
    ollamaService.killForLoad();        // 高负载 → 杀掉 AI 释放资源
  } else if (change.type === 'high_load_exited' || change.type === 'low_load_resumed') {
    ollamaService.tryLaunchOllama();    // 低负载 → 重新拉起 AI
  }
  // 系统通知用户
  if (change.type === 'high_load_entered') {
    ipcService.pushNotify({ title: '进入高负载', body: '应用已进入静默记录模式（仅首帧）' });
  }
},
```

**学习要点**：这就是"**事件驱动**"vs"**轮询**"的经典实战——负载是"事件源"，采集是"响应者"，用订阅-回调替代定时检查，资源开销降到最低。

### 6.4 主进程装配：index.ts（组合根）

最后看主进程入口 `index.ts` 怎么把所有东西串起来。它的模式叫 **组合根（Composition Root）**：所有服务在这里创建、注入依赖、启动。阅读顺序就是项目启动顺序：

```ts
async function assembleServices(): Promise<void> {
  // 1. 设置服务（先读配置，后面都依赖它）
  settingsService = new SettingsService();

  // 2. 核心服务
  cache = new ContextAwareCache({ dataRoot, ... });   // v3.1
  loadMonitor = new LoadMonitorService();              // v3.1
  screenshotService = new ScreenshotService();
  ollamaService = new OllamaService(settings.ollamaEndpoint, settings.ollamaModel);
  ollamaService.setAutoLaunch(settings.autoLaunchOllama !== false, ...);
  storageService = new StorageService();

  // 3. 调度器（注入依赖）
  scheduler = new CaptureScheduler({
    screenshotService, ollamaService, storageService, settingsService,
    dedupService, loadMonitor, cache,
    onLoadStatus: (s) => ipcService.pushLoadStatus(s),   // 回调注入
    onNewActivity: (r) => ipcService.pushNewActivity(r),
    // ...
  });

  // 4-6. 窗口 / 托盘 / IPC
  mainWindow = new MainWindow();
  trayService = new TrayService();
  ipcService = new IpcService({ scheduler, ollamaService, /* ... */ });
  ipcService.registerAll();

  // 7-8. 启动流程
  if (settings.runMode === 'extreme' && settings.loadAwareness?.enabled) {
    loadMonitor.start();                 // 启动负载监测
  }
  ollamaService.checkConnection();       // 连接 AI
  if (settings.autoStart) scheduler.start();  // 自动开始采集
  summaryScheduler.start();              // 汇总调度
  cleanupScheduler.start();              // 清理调度
}
```

生命周期管理（退出时按依赖逆序清理）：

```ts
app.on('before-quit', () => {
  scheduler?.destroy();
  summaryScheduler?.destroy();
  cleanupScheduler?.destroy();
  ollamaService?.destroy();   // 关闭自拉起的 Ollama（v3.1.3 双保险）
  trayService?.destroy();
  ipcService?.removeAll();
  mainWindow?.destroy();
  loadMonitor?.destroy();
  cache?.destroy();
});
```

> **学习要点**：回调注入（`onXxx` 参数）让服务之间**不直接引用**，只通过回调通知，低耦合、可测试。

---

## 7. 关键模块代码实战（四）：UI 层与"伪路由"

### 7.1 没有 react-router 的"路由"

本项目是单窗口应用，**没有用 react-router**，而是用 Zustand 里的 `activeTab` 状态 + 条件渲染实现"标签页切换"，配合 `React.lazy` 懒加载：

```tsx
// TabLayout.tsx（核心代码）
const TABS = [
  { key: 'dashboard', label: '仪表盘', Icon: LayoutDashboard },
  { key: 'activity',  label: '动态',   Icon: Activity },
  { key: 'diary',     label: '日记',   Icon: BookOpen },
  { key: 'analytics', label: '看板',   Icon: ChartPie },
  { key: 'qa',        label: '问答',   Icon: MessagesSquare },
];

export default function TabLayout(): JSX.Element {
  const activeTab = useAppStore((s) => s.activeTab);
  const setActiveTab = useAppStore((s) => s.setActiveTab);

  return (
    <div>
      {/* 胶囊导航栏 */}
      <div className="flex gap-1 ...">
        {TABS.map(({ key, label, Icon }) => (
          <button key={key} onClick={() => setActiveTab(key)}
            className={activeTab === key ? 'active样式' : '普通样式'}>
            <Icon size={18} strokeWidth={1.75} />
            <span>{label}</span>
          </button>
        ))}
      </div>

      {/* 内容区：条件渲染 + 懒加载 + Suspense */}
      <Suspense fallback={<TabLoading />}>
        {activeTab === 'dashboard' && <DashboardTab />}
        {activeTab === 'activity'  && <ActivityTab />}
        {activeTab === 'diary'     && <DiaryTab />}
        {activeTab === 'analytics' && <AnalyticsTab />}
        {activeTab === 'qa'        && <QATab />}
      </Suspense>
    </div>
  );
}
```

### 7.2 根组件 App.tsx 的关键模式

```tsx
export default function App(): JSX.Element {
  // 1. 挂载即订阅所有 IPC 推送（hooks 层）
  useCaptureStatus();
  useActivityStream();
  useSummary();
  useLoadStatus();

  // 2. 读设置（loading 时显示占位）
  const { settings, loading, updateSettings } = useSettings();

  // 3. 定时刷新统计（10 秒一次）
  useEffect(() => {
    refreshStatistics();
    const timer = setInterval(refreshStatistics, 10_000);
    return () => clearInterval(timer);
  }, [refreshStatistics]);

  if (loading) return <div>加载中...</div>;

  return (
    <div className="flex h-full flex-col">
      <StatusBar />                    {/* 顶部状态 */}
      <div className="flex flex-1 min-h-0">
        <main className="flex-1"><TabLayout /></main>   {/* 主内容区 */}
        <SettingsPanel ... />           {/* 右侧设置栏 */}
      </div>
    </div>
  );
}
```

**学习要点**：组件层只做三件事——**订阅数据（hooks）、组织布局（JSX）、响应用户操作（调用 window.api）**。没有业务逻辑，非常干净。

### 7.3 "认证授权"在本项目的对应物

桌面本地应用**没有用户登录体系**（不需要账号密码），但"安全边界"依然存在，对应到三个机制：

| 传统 Web 概念 | 本项目对应物 | 说明 |
|--------------|-------------|------|
| 认证（你是谁） | 无（单机本地应用） | 数据只在本地，无需身份验证 |
| 授权（你能干嘛） | contextBridge 白名单 | 渲染进程只能调用 preload 暴露的 `window.api`，无法碰 Node API |
| 会话管理 | 单实例锁 | `app.requestSingleInstanceLock()` 防止多开，二次启动唤起已有窗口 |

```ts
// 单实例锁 —— 防多开
const gotTheLock = app.requestSingleInstanceLock();
if (!gotTheLock) {
  app.quit();
} else {
  app.on('second-instance', () => mainWindow?.show());  // 再开就唤起已有窗口
}
```

---

## 8. 学习路线图：从零基础到读懂本项目

假设你从零开始，按以下路线学。每条标注**先修知识、学习重点、建议时长、对照本项目看什么**。

### 阶段 0：计算机与开发环境准备（0.5~1 周）

- **先修**：会用电脑、会装软件
- **学习重点**：
  - 装 Node.js（建议 ≥18）、VS Code
  - 理解"命令行"是什么（cd、ls、npm install）
  - 理解"包管理器"npm 的作用（装依赖、跑脚本）
- **建议时长**：3~5 天
- **对照本项目**：跑通 `npm install`，看看 `node_modules` 有多大

### 阶段 1：HTML + CSS + JavaScript 基础（2~3 周）

- **先修**：阶段 0
- **学习重点**：
  - HTML 语义化标签（div/span/button/input）
  - CSS 盒模型、flex 布局（本项目布局全靠 flex）
  - JS 核心：变量、函数、对象、数组、Promise、async/await（**重点！**）
  - 事件机制、DOM 操作
- **建议时长**：2~3 周（每天 1~2 小时）
- **对照本项目**：看渲染进程组件的 JSX 结构（本质是 HTML+JS）；看 `useEffect` 里的 async/await

### 阶段 2：TypeScript（1~2 周）

- **先修**：JS 基础扎实
- **学习重点**：
  - 接口 interface、类型别名 type、联合类型
  - 泛型（理解 `Promise<T>` 就够了）
  - 可选属性 `?`、`Partial<T>`、`Record<K,V>`
  - 类型收窄（typeof/interface 判断）
- **建议时长**：1~2 周
- **对照本项目**：通读 `src/shared/types.ts`——**这是全项目最好的 TS 教材**，看完你就能看懂所有数据结构

### 阶段 3：React 18（2~3 周）

- **先修**：JS + TS
- **学习重点**：
  - 组件、Props、State
  - **Hooks**：useState / useEffect / useCallback / useMemo（本项目全用到）
  - 条件渲染、列表渲染
  - 组件拆分思维
- **建议时长**：2~3 周
- **对照本项目**：读 `TabLayout.tsx`（列表渲染+条件渲染）、`App.tsx`（hooks 组合）、`hooks/useCaptureStatus.ts`（订阅模式）

### 阶段 4：Electron 三进程与 IPC（1~2 周）

- **先修**：React + Node.js 基础（知道 fs/path 是干嘛的）
- **学习重点**：
  - 主进程 / preload / 渲染进程的区别（第 2 章模型）
  - IPC 两种模式：invoke/handle 与 send/on
  - contextBridge 安全模型
  - 生命周期事件（whenReady、before-quit、window-all-closed）
- **建议时长**：1~2 周（这是理解本项目架构的关键阶段）
- **对照本项目**：精读 `preload/index.ts` + `IpcService.ts` + `index.ts`，画一遍数据流图

### 阶段 5：状态管理 Zustand（3~5 天）

- **先修**：React hooks
- **学习重点**：
  - 全局状态 vs 组件状态的区别
  - create / set / selector 用法
  - 为什么用 selector 订阅
- **建议时长**：3~5 天
- **对照本项目**：精读 `appStore.ts`，数一数里面有多少个状态，分别被哪些组件用

### 阶段 6：Node.js 服务端思维（1~2 周）

- **先修**：JS/TS
- **学习重点**：
  - fs 文件读写、path 路径处理
  - 事件发射器模式（EventEmitter / 回调注入）
  - 定时任务（setInterval/setTimeout）
  - 状态机设计（有限状态机概念）
- **建议时长**：1~2 周
- **对照本项目**：精读 `CaptureScheduler.ts`（状态机+定时+防重入）、`StorageService.ts`（原子写+缓冲）、`LoadMonitorService.ts`（事件驱动）

### 阶段 7：HTTP 客户端与 AI 集成（3~5 天）

- **先修**：async/await、REST 概念
- **学习重点**：
  - axios 实例配置、拦截器
  - 重试与指数退避
  - 连接检测（健康检查）
  - 什么是"降级处理"
- **建议时长**：3~5 天
- **对照本项目**：精读 `OllamaService.ts`（这是本项目代码质量最高的服务之一）

### 阶段 8：构建、测试与打包（1~2 周）

- **先修**：以上全部
- **学习重点**：
  - Vite 基本概念（入口/出口/alias/插件）
  - Vitest 单元测试（describe/it/expect/mock）
  - electron-builder 打包流程、NSIS 安装包
- **建议时长**：1~2 周
- **对照本项目**：读三份 `vite.build.*.config.ts`；跑一遍 `npm test`（366 个用例）；跑 `npm run electron:build`

### 路线总览图

```
阶段0 环境准备 ─▶ 阶段1 HTML/CSS/JS ─▶ 阶段2 TypeScript ─▶ 阶段3 React
   (3-5天)          (2-3周)              (1-2周)            (2-3周)
                                                              │
                    ┌─────────────────────────────────────────┘
                    ▼
              阶段4 Electron+IPC ─▶ 阶段5 Zustand ─▶ 阶段6 Node服务端思维
                (1-2周)              (3-5天)            (1-2周)
                    │                                       │
                    ▼                                       ▼
              阶段7 HTTP+AI 集成                    阶段8 构建/测试/打包
                (3-5天)                              (1-2周)
```

> **总时长估算**：每天投入 1~2 小时，约 2.5~3.5 个月可系统学完。如果想快速上手，优先精读第 4、5、6 章对应的代码（IPC 链路 + appStore + CaptureScheduler）。

---

## 9. 进阶专题（选读）

学完主线后，这几个项目里的"隐藏关卡"值得深入：

### 9.1 pHash 感知哈希去重（v3）

截图相似度如何判断？——把图片缩小到 32×32、灰度化、做 DCT 变换，取左上角 8×8 的低频系数生成 64 位哈希。两张图的哈希"汉明距离"小于阈值就认为画面没变。这是**图像指纹**的经典应用。

### 9.2 批量分析 + keep_alive 生命周期（v3.1）

高负载结束后，缓存的首尾帧需要批量送 AI。Ollama 的模型加载很慢（首次要加载几百 MB），所以批量分析时用 `keep_alive` 让模型常驻内存，批结束再 `keep_alive=0` 卸载。这是**资源生命周期管理**的实战。

### 9.3 孤儿会话恢复（v3.1）

如果应用在"高负载会话进行中"被强杀，会留下只有首帧没有尾帧的"孤儿会话"。启动时扫描 `.pending` 目录，按配置策略（询问/自动归档/删除）处理。这是**崩溃恢复/数据自愈**的设计。

### 9.4 自动拉起与杀进程（v3.1.2/3）

Ollama 不在运行时自动启动它；高负载时 `taskkill` 杀掉它释放资源；退出应用时确保关闭它（先 `child.kill()` 再 `taskkill /PID`，双保险）。这是**外部进程生命周期管理**。

### 9.5 安装器自定义脚本（installer.nsh）

升级安装时旧版应用还在运行会冲突。electron-builder 的 NSIS 脚本通过 `!macro customInit` 钩子在安装前 `taskkill /F /T /IM "拾光.exe"`，实现"装新版自动关旧版"。这是**桌面应用升级体验**的关键细节。

---

## 10. 术语表

| 术语 | 通俗解释 |
|------|---------|
| **Electron** | 用 Web 技术（HTML/CSS/JS）写桌面应用的框架，本质是"Chrome 内核 + Node.js"打包在一起 |
| **主进程** | Electron 应用的后台大脑，运行 Node.js，管窗口、文件、系统 |
| **渲染进程** | 一个窗口里的网页页面，运行 React UI |
| **Preload** | 渲染进程加载前运行的脚本，是主/渲染通信的安全桥梁 |
| **IPC** | 进程间通信。主进程和渲染进程交换数据的方式 |
| **contextBridge** | Electron 的安全 API，只把白名单函数暴露给渲染进程 |
| **invoke/handle** | IPC 请求-响应模式：渲染进程"打电话"，主进程"接电话" |
| **send/on** | IPC 事件推送模式：主进程"广播"，渲染进程"收听" |
| **Zustand** | 一个极简的 React 全局状态库（比 Redux 简单得多） |
| **Hook（React）** | 让函数组件"有状态"的函数，如 useState/useEffect |
| **TypeScript** | JavaScript 加类型。写代码时多写类型，编译器提前抓错 |
| **接口 interface** | TS 里描述"一个对象长什么样"的语法 |
| **Ollama** | 本地运行的 AI 推理服务，本项目用它跑视觉模型 |
| **视觉模型（VLM）** | 能"看图说话"的 AI 模型（本项目 qwen3-vl） |
| **REST API** | 用 HTTP 动词（GET/POST）操作资源的接口风格 |
| **axios** | JS 里的 HTTP 客户端库，发请求收响应 |
| **指数退避** | 重试间隔按 2 的幂增长（2s→4s→8s），避免打爆服务 |
| **原子写入** | 先写临时文件再改名替换，保证文件要么是旧的要么是新的，不会写一半 |
| **状态机** | 明确"有哪几种状态、什么事件触发什么转换"的程序结构 |
| **防重入** | 防止同一段代码被并发执行两次（如上一次还没完成又触发） |
| **pHash** | 感知哈希。图片的"指纹"，相似图片哈希也相似 |
| **汉明距离** | 两个二进制数不同的位数。pHash 用它判断图片相似度 |
| **DCT** | 离散余弦变换，图像压缩/指纹里用的数学变换 |
| **虚拟列表** | 只渲染屏幕可见区域的行，几千条数据也不卡 |
| **keep_alive** | Ollama 保持模型在内存中驻留的时间参数 |
| **NSIS** | Windows 安装程序制作工具，electron-builder 用它生成 exe 安装包 |
| **asar** | Electron 打包后的归档格式（类似 zip），保护源码 |
| **依赖注入** | 把依赖通过构造函数/参数传进去，而不是自己 new |
| **组合根** | 所有对象在一个地方创建和连接的入口（本项目的 index.ts） |
| **回调注入** | 通过传函数参数让两个模块通信，彼此不直接依赖 |
| **事件驱动** | 程序不是"定时检查"，而是"等事件发生了再反应" |

---

## 11. 难点答疑：初学者常见困惑

### Q1：主进程和渲染进程到底啥区别？为什么 React 不能在主进程里跑？

主进程跑在 Node.js 环境里，**没有 DOM**（没有 document/window 这些浏览器对象），所以 React 无法在主进程渲染。渲染进程是真正的浏览器环境，有 DOM，但**没有 Node API**（不能直接读写文件）。两者各有专长，靠 IPC 协作——这就是 Electron 的核心架构思想。

### Q2：为什么这个项目没有 react-router？

react-router 是给"多页面 URL"场景用的（比如 `/home`、`/settings`）。本项目是**单窗口单页面**应用，五个标签页只是界面切换，没有 URL 概念，用 Zustand 的 `activeTab` 状态就够了。**选技术要看场景，不是越多越好**。

### Q3：`window.api` 是从哪来的？为什么我在代码里搜不到它的定义？

它由 preload 脚本里的 `contextBridge.exposeInMainWorld('api', api)` 创建（`src/preload/index.ts` 最后一行）。它的**类型**定义在 `src/shared/types.ts` 的 `ElectronAPI` 接口里。所以你搜"定义"搜不到——它是运行时注入的，搜"exposeInMainWorld"才能找到。

### Q4：`Promise<T>` 里的 `T` 是什么意思？

泛型（Generic）。`Promise<CaptureStatus>` 表示"这个异步操作最终会给我一个 CaptureStatus 对象"。`window.api.settings.get()` 返回 `Promise<AppSettings>`，所以 `const s = await window.api.settings.get()` 后，`s` 的类型就是 `AppSettings`，能用 `s.dataRoot` 等字段，还有自动补全。

### Q5：为什么这么多回调？onXxx 到底是啥？

主进程的业务逻辑执行完，需要"告诉" UI 层结果。但两边是独立的进程，所以主进程通过 `webContents.send(通道, 数据)` 发消息；preload 包装成 `on.captureStatus(callback)`；渲染进程的 hook 里注册 callback，收到消息就调 callback 更新 store。**回调 = "有消息时请你调用我这个函数"**。

### Q6：366 个测试都是干嘛的？测试什么？

用 Vitest 写的**单元测试**，不启动真实应用，直接实例化服务类并模拟依赖（mock）。比如测试 StorageService 的原子写入、DedupService 的去重决策、LoadMonitorService 的状态转换、OllamaService 的重试逻辑。**测试的价值**：改代码不怕改坏——跑一遍就知道哪里炸了。

### Q7：为什么打包要拆成三份 Vite 配置？

因为 Electron 应用有三个独立构建目标：渲染进程（网页资源）、主进程（Node 代码）、preload（桥接代码）。它们的环境、入口、产物目录都不同，拆开配置更清晰。这也是 vite-plugin-electron 之外的生产级做法。

### Q8：这个项目的"数据库"在哪？为什么不用 SQLite？

数据只有"按小时追加记录 + 读时间段"，JSON 文件足够简单可靠（配合原子写入防损坏）。**对写多读少的时序数据，文件系统是合理选择**——技术选型永远服务于实际需求。

### Q9：v3.1 的"负载感知"到底解决了什么问题？

老版本固定每 20 秒截屏分析。如果用户在打 3A 游戏或编译大型工程（CPU 满负荷），截屏+AI 推理会跟用户抢资源，导致电脑更卡。负载感知让应用**察言观色**：高负载就退让（只抓首尾帧），低负载再补作业。**好软件要学会"让路"**。

### Q10：我想在这个项目上练手，第一件该改什么？

改 UI 是最安全的入口：
1. 改 `TabLayout.tsx` 的标签样式（Tailwind class 随便调）
2. 改 `App.tsx` 的渐变背景色
3. 给 `StatusBar.tsx` 加一个显示项
改完跑 `npm run dev` 看效果。**先改能看见的东西，建立正反馈，再深入主进程逻辑**。

---

## 附：项目文件速查表

| 想看什么 | 打开这个文件 |
|---------|------------|
| 全部数据类型（项目词汇表） | `src/shared/types.ts` |
| IPC 通道名、默认设置 | `src/shared/constants.ts` |
| 主进程启动装配顺序 | `src/main/index.ts` |
| 安全桥（window.api 来源） | `src/preload/index.ts` |
| IPC 通道注册中心 | `src/main/services/IpcService.ts` |
| 核心调度状态机 | `src/main/services/CaptureScheduler.ts` |
| AI 调用（重试/拉起/批量） | `src/main/services/OllamaService.ts` |
| 文件存储（原子写） | `src/main/services/StorageService.ts` |
| 负载感知状态机 | `src/main/services/LoadMonitorService.ts` |
| 全局状态管理 | `src/renderer/store/appStore.ts` |
| 根组件（订阅+布局） | `src/renderer/App.tsx` |
| 标签页"路由" | `src/renderer/components/tabs/TabLayout.tsx` |
| 数据订阅 hooks | `src/renderer/hooks/` |
| 测试（366 个用例） | `tests/` |
| 渲染层构建配置 | `vite.build.renderer.config.ts` |
| 打包配置 | `package.json` 的 `build` 字段 |

---

*本指南基于「拾光」项目 v3.1.x 真实源码编写。建议配合源码阅读，边看边跑 `npm run dev` 体验效果，学习效率最高。*
