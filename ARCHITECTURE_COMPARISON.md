# Claude Code 官方 vs Claudia 项目 - 工具执行架构对比报告

## 执行摘要

本报告对比了官方 claude-code CLI (`/opt/node22/lib/node_modules/@anthropic-ai/claude-code/cli.js`) 和 Claudia 项目的工具执行架构。

**核心发现**：
- 官方实现：完整的工具执行引擎，包含并发控制、钩子系统、权限管理
- Claudia 实现：**代理模式**，直接调用官方 claude 二进制，不实现工具执行逻辑
- 主要问题：Claudia 使用 `--dangerously-skip-permissions` 跳过所有安全检查

---

## 1. 官方 Claude Code 工具执行架构

### 1.1 核心组件

#### 1.1.1 EV0 类 - 工具执行队列管理器（cli.js:2925）

```javascript
class EV0 {
    toolDefinitions;      // 工具定义列表
    canUseTool;           // 权限检查函数
    tools = [];           // 工具执行队列
    toolUseContext;       // 工具使用上下文
    hasErrored = false;   // 错误标记

    constructor(A, Q, B) {
        this.toolDefinitions = A;
        this.canUseTool = Q;
        this.toolUseContext = B;
    }

    // 添加工具到队列
    addTool(A, Q) {
        // 1. 查找工具定义
        // 2. 验证输入 (inputSchema.safeParse)
        // 3. 检查并发安全性 (isConcurrencySafe)
        // 4. 添加到队列，状态设为 "queued"
        // 5. 触发队列处理
    }

    // 处理工具队列
    async processQueue() {
        for (let A of this.tools) {
            if (A.status !== "queued") continue;
            if (this.canExecuteTool(A.isConcurrencySafe))
                await this.executeTool(A);
            else if (!A.isConcurrencySafe)
                break; // 遇到非并发安全的工具，停止处理
        }
    }

    // 检查是否可以执行工具
    canExecuteTool(A) {
        let Q = this.tools.filter((B) => B.status === "executing");
        return Q.length === 0 || (A && Q.every((B) => B.isConcurrencySafe));
    }

    // 执行单个工具
    async executeTool(A) {
        A.status = "executing";
        // ... 调用 OY1 函数执行工具
        // ... 收集结果和上下文修改器
        A.status = "completed";
    }
}
```

**状态机**：
```
queued → executing → completed → yielded
```

#### 1.1.2 工具定义结构（cli.js:713）

```javascript
{
    name: "TodoWrite",                    // 工具名称
    inputSchema: zodSchema,               // Zod schema 用于输入验证
    outputSchema: zodSchema,              // 输出 schema

    // 判断是否可以并发执行
    isConcurrencySafe() { return false; },

    // 权限检查
    async checkPermissions(input) {
        return { behavior: "allow", updatedInput: input };
    },

    // 实际执行
    async call({todos}, context) {
        // 工具实现逻辑
        return { data: {...} };
    },

    // 渲染函数（用于 UI）
    renderToolUseMessage: fn,
    renderToolUseProgressMessage: fn,
    renderToolUseRejectedMessage: fn,
    renderToolUseErrorMessage: fn,
    renderToolResultMessage: fn,
}
```

#### 1.1.3 OY1 函数 - 工具执行核心（cli.js:2925+）

```javascript
async function* OY1(A, Q, B, G) {
    // A: tool_use block
    // Q: assistant message
    // B: canUseTool function
    // G: toolUseContext

    // 1. 输入验证
    let W = A.inputSchema.safeParse(A.input);
    if (!W.success) {
        // 返回验证错误
        yield { message: errorMessage };
        return;
    }

    // 2. 额外验证
    let X = await tool.validateInput?.(W.data, G);
    if (X?.result === false) {
        yield { message: X.message };
        return;
    }

    // 3. 执行 PreToolUse 钩子
    for await (let hook of PreToolUse(...)) {
        if (hook.blockingError) {
            yield { message: errorMessage };
            return;
        }
        if (hook.preventContinuation) {
            yield { message: stopMessage };
            return;
        }
    }

    // 4. 权限检查
    let E = await B(tool, input, G, message, toolUseId);
    if (E.behavior !== "allow") {
        yield { message: E.message };
        return;
    }

    // 5. 执行工具
    let N = await tool.call(E.updatedInput, context);

    // 6. 执行 PostToolUse 钩子
    for await (let hook of PostToolUse(...)) {
        if (hook.message) yield { message: hook.message };
    }

    // 7. 返回结果
    yield { message: toolResultMessage(N.data) };

    // 8. 返回上下文修改器（如果有）
    if (N.contextModifier) {
        yield { contextModifier: { toolUseID, modifyContext: N.contextModifier } };
    }
}
```

