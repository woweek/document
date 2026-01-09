# 构建决策选型与集成开发环境源码解析跳转

## 目录
- [决策起点：重新审视需求](#决策起点重新审视需求)
- [需求收敛：只剩下「安全可回溯的 minify」](#需求收敛只剩下安全可回溯的-minify)
- [SWC 在当前需求下的问题](#swc-在当前需求下的问题)
- [为什么 esbuild 能成为「平替但更干净」的选择](#为什么-esbuild-能成为平替但更干净的选择)
- [从「选 esbuild」到「不用 esbuild CLI」](#从选-esbuild到不用-esbuild-cli)
- [最终结论](#最终结论)
- [实现细节](#实现细节)
- [集成开发环境源码解析跳转](#集成开发环境源码解析跳转)

---

## 决策起点：重新审视需求

### 关键转折点：需求反向推导

决策的核心不是工具对比，而是**需求反向推导**。

我们的前提条件：

* 运行环境是 **Node.js 22**
* 代码本身：
  * 不需要降级（无需 transpile）
  * 不依赖 polyfill
  * 没有 tree-shaking 需求
* SWC 当前实际只承担了：
  * **压缩（minify）**
  * 而且在 Node 后端场景下，**mangle 还会破坏可观测性**

第一个重要的工程判断：

> **编译 ≠ 构建的必需步骤**
> 
> 在 Node 22 下，SWC 的「编译能力」几乎全部是冗余的

---

## 需求收敛：只剩下「安全可回溯的 minify」

### 1. mangle + sourcemap ≠ 可用的生产可观测性

在分析生产问题时，我们确认了几个关键结论：

* Node 后端：
  * 错误来自运行时（stack trace）
  * async / closure / dynamic import 会严重削弱 sourcemap 的还原能力
* 即便存在 sourcemap：
  * 变量名被 mangle 后，**逻辑语义已经丢失**
  * 线上错误无法快速定位

**关键约束条件**：

> **Node 后端服务：只能 minify，不能 mangle**

这一步直接影响了工具选型。

---

## SWC 在当前需求下的问题

### 1. ignore 与 copy-files 的冲突

SWC CLI 的行为限制：

* `--ignore`：
  * 会彻底跳过文件
  * **连 copy 都不会发生**
* `-D / --copy-files`：
  * 无法覆盖被 ignore 的路径

直接后果：

* SWC CLI 无法同时实现 ignore 和 copy，导致之前的方案必须先用 `pnpm -r exec` 遍历所有包，再用 `find` 过滤文件，**变相增加了构建步骤和复杂度**
* 不得不用 shell 手段（`mv / mktemp / restore`）临时移动文件
* 构建逻辑被迫"命令式 + 状态型"
* 可读性、可维护性显著下降

### 2. CLI 在 monorepo 下的表达力不足

在 monorepo + `pnpm -r exec` 场景下：

* SWC CLI：
  * 对 cwd、输出结构、日志上下文几乎不可控
  * 输出无法标识「当前是哪个 package 在构建」
* 为了解决这些问题：
  * 需要大量 shell glue
  * 与 zx 并发执行产生输出乱序问题

认知：

> **不是 SWC 不强，而是 CLI 这个抽象层已经不够了**

---

## 为什么 esbuild 能成为「平替但更干净」的选择

在只剩 **minify（不 mangle）** 这个核心需求后，需要在主流构建工具中选择最合适的方案。

### 备选方案对比

评估了以下工具：

* **Rollup / Rolldown**：
  * 定位：模块打包器（bundler）
  * 问题：我们不需要打包，只需要 minify
  * 结论：功能过重，不适用
* **Terser**：
  * 定位：纯 minify 工具，能力上对标 esbuild
  * 问题：JavaScript 实现，性能差
  * 结论：不适用
* **SWC**：
  * 优势：Rust 实现，性能强
  * 问题：CLI 在 monorepo 下表达力不足（见上文）
* **esbuild**：
  * Go 实现，冷启动快，IO 高效
  * API 灵活，与 monorepo 自然融合
  * 功能刚好命中需求，无冗余

**最终选择 esbuild 的核心原因**：在"只需要 minify"的约束下，esbuild 是唯一同时满足**性能、简洁、可控**三个条件的工具。

### 1. 功能层面：刚好命中最小需求集

* 支持：
  * `minifyWhitespace: true`
  * `minifySyntax: true`
* 可以**明确不启用 mangle**
* Node 22 下：
  * 不需要 loader hack
  * 不需要 target 降级

换句话说：

> esbuild 没有「多余能力」需要你刻意关掉

### 2. 性能与运行模型

在我们的场景下：

* 输入规模：中大型 monorepo
* 构建目标：JS → JS
* 不做 AST 级复杂转换

结论非常明确：

* **esbuild 的冷启动 + IO + minify 路径**
* 明显优于：
  * SWC CLI（初始化 Rust + 参数解析 + 目录扫描）
  * Babel / Terser（解释器级）

性能不是"锦上添花"，而是让我们敢于**在 pnpm -r 并发场景下放开跑**。

---

## 从「选 esbuild」到「不用 esbuild CLI」

这是整个思考链条中**最重要、也最成熟的一步**。

### 1. CLI 的根本问题不是功能，而是「上下文不可控」

遇到的问题包括：

* 无法可靠输出当前执行 package 的路径
* stdout 在并发下乱序
* 需要依赖 `pnpm exec` 的隐式环境变量
* 无法在输出中稳定注入 monorepo 语义（cwd / package name）

本质认知：

> **CLI 是给人用的，不是给"构建系统"用的**

### 2. esbuild API 恰好补齐了 CLI 的所有短板

#### （1）执行效率：消除进程启动开销

**CLI 模式（pnpm -r exec）：**
```bash
pnpm -r exec "esbuild src/**/*.js --outdir=dist"
```
- 每个包启动一个独立的 esbuild 进程
- monorepo 有 N 个包 → 启动 N 次 esbuild
- 每次进程启动都要：初始化 + 解析参数 + 加载配置

**API 模式（Promise.all）：**
```javascript
await Promise.all(packages.map(name => 
  esbuild.build({ absWorkingDir: `packages/${name}`, ... })
))
```
- 单进程内并发执行
- 只启动一次 Node.js 进程
- 多个构建任务共享内存，无额外进程开销

**性能差异：**
- CLI 模式：~5-8s（进程启动占 30-40% 时间）
- API 模式：~3s（无进程启动开销）

#### （2）cwd 完全可控

```javascript
esbuild.build({
  absWorkingDir: packagePath,
})
```

这一步直接解决了：

* monorepo 子包定位
* src / dist 相对路径问题
* glob 在"错误 cwd"下失效的问题

#### （3）输出路径、结构、日志完全可控

* 不再依赖终端输出猜测执行位置
* 可以：
  * 在 build 前后自己输出 package 名
  * 保证日志顺序
  * 与 zx / Promise 并发模型自然融合

#### （4）API 本身是「可组合的」

这对我们非常关键：

* 可以轻松：
  * 跳过没有 src 的包
  * 忽略 `packages/standalone`
  * 按 monorepo 规则做过滤
* 而不是把逻辑硬塞进 shell / glob / pnpm filter

---

## 最终结论

我们并不是"从 SWC 切换到 esbuild"，而是经历了**三次理性收敛**：

1. **从"我要编译" → "我只需要 minify"**
2. **从"工具能力" → "生产可观测性优先"**
3. **从"CLI 自动化" → "API 级构建控制"**

最终选择 esbuild API，是因为它：

* 在 Node 22 后端场景下：
  * 功能刚好
  * 行为确定
  * 性能足够
* 并且：
  * 能自然融入 monorepo
  * 不再需要 shell hack
  * 构建逻辑可读、可维护、可演进

---

## 实现细节

### 核心脚本：scripts/build.mjs

详见 `scripts/build.mjs` 源码。

**核心特性：**
- 交互式选择或 CI 全量构建
- 并发构建多个包
- 实时输出构建分析

### 错误处理策略：快速失败（Fail Fast）

**设计考量：**

最初考虑在 `Promise.all` 中使用 `try-catch` 来捕获错误，让构建在某个包失败时继续执行其他包。但这会导致：
* 错误不容易被发现，需要翻阅完整的 CI/CD 日志才能定位
* 无法快速定位是哪个包失败（日志可能被其他包的输出淹没）
* 调试困难，堆栈信息不完整

**最终方案：**

```javascript
// ✅ 直接抛错，不捕获
await Promise.all(packages.map(async name => {
  const dir = path.resolve(`packages/${name}`)
  await build({ absWorkingDir: dir, ... })
  console.log(`${chalk.green(name)}: built successfully`)
}))
```

**优势：**
1. 任一包失败，立即停止，错误立即暴露
2. 错误信息包含包名（通过 `absWorkingDir` 上下文）
3. 堆栈完整，便于调试
4. CI/CD 能正确捕获失败状态（exit code 1）

---

## 集成开发环境源码解析跳转

### jsconfig.json：代码跳转配置

**文件位置：** 项目根目录

**支持的 IDE：**
- ✅ **VSCode** - 原生支持
- ✅ **WebStorm / IntelliJ IDEA** - 自动识别
- ✅ **其他基于 LSP 的编辑器** - 大多支持

**作用：**
- 让 IDE 识别 monorepo 别名（`@common/*`、`@tokens/*` 等）
- Cmd+点击跳转到**源码**（`src/`）而非构建产物（`dist/`）

**重要：**
- ❌ `package.json` 的 `source` 字段对 IDE 跳转**无效**
- ✅ 跳转靠 `jsconfig.json`，运行时靠 `exports`

**配置详见：** `jsconfig.json` 源码

---

## 附录

### 相关资源

- [esbuild 官方文档](https://esbuild.github.io/)
- [VSCode jsconfig.json](https://code.visualstudio.com/docs/languages/jsconfig)
- [TypeScript paths 配置](https://www.typescriptlang.org/tsconfig#paths)
