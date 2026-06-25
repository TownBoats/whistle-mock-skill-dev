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

根据 RPC/API 接口定义生成 Whistle 代理 mock 规则。解析 proto 文件或 API 文档，自动生成 mock 响应数据，通过 whistle HTTP API（`curl` 调 `/cgi-bin/`，无需安装 MCP）直接写入 Whistle。

详细接口、curl 示例、排错等见 `references/`（见末尾「参考文档」）。

## 交互原则（严格两轮四步，禁止中途反复追问）

1. **【用户】发送文档** — 提供接口文档（Git/iWiki 链接或直接粘贴 proto）。
2. **【AI】解析 + 规划清单 + 结尾确认** — 解析文档，**只产出计划写入的 Rules 和 Values 名称清单**（含建议的业务大类、场景名、每个接口对应的 Value 名 + 业务变体名），**不产出 mock JSON 内容**，在回答末尾用编号列表集中确认下列问题：
   - **一、命名是否可以** — 建议的业务大类 / 场景名 / Value 命名（`{Scene}{MethodName}[-{Variant}]`）。
   - **二、是否需要业务变体（基于接口 message 的不同业务态）** — 根据接口响应 message 的实际字段语义推测可能的业务态变体（如订单接口 → `-已下单`/`-未下单`/`-已退款`；权益接口 → `-有奖`/`-无奖`；列表接口 → `-有数据`/`-空`）。**禁止套用刻板模板**，必须结合接口 message 内容给出针对性建议；无明显业务态时不强行添加。
   - **三、是否需要 error 变体（与 message 无关的协议/异常态）** — 是否补充 500/401/429/超时/业务异常 retcode≠0 等错误态变体（与具体业务无关，按需选用）。
   - **四、是否需要调整命名 / pattern / 协议** — 例如 Value 名想自定义、URL pattern 需限定域名、协议想用 `file://` 等。
3. **【用户】回答** — 一次性回复上述四类问题。
4. **【AI】开干** — 据确认结果**才开始生成 mock JSON 内容**并写入、启用、验证，结束流程。

> 能合理推测的项（大类、场景名、业务变体）一律先给**建议默认值**并标注"如不符可调整"，作为对应类别的问题呈现，而非逐条阻塞追问。
>
> **业务变体 ≠ error 变体**：业务变体来自 message 字段的合法业务态，error 变体来自协议/系统层异常；二者分开规划、分开确认，不要混为一谈。
>
> **🛟 跳过等待的两种情况**（避免在单轮 / 自动化场景被卡死）：
> 1. **用户已隐式同意**：若初始消息已包含「直接写入 / 按默认值 / 一把梭 / 不用问 / 你看着办」等推进语义，**跳过第二步等待**，直接给出建议默认值清单 + 立刻进入步骤 4 写入；过程中如有模糊点统一在最终汇报里说明。
> 2. **非交互式 CLI 模式**：若运行在 `-p` / `--input-format stream-json` 等单次喂入模式（看不到下一轮 user 消息），抛出清单后**不让出控制权**，按建议默认值连续走完步骤 4-6。

## 三级组织模型（命名核心）

| 层级 | 含义 | 对应 Whistle 实体 | 命名 |
|------|------|------------------|------|
| **业务大类** | 一组相关页面/场景 | Rule 的 `\r` 分组 | `{Category}`，如 `权益页面`、`运营页面` |
| **页面/场景** | 一个具体需求/页面 | 一个 Rule + 一个 Value Group | `{Scene}`，如 `专享理财金`、`财富私享会` |
| **接口变体** | 同接口的不同返回态 | Rule 行 + 一个 Value | `{Scene}{MethodName}[-{Variant}]` |

- **Value 命名**：`{Scene}{MethodName}[-{Variant}]`，**不加 `.json` 后缀**。如 `专享理财金queryPrizeLandingConfig`、`财富私享会WealthMeetingRecentInvite-没有`。
- **变体后缀**区分同接口不同返回，如 `-有奖`/`-无奖`、`-有数据`/`-空`、`-1`/`-2`；无变体可省略。
- 命名不能写死：根据会话上下文推测 `{Category}`/`{Scene}`，按「交互原则」第 2 步交开发者确认后再用。

## 工作流

### 步骤 1：检测 Whistle 端口

