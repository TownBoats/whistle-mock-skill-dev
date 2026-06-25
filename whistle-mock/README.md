# whistle-mock — Whistle Mock 规则生成

根据 RPC/API 接口定义，指导 AI 生成 Whistle 代理 mock 规则并写入本地 Whistle 实例，把一套跑通的前端联调 mock 流程规范化、可复用。

## 功能介绍

| 能力 | 说明 |
|------|------|
| **接口文档输入** | 支持 Git URL / iWiki 链接 / 直接粘贴；读取依赖对应 MCP 或用户粘贴 |
| **Mock 生成规范** | mock JSON 由 AI 生成；Skill 提供字段映射、多行格式、tRPC 封装、业务变体等约定 |
| **Whistle 写入流程** | 按固定 SOP 创建 Rule、Value Group、Value，写入后自动验证 |
| **业务/异常变体** | 支持业务态，也可补充 500/401/429/超时/业务异常等 error 变体 |
| **tRPC 兼容** | 约束 `retcode`/`retmsg` 封装、URL pattern、`resBody://` 等 tRPC mock 细节 |

> 定位说明：`whistle-mock` 的核心价值不是给 AI 新增底层能力，而是把一套跑通的 Whistle mock 流程沉淀成 SOP，减少重复说明和对话轮次。

## 系统要求

- **Whistle** — 已安装并运行（`npm install -g whistle && w2 start`）
- **Shell** — bash 或 zsh（需要 ANSI-C quoting `$'...'` 支持来创建 Value Group）
- **curl** — 用于调用 Whistle HTTP API
- **AI 编码助手** — 在 CodeBuddy / Claude Code 等 IDE 插件中使用

## 使用步骤

### 1. 确认 Whistle 已运行

```bash
# 安装（如未安装）
npm install -g whistle

# 启动（默认端口 8899）
w2 start

# 或指定端口
w2 start -p 8088
```

### 2. 在 AI 对话中触发 Skill

在 IDE 的 AI 对话框中，使用以下任一方式触发：

- "帮我生成这个接口的 mock 规则"
- "mock 这个 RPC 接口"
- "生成 whistle mock 数据"
- 提供 proto 文件路径或 Git/iWiki 链接，要求 mock

### 3. 提供接口文档

接口文档可以通过三种方式提供。Git/iWiki 内容读取依赖当前 AI 环境中的工蜂 MCP / iWiki MCP；无法读取时，直接粘贴文档即可。

| 来源 | 示例 | 说明 |
|------|------|------|
| **Git URL** | `https://git.woa.com/xxx/service.proto` | 适合 proto 文件 |
| **iWiki 链接** | iWiki 页面中的 API 定义 | 适合接口说明文档 |
| **直接粘贴** | 在对话中粘贴 proto 内容或 API 文档 | 兜底方式 |

### 4. 确认命名规划

Skill 会先输出一份**命名规划清单**（业务大类、场景名、每个接口的 Value 名称、可选变体），**不生成 mock JSON**。你需要一次性确认以下四类问题：

| 确认项 | 说明 |
|--------|------|
| **一、命名** | 建议的业务大类 / 场景名 / Value 命名是否可用 |
| **二、业务变体** | 基于接口 message 字段语义推测的业务态变体（如订单→已下单/未下单），**会结合 message 内容针对性建议**，不套固定模板 |
| **三、error 变体** | 是否需要 500/401/429/超时/业务异常 retcode≠0 等协议层异常态 |
| **四、调整** | 是否需要调整 pattern、协议（`resBody://` / `file://`）等 |

> 能合理推测的项一律先给建议默认值并标注"如不符可调整"，而非逐条追问。若用户已表达"直接写入/按默认值"等推进意图，Skill 会跳过等待直接写入。

### 5. 自动写入 Whistle

确认后，AI 会按 Skill 固化的规则生成 mock JSON 并写入：

1. 检测 Whistle 端口
2. 创建 Value Group（按场景分组）
3. 为每个 RPC 方法（及变体）创建 mock Value
4. 创建/更新 Whistle Rule（一个场景 = 一个 Rule）并启用
5. 验证 Rule 和 Value 已正确写入

