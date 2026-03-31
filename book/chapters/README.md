# 《Claude Code源码编程思想》

## Thinking in Claude Code Source

---

> "理解一个系统的最好方式，就是阅读它的源码。"

---

## 目录

### 前言
- [前言](book/chapters/00-preface.md)

### 第一篇：构建基石 —— 从零理解 AI 编程助手的诞生
- [第1章：初识 Claude Code —— 一个 AI 编程助手的设计哲学](book/chapters/ch01-introduction.md)
- [第2章：工程基础 —— 构建系统与项目结构](book/chapters/ch02-engineering.md)
- [第3章：启动与初始化 —— setup.ts 的引导艺术](book/chapters/ch03-setup.md)

### 第二篇：核心引擎 —— Agent 循环与查询引擎
- [第4章：Query Engine —— AI 交互的心脏](book/chapters/ch04-query-engine.md)
- [第5章：Tool 抽象 —— 万物皆工具的设计哲学](book/chapters/ch05-tool-abstraction.md)
- [第6章：文件操作工具 —— 代码感知的基础](book/chapters/ch06-file-tools.md)
- [第7章：执行与搜索工具 —— 连接外部世界的桥梁](book/chapters/ch07-exec-search-tools.md)

### 第三篇：多智能体协作 —— 从单体到群体智能
- [第8章：Agent Tool —— 嵌套智能体的递归之美](book/chapters/ch08-agent-tool.md)
- [第9章：Team 与 Task —— 群体智能的协调机制](book/chapters/ch09-team-task.md)
- [第10章：后台任务与调度 —— 异步世界的编排](book/chapters/ch10-async-tasks.md)

### 第四篇：终端UI引擎 —— React 在终端中的涅槃
- [第11章：Ink 框架 —— 终端中的 React 渲染引擎](book/chapters/ch11-ink-framework.md)
- [第12章：组件体系 —— 终端 UI 的组件化设计](book/chapters/ch12-components.md)
- [第13章：输入系统 —— 从按键到命令的旅程](book/chapters/ch13-input-system.md)

### 第五篇：通信与扩展 —— MCP 协议与插件系统
- [第14章：MCP 协议 —— Model Context Protocol 的设计与实现](book/chapters/ch14-mcp.md)
- [第15章：Hook 系统 —— 事件驱动的扩展机制](book/chapters/ch15-hooks.md)
- [第16章：Skill 与 Command —— 行为复用的艺术](book/chapters/ch16-skill-command.md)

### 第六篇：智能基础设施 —— 安全、记忆与状态
- [第17章：权限系统 —— 安全计算的守门人](book/chapters/ch17-permissions.md)
- [第18章：状态管理 —— 可预测的状态演化](book/chapters/ch18-state.md)
- [第19章：记忆系统 —— AI 的长期记忆机制](book/chapters/ch19-memory.md)

### 第七篇：通信架构 —— 跨进程与跨平台
- [第20章：Bridge 系统 —— 进程间的桥梁](book/chapters/ch20-bridge.md)
- [第21章：API 层 —— 与 Claude 的通信](book/chapters/ch21-api.md)

### 第八篇：进阶思想 —— 编程哲学的升华
- [第22章：设计模式映射 —— Claude Code 中的经典模式](book/chapters/ch22-patterns.md)
- [第23章：并发与异步 —— 异步世界的设计智慧](book/chapters/ch23-concurrency.md)
- [第24章：可扩展性设计 —— 面向未来的架构](book/chapters/ch24-extensibility.md)

### 附录
- [附录 A-E：源码路线图 / 类型速查 / 工具列表 / 配置参考 / 术语表](book/chapters/appendix.md)

---

## 统计信息

| 指标 | 数值 |
|------|------|
| 总章节 | 前言 + 24章 + 附录 |
| 总文件数 | 28 个 Markdown 文件 |
| 总行数 | 21,359 行 |
| 总大小 | 702 KB |
| 估算字数 | ~70 万字符 |

## 版权

本书基于 Claude Code 源码分析编写，仅供学习交流使用。