### 1.2 并发控制机制

**策略**：
1. 每个工具通过 `isConcurrencySafe(input)` 声明是否支持并发
2. 队列处理器检查当前执行中的工具
3. 如果有非并发安全的工具在执行，阻塞队列
4. 并发安全的工具可以同时执行

**代码示例（cli.js:2925）**：
```javascript
function mk3(A, Q) {
    return A.reduce((B, G) => {
        let Z = Q.options.tools.find((J) => J.name === G.name),
            I = Z?.inputSchema.safeParse(G.input),
            Y = I?.success ? Boolean(Z?.isConcurrencySafe(I.data)) : false;

        if (Y && B[B.length-1]?.isConcurrencySafe)
            B[B.length-1].blocks.push(G);  // 合并到当前批次
        else
            B.push({ isConcurrencySafe: Y, blocks: [G] });  // 新批次

        return B;
    }, []);
}
```

### 1.3 钩子系统

官方实现了完整的钩子系统（cli.js:3522）：

```javascript
{
    PreToolUse: {
        summary: "Before tool execution",
        description: "Input is JSON with tool_name, tool_input, and tool_use_id",
        // 可以：
        // - 修改输入
        // - 阻止执行
        // - 添加额外上下文
    },

    PostToolUse: {
        summary: "After tool execution",
        description: "Input is JSON with inputs and response",
        // 可以：
        // - 查看结果
        // - 添加额外上下文
        // - 阻止继续执行
    },

    PostToolUseFailure: {
        summary: "After tool execution fails",
        description: "Input is JSON with error information",
    },

    Stop: {
        summary: "When a stop hook is executed",
        description: "Executed at the end of a conversation turn",
    }
}
```

**钩子执行流程**：
```
PreToolUse → [阻止?] → 工具执行 → [成功?] → PostToolUse
                                      ↓ [失败]
                                PostToolUseFailure
```

### 1.4 权限管理

**多层权限检查**：
1. `checkPermissions` 方法（工具定义中）
2. `canUseTool` 函数（在 toolUseContext 中）
3. 钩子可以覆盖权限决策

**权限行为**：
```javascript
{
    behavior: "allow" | "deny" | "ask",
    message?: string,
    updatedInput?: any,
    decisionReason?: {
        type: "hook" | "user" | "policy",
        hookName?: string,
        reason?: string
    }
}
```

---

## 2. Claudia 项目架构

### 2.1 核心设计：代理模式

Claudia **不实现**工具执行逻辑，而是：
1. 调用官方 `claude` 二进制
2. 解析其 JSON 流输出
3. 转发到前端

**代码位置**：`/home/user/claudia/src-tauri/src/commands/claude.rs:787-886`

```rust
async fn execute_claude_code(
    app: AppHandle,
    project_path: String,
    prompt: String,
    model: String,
) -> Result<(), String> {
    let claude_path = find_claude_binary(&app)?;
    let mut cmd = create_command_with_env(&claude_path);

    cmd.arg("-p")
        .arg(&prompt)
        .arg("--model")
        .arg(&model)
        .arg("--output-format")
        .arg("stream-json")          // 获取 JSON 流
        .arg("--verbose")
        .arg("--dangerously-skip-permissions")  // ⚠️ 跳过所有权限检查！
        .current_dir(&project_path)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped());

    spawn_claude_process(app, cmd, prompt, model, project_path).await
}
```

### 2.2 进程管理

#### 2.2.1 ProcessRegistry（process/registry.rs）

```rust
pub struct ProcessRegistry {
    processes: Arc<Mutex<HashMap<i64, ProcessHandle>>>,
    next_id: Arc<Mutex<i64>>,
}

pub struct ProcessHandle {
    pub info: ProcessInfo,
    pub child: Arc<Mutex<Option<Child>>>,
    pub live_output: Arc<Mutex<String>>,  // 缓存实时输出
}

pub enum ProcessType {
    AgentRun { agent_id: i64, agent_name: String },
    ClaudeSession { session_id: String },
}
```

