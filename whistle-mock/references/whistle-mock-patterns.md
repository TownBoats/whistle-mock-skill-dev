# Whistle Mock 模式与最佳实践

## 核心 Mock 协议

### resBody 协议（推荐用于 tRPC）

`resBody` 协议是 tRPC/微服务 API mock 的**推荐默认选择**。它会将请求转发到服务器，然后替换响应体：

```
客户端请求 → Whistle → 服务器（请求已发出）
                ↓
          用 mock 数据替换响应体
                ↓
          客户端收到 mock 响应（带有真实响应头）
```

**tRPC 场景优先使用 resBody 的原因：**
- 保留真实响应头（Content-Type、CORS 等）
- 维持连接/会话上下文
- 避免因缺少响应头导致 tRPC 客户端解析异常
- 配合 `statusCode://200` 确保 HTTP 状态码正确

**模式：**
```txt
/MethodName/ resBody://{{Scene}MethodName} resType://json statusCode://200
```

> 🚨 `{XXX}` 花括号内**不能有空格**。`{ XXX }` 会让 Whistle 把整段当字面 URL 而非 Value 引用，导致 resBody 为空、命中空 hostname 的 DNS 查询并报 `DNS Lookup Failed`。

### file 协议（用于离线/纯 Mock）

`file` 协议拦截请求并直接返回指定内容，**不转发到服务器**：

```
客户端请求 → Whistle（匹配 file 规则）→ 直接返回 mock 数据 (200)
                  ↓
             （不请求服务器）
```

**使用 file:// 的场景：**
- 后端服务器不可用或已下线
- 想完全避免请求服务器
- mock 响应是自包含的，不依赖服务器头信息
- 测试离线场景

**模式：**
```txt
/MethodName/ file://{{Scene}MethodName}
```

### 协议对比

| 方面 | `resBody://` | `file://` |
|------|-------------|-----------|
| 服务器请求 | 是（发送到服务器） | 否（拦截） |
| 响应头 | 保留真实响应头 | 自动生成 |
| Content-Type | 来自真实响应 | 默认 text/plain，可用 resHeaders 指定 |
| 连接上下文 | 保持 | 不建立 |
| 适用场景 | 默认 mock、tRPC 服务 | 离线、纯 mock |
| 配合 statusCode | `resBody://{key} statusCode://200` | 不需要（始终 200） |

## tRPC URL Pattern 设计

### tRPC Pattern 示例（虚拟）

通过 HTTP 网关访问的 tRPC 服务使用**方法路径** pattern，无需域名：

```txt
# 简单方法名匹配（最常见）
/GetDemoItemInfo/ resBody://{示例页面GetDemoItemInfo} resType://json statusCode://200

# 小写方法名
/GetDemoStatus/ resBody://{示例活动页GetDemoStatus} resType://json statusCode://200

# Service.Method 带转义点号（推荐：精确匹配，避免误匹配）
/DemoItemService\.UpdateDemoAddress/ resBody://{示例页面UpdateDemoAddress} resType://json statusCode://200
```

> 🚨 **服务名 `.` 必须用 `\.` 转义**：pattern 是正则，未转义的 `.` 会匹配任意字符（如 `ServiceXMethod` 也会命中），可能误匹配其他方法或 Whistle 内部请求并触发 `DNS Lookup Failed`。
>
> 🚨 **`{ XXX }` 花括号内不能有空格**：必须写 `{XXX}`，带空格 Whistle 不识别为 Value 引用，resBody 为空，会触发空 hostname 的 DNS 报错。
>
> 💡 **强烈建议显式加 `resType://json`**：避免 Content-Type 推断异常（与项目中已跑通的 mock 规则对齐）。

### Pattern 匹配规则

| Pattern 类型 | 格式 | 示例 | 适用场景 |
|-------------|------|------|---------|
| 仅方法名 | `/MethodName/` | `/GetDemoItemInfo/` | 最常见，匹配任意域名 |
| 服务.方法 | `/ServiceName.MethodName/` | `/DemoItemService.GetDemoItem/` | 需要区分不同服务的同名方法 |
| 带域名 | `^domain.com/path/Method/` | `^api.example.com/trpc/Method/` | 需要限定特定域名 |
| 正则 | `/\/ServiceName\.MethodName\//` | `/\/DemoItemService\.UpdateDemoAddress\//` | 含特殊字符的复杂匹配 |

