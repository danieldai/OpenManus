# OpenManus 项目架构分析

## 项目概述

OpenManus 是一个开源的 AI Agent 框架，基于 ReAct (Think-Act) 模式实现。它提供了从简单的单 Agent 任务到复杂的多 Agent 协作工作流的完整解决方案。

## 1. 整体架构层次图

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用入口层                                │
├─────────────────────────────────────────────────────────────────┤
│  main.py          run_flow.py        run_mcp.py                 │
│  (单Agent模式)    (多Agent工作流)    (MCP工具模式)              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Agent 核心层                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BaseAgent (抽象基类)                         │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • state: IDLE/RUNNING/FINISHED/ERROR               │  │  │
│  │  │ • memory: Memory (消息历史)                         │  │  │
│  │  │ • llm: LLM (语言模型接口)                          │  │  │
│  │  │ • run(): 主执行循环                                 │  │  │
│  │  │ • step(): 单步执行 (抽象方法)                       │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │            ReActAgent (Think-Act模式)                   │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • think(): 思考下一步行动                          │  │  │
│  │  │ • act(): 执行行动                                  │  │  │
│  │  │ • step() = think() + act()                         │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         ToolCallAgent (工具调用Agent)                    │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • available_tools: ToolCollection                  │  │  │
│  │  │ • think(): LLM选择工具并生成tool_calls              │  │  │
│  │  │ • act(): 执行工具调用                               │  │  │
│  │  │ • execute_tool(): 执行单个工具                      │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Manus (通用Agent)                           │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • PythonExecute, BrowserUseTool, StrReplaceEditor │  │  │
│  │  │ • AskHuman, Terminate                              │  │  │
│  │  │ • MCP客户端支持                                    │  │  │
│  │  │ • BrowserContextHelper (浏览器上下文管理)           │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  其他专用Agent:                                                  │
│  • BrowserAgent  • SWEAgent  • DataAnalysis                    │
│  • MCPAgent      • SandboxAgent                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        工具系统层                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BaseTool (工具基类)                         │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • name, description, parameters                     │  │  │
│  │  │ • execute(): 执行工具 (抽象方法)                     │  │  │
│  │  │ • to_param(): 转换为函数调用格式                     │  │  │
│  │  │ • success_response() / fail_response()             │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         ToolCollection (工具集合)                        │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • tools: 工具列表                                  │  │  │
│  │  │ • tool_map: 工具名称映射                           │  │  │
│  │  │ • execute(): 执行指定工具                           │  │  │
│  │  │ • to_params(): 转换为LLM工具参数格式                │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  具体工具实现:                                                   │
│  • PythonExecute      • BrowserUseTool    • StrReplaceEditor    │
│  • FileOperators      • WebSearch         • AskHuman            │
│  • Terminate          • MCPClientTool     • PlanningTool         │
│  • Sandbox工具集 (sb_browser, sb_files, sb_shell, sb_vision)    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Flow 工作流层                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BaseFlow (工作流基类)                        │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • agents: Dict[str, BaseAgent]                     │  │  │
│  │  │ • primary_agent: 主Agent                            │  │  │
│  │  │ • execute(): 执行工作流 (抽象方法)                  │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         PlanningFlow (规划式工作流)                      │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • planning_tool: 规划工具                          │  │  │
│  │  │ • active_plan_id: 当前计划ID                        │  │  │
│  │  │ • _create_initial_plan(): 创建初始计划              │  │  │
│  │  │ • _get_current_step_info(): 获取当前步骤            │  │  │
│  │  │ • _execute_step(): 执行步骤                         │  │  │
│  │  │ • _mark_step_completed(): 标记步骤完成              │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        基础设施层                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │     LLM      │  │   Memory     │  │   Schema     │         │
│  │              │  │              │  │              │         │
│  │ • ask()      │  │ • messages   │  │ • Message    │         │
│  │ • ask_tool() │  │ • add_msg()  │  │ • ToolCall   │         │
│  │ • ask_with_ │  │ • get_recent │  │ • AgentState │         │
│  │   images()   │  │   _messages()│  │ • Memory     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Sandbox    │  │     MCP      │  │   Config     │         │
│  │              │  │              │  │              │         │
│  │ • Sandbox    │  │ • Server     │  │ • LLM配置    │         │
│  │   Manager    │  │ • Client     │  │ • MCP配置    │         │
│  │ • Terminal   │  │ • Tools      │  │ • Flow配置   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Agent 执行工作流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent 执行主循环                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  agent.run()    │
                    │  初始化状态      │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ state = RUNNING │
                    └─────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │  循环: current_step < max_steps      │
        │  且 state != FINISHED                │
        └─────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  agent.step()   │
                    └─────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │         ReActAgent.step()           │
        └─────────────────────────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
    ┌──────────────────┐        ┌──────────────────┐
    │   think()        │        │    act()         │
    │                  │        │                  │
    │ 1. 准备消息      │        │ 1. 执行工具调用   │
    │ 2. 调用LLM       │        │ 2. 处理工具结果   │
    │ 3. 解析tool_calls│        │ 3. 更新Memory     │
    │ 4. 更新Memory    │        │ 4. 返回结果       │
    └──────────────────┘        └──────────────────┘
                │                           │
                └─────────────┬─────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  检查是否卡住    │
                    │  is_stuck()?    │
                    └─────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                  是 │                   │ 否
                    │                   │
                    ▼                   ▼
        ┌──────────────────┐  ┌──────────────────┐
        │ handle_stuck()   │  │ 继续下一步       │
        │ 添加提示改变策略  │  │ current_step++   │
        └──────────────────┘  └──────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │  检查终止条件                        │
        │  • state == FINISHED?               │
        │  • current_step >= max_steps?       │
        │  • 调用Terminate工具?                │
        └─────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                  是 │                   │ 否
                    │                   │
                    ▼                   │
        ┌──────────────────┐           │
        │  清理资源         │           │
        │  cleanup()        │           │
        └──────────────────┘           │
                              │         │
                              └─────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  返回结果        │
                    └─────────────────┘
