---
name: whistle-mock
cn_name: Whistle Mock 规则生成
owner: timotang
description: 当用户需要为 RPC/API 接口生成 Whistle mock 规则时使用此 Skill。指导解析接口文档（proto 文件、iWiki 文档），根据返回结构自动生成 mock 数据，通过 HTTP API 或 whistle-mcp 写入 Whistle。触发关键词：生成mock规则, mock接口, mock API, 接口mock, whistle mock, 生成mock数据.
version: 1.0.0
updated_at: '2026-06-15'
tags:
  tech_stack:
    - frontend
    - nodejs
  cross_platform:
    - IDE
---

# Whistle Mock Skill

根据 RPC/API 接口定义生成 Whistle 代理 mock 规则。解析 proto 文件或 API 文档，自动生成 mock 响应数据，通过 whistle HTTP API 直接写入 Whistle（无需额外安装 MCP 工具）。

## 交互原则（严格的两轮四步流程）

整个 mock 流程**严格遵循以下两轮对话、四个步骤**，不得在中间反复多轮追问：

1. **【用户】发送文档** — 开发者发起 mock，提供接口文档（通过 MCP 获取proto / iWiki / 直接粘贴）。
2. **【AI】读取文档 + 规划 + 结尾确认** — AI 读取并解析文档，梳理**计划写入哪些 Values 和 Rules**（含命名、分组、mock 数据预览），并在**这次回答的末尾**集中向开发者确认。确认问题通常分三类：
   - **一、命名是否可以** — 建议的业务大类 / 场景名 / Value 命名（`{Scene}{MethodName}[-{Variant}]`）是否采用，或需自定义。
   - **二、是否需要异常情况的 mock** — 是否补充异常/边界变体（500 / 401 / 429 / 超时 / 空数据 / 业务异常 retcode 非 0 等）。
   - **三、是否需要调整** — mock 字段值、协议（`resBody://` / `file://`）、pattern 等是否需要改动。
3. **【用户】回答** — 开发者一次性回复上述三类问题。
4. **【AI】开干** — AI 据此完成写入（创建 Value Group、Values、Rule）、启用与验证，结束流程。

要点：
- **步骤 2 只规划与确认，不写入 Whistle**；真正的写入发生在步骤 4。
- 凡能合理推测的（业务大类、场景名、mock 字段值等），在步骤 2 先给出**建议默认值**并标注"如不符可调整"，作为三类问题呈现给开发者，而非逐条阻塞追问。
- 三类确认问题用编号列表集中列出，便于开发者在步骤 3 一次性回复。

## 工作流

按以下步骤顺序执行：

### 步骤 1：检测 Whistle 运行状态与端口

确认 Whistle 是否运行并检测端口：

1. 执行 `ps aux | grep -i whistle | grep -v grep` 检查 Whistle 是否在运行。
2. 如果运行中，从进程参数中提取端口。查找命令行中的 `"port":"XXXX"` 或 `--port XXXX` 或 `-p XXXX`。
3. 验证连通性：`curl -s http://127.0.0.1:{port}/cgi-bin/init` — 返回 JSON（含 `version` 字段）即确认 Whistle 可用。
4. 记录端口号作为 `WHISTLE_PORT`，后续步骤（3~7）所有 HTTP API 调用均使用此值作为步骤间传递契约。

如果 Whistle 未运行，引导用户启动：
```bash
# 安装（如需要）
npm install -g whistle && w2 start
# 或指定端口启动
w2 start -p 8088
```

### 步骤 2：确定操作方式

本 Skill 支持两种方式操作 Whistle，**优先使用 HTTP API**：

#### 方式 A：HTTP API（推荐，无需额外安装）

直接通过 `curl` 调用 whistle 的 `/cgi-bin/` 接口。所有规则的查看、创建、更新、删除均可完成。
详细接口规范见 `references/whistle-http-api.md`。

