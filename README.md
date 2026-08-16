# 拾光 ScreenAura

> **完全本地运行的桌面时间追踪工具**：定时截屏 → 本地 AI 视觉模型理解"你正在做什么" → 自动生成活动记录、10 分钟汇总、每日日记、数据看板与智能问答。

![version](https://img.shields.io/badge/version-3.1.0-blue)
![license](https://img.shields.io/badge/license-MIT-green)
![platform](https://img.shields.io/badge/platform-Windows-lightgrey)
![stack](https://img.shields.io/badge/stack-Electron%20%2B%20React%20%2B%20TypeScript-purple)

**开发代号**：screen-ai-tracker ｜ **许可**：MIT ｜ **版本**：v3.1.x（含 v3.2 部分 UI 预留）

> **仓库性质**：本仓库为从发布包 `app.asar` 解包的应用代码归档（编译后的主进程 / preload / 渲染进程产物 + 生产依赖清单 + 架构学习指南）。完整源码工程（`src/`、`tests/`、Vite 构建配置）不包含在内。

---

## ✨ 核心功能

| 功能 | 版本 | 说明 |
| --- | --- | --- |
| 定时截屏 | v1 | 每 20 秒（可配置）截取屏幕 |
| AI 活动总结 | v1 | 本地视觉模型把截图翻译成中文活动描述 |
| 小时/月度统计 | v1 | 按小时 JSON 文件组织记录 |
| 系统托盘 | v1 | 托盘图标 + 右键菜单控制采集 |
| 10 分钟汇总 | v2 | 多条记录合并为自然语言工作摘要 |
| 每日日记 | v2 | 基于当天汇总生成结构化日记 |
| 数据看板 | v2 | 饼图 / 柱状图展示分类、软件、项目专注度 |
| 智能问答 | v2 | 检索式问答（自动搜索记录 → 生成答案） |
| 截图清理 | v3 | 分级清理旧截图，控制磁盘占用 |
| pHash 去重 | v3 | 画面未变则跳过 AI 分析，省算力 |
| 负载感知 | v3.1 | 高负载时自动降级为"事件驱动"静默记录 |

## 🏗 架构一览

```
┌─────────────────────────────────────────────────┐
│              Electron 应用（三进程）             │
│  ┌────────────┐   IPC   ┌──────────────────┐   │
│  │ 主进程      │◀──────▶│ 渲染进程（React） │   │
│  │ 14 个服务   │         │ Zustand + Tailwind│   │
│  │ 状态机/调度  │         │ 5 个标签页        │   │
│  └──────┬─────┘         └──────────────────┘   │
│         │ contextBridge（preload 安全桥）       │
│  ┌──────▼─────┐                                 │
│  │ Ollama     │  POST /api/chat（视觉分析）     │
│  │ localhost  │  qwen3-vl:4b-instruct-q4_k_m    │
│  └────────────┘                                 │
└─────────────────────────────────────────────────┘
```

**核心服务**（主进程）：`CaptureScheduler` 采集状态机 ・ `OllamaService` AI 调用（重试/自动拉起/批量）・ `StorageService` 原子写入 ・ `DedupService` pHash 去重 ・ `LoadMonitorService` 负载感知 ・ `QAService` 智能问答 …

**数据存储**：文件系统即数据库 —— 按 `YYYY-MM/YYYYMMDD_HH.json` 组织记录，配合原子写入（`.tmp` + rename）与故障降级缓冲，保证数据不丢。

## 🛠 技术栈

- **运行时**：Electron 31 + Node.js ≥18
- **UI**：React 18 + TypeScript 5.5 + Zustand + Tailwind CSS 3.4
- **核心库**：axios / dayjs / screenshot-desktop / sharp / systeminformation / react-window / recharts / lucide-react
- **工程化**：Vite 5（三份构建配置）・ Vitest（366 个单元测试）・ electron-builder（NSIS 安装包）
- **AI**：Ollama + `qwen3-vl:4b-instruct-q4_k_m` 视觉模型

## 📁 仓库结构

```
├── dist-electron-main/       # 主进程编译产物（入口 index.js）
├── dist-electron-preload/    # preload 安全桥编译产物
├── dist-renderer2/           # 渲染进程编译产物（React UI）
├── resources/                # 应用图标等资源
├── docs/
│   └── LEARNING_GUIDE.md     # ★ 架构学习指南（三进程/IPC/核心服务/学习路线）
├── package.json              # 生产依赖清单
└── README.md
```

## 🚀 运行

**前提**：安装 [Ollama](https://ollama.com) 并拉取模型：

```bash
ollama pull qwen3-vl:4b-instruct-q4_k_m
```

**运行解包版**（需 Electron 运行时；`sharp` 等原生依赖按平台重新安装）：

```bash
npm install
npx electron dist-electron-main/index.js
```

> 完整构建请使用源码工程（见"仓库性质"说明）。

## 📖 学习指南

`docs/LEARNING_GUIDE.md` 提供从零到精通的完整拆解：

- **Electron 三进程模型**与 IPC 两种模式（invoke/handle + send/on）
- **核心服务精读**：CaptureScheduler 状态机、OllamaService 重试与自动拉起、StorageService 原子写入
- **v3.1 负载感知**：高负载"事件驱动"静默记录 + 首尾帧压缩 + 孤儿会话恢复
- **Zustand + hooks 订阅模式**、UI 层"伪路由"
- **8 阶段学习路线图**（从 HTML/JS 到读懂本项目，约 2.5~3.5 个月）

## 📄 License

MIT © 2026 fg6668