**关键功能**：
- 跟踪所有运行中的进程
- 提供进程终止能力
- 缓存实时输出（避免重复读取 JSONL）

#### 2.2.2 spawn_claude_process（claude.rs:1037-1203）

```rust
async fn spawn_claude_process(...) -> Result<(), String> {
    // 1. 启动进程
    let mut child = cmd.spawn()?;
    let pid = child.id();

    // 2. 获取 stdout/stderr
    let stdout = child.stdout.take()?;
    let stderr = child.stderr.take()?;

    // 3. 提取 session ID
    let session_id_holder = Arc::new(Mutex::new(None));

    // 4. 启动 stdout 读取任务
    let stdout_task = tokio::spawn(async move {
        let mut lines = stdout_reader.lines();
        while let Ok(Some(line)) = lines.next_line().await {
            // 解析 JSON 提取 session ID
            if let Ok(msg) = serde_json::from_str::<Value>(&line) {
                if msg["type"] == "system" && msg["subtype"] == "init" {
                    if let Some(sid) = msg["session_id"].as_str() {
                        *session_id_holder.lock().unwrap() = Some(sid.to_string());

                        // 注册到 ProcessRegistry
                        registry.register_claude_session(sid, pid, ...)?;
                    }
                }
            }

            // 缓存输出
            registry.append_live_output(run_id, &line)?;

            // 发送到前端
            app_handle.emit(&format!("claude-output:{}", session_id), &line)?;
            app_handle.emit("claude-output", &line)?;  // 向后兼容
        }
    });

    // 5. 监控进程完成
    tokio::spawn(async move {
        let _ = stdout_task.await;
        let _ = stderr_task.await;

        // 等待进程退出
        if let Some(mut child) = current_process.take() {
            match child.wait().await {
                Ok(status) => {
                    app_handle.emit("claude-complete", status.success())?;
                }
                Err(e) => {
                    log::error!("Failed to wait for Claude process: {}", e);
                }
            }
        }

        // 清理 ProcessRegistry
        registry.unregister_process(run_id)?;
    });

    Ok(())
}
```

### 2.3 Agent 执行（agents.rs:666-987）

```rust
async fn execute_agent(
    app: AppHandle,
    agent_id: i64,
    project_path: String,
    task: String,
    model: Option<String>,
    db: State<'_, AgentDb>,
    registry: State<'_, crate::process::ProcessRegistryState>,
) -> Result<i64, String> {
    // 1. 从数据库获取 Agent
    let agent = get_agent(db.clone(), agent_id).await?;

    // 2. 构建命令
    let mut cmd = create_command_with_env(&claude_path);
    cmd.arg("-p")
        .arg(&task)
        .arg("--system-prompt")            // ⭐ 使用自定义 system prompt
        .arg(&agent.system_prompt)
        .arg("--model")
        .arg(&execution_model)
        .arg("--output-format")
        .arg("stream-json")
        .arg("--verbose")
        .arg("--dangerously-skip-permissions")  // ⚠️ 跳过权限检查
        .current_dir(&project_path)
        .stdin(Stdio::null())              // ⭐ 不接受输入
        .stdout(Stdio::piped())
        .stderr(Stdio::piped());

    // 3. 启动进程
    let mut child = cmd.spawn()?;
    let pid = child.id();

    // 4. 注册到 ProcessRegistry 和数据库
    let run_id = db.insert_agent_run(...)?;
    registry.register_process(run_id, agent_id, pid, child)?;

    // 5. 读取输出并发送事件
    // （与 spawn_claude_process 类似）

    Ok(run_id)
}
```

### 2.4 输出处理

**事件系统**（Tauri Events）：
```rust
// 会话隔离事件（推荐）
app.emit(&format!("claude-output:{}", session_id), &line)?;
app.emit(&format!("claude-error:{}", session_id), &line)?;
app.emit(&format!("claude-complete:{}", session_id), success)?;
app.emit(&format!("claude-cancelled:{}", session_id), true)?;

// 全局事件（向后兼容）
app.emit("claude-output", &line)?;
app.emit("claude-error", &line)?;
app.emit("claude-complete", success)?;
app.emit("claude-cancelled", true)?;

// Agent 事件
app.emit(&format!("agent-output:{}", run_id), &line)?;
app.emit(&format!("agent-error:{}", run_id), &line)?;
app.emit(&format!("agent-complete:{}", run_id), success)?;
```

---

## 3. 架构对比总结