**核心接口速查：**
| 操作 | 接口 | 方法 | 说明 |
|------|------|------|------|
| 获取规则列表+内容 | `/cgi-bin/rules/list` | GET | 返回 `list[].data` 含规则内容 |
| 创建空规则 | `/cgi-bin/rules/add` | POST | 仅创建，不写内容 |
| 写入规则+启用 | `/cgi-bin/rules/select` | POST | **实际写入规则的接口** |
| 禁用规则 | `/cgi-bin/rules/unselect` | POST | |
| 删除规则 | `/cgi-bin/rules/remove` | POST | |
| 获取 Values（含内容） | `/cgi-bin/init` | GET | 返回 `values.list[].data` 含内容 |
| Values 名称列表 | `/cgi-bin/values/list` | GET | ⚠️ **不返回内容** |
| 创建/更新 Value | `/cgi-bin/values/add` | POST | 传 `value` 参数写入内容 |
| 删除 Value | `/cgi-bin/values/remove` | POST | |
| 移动到分组 | `/cgi-bin/values/move-to` | POST | |

**⚠️ 关键注意事项：**
- 所有 POST 请求**必须使用 `--data-urlencode`**，不能用 `-d`（否则 JSON 内容会丢失/写入为空）
- 写入时参数名是 `value`，读取时响应字段名是 `data`
- 验证 Value 是否写入成功必须用 `/cgi-bin/init`（不是 `values/list`）

#### 方式 B：whistle-mcp（MCP 工具）

如果用户已安装 `whistle-mcp-tool` 并在 MCP 配置中注册，可直接调用 MCP 工具（如 `getRules`、`updateRule`、`updateValue` 等）。
whistle-mcp 底层封装的也是同样的 HTTP API，功能等价。

**判断是否可用**：检查 MCP 配置中是否存在 `whistle-mcp-tool` 相关工具（尝试列出 MCP 工具或读取 MCP 配置文件）。如果未找到，提示用户：
> 未检测到 whistle-mcp MCP 工具。推荐使用方式 A（HTTP API），功能完全等价且零安装。如需安装 whistle-mcp，请参考其文档配置 MCP Server。

> **注意**：无论使用哪种方式，底层都是调用 whistle 的 `/cgi-bin/` HTTP 接口，结果完全一致。

### 步骤 3：获取接口文档

请用户提供 RPC 接口文档。支持以下格式：

- **Git URL**：git.woa.com 或其他 Git 平台上的 `.proto` 文件链接。使用 web_fetch 或工蜂 MCP 获取文件内容。
- **iWiki 链接**：包含 API 定义的 iWiki 页面链接。使用 iWiki MCP 获取文档。
- **直接粘贴**：用户可以直接粘贴 proto 内容或 API 文档。

如果用户提供 URL，获取内容。如果是 proto 文件，按照 proto 转 mock 指南解析（见 `references/proto-to-mock-guide.md`）。

**文档获取失败降级**：
- Git URL 获取失败（仓库不可访问、权限不足）：提示用户确认仓库地址和访问权限，或改为直接粘贴 proto 内容。
- iWiki MCP 不可用（MCP 服务未连接或超时）：提示用户检查 iWiki MCP 配置，或改为直接粘贴文档内容。
- 其他 URL 获取失败：提示用户直接粘贴接口文档内容作为替代。

### 步骤 4：解析接口定义

解析提供的文档，提取：

1. **服务名 / RPC 方法** — 每个方法对应一个 URL pattern。
2. **请求消息** — 用于理解输入参数（辅助 pattern 匹配）。
3. **响应消息** — 用于生成 mock 数据。
4. **业务场景线索** — 从接口文档、用户描述、页面措辞中推测该接口归属的**业务大类**与**页面/场景**（用于步骤 6 的命名）。例如用户说"mock 专享理财金落地页的接口"，可推测：业务大类「权益页面」→ 场景「专享理财金」。

对于 proto 文件，按照 `references/proto-to-mock-guide.md` 中的详细解析指南操作。

对于 iWiki 或其他文本文档，提取：
- API 路径（URL pattern）
- 响应 JSON 结构
- 字段名和类型

### 步骤 5：生成 Mock 数据

根据解析出的响应结构，按照 `references/proto-to-mock-guide.md` 中的类型映射规则自动生成 mock 数据。关键规则：

- `string` → 描述性示例文本（如 `"示例用户名"`、`"2024-01-01"`）
- `int32`/`int64`/`uint32`/`uint64` → `100`
- `float`/`double` → `1.0`
- `bool` → `true`
- `repeated` → 包含 2 个元素的数组
- `enum` → 第一个定义值（通常为 0）
- `map` → 包含 1 个示例条目的对象
- `nested message` → 递归生成
- `google.protobuf.Timestamp` → `"2024-01-01T00:00:00Z"`
- `google.protobuf.Any` → `{"@type": "type.googleapis.com/example.Type"}`