```

## 3. Think-Act 详细流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Think 阶段 (思考)                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 准备消息上下文  │
                    │ • system_prompt │
                    │ • next_step_    │
                    │   prompt        │
                    │ • memory.messages│
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  llm.ask_tool() │
                    │  • 传入工具列表  │
                    │  • tool_choice   │
                    │  • 消息历史      │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  LLM返回响应     │
                    │  • content       │
                    │  • tool_calls[]  │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 解析tool_calls   │
                    │ 保存到self.      │
                    │ tool_calls       │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 创建assistant    │
                    │ message并添加到  │
                    │ memory           │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 返回是否需要执行 │
                    │ bool(tool_calls) │
                    └─────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Act 阶段 (执行)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 遍历tool_calls   │
                    │ 对每个工具调用:  │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ execute_tool()   │
                    │ 1. 解析参数       │
                    │ 2. 查找工具       │
                    │ 3. 执行工具      │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 工具执行结果     │
                    │ • output         │
                    │ • error          │
                    │ • base64_image   │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 处理特殊工具     │
                    │ • Terminate      │
                    │   → state=FINISHED│
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 创建tool message │
                    │ 添加到memory     │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 返回结果字符串    │
                    └─────────────────┘
```

## 4. 工具调用详细流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    工具调用执行流程                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ ToolCall对象     │
                    │ • id             │
                    │ • function.name  │
                    │ • function.      │
                    │   arguments      │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 解析JSON参数     │
                    │ json.loads(     │
                    │   arguments)     │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 从ToolCollection │
                    │ 查找工具         │
                    │ tool_map[name]  │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 调用工具执行     │
                    │ await tool(**args)│
                    │ 或                │
                    │ await tool.      │
                    │   execute(**args)│
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 工具返回结果     │
                    │ ToolResult:      │
                    │ • output         │
                    │ • error          │
                    │ • base64_image   │
                    │ • system         │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 格式化观察结果   │
                    │ "Observed output│
                    │  of cmd `name`   │
                    │  executed: ..."  │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 返回结果字符串   │
                    └─────────────────┘
