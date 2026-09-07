> **示例输出。** 这是 `/improve` 针对
> [shadcn/ui](https://github.com/shadcn-ui/ui) 在 commit `1994caba0`
> （2026-06-10）生成的真实计划，作为格式示例保留。代码库已继续演进——不要执行本计划；请在你自己的仓库上运行 `/improve`。

# 计划 001：提取 search 与 view 共用的 shadow-config 解析

> **执行者须知**：按步骤执行本计划。每条验证命令都要跑完，并确认结果符合预期后再进入下一步。若「停止条件」一节中的任何情况发生，立即停止并汇报——不要即兴发挥。完成后，更新 `plans/README.md` 中本计划的状态行。
>
> **漂移检查（先跑）**：`git diff --stat 1994caba0..HEAD -- packages/shadcn/src/commands/search.ts packages/shadcn/src/commands/view.ts packages/shadcn/src/registry/config.ts`
> 若自本计划写成以来上述任一文件有改动，继续前请将「当前状态」中的摘录与实时代码对照；不一致则视为停止条件。

## 状态

- **优先级**：P2
- **工作量**：M
- **风险**：中
- **依赖**：无
- **类别**：tech-debt
- **规划于**：commit `1994caba0`，2026-06-10

## 为什么重要

`search.ts` 与 `view.ts` 各自手写了同一套 `shadow-config`（影子配置）回退逻辑——先构建默认配置，若存在则叠加上不完整的 `components.json`，再尝试完整的 `getConfig()`，失败则回退。代码中已承认这段重复（`search.ts:31`："TODO: We're duplicating logic for shadowConfig here. Revisit and properly abstract this."），而且两处副本**已经发生漂移**：search 用 `createConfig({style: "new-york", resolvedPaths: {cwd}})` 播种默认值，而 view 从空的 `configWithDefaults({})` 起步。今后对部分配置处理的任何改动（新默认值、校验规则）都必须改两处，并且可能静默分叉。

## 当前状态

- `packages/shadcn/src/commands/search.ts` — search/list 命令；shadow-config 代码块约在 91–115 行：

```ts
// search.ts ~91 (after `await loadEnvFiles(options.cwd)`)
// Start with a shadow config to support partial components.json.
// Use createConfig to get proper default paths
const defaultConfig = createConfig({
  style: "new-york",
  resolvedPaths: {
    cwd: options.cwd,
  },
})
let shadowConfig = configWithDefaults(defaultConfig)

// Check if there's a components.json file (partial or complete).
const componentsJsonPath = path.resolve(options.cwd, "components.json")
const hasComponentsJson = fsExtra.existsSync(componentsJsonPath)
if (hasComponentsJson) {
  const existingConfig = await fsExtra.readJson(componentsJsonPath)
  const partialConfig = rawConfigSchema.partial().parse(existingConfig)
  shadowConfig = configWithDefaults({
    ...defaultConfig,
    ...partialConfig,
  })
}

// Try to get the full config, but fall back to shadow config if it fails.
let config = shadowConfig
try {
  const fullConfig = await getConfig(options.cwd)
  if (fullConfig) {
    config = configWithDefaults(fullConfig)
  }
} catch {
  // Use shadow config if getConfig fails (partial components.json).
}
```

- `packages/shadcn/src/commands/view.ts` — view 命令；约 36–55 行是同一模式，但从 `configWithDefaults({})` 播种（没有 style/cwd 播种——这就是漂移）。
- `packages/shadcn/src/registry/config.ts:20` — `configWithDefaults(config?: DeepPartial<Config>)`，共享辅助函数的自然归属。同目录测试在 `packages/shadcn/src/registry/config.test.ts` — 以那些测试为模板。
- 约定：TypeScript ESM、`@/src/...` import 别名、来自 `@/src/schema` 的 zod schema、同目录 `*.test.ts` vitest 文件。风格对齐 `registry/config.ts`。

## 你将用到的命令

| 用途      | 命令                             | 成功时预期 |
|-----------|----------------------------------|-----------|
| 安装      | `pnpm install`                   | exit 0    |
| 测试      | `pnpm shadcn:test`               | 全部通过  |
| 检查+类型 | `pnpm check`                     | exit 0    |

在仓库根目录执行。

## 范围

**范围内**（只应修改这些文件）：
- `packages/shadcn/src/registry/config.ts`（添加共享辅助函数）
- `packages/shadcn/src/registry/config.test.ts`（为其编写测试）
- `packages/shadcn/src/commands/search.ts`（改用该辅助函数）
- `packages/shadcn/src/commands/view.ts`（改用该辅助函数）

**范围外**（不要动，即使看起来相关）：
- `packages/shadcn/src/commands/init.ts` — 通过交互提示构建配置，不是 shadow-config 模式；那里没有重复。
- `packages/shadcn/src/utils/get-config.ts` — `getConfig`/`createConfig` 保持原样；辅助函数组合调用它们。
- 不要改变*完整* `components.json` 的解析行为——当存在完整配置时，两个命令的行为必须与今天完全一致。

## Git 工作流

- 分支：`advisor/001-extract-shadow-config-resolution`
- 每步一次提交；提交信息遵循仓库的约定式风格（例如 `refactor(cli): extract shadow-config resolution` — 见 `git log` 中类似 `feat(cli): improve search command` 的例子）。
- 除非操作者明确要求，否则不要推送到远端，也不要打开拉取请求。

## 步骤

### 步骤 1：向 `registry/config.ts` 添加 `resolveShadowConfig`

添加一个导出的 async 函数：

```ts
export async function resolveShadowConfig(
  cwd: string,
  seed?: DeepPartial<Config>
): Promise<Config>
```

行为（从上文 search.ts 抽取）：构建 `configWithDefaults(createConfig({...seed, resolvedPaths: {cwd}}))`；若 `cwd` 下存在 `components.json`，用 `rawConfigSchema.partial()` 做部分解析并叠加；然后尝试 `getConfig(cwd)`，若返回配置则使用 `configWithDefaults(fullConfig)`；抛错时保留 shadow-config。`seed` 参数用于保留 search 的 `{style: "new-york"}` 播种。

在 `config.test.ts` 中添加测试（以该文件现有测试为模板）：无 components.json → 默认值；部分 components.json → 叠加；完整 components.json → getConfig 路径；格式错误的完整配置 → 回退到 shadow-config。

**验证**：`pnpm shadcn:test` → 全部通过，含 4 个新测试。

### 步骤 2：将 `search.ts` 改为使用该辅助函数

将约 91–115 行的代码块替换为对 `resolveShadowConfig(options.cwd, { style: "new-york" })` 的调用。删除因此不再使用的 import（`createConfig`、`rawConfigSchema`，以及若文件其余处也不再使用则删除 `fsExtra`/`path`）。

**验证**：`pnpm shadcn:test` → 通过；`pnpm check` → exit 0。

### 步骤 3：将 `view.ts` 改为使用该辅助函数

将约 36–55 行的代码块替换为 `resolveShadowConfig(options.cwd)`（不传 seed — 保留其当前的裸默认值行为）。清理未使用的 import。

**验证**：`pnpm shadcn:test` → 通过；`pnpm check` → exit 0；`grep -rn "shadow config" packages/shadcn/src/commands/` → 无匹配。

## 测试计划

- 针对 `resolveShadowConfig` 的 4 个新单元测试（步骤 1），放在 `registry/config.test.ts`，以该文件现有测试为模板。
- 现有命令测试必须保持通过：`pnpm shadcn:test`。
- 本计划不新增集成测试；两个命令的行为在抽取后保持不变（同一套逻辑，只放一处）。

## 完成标准

- [ ] `pnpm shadcn:test` 以 exit 0 结束；`resolveShadowConfig` 的 4 个新测试存在且通过
- [ ] `pnpm check` 以 exit 0 结束
- [ ] `grep -rn "TODO: We're duplicating logic for shadowConfig" packages/shadcn/src/` 无匹配（注释随重复逻辑一并删除）
- [ ] `search.ts` 与 `view.ts` 均调用 `resolveShadowConfig`；二者均不再包含内联的 shadow-config 代码块
- [ ] 范围内清单之外的文件未被修改（`git status`）
- [ ] 已更新 `plans/README.md` 的状态行

## 停止条件

出现以下情况时停止并汇报（不要即兴发挥）：

- 上述位置的代码与摘录不符（自 `1994caba0` 以来发生漂移）。
- search（`style: "new-york"`、cwd 解析路径）与 view（裸默认值）之间的播种差异被证明是不可省略的，且无法用 `seed` 参数表达——即除非辅助函数长出命令特有分支，否则测试失败。
- 从任一命令中移除该代码块需要改动范围内清单之外的文件。

## 维护说明

- 今后需要部分配置支持的命令应调用 `resolveShadowConfig`，不要复制该模式——评审应拒绝新的内联 shadow-config 代码块。
- 若 `init.ts` 日后需要部分配置续作，也应使用本辅助函数；此处推迟（范围外），因为 init 的提示驱动流程语义不同。
- 评审关注点：确认 view 在无 seed 路径上的行为逐字节一致——两处副本之间的漂移很可能是无意的，但如果 view *依赖* 裸默认值，不传 seed 的调用会保留该行为。