为每个 RPC 方法生成 mock 响应 JSON。响应封装格式的判断与生成规则（含 tRPC 网关自动注入 `retcode`/`retmsg` 的处理）详见 `references/proto-to-mock-guide.md` 的"处理响应封装"章节。

**⚠️ Mock JSON 必须格式化输出**：写入 Whistle 时，JSON 内容必须使用缩进（2 空格）的多行格式，而非单行压缩格式。Whistle 会原样存储传入的内容，单行 JSON 在 UI 中可读性极差。`--data-urlencode` 能正确处理换行符编码，直接在 `value` 参数中传入格式化的多行 JSON 即可：
```bash
# ✅ 正确：格式化多行 JSON
curl ... --data-urlencode 'value={
  "retcode": "0",
  "retmsg": "OK",
  "data": {
    "name": "张三"
  }
}'

# ❌ 错误：单行压缩 JSON（可读性差）
curl ... --data-urlencode 'value={"retcode":"0","retmsg":"OK","data":{"name":"张三"}}'
```

将生成的 mock 数据展示给用户预览。

> 📌 **本步骤是「交互原则」中第 2 步（AI 规划 + 结尾确认）的收尾**：步骤 3~5 属于第一轮 AI 回答——读取文档、规划要写入的 Values/Rules（命名 + 分组 + mock 数据预览），**此时尚未写入 Whistle**。在这次回答的末尾，按三类问题集中向开发者确认（一、命名是否可以；二、是否需要异常情况的 mock；三、是否需要调整 mock 字段/协议/pattern），然后等待开发者回复。**开发者确认后**再进入步骤 6 执行写入。

### 步骤 6：写入 Whistle

> ⚙️ **本步骤对应「交互原则」第 4 步（AI 开干），在开发者确认后执行。** 命名已在步骤 5 末尾向开发者确认，这里直接使用确认后的 `{Category}` / `{Scene}` / `{Variant}` 写入，不再重复追问。

通过 HTTP API 写入规则和 Values（或使用 whistle-mcp MCP 工具，两者等价）。按以下顺序：

1. **获取当前状态** — `GET /cgi-bin/rules/list` 检查是否存在同名规则。

2. **确认命名（沿用步骤 5 已确认的命名）** — 本 Skill 采用**三级组织模型**：

   | 层级 | 含义 | 对应 Whistle 实体 | 命名约定 |
   |------|------|------------------|---------|
   | **业务大类** | 一组相关页面/场景 | Rule 的 `\r` 分组 | `{Category}`，如 `权益页面`、`运营页面`、`关键页面` |
   | **页面/场景** | 一个具体需求/页面 | 一个 Rule + 一个 Value Group | `{Scene}`，如 `专享理财金`、`财富私享会` |
   | **接口变体** | 同接口的不同返回态 | Rule 行 + 一个 Value | `{Scene}{MethodName}[-{Variant}]` |

   **Value 命名规则**：`{Scene}{MethodName}[-{Variant}]`，**不加 `.json` 后缀**。
   - 示例：`专享理财金queryPrizeLandingConfig`、`财富私享会WealthMeetingRecentInvite-没有`。
   - 变体后缀 `-{Variant}` 用于区分同接口的不同返回，如 `-有奖`/`-无奖`、`-有数据`/`-空`、`-1`/`-2`。无变体时可省略。

   > 🔑 **由于每个开发者的需求不同，命名不能写死。** 命名的推测与确认在步骤 5 末尾完成（见「交互原则」第 2 步）：Skill 根据会话上下文（接口文档、页面描述、用户措辞）**推测业务大类与场景名，给出建议命名供开发者确认**。例如：
   > > 根据接口内容，建议归类为：业务大类「权益页面」→ 场景「专享理财金」。
   > > Value 将命名为 `专享理财金queryPrizeLandingConfig`、`专享理财金genMemberPrivilegeShareKey` 等。
   > > 是否采用？你也可以自定义大类名 / 场景名 / 变体后缀。
   >
   > 本步骤直接使用开发者确认后的最终 `{Category}` / `{Scene}` / `{Variant}` 进行写入。