```

## 5. PlanningFlow 多Agent工作流

```
┌─────────────────────────────────────────────────────────────────┐
│                  PlanningFlow 执行流程                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ execute(input)  │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 创建初始计划     │
                    │ _create_initial_ │
                    │   plan()         │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 使用LLM +        │
                    │ PlanningTool     │
                    │ 生成计划步骤     │
                    └─────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │  循环执行计划步骤                    │
        └─────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 获取当前步骤     │
                    │ _get_current_   │
                    │   step_info()    │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 选择执行Agent    │
                    │ get_executor()   │
                    │ 根据step_type    │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 执行步骤         │
                    │ _execute_step()  │
                    │ 1. 准备步骤上下文│
                    │ 2. agent.run()   │
                    │ 3. 获取执行结果  │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 标记步骤完成     │
                    │ _mark_step_     │
                    │   completed()    │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 还有未完成步骤?  │
                    └─────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                  是 │                   │ 否
                    │                   │
                    │                   ▼
                    │          ┌─────────────────┐
                    │          │ 完成计划         │
                    │          │ _finalize_plan() │
                    │          └─────────────────┘
                    │                   │
                    └───────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 返回最终结果     │
                    └─────────────────┘
```

## 6. 数据流图

```
┌──────────┐
│  用户输入 │
└────┬─────┘
     │
     ▼
┌─────────────────┐
│  Agent.run()    │
│  初始化Memory   │
└────┬────────────┘
     │
     ▼
┌─────────────────┐      ┌──────────────┐
│  Memory.messages│◄─────│  Message历史  │
│  [user_msg]     │      │  • user       │
└────┬────────────┘      │  • assistant  │
     │                   │  • tool       │
     ▼                   │  • system     │
┌─────────────────┐      └──────────────┘
│  think()        │
│  准备消息       │
└────┬────────────┘
     │
     ▼
┌─────────────────┐      ┌──────────────┐
│  LLM.ask_tool() │─────►│  LLM API     │
│  • messages     │      │  (OpenAI/    │
│  • tools        │      │   Azure/     │
│  • tool_choice  │      │   Bedrock)   │
└────┬────────────┘      └──────────────┘
     │
     ▼
┌─────────────────┐
│  LLM响应        │
│  • content      │
│  • tool_calls[] │
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  解析tool_calls │
│  保存到memory   │
└────┬────────────┘
     │
     ▼
┌─────────────────┐      ┌──────────────┐
│  act()          │─────►│  ToolCollection│
│  执行工具调用   │      │  • execute()  │
└────┬────────────┘      └──────┬───────┘
     │                         │
     │                         ▼
     │                ┌─────────────────┐
     │                │  BaseTool.      │
     │                │  execute()      │
     │                └──────┬──────────┘
     │                       │
     │                       ▼
     │                ┌─────────────────┐
     │                │  工具执行结果   │
     │                │  ToolResult      │
     │                └──────┬──────────┘
     │                       │
     ▼                       │
┌─────────────────┐         │
│  创建tool message│◄────────┘
│  添加到memory    │
└────┬─────────────┘
     │
     ▼
┌─────────────────┐
│  更新Memory     │
│  形成循环       │
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  检查终止条件    │
│  继续或结束      │
└─────────────────┘
```

## 7. 关键组件关系图

```
                    ┌──────────────┐
                    │   BaseAgent  │
                    │  (核心抽象)   │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ ReActAgent   │  │ 其他Agent    │  │ 专用Agent    │
│              │  │              │  │              │
│ • think()    │  │              │  │ • Browser    │
│ • act()      │  │              │  │ • SWE        │
│ • step()     │  │              │  │ • DataAnalysis│
└──────┬───────┘  └──────────────┘  └──────────────┘
       │
       ▼
┌──────────────┐
│ToolCallAgent │
│              │
│ • tool_calls │
│ • available_│
│   tools      │
└──────┬───────┘
       │
       ▼
┌──────────────┐      ┌──────────────┐
│    Manus     │─────►│ ToolCollection│
│              │      │              │
│ • MCP支持    │      │ • tools[]    │
│ • Browser    │      │ • tool_map{} │
│   上下文      │      │ • execute()  │
└──────────────┘      └──────┬───────┘
                              │
                              ▼
                    ┌──────────────┐
                    │  BaseTool    │
                    │              │
                    │ • execute()  │
                    │ • to_param() │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│PythonExecute │  │BrowserUseTool│  │StrReplaceEdit│
