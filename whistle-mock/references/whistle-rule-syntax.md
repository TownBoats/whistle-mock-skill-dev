# Whistle 规则语法速查

## 规则结构

每条 Whistle 规则遵循以下格式：

```
pattern operation [lineProps...] [filters...]
```

- **pattern**：URL 匹配表达式
- **operation**：格式 `protocol://value`，定义操作
- **lineProps**：可选，行级属性
- **filters**：可选，对匹配请求的二次过滤

## Pattern 匹配

| 类型 | 格式 | 示例 |
|------|------|------|
| 域名 | `domain` / `//domain` | `www.example.com` |
| 域名 + 端口 | `domain:port` | `www.example.com:8080` |
| 带协议 | `schema://domain[:port]` | `https://www.example.com` |
| 路径匹配 | `domain/path` | `www.example.com/api/users` |
| 通配符域名 | `*.example.com` / `**.example.com` | 匹配子域名 |
| 通配符路径（前缀 `^`） | `^domain/path/*/file` | `*` = 单层，`**` = 多层 |
| 正则 | `/pattern/[flags]` | `/\/api\/v1\/data/i` |
| 捕获组 | `$0`~`$9` | 用于操作值替换 |

## 操作值来源

| 来源 | 语法 | 用途 |
|------|------|------|
| 内联 | `protocol://(value)` | 短内容，不含空格/换行 |
| 嵌入 | `protocol://{key}` + 代码块 | 规则文件中的中等长度内容 |
| Values 引用 | `protocol://{key}` | Values 面板中的可共享/复用内容 |
| 本地文件 | `protocol:///path/to/file` | 来自文件系统的大内容 |
| 远程 URL | `protocol://https://url` | 来自远程服务器的内容 |

## Mock 相关的核心协议

| 协议 | 语法 | 行为 | Mock 用途 |
|------|------|------|----------|
| `file` | `pattern file://{key}` | 直接返回内容，不请求服务器 | 离线 mock |
| `statusCode` | `pattern statusCode://500` | 返回指定状态码，不请求服务器 | 错误模拟 |
| `resBody` | `pattern resBody://{key}` | 请求服务器后替换响应体 | 默认 mock 方式（推荐） |
| `resMerge` | `pattern resMerge://{key}` | 将数据合并到服务器响应中 | 字段级覆盖 |
| `resReplace` | `pattern resReplace://{old=new}` | 响应中的正则/关键词替换 | 内容替换 |
| `resHeaders` | `pattern resHeaders://({"k":"v"})` | 设置响应头 | Content-Type、CORS |
| `resDelay` | `pattern resDelay://3000` | 延迟响应 N 毫秒 | 超时模拟 |
| `reqDelay` | `pattern reqDelay://3000` | 延迟请求 N 毫秒 | 超时模拟 |

## 数据对象格式

Whistle 支持 3 种数据对象格式：

### JSON 格式
```json
{
  "key1": "value1",
  "key2": "value2"
}
```

### 行格式
```txt
key1: value1
key2: value2
a.b.c: 123
```

### 内联格式
```txt
key1=value1&key2=value2
```

## 模板变量

使用反引号包裹的表达式实现动态值：

```txt
pattern protocol://`...${variable}...`
```

| 变量 | 值 |
|------|-----|
| `${now}` | 当前时间戳（毫秒） |
| `${random}` | 0~1 随机数 |
| `${randomUUID}` | UUID v4 |
| `${randomInt(n)}` | [0, n] 随机整数 |
| `${reqId}` | Whistle 请求 ID |
| `${url}` | 完整请求 URL |
| `${query.xxx}` | Query 参数值 |
| `${method}` | 请求方法 |
| `${reqHeaders.xxx}` | 请求头值 |
| `${clientIp}` | 客户端 IP |

## 过滤器

| 过滤器 | 语法 | 说明 |
|--------|------|------|
| 按方法包含 | `includeFilter://m:POST` | 仅匹配 POST 请求 |
| 按请求体包含 | `includeFilter://b:keyword` | 请求体包含关键词 |
| 按请求头包含 | `includeFilter://reqH:cookie=/env=dev/` | 请求头匹配正则 |
| 按状态码包含 | `includeFilter://s:200` | 响应状态码为 200 |
| 按路径排除 | `excludeFilter://*/api/auth` | 排除匹配路径 |
| 随机概率 | `includeFilter://chance:0.5` | 50% 概率匹配 |

## 行属性

| 属性 | 语法 | 说明 |
|------|------|------|
| 重要 | `lineProps://important` | 比其他规则优先级更高 |
| 弱规则 | `lineProps://weakRule` | 优先级更低（代理仍然生效） |

## 容量指南

| 大小 | 推荐来源 |
|------|---------|
| < 2KB | 内联 `()` 或嵌入 `{key}` |
| 2KB ~ 200KB | Values 引用 `{key}` |
| > 200KB | 本地文件 `/path/to/file` |