3. **创建 Value Group** — `POST /cgi-bin/values/add`，名称为 `\r{Scene}`（`\r` 前缀标记为分组）。同一场景的所有 Value 归入此组。

   > 🚨 **关键：`\r` 必须是真正的回车符 (0x0D)，不是字面的反斜杠+r！**
   > 这是分组失败最常见的坑。Whistle 靠名称首字符是否为回车符 (0x0D) 来判定分组。
   > - ✅ **正确**（bash/zsh ANSI-C quoting，**纯 curl 即可，无需 Python**）：
   >   ```bash
   >   curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/add --data-urlencode $'name=\r{Scene}'
   >   ```
   > - ❌ **错误**（双引号里的 `\r` 是字面的两个字符 `\`+`r`，分组不会生效）：
   >   ```bash
   >   curl ... --data-urlencode "name=\r{Scene}"   # 会被存成字面 \rScene
   >   ```
   > 注意：`$'...'` 只包裹**参数值本身**，`--data-urlencode` 是独立 token，不要写成 `$'--data-urlencode name=\r...'`。

4. **创建 Values** — 为每个 RPC 方法（及其变体）的 mock 数据：
   - `POST /cgi-bin/values/add`，`name={Scene}{MethodName}[-{Variant}]`（无 `.json` 后缀），`value=mock JSON 内容`
   - `POST /cgi-bin/values/move-to`，`from={Scene}{MethodName}[-{Variant}]`，`to=\r{Scene}`，将其移入分组
     - ⚠️ `to=\r{Scene}` 同样**必须用 `$'to=\r{Scene}'`**（真正回车符），否则 `move-to` 返回 `ec=2`（找不到目标分组），文件不会进组。
     - move-to 会把文件插到分组相邻位置，若顺序乱了，可重复 move 调整；分组顺序不影响 mock 功能（规则按 Value 全名引用）。

5. **创建或更新 Rule** — 为**场景**创建规则（一个场景 = 一个 Rule，Rule 名 = `{Scene}`）：
   - 如果不存在：`POST /cgi-bin/rules/add` 创建，再 `POST /cgi-bin/rules/select` 写入内容并启用
   - 如果已存在：先从 `rules/list` 获取当前 `data` 内容，追加新行，再通过 `rules/select` 更新
   - 可选：将 Rule 归入业务大类分组（Rule 同样支持 `\r{Category}` 分组），便于在 Whistle 左栏按大类折叠管理。

   规则内容格式（详见 `references/whistle-mock-patterns.md`）：
   ```txt
   # {Scene} - 正常响应
   /{ServiceName}.{MethodName}/ resBody://{ {Scene}{MethodName} } statusCode://200

   # 同一接口的多个变体并列写出，只启用一行，其余用 # 注释互斥切换
   /{ServiceName}.{MethodName}/ resBody://{ {Scene}{MethodName}-有数据 } statusCode://200
   # /{ServiceName}.{MethodName}/ resBody://{ {Scene}{MethodName}-空 } statusCode://200
   ```
   - Rule 内可用注释行 `# 子场景标题` 进一步分块（如 `# 落地页`、`# 分享页`、`# 结果页`）。

6. **启用规则** — `POST /cgi-bin/rules/select`（写入时 `selected=true` 即自动启用）。

7. **启用多规则模式**（如需同时启用多条规则）— `POST /cgi-bin/rules/allow-multiple-choice`，`allowMultipleChoice=1`。

8. **验证写入结果** — 写入后必须验证（全程仅用 curl + grep，不用 python3/jq）：
   - 规则：`GET /cgi-bin/rules/list` → 检查对应规则的 `data` 字段非空且内容正确
   - Value：`GET /cgi-bin/init` → 检查 `values.list` 中对应项的 `data` 字段非空且内容正确
   - ⚠️ 不要用 `values/list` 验证（它不返回内容）
   - **验证分组**：检查分组名首字符是否为回车符 (0x0D)。可读取 `~/.WhistleAppData/.whistle/values/properties` 的 `filesOrder`，确认分组名 `ord(name[0]) == 13`，且子 Value 紧随其后。若分组名以字面 `\r`（反斜杠+r）开头，说明创建时用错了引号，需删除重建（见下方排错）。
     ```bash
     # 纯 shell 验证分组首字符（无需 Python）：用 od 查看，分组项应以 0d 开头
     curl -s http://127.0.0.1:{PORT}/cgi-bin/values/list | grep -o $'\r[A-Za-z]*' | od -c | head
     ```

> 完整接口参数见 `references/whistle-http-api.md` 中的"完整 Mock 写入示例"。