执行细节（如 `rules/add` / `rules/select`、`\r` 分组、花括号空格等坑点）已沉淀在 `SKILL.md` 和 `references/` 中，README 只保留使用路径。

### 6. 可选：生成异常/边界变体

写入完成后，可以要求生成以下 error 变体（与正常返回并列写入同一 Rule，默认 `#` 注释，不影响正常开发）：

- 服务器错误 (500)
- 超时 (3s delay)
- 认证失败 (401)
- 限流 (429)
- 业务异常 (HTTP 200 + retcode 非 0)

## 支持的 Whistle 操作方式

| 方式 | 说明 | 推荐度 |
|------|------|--------|
| **HTTP API** | 直接 curl 调用 Whistle 的 `/cgi-bin/` 接口，零安装 | ⭐ 推荐 |
| **whistle-mcp** | 通过 MCP 工具操作，需额外安装配置 | 可选 |

两种方式底层一致，结果完全相同。

## 协议选择指南

| 协议 | 适用场景 |
|------|---------|
| `resBody://` | **默认** — 请求仍发到服务器，替换响应体；与 tRPC 网关兼容好 |
| `file://` | 后端不可用、纯 mock 不依赖服务器 |
| `resMerge://` | 仅覆盖部分字段，保留其他真实响应 |
| `resReplace://` | 简单的值替换 |

## Mock 数据生成约定

AI 根据接口结构生成 mock JSON，Skill 用以下默认约定保证结果稳定、一致、可读。

| Proto 类型 | Mock 值 |
|-----------|---------|
| `string` | `"示例用户名"` / `"2024-01-01"` 等描述性文本 |
| `int32` / `int64` / `uint32` / `uint64` | `100` |
| `float` / `double` | `1.0` |
| `bool` | `true` |
| `repeated` | 包含 2 个元素的数组 |
| `enum` | 第一个定义值（通常为 0） |
| `map` | 包含 1 个示例条目的对象 |
| `google.protobuf.Timestamp` | `"2024-01-01T00:00:00Z"` |

## 目录结构

```
x-whistle-mock/
├── SKILL.md                          # Skill 核心指令
├── README.md                         # 本文件
└── references/                       # 参考文档
    ├── whistle-http-api.md           # Whistle HTTP API 完整接口规范
    ├── whistle-rule-syntax.md        # Whistle 规则语法速查
    ├── whistle-mock-patterns.md      # Mock 专用模式与最佳实践
    └── proto-to-mock-guide.md        # Proto 转 Mock 详细指南
```

## 注意事项

1. **一个场景 = 一个 Rule + 一个 Value Group**，同接口的不同返回态作为多个 Value 并列管理。
2. **命名先建议、再确认**：Skill 会推测业务大类、场景名和 Value 名，确认后再写入。
3. **默认 tRPC 友好**：优先使用 `resBody://`，并按需补齐 `retcode` / `retmsg`。
4. **写入后会验证**：Skill 会检查 Rule 和 Value 是否真正写入，避免只创建空壳规则。
5. **详细坑点不放在 README**：`rules/add`、`\r` 分组、花括号空格、POST 编码等细节见 `SKILL.md` 和 `references/`。

## 常见问题

<details>
<summary>Whistle 未运行怎么办？</summary>

手动启动：

```bash
w2 start        # 默认端口 8899
w2 start -p 8088  # 指定端口
```
</details>

<details>
<summary>proto / iWiki 内容读取失败？</summary>

通常是 MCP 权限或网络问题，直接把 proto/API 文档粘贴到对话中即可继续。
</details>

<details>
<summary>想测试异常或边界场景？</summary>

让 Skill 补充业务变体或 error 变体即可，例如空数据、业务异常、500、401、429、超时等。
</details>

<details>
<summary>只想 mock 部分字段？</summary>

使用 `resMerge://` 协议，仅覆盖指定字段，保留其他真实响应数据。
</details>