│              │  │              │  │              │
│              │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

## 8. 状态转换图

```
                    ┌─────────┐
                    │  IDLE   │
                    │ (空闲)   │
                    └────┬────┘
                         │
                         │ run()
                         ▼
                    ┌─────────┐
                    │ RUNNING │
                    │ (运行中) │
                    └────┬────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        │                │                │
        ▼                ▼                ▼
┌───────────┐    ┌───────────┐    ┌───────────┐
│ FINISHED  │    │  ERROR    │    │ 继续循环   │
│ (已完成)   │    │ (错误)     │    │            │
│           │    │           │    │ step()     │
│ • 正常完成 │    │ • 异常捕获 │    │            │
│ • Terminate│   │ • 状态回滚 │    │            │
└───────────┘    └───────────┘    └────────────┘
```

## 核心组件说明

### BaseAgent
- **职责**: 所有 Agent 的抽象基类
- **核心属性**:
  - `state`: Agent 状态 (IDLE/RUNNING/FINISHED/ERROR)
  - `memory`: 消息历史存储
  - `llm`: 语言模型接口
  - `max_steps`: 最大执行步数
- **核心方法**:
  - `run()`: 主执行循环
  - `step()`: 单步执行（抽象方法，由子类实现）
  - `update_memory()`: 更新消息历史
  - `is_stuck()`: 检测是否陷入循环
  - `handle_stuck_state()`: 处理卡住状态

### ReActAgent
- **职责**: 实现 Think-Act 模式的 Agent 基类
- **核心方法**:
  - `think()`: 思考阶段，分析当前状态并决定下一步行动（抽象方法）
  - `act()`: 执行阶段，执行决定的行动（抽象方法）
  - `step()`: 组合 think() 和 act() 的完整步骤

### ToolCallAgent
- **职责**: 支持工具调用的 Agent
- **核心属性**:
  - `available_tools`: 可用工具集合
  - `tool_calls`: 当前步骤的工具调用列表
  - `tool_choices`: 工具选择策略 (AUTO/NONE/REQUIRED)
- **核心方法**:
  - `think()`: 调用 LLM 选择工具并生成 tool_calls
  - `act()`: 执行工具调用并处理结果
  - `execute_tool()`: 执行单个工具调用
  - `_handle_special_tool()`: 处理特殊工具（如 Terminate）

### Manus
- **职责**: 通用 Agent，支持多种工具和 MCP
- **核心特性**:
  - 内置工具: PythonExecute, BrowserUseTool, StrReplaceEditor, AskHuman, Terminate
  - MCP 客户端支持，可连接外部 MCP 服务器
  - BrowserContextHelper 管理浏览器上下文
  - 动态工具加载和卸载

### ToolCollection
- **职责**: 工具集合管理器
- **核心功能**:
  - 工具注册和查找
  - 工具执行调度
  - 工具参数格式转换（用于 LLM 函数调用）

### BaseTool
- **职责**: 所有工具的抽象基类
- **核心方法**:
  - `execute()`: 执行工具（抽象方法）
  - `to_param()`: 转换为 LLM 函数调用格式
  - `success_response()` / `fail_response()`: 创建标准化的工具结果

### PlanningFlow
- **职责**: 规划式多 Agent 工作流
- **核心功能**:
  - 使用 LLM 创建任务计划
  - 按步骤执行计划
  - 根据步骤类型选择合适的 Agent
  - 跟踪计划执行进度

### LLM
- **职责**: 语言模型接口封装
- **支持**:
  - OpenAI API
  - Azure OpenAI
  - AWS Bedrock
- **核心方法**:
  - `ask()`: 普通对话
  - `ask_tool()`: 工具调用模式
  - `ask_with_images()`: 多模态输入
  - Token 计数和限制管理

### Memory
- **职责**: 消息历史管理
- **核心功能**:
  - 存储对话消息（user/assistant/tool/system）
  - 消息数量限制
  - 获取最近消息

## 工作流程总结

### 单 Agent 模式 (main.py)
1. 创建 Manus Agent
2. 接收用户输入
3. Agent 执行 Think-Act 循环
4. 返回结果

