# Claude Code 文档体系设计

**日期**：2026-06-04  
**目标**：将 claude-code 架构分析整理成结构化文档体系，替换现有 `internals.md`

---

## 背景

现状：`docs/internals.md`（366行）是单一混合文档，覆盖多个子系统但结构松散。  
目标：重组为 13 篇文档，每篇聚焦一个子系统，兼顾架构概述和源码级深挖。

---

## 文档定位

- **受众**：个人学习 + 对外分享兼顾
- **深度**：每篇分层——开头架构概述（快速理解），后半段源码级深挖（深度参考）
- **位置**：全部平铺在 `docs/`，用编号文件名区分
- **处理现有文档**：`internals.md` 废弃，内容迁移至新结构

---

## 文件结构

```
docs/
  00-overview.md            # 整体架构：设计哲学、三层模型、核心数据流、子系统关系图
  01-query-engine.md        # 对话引擎：主循环、消息组装、上下文管理、压缩
  02-tool-system.md         # 工具系统：Tool 基类、注册表、45+ 工具分类、执行流程
  03-task-system.md         # 任务系统：后台任务类型、调度、通知、生命周期
  04-command-system.md      # 命令系统：斜杠命令注册、分派、100+ 命令分类
  05-hook-system.md         # Hook 系统：87个 hooks、权限检查、IDE 集成、快捷键
  06-state-management.md    # 状态管理：AppState 结构、响应式更新、数据流向
  07-memory-system.md       # Memory：4种类型、文件格式、MEMORY.md索引、相关性召回
  08-ui-rendering.md        # UI 渲染：自定义 Ink 框架、146个组件、终端渲染原理
  09-bridge-remote.md       # 桥接与远程：Agent 间通信5种方式、WebSocket、ListPeers
  10-skills-plugins.md      # Skills + Plugins：加载机制、内置技能、插件架构
  11-prompt-cache.md        # 专题：KV Cache 原理、cache_control、Break 检测
  12-speculation.md         # 专题：推测执行原理、文件 Overlay、Boundary 类型
```

---

## 文档模板

### 子系统文档（01~10）

```markdown
# [系统名称]

## 概述
一段话：这个系统是什么、解决什么问题、在整体架构中的位置

## 核心职责
3~5 条，bullet 列表

## 架构概览（快速理解）
- 核心组件图（文字版关系图）
- 关键数据流（箭头描述）

## 核心实现（源码级深挖）
### [模块1]
- 文件路径 + 关键函数/类
- 实现要点

### [模块2]
...

## 关键设计决策
值得注意的设计取舍、Why 这样设计

## 与其他系统的交互
列出与哪些系统有依赖关系，如何交互
```

### 专题文档（11~12）

```markdown
# [专题名]

## 是什么 & 解决什么问题
## 工作原理（核心机制）
## 实现细节（源码级）
## 边界条件 & 注意事项
```

---

## `00-overview.md` 结构

```markdown
# Claude Code 架构总览

## 什么是 Claude Code
## 整体架构：三层模型（UI 层 → Harness 层 → LLM 层）
## 子系统全景图
## 核心数据流（端到端一次对话流转）
## 设计哲学
## 文档导航（13 篇索引表）
```

---

## 实施说明

1. 按 `00 → 01 → 02 ...` 顺序逐篇创建
2. 每篇先从 `internals.md` 提取相关内容，再根据源码补充细节
3. 全部完成后删除 `internals.md`
4. 在 `docs/README.md`（或 `00-overview.md`）维护文档导航入口
