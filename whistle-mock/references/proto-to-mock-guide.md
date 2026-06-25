# Proto 转 Mock 指南

## Proto 文件结构

一个典型的 proto 文件包含：

```protobuf
syntax = "proto3";

package example.demo.item;

import "google/protobuf/timestamp.proto";

// 服务定义
service DemoItemService {
  rpc GetDemoItem(GetDemoItemReq) returns (GetDemoItemResp);
  rpc ListDemoItems(ListDemoItemsReq) returns (ListDemoItemsResp);
  rpc CreateDemoItem(CreateDemoItemReq) returns (CreateDemoItemResp);
}

// 请求消息
message GetDemoItemReq {
  string item_id = 1;
}

message ListDemoItemsReq {
  int32 page = 1;
  int32 page_size = 2;
  string keyword = 3;
}

// 响应消息
message GetDemoItemResp {
  int32 code = 1;
  string msg = 2;
  DemoItemData data = 3;
}

message ListDemoItemsResp {
  int32 code = 1;
  string msg = 2;
  DemoItemListData data = 3;
}

// 数据消息
message DemoItemData {
  string item_id = 1;
  string title = 2;
  string content = 3;
  repeated string tags = 4;
  DemoItemStatus status = 5;
  google.protobuf.Timestamp create_time = 6;
}

message DemoItemListData {
  int32 total = 1;
  bool has_more = 2;
  repeated DemoItemData list = 3;
}

enum DemoItemStatus {
  UNKNOWN = 0;
  DRAFT = 1;
  PUBLISHED = 2;
  ARCHIVED = 3;
}
```

## 解析步骤

### 1. 提取服务名

从 `package` 声明和 `service` 块识别完整 RPC 路径，如 `example.demo.item.DemoItemService.GetDemoItem`。

服务路径主要用于派生 **URL pattern**，默认取最后两层（`/DemoItemService.GetDemoItem/`），`.` 不转义。注意：Rule 名与 Value Group 名采用**页面/场景名**（由 Skill 根据上下文推测、用户确认），不再使用服务名，详见 `whistle-mock-patterns.md` 的"规则组织最佳实践"。

### 2. 提取 RPC 方法

从 `service` 块中提取每个 `rpc`：
- `GetDemoItem` → 方法名
- `GetDemoItemReq` → 请求消息名
- `GetDemoItemResp` → 响应消息名

### 3. 派生 URL Pattern

通过 HTTP 网关访问的 tRPC 服务，URL pattern 使用**方法路径**，默认取最后两层 `{Service}.{Method}`：

```txt
# 推荐：最后两层，前后加斜杠，点号不转义
/DemoItemService.GetDemoItem/
/DemoItemService.GetDemoStatus/
/DemoItemService.UpdateDemoAddress/
```

**从 proto 派生 pattern 的方法：**

1. **默认：最后两层匹配**
   ```
   Proto: package example.demo.item; service DemoItemService { rpc GetDemoItem(...) }
   Full: example.demo.item.DemoItemService.GetDemoItem
   Pattern: /DemoItemService.GetDemoItem/
   ```

2. **没有服务名线索时，退化为方法名匹配**
   ```
   Proto: rpc GetDemoItem(...) returns (...);
   Pattern: /GetDemoItem/
   ```

3. **如果用户提供了特定域名或路径前缀**，则添加前缀：
   ```
   ^api.example.com/trpc/ServiceName.MethodName/
   ```

如果 URL pattern 不确定，请向用户确认：
- 域名（如需要）
- URL 路径 pattern（检查项目的网关配置）

常见 pattern：
- tRPC 网关：`/MethodName/` 或 `/ServiceName.MethodName/`
- tRPC 直连：`/trpc/package.Service/Method`
- HTTP 网关：`/api/v1/resource/action`
- 自定义：因项目而异

### 4. 从响应消息生成 Mock 数据

根据字段类型递归生成 mock 数据。

## 类型映射：Proto → Mock 值

### 标量类型

| Proto 类型 | Mock 值 | 说明 |
|-----------|---------|------|
| `string` | `"示例文本"` | 使用描述性中文文本，更易读 |
| `int32` | `100` | |
| `int64` | `100` | |
| `uint32` | `100` | |
| `uint64` | `100` | |
| `sint32` | `100` | |
| `sint64` | `100` | |
| `fixed32` | `100` | |
| `fixed64` | `100` | |
| `sfixed32` | `100` | |
| `sfixed64` | `100` | |
| `float` | `1.0` | |
| `double` | `1.0` | |
| `bool` | `true` | |
| `bytes` | `""` | 为简洁起见使用空字符串 |