**关键约定：**
- 尾部 `/` 确保精确匹配方法边界
- 大多数情况无需域名前缀 — Whistle 匹配任意域名的路径
- **Service.Method 中必须用 `\.` 转义点号**（未转义会作为正则任意字符匹配，可能误匹配触发 DNS 报错）
- **Value 引用 `{XXX}` 内零空格**（带空格 Whistle 不识别为引用）
- **建议显式 `resType://json`**（与可跑通规则对齐）
- 大小写敏感 — 与 RPC 定义中的大小写完全一致

### RESTful API Pattern
```txt
# 精确路径匹配
example.com/api/v1/users resBody://{users.json} statusCode://200

# 带参数的路径（使用通配符或正则）
^example.com/api/v1/users/*/profile resBody://{userProfile.json} statusCode://200

# 基于 ID 的路径正则
/^https?:\/\/example\.com\/api\/v1\/users\/(\d+)/ resBody://{userDetail.json} statusCode://200
```

### HTTP + JSONP Mock
```txt
# 带 callback 的 JSONP
example.com/api/jsonp file://`(${query.callback}({"status":"ok"}))`

# 或使用 tpl 协议模板
example.com/api/jsonp tpl://{jsonp-template.json}
```

## 异常场景 Mock 模式

### 服务器错误（5xx）
```txt
# 直接返回 500 状态码
/MethodName/ statusCode://500

# 返回 500 并附带自定义错误体
/MethodName/ statusCode://500 resBody://({"retcode":"500","retmsg":"internal error"})
```

### 超时模拟
```txt
# 延迟 3 秒后返回正常数据
/MethodName/ resDelay://3000 resBody://{{Scene}MethodName} resType://json statusCode://200

# 延迟 + 错误响应
/MethodName/ resDelay://5000 file://{{Scene}MethodName-error}
```

### 网络错误模拟
```txt
# 401 未授权
/MethodName/ statusCode://401

# 403 禁止访问
/MethodName/ statusCode://403

# 404 未找到
/MethodName/ statusCode://404

# 429 限流
/MethodName/ statusCode://429

# 503 服务不可用
/MethodName/ statusCode://503
```

### 空数据 / 空响应
```txt
# 返回成功但数据为空
/MethodName/ resBody://{{Scene}MethodName-空} resType://json statusCode://200
```
其中 `{Scene}MethodName-空` 内容：
```json
{
  "retcode": "0",
  "retmsg": "OK",
  "data": null
}
```

### 不稳定 / 间歇性故障
```txt
# 50% 概率错误，50% 概率成功
/MethodName/ statusCode://500 includeFilter://chance:0.5
/MethodName/ resBody://{{Scene}MethodName} resType://json statusCode://200 includeFilter://chance:0.5
```

## 部分 Mock 模式

需要保留真实服务器响应但修改特定字段时：

### resMerge — 合并字段
```txt
# 保留真实响应但覆盖/添加特定字段
/MethodName/ resMerge://({"data":{"role":"admin"}})
```

### resBody — 完整替换
```txt
# 替换整个响应体（请求仍发送到服务器）
/MethodName/ resBody://{{Scene}MethodName-override} resType://json statusCode://200
```

### resReplace — 文本替换
```txt
# 替换响应中的特定文本（字段值替换）
/MethodName/ resReplace://"user_goods_status":4="user_goods_status":10
```

## 规则组织最佳实践

采用**三级组织模型**：业务大类 → 页面/场景 → 接口变体。

### 一个场景 = 一个 Rule

将一个**页面/业务场景**涉及的所有接口（及变体）归入同一个 Rule，Rule 名 = 场景名。Rule 内可用注释行细分子场景，同接口的多个变体并列、互斥（只启用一行，其余注释）：