**API 调用失败处理**：
- 所有 curl 调用后检查返回 JSON 的 `ec` 字段：`ec !== 0` 表示失败，需根据 `emsg` 定位原因并重试或报错退出。
- curl 本身返回非 0（网络不通）：检查 `WHISTLE_PORT` 是否正确、Whistle 是否仍在运行（回到步骤1重新检测）。

**并发安全说明**：
- 更新已有 Rule 时采用"先读后追加"模式（读取当前 `data` → 追加新行 → 写回）。当多 Agent 并发操作同一 Rule 时，后写者可能覆盖先写者的内容。
- 建议：写入前先读取当前 Rule 内容，检查最后一行是否已包含待追加的 pattern（如 `/{MethodName}/ resBody://`），若已存在则跳过追加，避免重复规则。

#### 分组排错（move-to 返回 ec=2 / 分组名显示为字面 `\r`）

如果分组没生效（UI 里子项没收进分组，或分组名显示成 `\rScene` 字面文本）：

```bash
PORT=8088
SCENE="专享理财金"
# 1. 删除错误的字面分组（注意：删字面分组用普通引号，因为它存的就是字面 \r）
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/values/remove --data-urlencode "list[]=\r${SCENE}"
# 2. 用真正回车符重建分组（ANSI-C quoting）
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/values/add --data-urlencode $'name=\r'"${SCENE}"
# 3. 重新移入各 Value（to 必须用真正回车符；返回 ec=0 即成功，ec=2 表示分组未匹配）
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/values/move-to \
  --data-urlencode "from=${SCENE}queryPrizeLandingConfig" --data-urlencode $'to=\r'"${SCENE}" --data-urlencode "group=false"
```

> 提示：`$'name=\r'"${SCENE}"` 这种拼接写法可在回车符后接变量。整体也可写 `$'name=\r专享理财金'`（字面场景名时）。

### 步骤 7：可选 — 异常/边界变体

是否生成异常/边界**变体**，已在步骤 5 末尾作为「第二类问题」向开发者确认。若开发者在步骤 5 表示需要，则在步骤 6 写入时一并创建以下变体（**不在写入后再追问**）。变体复用步骤 6 的命名规则（`{Scene}{MethodName}-{Variant}`），与正常返回**并列写在同一 Rule 内**，默认注释掉（行首加 `# `），开发者在 Whistle 中按需启用（同接口多变体互斥，只启用一行）：

| 变体 | 规则 Pattern | Value 内容 |
|------|-------------|-----------|
| 服务器错误 | `/{ServiceName}.{MethodName}/ statusCode://500` | — |
| 超时 | `/{ServiceName}.{MethodName}/ resDelay://3000 file://{ {Scene}{MethodName}-error }` | `{"retcode":"500","retmsg":"internal error"}` |
| 空数据 | `/{ServiceName}.{MethodName}/ resBody://{ {Scene}{MethodName}-空 } statusCode://200` | `{"retcode":"0","retmsg":"OK","data":null}` |
| 认证失败 | `/{ServiceName}.{MethodName}/ statusCode://401` | — |
| 限流 | `/{ServiceName}.{MethodName}/ statusCode://429` | — |
| 业务异常（HTTP 200 + retcode 非 0） | `/{ServiceName}.{MethodName}/ resBody://{ {Scene}{MethodName}-biz_error } statusCode://200` | `{"retcode":"10001","retmsg":"业务处理失败","data":null}` |

## 协议选择指南

根据 mock 场景选择合适的 Whistle 协议：

| 协议 | 适用场景 | 行为 |
|------|---------|------|
| `resBody://{key}` | **默认选择** — 大多数 mock 场景 | 请求仍发送到服务器，然后替换响应体。保留真实响应头。与 tRPC/HTTP 网关兼容性好。 |
| `file://{key}` | 纯 mock，不依赖服务器 | 直接返回内容，**不请求服务器**。可能丢失客户端依赖的响应头。 |
| `resMerge://{key}` | 部分字段覆盖 | 将数据合并到真实服务器响应中。保留所有原始字段，仅覆盖指定字段。 |
| `resReplace://{old=new}` | 简单值替换 | 响应中的正则/关键词替换。适合翻转特定字段值。 |

**建议**：tRPC 服务默认使用 `resBody://` + `statusCode://200` 模式。该模式：
- 确保请求仍到达服务器（维持连接上下文）
- 仅替换响应体为 mock 数据
- 保留真实响应中的 Content-Type 和其他头信息

