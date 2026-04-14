# Super Productivity 项目分析

## 项目定位

一个**高级待办事项 + 时间追踪应用**（v18.1.2），支持 Web/PWA、桌面（Electron）和移动端（Capacitor/Android/iOS）三端运行。MIT 许可证，作者 Johannes Millan。

---

## 技术栈

| 层面 | 技术 |
|---|---|
| **语言** | TypeScript（主体）、HTML、SCSS |
| **前端框架** | Angular（standalone components，信号优先） |
| **状态管理** | NgRx（Redux 模式：actions → reducers → effects → selectors） |
| **UI 组件库** | Angular Material |
| **桌面端** | Electron（主进程 `electron/main.ts`，预加载 `preload.ts`） |
| **移动端** | Capacitor（Android/iOS 桥接） |
| **本地存储** | IndexedDB（通过持久化层） |
| **数据同步** | Dropbox、WebDAV、本地文件、SuperSync Server（自建服务端） |
| **运行时类型校验** | Typia |
| **单元测试** | Jasmine + Karma |
| **E2E 测试** | Playwright |
| **代码质量** | ESLint + Stylelint + Prettier + Husky（git hooks） |
| **构建/打包** | Angular CLI + electron-builder + Vite（插件） |
| **容器化** | Docker / Docker Compose（E2E、SuperSync Server） |

---

## 顶层目录结构

```
super-productivity/
├── src/app/           # Angular 前端主体
│   ├── core/          # 核心服务（持久化、启动、通知、错误处理、平台抽象）
│   ├── core-ui/       # 核心 UI 组件
│   ├── features/      # 功能模块（40+ 个，每个自包含）
│   ├── root-store/    # NgRx 根 Store + meta-reducers
│   ├── op-log/        # 操作日志（operation log）持久化层
│   ├── pfapi/         # 平台 API 抽象
│   ├── plugins/       # 插件系统
│   ├── imex/          # 导入/导出 & 同步
│   ├── pages/         # 页面级路由组件
│   ├── routes/        # 路由定义
│   ├── ui/            # 通用 UI 组件
│   └── util/          # 工具函数
├── electron/          # Electron 主进程代码
├── android/ & ios/    # Capacitor 原生壳
├── packages/          # 独立子包
│   ├── super-sync-server/  # 自建同步服务器
│   ├── shared-schema/      # 共享数据模型 Schema
│   ├── plugin-api/         # 插件 API
│   └── plugin-dev/         # 插件开发工具
├── e2e/               # Playwright E2E 测试
└── docs/              # 架构文档
```

---

## 核心设计思路

### 1. 功能模块化（Feature Modules）

`src/app/features/` 下有 40+ 个独立模块，每个包含自己的：
- **model**（数据模型）
- **store**（NgRx actions/reducers/selectors/effects）
- **service**（业务逻辑）
- **components**（UI 组件）

关键功能模块包括：`tasks`（任务）、`project`（项目）、`tag`（标签）、`time-tracking`（时间追踪）、`planner`（计划）、`boards`（看板）、`issue`（第三方 Issue 集成，如 Jira/GitHub）、`config`（全局配置）、`work-context`（工作上下文）等。

### 2. 单向数据流（NgRx / Redux）

```
用户操作 → dispatch Action → Reducer（纯函数，不可变更新）→ Store
                                ↓
                           Selectors → 组件订阅
                                ↓
                           Effects → 副作用（持久化、同步、通知）
```

特别的：所有 Effects 必须使用 `LOCAL_ACTIONS` 而非 `Actions`，确保远程同步操作不会重复触发副作用。

### 3. 操作日志 & 同步架构

- `op-log/`：基于操作日志（Operation Log）的持久化层，记录所有状态变更
- 使用**向量时钟（Vector Clock）** 做冲突检测和解决
- 支持多种同步后端：Dropbox、WebDAV、本地文件、SuperSync Server
- `imex/sync/`：统一的同步入口

### 4. 跨平台抽象

```
Angular App（统一前端）
    ├── Web/PWA：Service Worker
    ├── Electron：主进程 IPC 通信（系统托盘、快捷键、空闲检测）
    └── Capacitor：原生桥接 Android/iOS
```

通过 `IS_ELECTRON` 等标志做平台条件判断，`core/platform/` 提供平台抽象层。

### 5. Meta-Reducers 处理跨实体原子操作

当一个 Action 影响多个实体时（如删除标签需同时从任务中移除），使用 meta-reducers 在单次 reducer pass 中完成所有变更，避免同步日志中出现不一致的部分操作。

### 6. 插件系统

`packages/plugin-api/` 定义插件接口，`packages/plugin-dev/` 提供开发工具，支持通过插件扩展功能。

---

## 数据模型核心实体

- **Task**：任务（支持子任务、重复任务、时间追踪）
- **Project**：项目（任务容器）
- **Tag**：标签（虚拟分组，TODAY_TAG 是特殊的虚拟标签，由 `dueDay` 决定）
- **WorkContext**：当前工作上下文（可以是 Project 或 Tag）
- **GlobalConfig**：全局用户配置

---

## 总结

这是一个**架构成熟、设计严谨的中大型 Angular 应用**。核心设计亮点：
1. 严格的 NgRx 单向数据流 + 操作日志实现可靠的多端同步
2. 向量时钟冲突解决机制
3. 高度模块化的 Feature Module 组织
4. 单一代码库支持 Web/桌面/移动三端
5. 完善的插件系统和第三方集成（Jira、GitHub 等）
