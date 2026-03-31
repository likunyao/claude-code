# 《Claude Code源码编程思想》

## 参考结构：C++编程思想（Thinking in C++）

---

## 前言
- 为什么要读 Claude Code 源码
- 本书结构与阅读指南
- 源码版本说明与获取方式

## 第一篇：构建基石 —— 从零理解 AI 编程助手的诞生

### 第1章：初识 Claude Code —— 一个 AI 编程助手的设计哲学
- 1.1 AI 编程助手的本质：对话即编程
- 1.2 Claude Code 的设计目标：安全、高效、可扩展
- 1.3 源码全景：1902 个源码文件的地图
- 1.4 从入口到交互：一次完整对话的生命周期
- 1.5 核心抽象三剑客：Query、Tool、Command
- 1.6 小结与思考

### 第2章：工程基础 —— 构建系统与项目结构
- 2.1 Bun 运行时与构建管线（bun:bundle、feature flags）
- 2.2 死代码消除：feature() 函数的多态构建
- 2.3 模块化设计：src/ 目录的职责划分
- 2.4 类型系统哲学：TypeScript 类型定义的组织（types/）
- 2.5 常量管理：constants/ 的分层策略
- 2.6 小结与思考

### 第3章：启动与初始化 —— setup.ts 的引导艺术
- 3.1 从 main.tsx 到 setup.ts：启动流程分析
- 3.2 配置加载的层次：全局 → 项目 → 会话
- 3.3 远程托管设置与策略限制：企业能力如何嵌入启动流程
- 3.4 环境探测：终端能力检测与适配
- 3.5 迁移系统：migrations/ 的版本演进策略
- 3.6 入口分流：CLI、REPL、Bridge、Assistant 模式的路由
- 3.7 小结与思考

## 第二篇：核心引擎 —— Agent 循环与查询引擎

### 第4章：Query Engine —— AI 交互的心脏
- 4.1 query.ts（68KB）与 QueryEngine.ts（46KB）的协作
- 4.2 消息流转模型：用户输入 → API 调用 → 工具执行 → 响应渲染
- 4.3 上下文窗口管理：token 计数与自动压缩
- 4.4 流式响应处理：SSE 解析与增量渲染
- 4.5 错误恢复与重试机制：categorizeRetryableAPIError
- 4.6 费用追踪：cost-tracker.ts 的实时成本计算
- 4.7 小结与思考

### 第5章：Tool 抽象 —— 万物皆工具的设计哲学
- 5.1 Tool 接口的本质：输入验证 → 权限检查 → 执行 → 结果
- 5.2 Tool.ts（29KB）：类型系统的精密设计
- 5.3 ToolUseContext：工具执行的上下文宇宙
- 5.4 tools.ts：工具注册与动态加载机制
- 5.5 工具匹配：toolMatchesName 的模糊查找
- 5.6 条件工具：feature flags、平台与产品能力的联合裁剪
- 5.7 小结与思考

### 第6章：文件操作工具 —— 代码感知的基础
- 6.1 FileReadTool：文件读取与安全边界
- 6.2 FileEditTool：精确编辑的字符串替换模型
- 6.3 FileWriteTool：文件创建与覆盖策略
- 6.4 GlobTool：模式匹配的文件搜索
- 6.5 GrepTool：基于 ripgrep 的内容搜索
- 6.6 LSPTool：语言服务协议的深度集成
- 6.7 NotebookEditTool：Jupyter Notebook 的结构化编辑
- 6.8 工具设计哲学：为什么是这些工具？
- 6.9 小结与思考

### 第7章：执行与搜索工具 —— 连接外部世界的桥梁
- 7.1 BashTool：沙箱化的命令执行
- 7.2 WebSearchTool：实时信息检索
- 7.3 WebFetchTool：网页内容的获取与解析
- 7.4 ToolSearchTool：工具自发现机制
- 7.5 SyntheticOutputTool：合成输出的设计意图
- 7.6 小结与思考

## 第三篇：多智能体协作 —— 从单体到群体智能

### 第8章：Agent Tool —— 嵌套智能体的递归之美
- 8.1 AgentTool：子智能体的创建与生命周期
- 8.2 Agent 定义：loadAgentsDir 的自定义 Agent 系统
- 8.3 子上下文隔离：createSubagentContext 的沙箱设计
- 8.4 递归查询：子 Agent 中的嵌套 Agent 调用
- 8.5 Agent 类型：Explore、Plan、general-purpose 的角色分工
- 8.6 小结与思考

