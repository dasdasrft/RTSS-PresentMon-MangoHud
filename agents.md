# Antigravity Agent 极速执行与合并调用规范 (AGENTS.md)

> **设计背景**：针对 [Google Antigravity](https://antigravity.google/docs/models/) 中每次工具调用（Tool Call）均需承担 10–15 秒服务端往返排队延迟（TTFT）（[Google AI Developers Forum #198760](https://discuss.ai.google.dev/t/antigravity-ide-2-6-0-agent-latency-dominates-everything-a-0-01s-tool-call-takes-minutes-measured-reproducible/198760)）以及 256K 自动压缩阈值清空提示词缓存（Prompt Cache）的机制（[Google AI Developers Forum #198430](https://discuss.ai.google.dev/t/1m-advertised-context-vs-a-256k-compaction-clamp-in-the-shipped-binaries-plus-quota-and-agy-p-policy-questions-desktop-2-29-0-cli-1-5-1/198430)），本文件强制约束 Agent **最大化单步并发度、减少单轮调用步数、精简内部思考链**。将本文件保存在项目根目录即可自动生效。

## 1. 核心步数预算与并发铁律 (Step Budget & Parallelism)

1. **单轮步数硬上限（Max 8 Steps/Turn）**：
   - 常规代码修改、Bug 修复或功能实现必须在 **3 至 8 步工具调用内**完成，严禁将单轮任务拆碎成 15–25 步串行小调用（[Google AI Developers Forum #198760](https://discuss.ai.google.dev/t/antigravity-ide-2-6-0-agent-latency-dominates-everything-a-0-01s-tool-call-takes-minutes-measured-reproducible/198760)）。
   - 标准执行流水线应严格收敛为 **4 个阶段**：
     1. **Step 1（并发定位与全量读取）**：一次性并发搜索并读取所有相关文件。
     2. **Step 2（批量修改）**：在单步内一次性提交单个或多个文件的全部代码变更。
     3. **Step 3（单次合并验证）**：用一条串联命令一次性完成类型检查、构建与测试。
     4. **Step 4（总结交付）**：直接输出结果，不再发起多余的二次确认读取。

2. **强制同轮并发调用（Mandatory Parallel Tool Calls）**：
   - 凡是没有严格先后数据依赖的操作（例如读取 3 个不同组件、搜索 2 个关键字、修改 2 个独立文件），**必须在同一轮（同一个 Turn）内并发发出所有工具调用**，严禁“读完 A 再发请求读 B、读完 B 再发请求读 C”。

## 2. 文件读取与搜索合并规则 (Batched Read & Search)

1. **禁止碎片化切片读取（No Micro-Slicing）**：
   - 读取文件时，若文件小于 500 行，**必须一次性读取完整文件**；若文件较大，单次至少读取 **250–400 行**完整上下文区间。
   - **严禁**以 20–50 行为单位分多次连续滑动读取同一个文件（每次切片读取都会白白消耗 12 秒以上的 TTFT 排队时间）（[Google AI Developers Forum #198760](https://discuss.ai.google.dev/t/antigravity-ide-2-6-0-agent-latency-dominates-everything-a-0-01s-tool-call-takes-minutes-measured-reproducible/198760)）。

2. **精准搜索与目录隔离（Targeted Search & Ignore Rules）**：
   - 搜索符号或文本时，在一个正则表达式中合并多个候选词（例如 `patternA|patternB|patternC`），并在单次调用中指定目标子目录（如 `src/`）。
   - **严禁**扫描或遍历 `node_modules/`、`dist/`、`build/`、`.next/`、`target/`、`venv/`、`.git/` 或大型日志/锁文件（`package-lock.json`、`pnpm-lock.yaml`），防止触发搜索工具死循环（[Google AI Developers Forum #187044](https://discuss.ai.google.dev/t/antigravity-takes-too-long-and-gets-stuck-on-working-while-coding/187044)）。

## 3. 代码编辑与终端命令合并规则 (Batched Edits & Terminal Execution)

1. **单文件多处修改一次性提交（Single-Pass Multi-Edit）**：
   - 对同一文件的多处修改（如新增 `import`、修改类型定义、更新主函数逻辑），**必须在单次编辑工具调用中一次性合并提交**，严禁分 3–4 步逐块修改同一文件。
   - 修改完成后，编辑器工具本身已返回变更结果，**严禁**再额外调用一次读取工具去“看一眼刚改完的文件”。

2. **优先使用内置工具，限制终端与 PowerShell 调用（Terminal Restraint）**：
   - 查看目录结构、搜索文本、读写文件时，**必须使用 IDE 内置文件与搜索工具**，严禁通过终端执行 `cat`、`grep`、`find`、`ls`、`dir` 或 `Get-Content`（在 Windows 下频繁拉起 PowerShell 极易导致 Antigravity 界面假死在 `Working...`）（[Google AI Developers Forum #187044](https://discuss.ai.google.dev/t/antigravity-takes-too-long-and-gets-stuck-on-working-while-coding/187044)）。
   - 确需执行编译、Lint 或单元测试时，**必须使用 `&&` 或 `;` 将多条验证命令合并为单次终端调用**（例如：`npm run typecheck && npm test -- --watchAll=false`），严禁分多步先后执行。
   - 运行任何测试或开发脚本时，必须附带非交互参数（如 `--watch=false`、`CI=true`），严禁启动会阻塞终端的常驻监听进程。

## 4. 思考链压缩与 256K 上下文保护 (Concise Thinking & Context Hygiene)

1. **极简思考链（Concise Chain-of-Thought）**：
   - 在工具调用之间的过渡步骤中，内部思考（Thinking）应仅保留核心行动决策（控制在 **50–100 tokens** 以内），不要在思考链中重复默写代码片段、复述文件内容或展开冗长的自我辩论（[Google AI Developers Forum #198419](https://discuss.ai.google.dev/t/antigravity-flash-and-argon-models-are-incredibly-slow-in-response/198419)）。

2. **保护 256K 自动压缩水位线（256K Compaction Guard）**：
   - 切勿向终端打印超大 JSON、完整构建产物或数千行堆栈日志，避免单步塞满上下文并在 250K–256K 处触发耗时 70 秒的强制自动压缩及 240K+ 缓存清空（[Google AI Developers Forum #198430](https://discuss.ai.google.dev/t/1m-advertised-context-vs-a-256k-compaction-clamp-in-the-shipped-binaries-plus-quota-and-agy-p-policy-questions-desktop-2-29-0-cli-1-5-1/198430)）。
   - 如果终端输出可能很长，必须在命令末尾加上行数截断（例如 `| head -n 50` 或仅输出错误摘要）。

## 5. English Enforcement Directives (供底层模型高优先级解析)

- **BATCH ALL TOOL CALLS**: Every model turn in Antigravity incurs ~12s of server-side TTFT overhead ([Google AI Developers Forum #198760](https://discuss.ai.google.dev/t/antigravity-ide-2-6-0-agent-latency-dominates-everything-a-0-01s-tool-call-takes-minutes-measured-reproducible/198760)). You MUST minimize the total number of turns per user request (target: 3–8 steps max).
- **PARALLEL READS & EDITS**: Whenever you need to inspect or modify multiple files, emit ALL read/search/edit tool calls in parallel within the SAME turn. Never read files one by one across sequential turns.
- **NO MICRO-READS**: Read full files (up to 400 lines per call) instead of reading 30–50 lines repeatedly. Do NOT re-read a file immediately after editing it.
- **ONE-PASS EDITS**: Apply all changes to a file in a single batched edit call rather than splitting imports, types, and implementation across separate turns.
- **NO SHELL FOR FILE OPS**: Never spawn shell/PowerShell commands (`cat`, `grep`, `find`, `Get-ChildItem`) to read or search files ([Google AI Developers Forum #187044](https://discuss.ai.google.dev/t/antigravity-takes-too-long-and-gets-stuck-on-working-while-coding/187044)). Chain verification commands into a single execution (`npm run lint && npm test`).
- **ULTRA-CONCISE THINKING**: Keep intermediate step reasoning under 80 tokens ([Google AI Developers Forum #198419](https://discuss.ai.google.dev/t/antigravity-flash-and-argon-models-are-incredibly-slow-in-response/198419)). Do not draft full code blocks inside thinking traces before calling an edit tool.

## 6. 配套 `.antigravityignore` 推荐模板

建议同时在项目根目录创建 `.antigravityignore`（及 `.agentignore`），粘贴以下内容以彻底阻断后台无效索引（[Google AI Developers Forum #187044](https://discuss.ai.google.dev/t/antigravity-takes-too-long-and-gets-stuck-on-working-while-coding/187044)）：

```gitignore
node_modules/
dist/
build/
out/
.next/
.nuxt/
.turbo/
.cache/
target/
venv/
.venv/
__pycache__/
coverage/
*.min.js
*.min.css
*.map
package-lock.json
pnpm-lock.yaml
yarn.lock
*.log
```