1. `ps aux | grep -i whistle | grep -v grep` 确认运行，并从命令行参数（`"port":"XXXX"` / `--port` / `-p`）提取端口。
2. 验证：`curl -s http://127.0.0.1:{port}/cgi-bin/init` 返回含 `version` 的 JSON 即可用。记为 `WHISTLE_PORT`，后续所有调用复用。
3. 未运行则引导启动：`npm install -g whistle && w2 start`（或 `w2 start -p 8088`）。

> 若已配置 `whistle-mcp` MCP 工具，可改用其 `getRules`/`updateRule`/`updateValue` 等，底层与 HTTP API 等价。

### 步骤 2：获取并解析接口文档

- **来源**：Git URL（工蜂 MCP 取 `.proto`）/ iWiki 链接（iWiki MCP）/ 直接粘贴。获取失败时一律降级为请用户直接粘贴内容。
- **解析提取**：服务名 + RPC 方法（→ URL pattern）、响应消息（→ mock 数据）、业务场景线索（→ `{Category}`/`{Scene}` 命名）。
- proto 解析细节见 `references/proto-to-mock-guide.md`。

### 步骤 3：输出规划清单并确认（第一轮回答收尾，**不出 JSON**）

第一轮回答**只输出命名规划清单**，**不生成任何 mock JSON 内容**（JSON 在用户确认后的步骤 4 才生成）。清单形式建议如下：

```
业务大类：{Category}
场景：{Scene}（Rule 名 = Value Group 名）

Values 清单：
- {Scene}{Method1}                        # 默认正常返回
- {Scene}{Method1}-{业务变体A}             # 业务变体（基于 message 推测）
- {Scene}{Method2}                        # …
- ...（如确认需要 error 变体，再增列 -error / -biz_error 等）

Rules 清单：
- Rule「{Scene}」包含以上 Values 对应的规则行（同接口多变体并列，默认只启用一行）
```

**业务变体的推测原则**：必须基于接口响应 message 的实际字段含义来设计，而非套用固定模板。示例：
- 接口含 `order_status` 枚举 → 变体可能是 `-已下单` / `-已支付` / `-已退款`。
- 接口含 `prize_info` 可选字段 → 变体可能是 `-有奖` / `-无奖`。
- 接口返回 `list` 数组 → 变体可能是 `-有数据` / `-空`。
- 接口字段单一无明显业务态 → **不强行添加业务变体**，只给默认正常返回。

随后按「交互原则」的四类问题在回答末尾集中确认，等待回复后再进入步骤 4。

### 步骤 4：写入 Whistle（开发者确认后执行）

使用确认后的 `{Category}`/`{Scene}`/`{Variant}` 写入。**此时才生成 mock JSON 内容**。完整 curl 参数见 `references/whistle-http-api.md` 的"完整 Mock 写入示例"。

**Mock JSON 生成规则**：
- 按 `references/proto-to-mock-guide.md` 的类型映射（`string→示例文本`、`int→100`、`repeated→2 元素数组`、`enum→首值`、`Timestamp→"2024-01-01T00:00:00Z"` 等）。
- **腾讯内部 tRPC 服务**：mock 必须含 `"retcode": "0", "retmsg": "OK"`（网关自动注入，即使 proto 未定义）。封装判断见 reference 的"处理响应封装"。
- **业务变体**：按确认的业务态生成对应字段差异（如 `-已下单` 则 `order_status=PAID`、`-空` 则 `list=[]`）。
- **error 变体**：按下方步骤 5 表格生成。
- **🚨 Value JSON 必须多行（2 空格缩进），禁止单行压缩**：单行 JSON 在 Whistle 编辑器中完全不可读，违背 mock 的「可调试」初衷。
  - ❌ `--data-urlencode 'value={"retcode":"0","data":{"a":1,"b":2}}'`
  - ✅ `--data-urlencode $'value={\n  "retcode": "0",\n  "data": {\n    "a": 1,\n    "b": 2\n  }\n}'`（ANSI-C quoting，`\n` 会被解析为真实换行）

**低侵入执行约束**：优先用已配置的 `whistle-mcp`；无 MCP 时直接用 `curl` 调 HTTP API。禁止为批量写入主动创建临时脚本、写入工作区文件、`chmod +x`、执行本地脚本。

**批次大小**：批量 Values 按 **3-5 个/批** 串成单次 Bash（用 `;` 分隔，**不要 `&&`**）。**禁止在批次中使用 shell 变量赋值**（如 `SCENE='xxx';...` 或 `for x in ...; do curl ...`）——CodeBuddy CLI 的 permission 系统无法识别这种命令的 root，会直接拒绝执行。变量需要复用时，**直接把字面值拼进每条 curl**。若整批被拒，再降级为 1 Value/批。