| 特性 | 官方 Claude Code | Claudia 项目 |
|-----|----------------|------------|
| **工具执行** | 完整实现，包含 EV0 队列管理器 | 委托给官方二进制 |
| **并发控制** | 智能并发，基于 `isConcurrencySafe` | 由官方二进制处理 |
| **钩子系统** | PreToolUse, PostToolUse, PostToolUseFailure, Stop | 无（依赖官方） |
| **权限管理** | 多层检查：工具定义 + canUseTool + 钩子 | **跳过所有检查** (`--dangerously-skip-permissions`) |
| **输入验证** | Zod schema + validateInput 方法 | 由官方二进制处理 |
| **错误处理** | 详细的错误分类和恢复 | 依赖官方二进制 |
| **进度报告** | 工具可以报告进度 | 解析官方输出 |
| **上下文修改** | contextModifier 机制 | 无法修改 |
| **流式输出** | async generator 模式 | 读取 stdout 并转发 |
| **进程管理** | Node.js child_process | Tokio async process |
| **会话管理** | 内存 + JSONL | JSONL + ProcessRegistry |
| **实时输出** | 不缓存，直接流式 | 缓存在 ProcessRegistry |

---

## 4. 发现的 Bug 和问题

### 4.1 🔴 严重安全问题

#### 问题 1：全局跳过权限检查

**位置**：`claude.rs:811`, `agents.rs:713`

```rust
.arg("--dangerously-skip-permissions")  // ⚠️ 危险！
```

**影响**：
- 所有工具执行不经过任何权限检查
- 用户无法控制 Agent 的操作
- 潜在的数据泄露或破坏风险

**官方的权限管理**：
```javascript
// 用户可以：
// - 允许 (allow)
// - 拒绝 (deny)
// - 每次询问 (ask)
// - 通过钩子自定义策略

const E = await canUseTool(tool, input, context, message, toolUseId);
if (E.behavior !== "allow") {
    yield { message: E.message };  // 拒绝执行
    return;
}
```

**建议**：
1. 移除 `--dangerously-skip-permissions`
2. 实现自己的权限 UI
3. 或至少提供一个设置让用户选择是否跳过

---

### 4.2 🟡 功能限制

#### 问题 2：无法自定义工具执行逻辑

由于 Claudia 完全依赖官方二进制，它无法：
- 添加自定义工具
- 修改现有工具的行为
- 实现自定义钩子
- 控制并发策略

**官方的灵活性**：
```javascript
// 可以注册自定义工具
toolDefinitions.push({
    name: "MyCustomTool",
    async call(input, context) {
        // 自定义逻辑
        return { data: result };
    }
});

// 可以注册钩子
registerHook("PreToolUse", async (context) => {
    // 自定义验证
    if (shouldBlock) {
        return { blockingError: "Blocked by policy" };
    }
});
```

**Claudia 的限制**：
- 只能使用官方提供的工具
- 无法拦截或修改工具调用
- 无法实现自定义策略

---

#### 问题 3：Agent 不支持交互式输入

**位置**：`agents.rs:715`

```rust
.stdin(Stdio::null())  // ⭐ 不接受输入
```

**问题**：
- Agent 执行时如果 Claude 请求用户输入会卡住
- 没有超时检测（虽然有 30 秒超时，但会终止进程）

**代码中的超时处理**（agents.rs:883-952）：
```rust
// 等待首次输出，超时 30 秒
for i in 0..300 {
    if first_output.load(Ordering::Relaxed) {
        break;
    }
    tokio::time::sleep(Duration::from_millis(100)).await;
}

if !first_output.load(Ordering::Relaxed) {
    warn!("TIMEOUT: No output from Claude process after 30 seconds");
    warn!("This usually means:");
    warn!("   1. Claude process is waiting for user input");
    // ... 强制终止进程
}
```

**影响**：
- 如果 Claude 因为 API 问题需要用户确认，Agent 会超时失败
- 无法支持需要用户反馈的工作流

---

### 4.3 🟢 轻微问题

#### 问题 4：重复的事件发送

**位置**：`claude.rs:1129-1133`, `agents.rs:810-812`

```rust
// 会话隔离事件
app_handle.emit(&format!("claude-output:{}", session_id), &line)?;
// 全局事件（向后兼容）
app_handle.emit("claude-output", &line)?;
```

**问题**：
- 每条消息发送两次
- 增加事件系统开销
- 可能导致前端重复处理

