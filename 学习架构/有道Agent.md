# LobsterAI 子 Agent 建立过程

## 一、概述

LobsterAI 使用 `@anthropic-ai/claude-agent-sdk` 作为底层 Agent 执行引擎。每次新建任务时，会创建一个 Agent 实例，注入提示词和模型配置。

---

## 二、Agent 创建流程

```
CoworkRunner.startSession()
    │
    ├─► composeEffectiveSystemPrompt()
    │     └─► 组装 5 部分提示词
    │
    ├─► runClaudeCode()
    │     ├─► 判断执行模式
    │     │
    │     └─► runClaudeCodeLocal() 或 runClaudeCodeInSandbox()
    │           │
    │           ├─► getCurrentApiConfig('local' | 'sandbox')
    │           │     └─► 解析 API Key、BaseURL、Model
    │           │
    │           ├─► getEnhancedEnv()
    │           │     └─► 注入环境变量
    │           │
    │           ├─► loadClaudeSdk()
    │           │     └─► 动态加载 SDK
    │           │
    │           └─► query({ prompt, options })
    │                 └─► 启动 Agent 执行
```

---

## 三、提示词注入过程

### 3.1 提示词组装

**方法**: `CoworkRunner.composeEffectiveSystemPrompt()`

**代码位置**: `src/main/libs/coworkRunner.ts:1788`

```typescript
private composeEffectiveSystemPrompt(
  baseSystemPrompt: string,
  workspaceRoot: string,
  cwd: string,
  confirmationMode: 'modal' | 'text',
  userMemoriesXml: string,
  memoryEnabled: boolean
): string {
  const safetyPrompt = this.buildWorkspaceSafetyPrompt(workspaceRoot, cwd, confirmationMode);
  const localTimePrompt = this.buildLocalTimeContextPrompt();
  const memoryRecallPrompt = [...];

  return [safetyPrompt, localTimePrompt, userMemoriesXml, memoryRecallPrompt, baseSystemPrompt]
    .filter((section): section is string => Boolean(section?.trim()))
    .join('\n\n');
}
```

### 3.2 提示词注入

**代码位置**: `src/main/libs/coworkRunner.ts:2468-2469`

```typescript
if (systemPrompt) {
  options.systemPrompt = systemPrompt;
}
```

**注入时机**: 在调用 SDK `query()` 之前

---

## 四、模型配置过程

### 4.1 API 配置解析

**方法**: `resolveCurrentApiConfig()`

**代码位置**: `src/main/libs/claudeSettings.ts:200`

**解析流程**:

```
resolveCurrentApiConfig()
    │
    ├─► 从 SQLite 读取 app_config
    │
    ├─► resolveMatchedProvider()
    │     ├─► 获取 defaultModel
    │     ├─► 查找匹配的 provider
    │     ├─► 处理 Coding Plan 端点
    │     └─► 返回 { providerName, modelId, apiFormat, baseURL }
    │
    └─► 返回 { apiKey, baseURL, model, apiType }
```

### 4.2 环境变量注入

**方法**: `getEnhancedEnv()`

**代码位置**: `src/main/libs/coworkUtil.ts`

**注入的环境变量**:

| 变量名 | 来源 | 说明 |
|-------|------|------|
| `ANTHROPIC_API_KEY` | provider.apiKey | API 密钥 |
| `ANTHROPIC_BASE_URL` | provider.baseUrl | API 端点 |
| `ANTHROPIC_MODEL` | defaultModel | 模型 ID |
| `HTTP_PROXY` / `HTTPS_PROXY` | 系统代理 | 代理配置 |
| `NO_PROXY` | 运行时计算 | 跳过代理的地址 |

**注入代码**: `src/main/libs/coworkRunner.ts:2484-2485`

```typescript
ANTHROPIC_BASE_URL: envVars.ANTHROPIC_BASE_URL,
ANTHROPIC_MODEL: envVars.ANTHROPIC_MODEL,
```

### 4.3 Coding Plan 端点

**特殊端点配置** (`claudeSettings.ts:16-22`):

| 提供商 | Coding Plan 端点 |
|-------|------------------|
| Qwen | `https://coding.dashscope.aliyuncs.com/apps/anthropic` |
| 智谱 | `https://open.bigmodel.cn/api/coding/paas/v4` |
| 火山引擎 | `https://ark.cn-beijing.volces.com/api/coding` |
| Moonshot/Kimi | `https://api.kimi.com/coding` |