顺序：

1. **读现状** — `GET /cgi-bin/rules/list` 检查是否已存在同名 Rule。
2. **建 Value Group** — `POST /cgi-bin/values/add`，`name=\r{Scene}`。
   > 🚨 `\r` 必须是**真正的回车符 (0x0D)**：bash/zsh 用 ANSI-C quoting `--data-urlencode $'name=\r{Scene}'`，**切勿用双/单引号**（会变字面 `\r`，分组失效）。详见 reference「分组管理」。
3. **建 Values** — 对每个方法/变体：`values/add`（`name={Scene}{MethodName}[-{Variant}]`，`value=格式化 JSON`），再 `values/move-to`（`to=\r{Scene}`，同样需 `$'to=\r{Scene}'`）移入分组。
4. **建/更新 Rule（写入内容 + 启用，必须一气呵成）** — 一个场景 = 一个 Rule（Rule 名 = `{Scene}`）。
   > 🚨🚨 **`rules/add` 只创建空壳 Rule，不写内容！必须立刻调 `rules/select` 才真正落盘。**
   > **两步必须串在同一次 Bash 调用里**（用 `;` 串联，不要分批），杜绝在 `rules/add` 之后被中断（turn 上限/超时/网络）留下空壳。
   > 若不存在则 `rules/add; rules/select`；已存在则读 `data` **追加新行**（不覆盖），再 `rules/select`。可选用 `\r{Category}` 分组归类。
   > 若发现已存在内容为空的同名 Rule（之前被截断的空壳），先 `rules/remove` 删除再重做。

   规则格式（**`{Value}` 引用必须零空格、服务名 `.` 必须转义**，否则 Value 匹配不到 → resBody 为空 → 触发 DNS 报错）：
   ```txt
   # {Scene} - 正常响应
   /{ServiceName}\.{MethodName}/ resBody://{{Scene}{MethodName}} resType://json statusCode://200
   # 同接口多变体并列、互斥（只启用一行，其余 # 注释）
   /{ServiceName}\.{MethodName}/ resBody://{{Scene}{MethodName}-有数据} resType://json statusCode://200
   # /{ServiceName}\.{MethodName}/ resBody://{{Scene}{MethodName}-空} resType://json statusCode://200
   ```
   Rule 内可用 `# 子场景标题` 分块（如 `# 落地页`、`# 分享页`）。

   > 🚨 **三处常踩坑**（同时违反会导致 Whistle 报 `DNS Lookup Failed`，hostname 为空字符串）：
   > 1. **`resBody://{XXX}` 花括号内不能有空格**：`{ XXX }` ❌ → `{XXX}` ✅。带空格时 Whistle 把整段当字面 URL 而非 Value 引用，resBody 为空。
   > 2. **服务名中的 `.` 必须转义为 `\.`**：`/Service.Method/` ❌ → `/Service\.Method/` ✅。pattern 是正则，`.` 会匹配任意字符。
   > 3. **强烈建议加 `resType://json`**：与已知可跑通的规则对齐，避免 Content-Type 推断异常。

5. **批量启用与互斥** — `selected=true` 已在步骤 4 的 `rules/select` 里写入。需同时启用多条规则用 `rules/allow-multiple-choice`（`allowMultipleChoice=1`）。

6. **强制验证（不可省略，结束前必跑）** — 这是收尾的硬性 gate，**任何"已完成"声明都必须先通过这一步**：
   - `GET /cgi-bin/rules/list`：抓本场景 Rule，**断言 `data` 字段长度 > 0 且包含 `resBody://`**。若 data 为空字符串或缺 `resBody://`，说明 `rules/select` 漏调，**立即补救**：重新调 `rules/select` 写入完整内容；若失败则 `rules/remove` 删空壳 Rule 并重新走步骤 4。
   - **断言 Rule data 中不存在 `resBody://{ ` 或 ` }`**（即花括号内带空格）—— 一旦匹配到，说明会触发 Value 匹配失败 + DNS 报错，立刻 `rules/select` 用无空格版本覆盖。
   - **断言服务名中的 `.` 已转义为 `\.`**（grep `Service\.Method` vs `Service.Method`）—— 未转义时立刻覆盖。
   - `GET /cgi-bin/init`：抓 Values，**断言每个 value 的 `data` 字段非空、且包含换行符 `\n`**（单行 JSON 视为不合格，需重写）。任一不达标则重新 `values/add` 写入。
   - **勿用 `values/list`**（不返回内容）。
   - 仅当 Rule data 非空 + `resBody://{无空格}` + 服务名 `\.` 转义 + 全部 Values 内容齐全，才能宣布"写入完成"。