**建议**：
- 在前端统一使用会话隔离事件
- 移除全局事件

---

#### 问题 5：缺少工具进度反馈

官方支持工具进度报告：
```javascript
// 工具可以报告进度
async call(input, context, canUseTool, message, reportProgress) {
    for (let i = 0; i < 100; i++) {
        reportProgress({
            toolUseID: "xxx",
            data: { percent: i }
        });
        await doWork();
    }
    return { data: result };
}
```

Claudia 只能：
- 等待官方二进制返回进度消息
- 无法自定义进度格式
- 无法为自定义工具添加进度

---

#### 问题 6：ProcessRegistry 的内存泄漏风险

**位置**：`process/registry.rs:431-450`

```rust
pub fn append_live_output(&self, run_id: i64, output: &str) -> Result<(), String> {
    let processes = self.processes.lock()?;
    if let Some(handle) = processes.get(&run_id) {
        let mut live_output = handle.live_output.lock()?;
        live_output.push_str(output);  // ⚠️ 无限增长！
        live_output.push('\n');
    }
    Ok(())
}
```

**问题**：
- 输出会持续累积，直到进程结束
- 长时间运行的 Agent 会消耗大量内存
- 没有清理机制

**建议**：
1. 限制缓存大小（如最后 10MB）
2. 或只缓存最近的 N 行
3. 或完全依赖 JSONL 文件

---

### 4.4 🔵 设计建议

#### 建议 1：实现部分工具执行逻辑

虽然完全复制官方架构不现实，但可以：
1. 实现一个简化的工具注册系统
2. 支持 Rust 侧的工具（如文件操作、系统信息）
3. 将这些工具注入到 Claude 对话中

**示例架构**：
```rust
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;

    async fn call(&self, input: serde_json::Value) -> Result<ToolResult, ToolError>;

    fn is_concurrency_safe(&self) -> bool { false }
    fn requires_permission(&self) -> bool { true }
}

pub struct ToolRegistry {
    tools: HashMap<String, Box<dyn Tool>>,
}

impl ToolRegistry {
    pub fn register<T: Tool + 'static>(&mut self, tool: T) {
        self.tools.insert(tool.name().to_string(), Box::new(tool));
    }

    pub async fn execute(&self, name: &str, input: Value) -> Result<ToolResult> {
        let tool = self.tools.get(name)?;

        // 权限检查
        if tool.requires_permission() {
            check_permission(name, &input).await?;
        }

        tool.call(input).await
    }
}
```

---

#### 建议 2：添加权限管理 UI

即使使用官方二进制，也可以：
1. 移除 `--dangerously-skip-permissions`
2. 实现一个简单的权限弹窗
3. 让用户确认每个工具调用

**实现思路**：
```rust
// 监听 official 二进制的输出
if msg["type"] == "permission_request" {
    let tool_name = msg["tool_name"].as_str()?;
    let input = &msg["input"];

    // 显示权限弹窗
    let decision = show_permission_dialog(tool_name, input).await?;

    if decision == "allow" {
        // 向 stdin 发送允许命令（如果官方支持）
        child.stdin.write(b"allow\n")?;
    } else {
        // 拒绝
        child.stdin.write(b"deny\n")?;
    }
}
```

**问题**：官方 claude 可能不支持这种交互模式

---

#### 建议 3：改进进程生命周期管理

**当前问题**：
- 进程终止逻辑分散在多个地方
- ClaudeProcessState 和 ProcessRegistry 有重叠
- 缺少统一的清理机制

**建议架构**：
```rust
pub struct UnifiedProcessManager {
    registry: ProcessRegistry,

    // 统一的进程启动
    pub async fn spawn_claude(&self, config: ClaudeConfig) -> Result<ProcessHandle> {
        // 1. 创建进程
        // 2. 注册到 registry
        // 3. 设置输出处理
        // 4. 返回 handle
    }

    // 统一的进程终止
    pub async fn kill(&self, handle: ProcessHandle) -> Result<()> {
        // 1. 发送 SIGTERM
        // 2. 等待 5 秒
        // 3. 发送 SIGKILL
        // 4. 清理资源
        // 5. 从 registry 移除
    }

    // 自动清理
    pub async fn cleanup_finished(&self) -> Result<Vec<i64>> {
        // 定期调用，清理已结束的进程
    }
}
```

---

## 5. 性能对比