### 第9章：Team 与 Task —— 群体智能的协调机制
- 9.1 TeamCreateTool / TeamDeleteTool：团队的创建与销毁
- 9.2 SendMessageTool：Agent 间的消息传递协议
- 9.3 Task v2：TaskCreate、TaskGet、TaskUpdate、TaskList
- 9.4 任务列表与后台运行任务：两套 task 概念的分层
- 9.5 任务依赖与阻塞：blocks / blockedBy 的 DAG 模型
- 9.6 协调者模式：coordinator/ 的中心调度
- 9.7 小结与思考

### 第10章：后台任务与调度 —— 异步世界的编排
- 10.1 TaskOutputTool：后台任务的输出管理
- 10.2 TaskStopTool：任务取消与中止
- 10.3 后台任务类型：shell、agent、remote、workflow 的统一状态面
- 10.4 ScheduleCronTool：定时任务与 Cron 表达式
- 10.5 Worktree 隔离：EnterWorktree / ExitWorktree
- 10.6 BriefTool：上下文摘要与任务交接
- 10.7 小结与思考

## 第四篇：终端UI引擎 —— React 在终端中的涅槃

### 第11章：Ink 框架 —— 终端中的 React 渲染引擎
- 11.1 React Reconciler 在终端：reconciler.ts 的适配层
- 11.2 Yoga 布局引擎：Flexbox 在终端的实现
- 11.3 终端渲染管线：虚拟DOM → ANSI 转义序列
- 11.4 事件系统：键盘、鼠标、焦点、点击事件
- 11.5 termio 解析器：CSI、ESC、OSC、SGR 序列解析
- 11.6 小结与思考

### 第12章：组件体系 —— 终端 UI 的组件化设计
- 12.1 components/：组件的组织哲学
- 12.2 消息渲染：AssistantMessage、ToolResult 组件
- 12.3 交互组件：PermissionDialog、AskUserQuestion
- 12.4 代码展示：语法高亮、diff 视图、搜索高亮
- 12.5 状态指示器：Spinner、Progress、Cost 显示
- 12.6 interactiveHelpers.tsx（57KB）：交互逻辑的集中管理
- 12.7 小结与思考

### 第13章：输入系统 —— 从按键到命令的旅程
- 13.1 Vim 模式：模态编辑器的完整实现（motions、operators、textObjects）
- 13.2 按键绑定系统：keybindings/ 的可配置映射
- 13.3 parse-keypress：按键事件的解析与分发
- 13.4 多行编辑与历史：history.ts 的会话管理
- 13.5 自动补全与建议：PromptSuggestion 服务
- 13.6 小结与思考

## 第五篇：通信与扩展 —— MCP 协议与插件系统

### 第14章：MCP 协议 —— Model Context Protocol 的设计与实现
- 14.1 MCP 架构概览：客户端、服务端、传输层
- 14.2 服务端管理：mcp/ 的连接生命周期
- 14.3 MCPTool：动态工具的注册与执行
- 14.4 资源系统：ListMcpResourcesTool、ReadMcpResourceTool
- 14.5 认证与安全：McpAuthTool 的权限控制
- 14.6 小结与思考

### 第15章：Hook 系统 —— 事件驱动的扩展机制
- 15.1 Hook 事件面：生命周期、工具、权限、任务与环境信号
- 15.2 Hook 生命周期：pre、post、error 阶段
- 15.3 Hook 执行引擎：shell 命令的调度与超时
- 15.4 Hook 与工具的协作：useCanUseTool 的权限拦截
- 15.5 自定义 Hook 开发指南
- 15.6 小结与思考

### 第16章：Skill 与 Command —— 行为复用的艺术
- 16.1 skills/：技能注册与执行系统
- 16.2 commands.ts（25KB）：命令系统的注册表模式
- 16.3 插件命令：Markdown + frontmatter 的内容驱动装载
- 16.4 命令实现分析：commit、review、mcp 等核心命令
- 16.5 斜杠命令：用户可调用技能的 DSL
- 16.6 命令与工具的协作模型
- 16.7 小结与思考