**触发条件**: `provider.codingPlanEnabled === true`

**代码位置**: `src/main/libs/claudeSettings.ts:144-170`

```typescript
if (providerName === 'qwen' && providerConfig.codingPlanEnabled) {
  if (apiFormat === 'anthropic') {
    baseURL = QWEN_CODING_PLAN_ANTHROPIC_BASE_URL;
  } else {
    baseURL = QWEN_CODING_PLAN_OPENAI_BASE_URL;
    apiFormat = 'openai';
  }
}
```

---

## 五、SDK 加载过程

### 5.1 SDK 路径解析

**方法**: `loadClaudeSdk()`

**代码位置**: `src/main/libs/claudeSdk.ts`

```typescript
function getClaudeSdkPath(): string {
  if (app.isPackaged) {
    return join(
      process.resourcesPath,
      'app.asar.unpacked',
      'node_modules',
      '@anthropic-ai/claude-agent-sdk',
      'sdk.mjs'
    );
  }
  // 开发环境
  return join(rootDir, 'node_modules/@anthropic-ai/claude-agent-sdk/sdk.mjs');
}
```

### 5.2 SDK 调用

**代码位置**: `src/main/libs/coworkRunner.ts:2528-2530`

```typescript
const { query, createSdkMcpServer, tool } = await loadClaudeSdk();
// ...
const result = await query({ prompt, options });
```

---

## 六、执行模式

### 6.1 Local 模式

**特点**: 直接在本地进程执行

**流程**:
```
runClaudeCodeLocal()
    ├─► 加载 SDK
    ├─► 创建 MCP Server (memory tools)
    ├─► 调用 query({ prompt, options })
    └─► 迭代处理事件流
```

### 6.2 Sandbox 模式

**特点**: 在虚拟机中隔离执行

**流程**:
```
runClaudeCodeInSandbox()
    ├─► 准备沙箱环境
    ├─► spawnCoworkSandboxVm() 启动虚拟机
    ├─► 通过 IPC/文件 与沙箱通信
    └─► 处理沙箱返回的事件
```

---

## 七、配置文件路径汇总

| 配置项 | 文件路径 | 说明 |
|-------|---------|------|
| SDK 路径 | `node_modules/@anthropic-ai/claude-agent-sdk/sdk.mjs` | Agent SDK |
| CLI 路径 | `node_modules/@anthropic-ai/claude-agent-sdk/cli.js` | CLI 入口 |
| 系统提示词 | `sandbox/agent-runner/AGENT_SYSTEM_PROMPT.md` | 基础提示词 |
| 模型配置 | `src/renderer/config.ts` (defaultConfig) | 默认模型列表 |
| API 配置 | SQLite `kv` 表 (`app_config`) | 运行时配置 |

---

## 八、定制要点

### 8.1 修改默认模型

**文件**: `src/renderer/config.ts`

```typescript
model: {
  availableModels: [
    { id: 'your-model-id', name: 'Your Model', supportsImage: true },
  ],
  defaultModel: 'your-model-id',
}
```

### 8.2 新增提供商

**文件**: `src/renderer/config.ts`

```typescript
providers: {
  'your-provider': {
    enabled: false,
    apiKey: '',
    baseUrl: 'https://your-api.example.com',
    apiFormat: 'anthropic',
    models: [
      { id: 'model-1', name: 'Model 1', supportsImage: true },
    ]
  }
}
```

### 8.3 新增 Coding Plan 端点

**文件**: `src/main/libs/claudeSettings.ts`

1. 定义端点常量:
```typescript
const YOUR_PROVIDER_CODING_PLAN_BASE_URL = 'https://your-coding-endpoint.example.com';
```

2. 在 `resolveMatchedProvider()` 中添加处理:
```typescript
if (providerName === 'your-provider' && providerConfig.codingPlanEnabled) {
  baseURL = YOUR_PROVIDER_CODING_PLAN_BASE_URL;
}
```

### 8.4 修改系统提示词

**文件**: `sandbox/agent-runner/AGENT_SYSTEM_PROMPT.md`

直接编辑文件内容，修改 AI 人格、行为规范等。