```txt
# 场景：示例页面

# 落地页
/DemoItemService\.GetDemoItemInfo/ resBody://{示例页面GetDemoItemInfo} resType://json statusCode://200
/DemoItemService\.GetDemoStatus/ resBody://{示例页面GetDemoStatus} resType://json statusCode://200

# 同接口多变体（互斥，只启用一行）
/DemoItemService\.UpdateDemoAddress/ resBody://{示例页面UpdateDemoAddress-有数据} resType://json statusCode://200
# /DemoItemService\.UpdateDemoAddress/ resBody://{示例页面UpdateDemoAddress-空} resType://json statusCode://200

# 异常变体（注释掉，需要时启用）
# /DemoItemService\.GetDemoItemInfo/ statusCode://500
# /DemoItemService\.GetDemoStatus/ resDelay://3000 file://{示例页面GetDemoStatus-error}
```

> 不同场景间互不重叠时可同时启用多个 Rule（开启 `allow-multiple-choice`）；在 Whistle 左栏可把 Rule 按**业务大类**用 `\r` 前缀分组折叠（如 `示例业务`、`测试业务`）。

### Values 按场景分组

将 Values 按场景名组织到 Group 中（Group 名 = 场景名）：
- Group：`示例页面`
  - `示例页面GetDemoItemInfo` — 正常响应
  - `示例页面GetDemoStatus` — 正常响应
  - `示例页面UpdateDemoAddress-有数据` — 有数据变体
  - `示例页面UpdateDemoAddress-空` — 空数据变体
  - `示例页面GetDemoStatus-error` — 错误变体

### 命名约定

**Value 名 = `{Scene}{MethodName}[-{Variant}]`，不加 `.json` 后缀。**

| 类型 | 命名格式 | 示例 |
|------|---------|------|
| 正常响应 | `{Scene}{MethodName}` | `示例页面GetUser` |
| 有/无数据 | `{Scene}{MethodName}-有数据` / `-空` | `示例活动页GetInviteStatus-没有` |
| 有奖/无奖 | `{Scene}{MethodName}-有奖` / `-无奖` | `示例活动页QueryRewardInfo-有奖` |
| 错误/异常 | `{Scene}{MethodName}-error` / `-biz_error` | `示例页面GetUser-error` |
| 多版本 | `{Scene}{MethodName}-1` / `-2` | `示例活动页GetFeaturedReview-2` |

> 业务大类与场景名由 Skill 根据上下文推测、给出建议，经用户确认后使用，不要写死。

## Content-Type 处理

Value 名不再依赖 `.json` 后缀来推断 Content-Type：

- **`resBody://` 模式**（推荐）：请求仍发到真实服务器，Whistle 仅替换响应体，**Content-Type 沿用真实服务器响应头**，无需后缀即可正确解析。
- **`file://` 模式**（纯 mock）：若客户端对 Content-Type 敏感，可显式追加 `resHeaders://` 设置：
  ```txt
  /Service\.MethodName/ file://{场景MethodName} resHeaders://(content-type=application/json)
  ```

## 常见响应封装格式

许多 tRPC/微服务框架使用标准响应封装。请匹配项目使用的格式：

### 格式 1：retcode + retmsg（腾讯内部服务常见）
```json
{
  "retcode": "0",
  "retmsg": "OK"
}
```
错误：`{"retcode": "1612701126", "retmsg": "抱歉，当前使用人数较多导致系统繁忙"}`

注意：`retcode` 通常是**字符串**类型，不是整数。请检查项目约定。

### 格式 2：code + msg + data
```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```
错误：`{"code": 500, "msg": "internal error", "data": null}`

### 格式 3：ret + msg + data
```json
{
  "ret": 0,
  "msg": "success",
  "data": { ... }
}
```

### 格式 4：status + message + result
```json
{
  "status": 0,
  "message": "success",
  "result": { ... }
}
```

检测项目使用的封装格式：
1. 查看用户 Whistle 中已有的 Values 的响应格式
2. 检查 proto 响应消息的字段名
3. 查看项目的错误码约定

在 mock 数据中**使用相同的结构**。