### 多 Agent 工作流模式 (run_flow.py)
1. 创建多个 Agent（Manus, DataAnalysis 等）
2. 创建 PlanningFlow
3. Flow 使用 LLM 创建任务计划
4. 按计划步骤执行，选择合适的 Agent
5. 跟踪进度并完成计划

### MCP 工具模式 (run_mcp.py)
1. 创建 Manus Agent
2. 连接配置的 MCP 服务器
3. 加载 MCP 工具到 Agent
4. 执行任务时可以使用 MCP 工具

## 关键步骤中的提示词 (Prompts)

提示词是 Agent 执行过程中的核心指令，定义了 Agent 的行为模式和决策逻辑。以下是各个关键步骤中使用的提示词：

### 1. Manus Agent (通用Agent)

**System Prompt:**
```
You are OpenManus, an all-capable AI assistant, aimed at solving any task presented by the user.
You have various tools at your disposal that you can call upon to efficiently complete complex requests.
Whether it's programming, information retrieval, file processing, web browsing, or human interaction
(only for extreme cases), you can handle it all.
The initial directory is: {directory}
```

**Next Step Prompt:**
```
Based on user needs, proactively select the most appropriate tool or combination of tools.
For complex tasks, you can break down the problem and use different tools step by step to solve it.
After using each tool, clearly explain the execution results and suggest the next steps.

If you want to stop the interaction at any point, use the `terminate` tool/function call.
```

**使用位置**: `app/agent/manus.py` - 通用任务处理

---

### 2. ToolCallAgent (工具调用Agent)

**System Prompt:**
```
You are an agent that can execute tool calls
```

**Next Step Prompt:**
```
If you want to stop interaction, use `terminate` tool/function call.
```

**使用位置**: `app/agent/toolcall.py` - 所有工具调用Agent的基类

---

### 3. BrowserAgent (浏览器自动化Agent)

**System Prompt:**
```
You are an AI agent designed to automate browser tasks. Your goal is to accomplish the ultimate task following the rules.

# Input Format
Task
Previous steps
Current URL
Open Tabs
Interactive Elements
[index]<type>text</type>
- index: Numeric identifier for interaction
- type: HTML element type (button, input, etc.)
- text: Element description

# Response Rules
1. RESPONSE FORMAT: You must ALWAYS respond with valid JSON in this exact format:
{"current_state": {"evaluation_previous_goal": "Success|Failed|Unknown - Analyze the current elements...",
"memory": "Description of what has been done...",
"next_goal": "What needs to be done with the next immediate action"}},
"action":[{"one_action_name": {action-specific parameter}}, ...]}

2. ACTIONS: You can specify multiple actions in the list to be executed in sequence.
3. ELEMENT INTERACTION: Only use indexes of the interactive elements
4. NAVIGATION & ERROR HANDLING: Handle popups, scroll, wait for page load
5. TASK COMPLETION: Use the done action as the last action when complete
6. VISUAL CONTEXT: When an image is provided, use it to understand the page layout
```

**Next Step Prompt:**
```
What should I do next to achieve my goal?

When you see [Current state starts here], focus on the following:
- Current URL and page title
- Available tabs
- Interactive elements and their indices
- Content above or below the viewport
- Any action results or errors

For browser interactions:
- To navigate: browser_use with action="go_to_url", url="..."
- To click: browser_use with action="click_element", index=N
- To type: browser_use with action="input_text", index=N, text="..."
- To extract: browser_use with action="extract_content", goal="..."
- To scroll: browser_use with action="scroll_down" or "scroll_up"

If you want to stop the interaction at any point, use the `terminate` tool/function call.
```

**使用位置**: `app/agent/browser.py` - 浏览器自动化任务

---

### 4. SWEAgent (软件工程Agent)

