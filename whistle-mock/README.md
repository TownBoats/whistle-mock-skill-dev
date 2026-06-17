# whistle-mock — Whistle Mock 规则生成

根据 RPC/API 接口定义，自动生成 Whistle 代理 mock 规则并写入本地 Whistle 实例，一键完成前端开发联调时的接口 mock。

## 功能介绍

| 能力 | 说明 |
|------|------|
| **接口文档解析** | 支持从 proto 文件、iWiki 文档、直接粘贴三种方式获取接口定义 |
| **Mock 数据自动生成** | 根据字段类型自动映射生成合理的 mock 值（string → 示例文本、int → 100、repeated → 2 元素数组等） |
| **Whistle 规则写入** | 通过 HTTP API 直接写入 Whistle，自动创建 Rule、Value Group、Value 并启用 |
| **异常场景 Mock** | 可选生成 500/401/429/超时/空数据等异常场景规则（默认注释） |
| **tRPC 兼容** | 自动处理 tRPC 网关注入的 `retcode`/`retmsg` 封装 |

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

Skill 支持三种文档来源：

| 来源 | 示例 |
|------|------|
| **Git URL** | `https://git.woa.com/xxx/service.proto` |
| **iWiki 链接** | iWiki 页面中的 API 定义 |
| **直接粘贴** | 在对话中粘贴 proto 内容或 API 文档 |

### 4. 确认 Mock 数据

Skill 解析接口定义后会自动生成 mock 响应 JSON，展示给你预览。你可以在写入前调整特定字段的值。

### 5. 自动写入 Whistle

确认后 Skill 会自动：

1. 检测 Whistle 端口
2. 创建 Value Group（按服务名分组）
3. 为每个 RPC 方法创建 mock JSON Value
4. 创建/更新 Whistle Rule 并启用
5. 验证写入结果

### 6. 可选：生成异常场景

写入完成后，可以要求生成以下异常场景（默认注释，不影响正常开发）：

- 服务器错误 (500)
- 超时 (3s delay)
- 空数据 (data: null)
- 认证失败 (401)
- 限流 (429)

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

## Mock 数据类型映射

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

1. **Value Group 分组名**：`\r` 前缀必须是真正的回车符 (0x0D)，不能用字面 `\r`。Skill 会自动处理，手动操作时需用 `$'name=\rServiceName'` 语法。
2. **tRPC 服务**：mock 响应必须包含 `"retcode": "0", "retmsg": "OK"`（网关自动注入的字段）。
3. **追加不覆盖**：更新已有 Rule 时追加新行，不会覆盖已有 mock 规则。
4. **并发安全**：多 Agent 同时操作同一 Rule 时可能覆盖，Skill 会检查是否已存在再决定是否追加。
5. **验证方式**：写入后用 `/cgi-bin/init` 验证 Value 内容（`/cgi-bin/values/list` 不返回内容）。
6. **POST 编码**：所有 curl POST 必须用 `--data-urlencode`，不能用 `-d`（否则 JSON 内容会丢失）。

## 常见问题

<details>
<summary>Whistle 未运行怎么办？</summary>

Skill 会自动检测并提示启动命令。手动启动：

```bash
w2 start        # 默认端口 8899
w2 start -p 8088  # 指定端口
```
</details>

<details>
<summary>Value Group 没生效（UI 里没分组）？</summary>

通常是分组名首字符不是真正的回车符。Skill 使用 ANSI-C quoting 保证正确性。手动修复见 SKILL.md 中的"分组排错"章节。
</details>

<details>
<summary>proto 文件获取失败？</summary>

如果是 Git 权限问题，可直接将 proto 内容粘贴到对话中，Skill 同样能解析。
</details>

<details>
<summary>只想 mock 部分字段？</summary>

使用 `resMerge://` 协议代替 `resBody://`，仅覆盖指定字段，保留其他真实响应数据。
</details>