## 第六篇：智能基础设施 —— 安全、记忆与状态

### 第17章：权限系统 —— 安全计算的守门人
- 17.1 PermissionMode：权限模式的分类（default、bypass、auto）
- 17.2 ToolPermissionContext：细粒度的权限规则
- 17.3 alwaysAllow / alwaysDeny / alwaysAsk 的三级策略
- 17.4 Auto Mode 的危险规则剥离与安全边界
- 17.5 远程托管设置、策略限制与本地权限规则的三层治理
- 17.6 小结与思考

### 第18章：状态管理 —— 可预测的状态演化
- 18.1 AppState 与 AppStateStore：全局状态的单一真相源
- 18.2 不可变更新：DeepImmutable 类型与状态不可变性
- 18.3 React Context 在终端中的应用
- 18.4 会话持久化：sessionStorage 与 sessionHistory
- 18.5 文件状态缓存：FileStateCache 的增量追踪
- 18.6 小结与思考

### 第19章：记忆系统 —— AI 的长期记忆机制
- 19.1 memdir/：文件式记忆的存储设计
- 19.2 记忆类型：user、feedback、project、reference
- 19.3 loadMemoryPrompt：记忆的加载与注入
- 19.4 extractMemories：自动记忆提取服务
- 19.5 SessionMemory：长期记忆之外的会话级记忆层
- 19.6 团队记忆同步：从个人记忆走向共享记忆
- 19.7 记忆检索与相关性选择
- 19.8 MEMORY.md：记忆索引的结构与管理
- 19.9 小结与思考

## 第七篇：通信架构 —— 跨进程与跨平台

### 第20章：Bridge 系统 —— 进程间的桥梁
- 20.1 bridge/：桥接通信的完整架构
- 20.2 传输层：HybridTransport 与 SSE/CCR v2 的双栈设计
- 20.3 消息协议：会话恢复、投递确认与 worker 状态上报
- 20.4 远程控制：remote/ 的远程管理能力
- 20.5 桌面集成：与 Claude Desktop 的协同
- 20.6 小结与思考

### 第21章：API 层 —— 与 Claude 的通信
- 21.1 services/api/：API 客户端的分层设计
- 21.2 认证流程：oauth/ 的 OAuth 2.0 实现
- 21.3 速率限制：rateLimitMessages 的优雅降级
- 21.4 流式处理：SSE 的解析与重组
- 21.5 模型管理：模型选择与 Fast Mode
- 21.6 小结与思考

## 第八篇：进阶思想 —— 编程哲学的升华

### 第22章：设计模式映射 —— Claude Code 中的经典模式
- 22.1 命令模式（Command Pattern）：Tool 与 Command
- 22.2 策略模式（Strategy Pattern）：权限与模式切换
- 22.3 观察者模式（Observer Pattern）：React 状态与事件
- 22.4 中间件模式（Middleware）：Hook 系统的管道
- 22.5 工厂模式（Factory）：工具的动态创建
- 22.6 代理模式（Proxy）：MCP 工具的透明代理
- 22.7 小结与思考

### 第23章：并发与异步 —— 异步世界的设计智慧
- 23.1 AbortController：取消传播的设计
- 23.2 并行工具执行：独立调用的并行化策略
- 23.3 后台任务管理：异步任务的生命周期
- 23.4 Context 压缩：compact/ 的上下文窗口管理
- 23.5 流式处理的背压与缓冲
- 23.6 小结与思考

### 第24章：可扩展性设计 —— 面向未来的架构
- 24.1 插件：多入口能力包，而不是单一命令扩展
- 24.2 Feature Flags 与运行时能力裁剪
- 24.3 治理面扩展：managed settings、policy limits、permission rules
- 24.4 自定义 Agent：.claude/agents/ 的用户扩展
- 24.5 配置的热更新：运行时配置变更
- 24.6 MCP 生态：第三方工具的集成范式
- 24.7 小结与思考

## 附录
- 附录A：源码阅读路线图（推荐阅读顺序）
- 附录B：核心类型速查表
- 附录C：工具完整列表与接口说明
- 附录D：配置文件参考
- 附录E：术语表

---

## 全书约 24 章 + 5 个附录
## 预计总字数：15-20 万字