### Well-Known 类型

| Proto 类型 | Mock 值 |
|-----------|---------|
| `google.protobuf.Timestamp` | `"2024-01-01T00:00:00Z"` |
| `google.protobuf.Any` | `{"@type": "type.googleapis.com/example.Type"}` |
| `google.protobuf.Struct` | `{}` |
| `google.protobuf.Value` | `"example"` |
| `google.protobuf.ListValue` | `[]` |
| `google.protobuf.Empty` | `{}` |
| `google.protobuf.Duration` | `"0s"` |
| `google.protobuf.FieldMask` | `"*"` |
| `google.protobuf.Int32Value` | `100` |
| `google.protobuf.StringValue` | `"示例文本"` |
| `google.protobuf.BoolValue` | `true` |

### 复合类型

| 类型 | Mock 值 | 示例 |
|------|---------|------|
| `repeated Type` | 包含 2 个元素的数组 | `[item1, item2]` |
| `map<K,V>` | 包含 1 个条目的对象 | `{"key1": value1}` |
| `enum` | 第一个定义值（通常为 0） | `"UNKNOWN"` 或 `0` |
| `nested message` | 递归生成 | `{...}` |

### 语义字段名检测

生成 mock 数据时，检测常见字段名模式以提供更真实的值：

| 字段名模式 | Mock 值 |
|-----------|---------|
| `*_id` / `*Id` | `"10001"` |
| `*_name` / `*Name` | `"示例名称"` |
| `*_title` / `*Title` | `"示例标题"` |
| `*_url` / `*Url` / `*_link` | `"https://example.com"` |
| `*_email` / `*Email` | `"example@test.com"` |
| `*_phone` / `*Phone` | `"13800138000"` |
| `*_time` / `*Time` / `*_date` / `*Date` | `"2024-01-01T00:00:00Z"` |
| `*_status` / `*Status` | `1` |
| `*_type` / `*Type` | `1` |
| `*_count` / `*Count` / `*_total` / `*Total` | `100` |
| `*_page` / `*Page` | `1` |
| `*_size` / `*Size` / `page_size` / `pageSize` | `10` |
| `*_image` / `*Image` / `*_img` / `*Img` | `"https://via.placeholder.com/150"` |
| `*_avatar` / `*Avatar` | `"https://via.placeholder.com/80"` |
| `*_price` / `*Price` / `*_amount` / `*Amount` | `99.9` |
| `*_desc` / `*Desc` / `*_description` | `"这是示例描述文本"` |
| `*_code` / `*Code` | `"0"` 或 `"SUCCESS"` |
| `retcode` / `ret_code` | `"0"`（字符串类型，腾讯服务常见） |
| `retmsg` / `ret_msg` | `"OK"` |
| `errorcode` / `error_code` | `""` |
| `*_msg` / `*Msg` / `*_message` / `*Message` | `"ok"` |
| `has_more` / `hasMore` | `false` |
| `is_*` / `is*` | `false` |

## 生成算法

```
function generateMock(message, depth=0):
    if depth > 5: return {}  // 防止无限递归

    result = {}
    for field in message.fields:
        if field is repeated:
            result[field.name] = [generateValue(field.type, depth+1), generateValue(field.type, depth+1)]
        else if field is map:
            result[field.name] = {"sample_key": generateValue(field.valueType, depth+1)}
        else:
            result[field.name] = generateValue(field.type, depth+1)
    return result

function generateValue(type, depth):
    if type is scalar: return SCALAR_MAP[type]
    if type is wellKnown: return WELLKNOWN_MAP[type]
    if type is enum: return firstEnumValue(type)
    if type is message: return generateMock(type, depth)
```

## 处理响应封装

许多服务使用通用响应封装。从 proto 或已有 Values 中检测格式：

> ⚠️ **关键：tRPC HTTP 网关自动注入 `retcode`/`retmsg`**
> 腾讯内部 tRPC 服务通过 HTTP 网关暴露时，网关会**自动**在响应 JSON 顶部注入 `retcode` 和 `retmsg` 字段，**即使 proto 定义中没有这两个字段**。这是网关层行为，不属于业务 proto 的一部分。因此：
> - **proto 里找不到 `retcode`/`retmsg` 是正常的**，不代表 mock 中不需要加
> - 腾讯内部 tRPC 服务 mock **默认应加上** `"retcode": "0", "retmsg": "OK"`
> - `retcode` 是**字符串**类型（`"0"` 不是 `0`）