**System Prompt:**
```
SETTING: You are an autonomous programmer, and you're working directly in the command line with a special interface.

The special interface consists of a file editor that shows you {WINDOW} lines of a file at a time.
In addition to typical bash commands, you can also use specific commands to help you navigate and edit files.
To call a command, you need to invoke it with a function call/tool call.

Please note that THE EDIT COMMAND REQUIRES PROPER INDENTATION.
If you'd like to add the line '        print(x)' you must fully write that out, with all those spaces before the code!
Indentation is important and code that is not indented correctly will fail and require fixing before it can be run.

RESPONSE FORMAT:
Your shell prompt is formatted as follows:
(Open file: <path>)
(Current directory: <cwd>)
bash-$

First, you should _always_ include a general thought about what you're going to do next.
Then, for every response, you must include exactly _ONE_ tool call/function call.

Remember, you should always include a _SINGLE_ tool call/function call and then wait for a response from the shell
before continuing with more discussion and commands. Everything you include in the DISCUSSION section will be saved
for future reference.
If you'd like to issue two commands at once, PLEASE DO NOT DO THAT! Please instead first submit just the first tool call,
and then after receiving a response you'll be able to issue the second tool call.
Note that the environment does NOT support interactive session commands (e.g. python, vim), so please do not invoke them.
```

**使用位置**: `app/agent/swe.py` - 软件工程任务（代码编辑、调试等）

---

### 5. MCPAgent (MCP协议Agent)

**System Prompt:**
```
You are an AI assistant with access to a Model Context Protocol (MCP) server.
You can use the tools provided by the MCP server to complete tasks.
The MCP server will dynamically expose tools that you can use - always check the available tools first.

When using an MCP tool:
1. Choose the appropriate tool based on your task requirements
2. Provide properly formatted arguments as required by the tool
3. Observe the results and use them to determine next steps
4. Tools may change during operation - new tools might appear or existing ones might disappear

Follow these guidelines:
- Call tools with valid parameters as documented in their schemas
- Handle errors gracefully by understanding what went wrong and trying again with corrected parameters
- For multimedia responses (like images), you'll receive a description of the content
- Complete user requests step by step, using the most appropriate tools
- If multiple tools need to be called in sequence, make one call at a time and wait for results

Remember to clearly explain your reasoning and actions to the user.
```

**Next Step Prompt:**
```
Based on the current state and available tools, what should be done next?
Think step by step about the problem and identify which MCP tool would be most helpful for the current stage.
If you've already made progress, consider what additional information you need or what actions would move you
closer to completing the task.
```

**特殊提示词**:

**Tool Error Prompt:**
```
You encountered an error with the tool '{tool_name}'.
Try to understand what went wrong and correct your approach.
Common issues include:
- Missing or incorrect parameters
- Invalid parameter formats
- Using a tool that's no longer available
- Attempting an operation that's not supported

Please check the tool specifications and try again with corrected parameters.
```

**Multimedia Response Prompt:**
```
You've received a multimedia response (image, audio, etc.) from the tool '{tool_name}'.
This content has been processed and described for you.
Use this information to continue the task or provide insights to the user.
```

**使用位置**: `app/agent/mcp.py` - MCP协议工具调用

---

### 6. DataAnalysis Agent (数据分析Agent)

**System Prompt:**
```
You are an AI agent designed to data analysis / visualization task.
You have various tools at your disposal that you can call upon to efficiently complete complex requests.
# Note:
1. The workspace directory is: {directory}; Read / write file in workspace
2. Generate analysis conclusion report in the end
```

**Next Step Prompt:**
```
Based on user needs, break down the problem and use different tools step by step to solve it.
# Note
1. Each step select the most appropriate tool proactively (ONLY ONE).
2. After using each tool, clearly explain the execution results and suggest the next steps.
3. When observation with Error, review and fix it.
```

**使用位置**: `app/agent/data_analysis.py` - 数据分析和可视化任务

---

### 7. PlanningFlow (规划工作流)

**Planning System Prompt:**
```
You are an expert Planning Agent tasked with solving problems efficiently through structured plans.
Your job is:
1. Analyze requests to understand the task scope
2. Create a clear, actionable plan that makes meaningful progress with the `planning` tool
3. Execute steps using available tools as needed
4. Track progress and adapt plans when necessary
5. Use `finish` to conclude immediately when the task is complete

Available tools will vary by task but may include:
- `planning`: Create, update, and track plans (commands: create, update, mark_step, etc.)
- `finish`: End the task when complete
Break tasks into logical steps with clear outcomes. Avoid excessive detail or sub-steps.
Think about dependencies and verification methods.
Know when to conclude - don't continue thinking once objectives are met.
```