| 指标 | 官方 Claude Code | Claudia 项目 |
|-----|----------------|------------|
| **启动延迟** | 低（Node.js 内存中） | 高（启动子进程） |
| **内存占用** | 单进程 | 多进程（官方 + Tauri） |
| **并发性能** | 智能并发调度 | 依赖官方实现 |
| **流式响应** | 即时 | 即时（转发） |
| **工具执行** | 高效（同进程） | 额外 IPC 开销 |

---

## 6. 可维护性对比

| 方面 | 官方 Claude Code | Claudia 项目 |
|-----|----------------|------------|
| **复杂度** | 高（完整实现） | 低（代理模式） |
| **版本同步** | 不需要 | **必须跟随官方版本** |
| **自定义能力** | 高 | 低 |
| **调试难度** | 中 | 高（跨进程） |
| **测试覆盖** | 可以单元测试工具 | 只能集成测试 |

---

## 7. 总体评价

### 7.1 Claudia 的优势

1. **快速开发**：不需要重新实现工具执行引擎
2. **功能完整**：自动获得官方的所有工具
3. **稳定性**：工具执行逻辑由官方维护
4. **易于维护**：代理模式降低了代码复杂度

### 7.2 Claudia 的劣势

1. **安全风险**：跳过所有权限检查
2. **功能受限**：无法自定义工具或钩子
3. **性能损失**：额外的进程创建和 IPC 开销
4. **版本依赖**：必须与官方二进制版本同步
5. **调试困难**：工具执行在另一个进程中

### 7.3 建议的改进优先级

#### 🔴 高优先级（安全问题）
1. 移除 `--dangerously-skip-permissions` 或提供配置选项
2. 实现基本的权限确认 UI
3. 添加 Agent 操作日志审计

#### 🟡 中优先级（功能增强）
1. 支持交互式 Agent（处理 stdin）
2. 实现自定义工具注册系统
3. 优化 ProcessRegistry 的内存使用

#### 🟢 低优先级（体验优化）
1. 统一事件系统（移除重复发送）
2. 改进进程生命周期管理
3. 添加工具执行统计和分析

---

## 8. 结论

Claudia 项目采用的**代理模式**是一个务实的选择，避免了重新实现复杂的工具执行引擎。然而，**使用 `--dangerously-skip-permissions` 是一个严重的安全问题**，应该优先解决。

**建议的演进路径**：
1. 短期：实现权限确认 UI，移除 `--dangerously-skip-permissions`
2. 中期：添加自定义工具注册系统，支持 Rust 侧工具
3. 长期：考虑实现轻量级的工具执行引擎，减少对官方二进制的依赖

**架构选择的权衡**：
- 如果追求**快速迭代和稳定性**：保持代理模式，修复安全问题
- 如果追求**功能灵活性和性能**：实现自己的工具执行引擎（工作量大）
- **推荐**：混合模式 - 官方二进制处理标准工具，Rust 侧处理自定义工具

---

## 附录 A：关键代码位置

### 官方 Claude Code
- 工具执行队列：`cli.js:2925` (class EV0)
- 工具执行核心：`cli.js:2925+` (function OY1)
- 工具定义示例：`cli.js:713` (TodoWrite)
- 钩子系统：`cli.js:3522`
- 并发控制：`cli.js:2925` (mk3, ck3, dk3 functions)

### Claudia 项目
- Claude 执行：`src-tauri/src/commands/claude.rs:787-886`
- Agent 执行：`src-tauri/src/commands/agents.rs:666-987`
- 进程管理：`src-tauri/src/process/registry.rs`
- 进程启动：`src-tauri/src/commands/claude.rs:1037-1203`

---

## 附录 B：官方工具并发策略示例

```javascript
// Read 工具：并发安全（只读操作）
{
    name: "Read",
    isConcurrencySafe: () => true,
}

// Write 工具：不并发安全（修改文件）
{
    name: "Write",
    isConcurrencySafe: () => false,
}

// Bash 工具：取决于命令
{
    name: "Bash",
    isConcurrencySafe: (input) => {
        // 只读命令可以并发
        if (input.command.startsWith("cat") ||
            input.command.startsWith("ls")) {
            return true;
        }
        return false;
    }
}
```

---

**报告生成时间**：2025-12-29
**官方版本**：cli.js (10.4MB)
**Claudia 版本**：当前 main 分支
**分析工具**：代码静态分析 + 架构对比