### 如果响应使用 `retcode`/`retmsg` 结构（腾讯内部约定，tRPC 网关默认）：
```json
{
  "retcode": "0",
  "retmsg": "OK",
  "data": {
    // ... 从 data 消息生成
  }
}
```
注意：腾讯服务中 `retcode` 通常是**字符串**类型（非整数）。不要因为 proto 中没有这两个字段就省略它们——它们是网关自动注入的。

### 如果响应使用 `code`/`msg`/`data` 结构：
```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    // ... 从 data 消息生成
  }
}
```

### 如果响应是扁平消息：
```json
{
  // ... 所有字段直接生成
}
```

`data` 字段的类型应从其消息定义中解析。

**检测封装格式的方法：**
1. 查看用户 Whistle 中已有的 Values 的响应格式
2. 查看 proto 响应消息的字段名
3. 检查项目是否使用了通用封装 proto（如 `CommonResp`、`BaseResponse`）
4. **如果是腾讯内部 tRPC 服务，默认就应加 `retcode`/`retmsg`**——即使 proto 里没有定义，因为 HTTP 网关会自动注入

## 完整示例

### 输入 Proto：
```protobuf
syntax = "proto3";
package example.demo.item;

service DemoItemService {
  rpc GetDemoItem(GetDemoItemReq) returns (GetDemoItemResp);
}

message GetDemoItemReq {
  string item_id = 1;
}

message GetDemoItemResp {
  int32 code = 1;
  string msg = 2;
  DemoItemData data = 3;
}

message DemoItemData {
  string item_id = 1;
  string title = 2;
  string content = 3;
  repeated string tags = 4;
  DemoItemStatus status = 5;
}

enum DemoItemStatus {
  UNKNOWN = 0;
  DRAFT = 1;
  PUBLISHED = 2;
}
```

### 生成的 Mock：

**Value：`示例商品GetDemoItem`**
```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "item_id": "10001",
    "title": "示例标题",
    "content": "这是示例描述文本",
    "tags": ["标签1", "标签2"],
    "status": 0
  }
}
```

**Value：`示例商品GetDemoItem-error`**
```json
{
  "code": 500,
  "msg": "internal error",
  "data": null
}
```

**Value：`示例商品GetDemoItem-空`**
```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**Rule：`示例商品`（场景名）**
```txt
# 正常响应
/DemoItemService.GetDemoItem/ resBody://{示例商品GetDemoItem} resType://json statusCode://200

# 异常/边界变体（取消注释以启用）
# /DemoItemService.GetDemoItem/ statusCode://500
# /DemoItemService.GetDemoItem/ resDelay://3000 file://{示例商品GetDemoItem-error}
# /DemoItemService.GetDemoItem/ resBody://{示例商品GetDemoItem-空} resType://json statusCode://200
```

> 🚨 三处必守约束：
> 1. **`{XXX}` 花括号内零空格**：`{ XXX }` ❌ → `{XXX}` ✅
> 2. **pattern 取最后两层且 `.` 不转义**：`/package.Service.Method/` ❌ → `/Service.Method/` ✅
> 3. **显式 `resType://json`**：与已知可跑通规则对齐，避免响应类型推断异常

### 另一示例：retcode/retmsg 格式（腾讯约定）

**Value：`示例页面GetDemoItemInfo`**
```json
{
  "retcode": "0",
  "retmsg": "OK",
  "item_reward_info": {
    "reward_id": "REWARD_10001",
    "reward_name": "示例奖品",
    "reward_url": "https://via.placeholder.com/300x300"
  },
  "item_order_state": "NOT_ORDERED"
}
```

**Rule：`示例页面`（场景名）**
```txt
/DemoItemService.GetDemoItemInfo/ resBody://{示例页面GetDemoItemInfo} resType://json statusCode://200
```

## iWiki 文档解析

当输入来自 iWiki（而非 proto 文件）时，文档通常包含：

1. **API 路径** — 如 `/api/v1/items/get`
2. **请求方法** — GET/POST
3. **请求参数** — Query 参数或请求体字段
4. **响应结构** — 带字段描述的 JSON 结构

解析文档时提取：
- URL 路径 → 用作规则中的 pattern
- 响应 JSON → 直接用作 mock 数据，为占位字段填充示例值
- 字段描述 → 作为生成真实 mock 值的参考

如果 iWiki 文档包含完整的示例响应，直接用作 mock 数据。如果仅有字段定义（名称 + 类型），则应用与 proto 解析相同的类型映射和语义检测规则。