### 步骤 5：error 变体（按步骤 3「第三类问题」的确认结果）

> ⚠️ 本步骤**只处理 error 变体**（协议/系统层异常态），与 message 字段无关。基于 message 的业务变体已在步骤 3 规划清单中确认、步骤 4 中与正常返回一并写入，不在此处重复。

需要时与正常返回**并列写入同一 Rule**，默认 `#` 注释，开发者按需启用（同接口多变体互斥）：

| error 变体 | 规则行 | Value 内容 |
|------|--------|-----------|
| 服务器错误 | `/{Svc}\.{Method}/ statusCode://500` | — |
| 超时 | `/{Svc}\.{Method}/ resDelay://3000 file://{{Scene}{Method}-error}` | `{"retcode":"500","retmsg":"internal error"}` |
| 认证失败 | `/{Svc}\.{Method}/ statusCode://401` | — |
| 限流 | `/{Svc}\.{Method}/ statusCode://429` | — |
| 业务异常(200+retcode≠0) | `/{Svc}\.{Method}/ resBody://{{Scene}{Method}-biz_error} resType://json statusCode://200` | `{"retcode":"10001","retmsg":"业务处理失败","data":null}` |

## 重要规则

- **🚨 `resBody://{XXX}` 花括号内零空格**：`{ XXX }` ❌ → `{XXX}` ✅。带空格 Whistle 不识别为 Value 引用，resBody 为空，触发 hostname 为空的 DNS 报错（`DNS Lookup Failed`）。
- **🚨 URL pattern**：tRPC 用方法路径 `/ServiceName\.MethodName/`，服务名中的 `.` **必须**用 `\.` 转义（pattern 是正则），未转义会误匹配；末尾加 `/` 精确匹配；大小写敏感。**建议显式加 `resType://json`** 与已知可跑通的规则对齐。详见 patterns.md。
- **协议**：tRPC 服务默认 `resBody://` + `statusCode://200`（请求仍到服务器、保留真实响应头）；仅后端不可用时用 `file://`。部分字段覆盖用 `resMerge://`。详见 `references/whistle-mock-patterns.md`。
- **🚨 `rules/add` ≠ 写入规则内容**：`rules/add` 仅创建空壳。**真正写入规则内容靠 `rules/select` 的 `value=` 参数**。两者必须串在同一次 Bash 调用里（`rules/add; rules/select`），否则一旦中途被截断（turn 上限/超时）就会留下空壳 Rule。结束前必须用步骤 4-6 的"强制验证"确认 `data` 字段非空。
- **一个场景 = 一个 Rule**，所有接口及变体作为独立行写入同一 Rule；更新已有 Rule 时**追加不覆盖**（先读 `rules/list` 检查是否已有该 pattern，避免重复/并发互覆）。
- **Value 命名** `{Scene}{MethodName}[-{Variant}]`，无 `.json`；命名由 Skill 推测、开发者确认。
- **分组 `\r` 必须真回车符 (0x0D)**：用 `$'name=\r{Scene}'`，切勿双/单引号。**Whistle 没有 `values/add-group` 接口**，建分组只能用 `values/add` + `$'name=\r{Group}'`，无需做接口探活实验。
- **POST 必须 `--data-urlencode`**，不能用 `-d`（JSON 会丢失）；写入参数名 `value`、读取响应字段名 `data`。
- **执行约束**：只用 `curl` + `grep` + `head/tail`；不要用 `python3 -c`、 `python3`、`python3 -m`/ `jq` / 临时脚本 / 命令替换 `$(...)`。clientId 用固定串（如 `1718600000000-1`）。批量写入按 3-5 个/批拆 `curl`，不要在批次中用 shell 变量赋值（permission 系统会拒）。

## 参考文档

按需从 `references/` 加载：

- **`whistle-http-api.md`** — 完整 HTTP API、curl 示例、`\r` 分组规范、完整写入示例、分组排错。**写入时必读。**
- **`whistle-rule-syntax.md`** — Whistle 规则语法速查（pattern、操作、数据源、过滤器）。
- **`whistle-mock-patterns.md`** — mock 模式、协议对比、tRPC pattern、组织最佳实践与命名约定。
- **`proto-to-mock-guide.md`** — proto 转 mock、类型映射、响应封装处理。