**Next Step Prompt:**
```
Based on the current state, what's your next action?
Choose the most efficient path forward:
1. Is the plan sufficient, or does it need refinement?
2. Can you execute the next step immediately?
3. Is the task complete? If so, use `finish` right away.

Be concise in your reasoning, then select the appropriate tool or action.
```

**计划创建提示词** (在 `_create_initial_plan` 中使用):
```
You are a planning assistant. Create a concise, actionable plan with clear steps.
Focus on key milestones rather than detailed sub-steps.
Optimize for clarity and efficiency.

[如果有多Agent] Now we have {agents_description} agents.
The information of them are below: {json.dumps(agents_description)}
When creating steps in the planning tool, please specify the agent names using the format '[agent_name]'.

Create a reasonable plan with clear steps to accomplish the task: {request}
```

**步骤执行提示词** (在 `_execute_step` 中使用):
```
CURRENT PLAN STATUS:
{plan_status}

YOUR CURRENT TASK:
You are now working on step {step_index}: "{step_text}"

Please only execute this current step using the appropriate tools.
When you're done, provide a summary of what you accomplished.
```

**使用位置**: `app/flow/planning.py` - 多Agent规划工作流

---

### 8. 提示词使用流程

提示词在 Agent 执行过程中的使用流程：

```
┌─────────────────────────────────────────────────────────┐
│              提示词在Think阶段的使用                      │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  think() 方法   │
                    └────┬────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │  1. 添加 next_step_prompt      │
        │     (作为user message)         │
        └────┬───────────────────────────┘
             │
             ▼
        ┌────────────────────────────────┐
        │  2. 调用 llm.ask_tool()         │
        │     • system_msgs:              │
        │       [system_prompt]           │
        │     • messages:                 │
        │       [历史消息 + next_step]    │
        │     • tools: 可用工具列表        │
        └────┬───────────────────────────┘
             │
             ▼
        ┌────────────────────────────────┐
        │  3. LLM根据提示词和上下文      │
        │     生成工具调用决策            │
        └────────────────────────────────┘
```

### 9. 提示词设计原则

1. **明确角色定位**: 每个提示词都明确定义了 Agent 的角色和能力
2. **任务导向**: 提示词指导 Agent 如何分解和执行任务
3. **工具使用指导**: 明确说明如何使用可用工具
4. **错误处理**: 包含错误处理和恢复策略
5. **终止条件**: 明确说明何时以及如何终止任务
6. **上下文感知**: 提示词会动态包含当前状态信息（如目录、URL、计划状态等）

### 10. 提示词文件位置

所有提示词定义在 `app/prompt/` 目录下：

- `app/prompt/manus.py` - Manus Agent 提示词
- `app/prompt/toolcall.py` - ToolCallAgent 基础提示词
- `app/prompt/browser.py` - BrowserAgent 提示词
- `app/prompt/swe.py` - SWEAgent 提示词
- `app/prompt/mcp.py` - MCPAgent 提示词
- `app/prompt/planning.py` - PlanningFlow 提示词
- `app/prompt/visualization.py` - DataAnalysis Agent 提示词

## 关键设计模式

1. **ReAct 模式**: Think-Act 循环，Agent 先思考再行动
2. **策略模式**: 不同的 Agent 实现不同的策略
3. **工厂模式**: FlowFactory 创建不同类型的工作流
4. **观察者模式**: Memory 记录所有交互历史
5. **模板方法模式**: BaseAgent.run() 定义执行流程，子类实现 step()

## 总结

OpenManus 是一个设计良好的 AI Agent 框架，具有以下特点：

1. **分层架构清晰**: Agent → Tool → Flow，职责明确
2. **执行模式标准**: ReAct (Think-Act) 模式，易于理解和扩展
3. **工具系统灵活**: 支持本地工具和 MCP 远程工具
4. **多 Agent 协作**: 通过 Flow 实现复杂的多 Agent 工作流
5. **状态管理完善**: 明确的状态机，支持错误处理和资源清理
6. **内存管理智能**: Memory 保存对话历史，支持上下文理解

该架构支持从简单的单 Agent 任务到复杂的多 Agent 协作工作流，具有良好的扩展性和可维护性。