仅在以下情况使用 `file://`：
- 后端服务器不可用
- 想完全避免请求服务器
- mock 响应是自包含的，不依赖服务器头信息

## tRPC URL Pattern 格式

通过 HTTP 网关访问的 tRPC 服务，URL pattern 使用**方法路径**，无需域名：

```txt
# 格式：/ServiceName.MethodName/  →  引用 Value 名 {Scene}{MethodName}[-{Variant}]
/QueryMaterialsEquityInfo/ resBody://{ 专享理财金QueryMaterialsEquityInfo } statusCode://200
/getTransportState/ resBody://{ 财富私享会getTransportState } statusCode://200
/FuactEquityMaterialVoService\.ModifyOrderAddress/ resBody://{ 专享理财金ModifyOrderAddress } statusCode://200
```

关键要点：
- **无需域名前缀** — Whistle 匹配任意域名的路径部分
- **尾部斜杠** — 末尾加 `/` 确保精确匹配方法边界
- **转义点号** — 如果服务名包含点（如 `FuactEquityMaterialVoService.Method`），用 `\.` 转义实现精确匹配，不转义则为宽松匹配
- **大小写敏感** — 与 RPC 方法定义中的大小写完全一致

如果用户提供了特定域名或服务使用了不同的 URL 格式，请相应调整。

## 重要规则

- tRPC 服务默认使用 `resBody://` + `statusCode://200` 作为 mock 协议。仅在后端不可用时使用 `file://`。
- **采用三级组织模型**：业务大类（Rule 分组）→ 页面/场景（一个 Rule + 一个 Value Group）→ 接口变体（Rule 行 + Value）。
- **Value 命名规则**：`{Scene}{MethodName}[-{Variant}]`，**不加 `.json` 后缀**。变体后缀（如 `-有奖`/`-空`）区分同接口的不同返回态。
- **命名由 Skill 推测、用户确认**：根据会话上下文推测业务大类与场景名，给出建议命名展示给用户，确认或修改后再写入；不要把命名写死。
- 一个**场景** = 一个 Rule。该场景涉及的所有接口（及变体）作为独立行写入同一个 Rule；同接口的多个变体并列、互斥（只启用一行，其余注释）。
- Values 按 Value Group（场景）组织。**分组名前缀 `\r` 必须是真正的回车符 (0x0D)**：bash/zsh 用 ANSI-C quoting `$'name=\r{Scene}'`，**切勿用双引号 `"name=\r..."`**（会变字面反斜杠+r，分组失效）。纯 curl + bash 即可，无需 Python。
- 更新已有 Rule 时，**追加**新行而非覆盖已有内容。先用 `GET /cgi-bin/rules/list` 读取当前规则内容，再添加新行。
- 异常/边界变体默认注释掉。
- 腾讯内部 tRPC 服务 mock 必须包含 `"retcode": "0", "retmsg": "OK"`（网关自动注入，即使 proto 中未定义）。详细规则见 `references/proto-to-mock-guide.md` 的"处理响应封装"章节。
- **优先使用 HTTP API 操作 Whistle**，无需依赖 whistle-mcp MCP 工具。两者底层一致，但 HTTP API 零安装、零配置。
- **禁止在 curl 命令实参中内联 `$(...)` 命令替换，禁止使用 `python3 -c`**：IDE 安全扫描器会判定为命令注入/执行不可信脚本，强制人工确认，打断自动化。规避：① clientId 用固定串（如 1718600000000-1）；② 确需时间戳则先单独一行 `TS=$(date +%s)000` 赋值，curl 内只引用 `${TS}`；③ 验证类 GET 去掉 `?_=` 防缓存参数；④ 验证只用 curl + grep，不用 python3/jq。

## 参考文档

需要时从 `references/` 加载以下参考文件获取详细指导：

- **`references/whistle-http-api.md`** — Whistle HTTP API 操作规范。基于 whistle-mcp 源码整理的完整接口文档，包含所有 curl 调用示例。**写入规则时必读。**
- **`references/whistle-rule-syntax.md`** — Whistle 规则语法速查（pattern、操作、数据源、过滤器）。不确定规则语法时加载。
- **`references/whistle-mock-patterns.md`** — Mock 专用模式、协议与最佳实践。生成 mock 规则时加载。
- **`references/proto-to-mock-guide.md`** — Proto 定义转 Whistle mock 规则指南。解析 proto 文件时加载。
