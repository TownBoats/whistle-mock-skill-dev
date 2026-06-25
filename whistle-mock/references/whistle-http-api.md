# Whistle HTTP API 操作规范

本文档基于 [whistle-mcp](https://github.com/7gugu/whistle-mcp) 源码整理，记录所有通过 HTTP 直接操作 whistle 的接口规范。
**无需安装任何 MCP 工具，直接用 `curl` 即可完成所有操作。**

## 基础信息

- **Base URL**: `http://127.0.0.1:{PORT}`（端口通过 `ps aux | grep whistle` 检测）
- **请求格式**: `application/x-www-form-urlencoded`（POST 请求）
- **编码方式**: POST 请求**必须使用 `--data-urlencode`**（不能用简单 `-d`，否则包含特殊字符的内容会丢失）
- **认证**: 如果 whistle 设置了用户名密码，使用 HTTP Basic Auth（`-u username:password`）
- **成功响应**: `{"ec": 0}` 表示成功

## 🚨 常见陷阱速查（写入前必读）

> 这些是反复踩坑后总结的 Top 陷阱，**写入前先扫一遍能避免 80% 的失败**：

1. **`rules/add` 只创建空壳，不写内容**——真正写入靠 `rules/select` 的 `value=` 参数。两步**必须串在同一次 Bash 调用里**（用 `;` 串联），否则一旦中途被截断就会留下空壳 Rule。结束前用 `rules/list` 验证 `data` 字段非空。
2. **🚨 `resBody://{XXX}` 花括号内零空格 + 服务名 `.` 必须转义**——这三件事任一违反，Whistle 会返回 `DNS Lookup Failed`（hostname 为空字符串）：
   - `resBody://{ XXX }` ❌（带空格） → `resBody://{XXX}` ✅
   - `/Service.Method/` ❌（未转义 `.`） → `/Service\.Method/` ✅
   - 强烈建议显式加 `resType://json` 与已跑通的规则对齐
3. **Whistle 没有 `values/add-group` 接口**——别浪费 turn 探活。建分组的**唯一**方式是 `values/add` + `$'name=\r{Group}'`（必须 ANSI-C quoting）。
4. **`\r` 必须是真回车符 (0x0D)**——用 `$'name=\r分组名'`。双引号 `"...\r..."` 或单引号 `'...\r...'` 都会变成字面反斜杠+r，分组失效。
5. **POST 必须 `--data-urlencode`**——用 `-d` 传 JSON 时特殊字符会丢，写入为空。
6. **写入用 `value` 字段，读取用 `data` 字段**——字段名不对称，别搞混。
7. **`values/list` 不返回内容**——查 value 内容必须用 `/cgi-bin/init` 看 `values.list[].data`。
8. **批量调用 + shell 变量赋值会被 CodeBuddy CLI 拒绝**（`SCENE='xxx';...` 这种命令 root 无法识别）——直接把字面值拼进每条 curl，按 3-5 个/批拆。

---

## 端口检测

```bash
# 方法 1：从进程参数中获取端口
ps aux | grep -i whistle | grep -v grep
# 查找 "port":"8088" 或 --port 8088

# 方法 2：验证连通性
curl -s http://127.0.0.1:8088/cgi-bin/init | grep -o '"version":"[^"]*"'
```

---

## 规则管理（Rules）

### 获取规则列表

```bash
curl -s http://127.0.0.1:{PORT}/cgi-bin/rules/list
```

**响应结构：**
```json
{
  "ec": 0,
  "defaultRules": "规则内容...",
  "defaultRulesIsDisabled": 0,
  "allowMultipleChoice": 1,
  "list": [
    {
      "name": "规则名",
      "data": "规则内容",
      "selected": true,
      "index": 0
    }
  ]
}
```

### ⚠️ 创建规则（rules/add 仅占位，不写内容；必须紧跟 rules/select）

> 🚨🚨 **`rules/add` 不会写入规则内容！它只是创建一个空壳 Rule（name 有了、data 是空的）。**
> 要让 mock 真正生效，**必须立即调用 `rules/select` 写入 `value=` 内容**。两步必须串在同一次 Bash 调用里（`;` 分隔），不可分批。
>
> ❌ **错误用法**（仅调 `rules/add` 就结束）：Whistle 里建了一个名字，但 mock 完全无效。这是 mock 不生效最常见的根因。
> ✅ **正确用法**：`rules/add; rules/select`（两条 curl 用分号串联）。

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=规则名"
```

### 更新规则内容（并启用）

这是**最关键的接口**，相当于"写入规则+启用"一步完成：

```bash
# 普通规则（非 Default）
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/select \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=规则名" \
  --data-urlencode "value=/Service\.MethodName/ resBody://{mock.json} resType://json statusCode://200" \
  --data-urlencode "selected=true" \
  --data-urlencode "active=true" \
  --data-urlencode "key=w-reactkey-$(( RANDOM % 1000 ))" \
  --data-urlencode "hide=false" \
  --data-urlencode "changed=true"
```

```bash
# Default 规则
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/enable-default \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=Default" \
  --data-urlencode "value=规则内容" \
  --data-urlencode "selected=true" \
  --data-urlencode "active=true" \
  --data-urlencode "key=w-reactkey-$(( RANDOM % 1000 ))" \
  --data-urlencode "hide=false" \
  --data-urlencode "changed=true"
```

### 禁用规则

```bash
# 普通规则
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/unselect \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=规则名" \
  --data-urlencode "value=规则内容" \
  --data-urlencode "selected=true" \
  --data-urlencode "active=true" \
  --data-urlencode "key=w-reactkey-$(( RANDOM % 1000 ))" \
  --data-urlencode "hide=false" \
  --data-urlencode "changed=true"

# Default 规则
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/disable-default \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=Default" \
  --data-urlencode "value=规则内容" \
  --data-urlencode "selected=true" \
  --data-urlencode "active=true" \
  --data-urlencode "key=w-reactkey-$(( RANDOM % 1000 ))" \
  --data-urlencode "hide=false" \
  --data-urlencode "changed=true"
```

### 删除规则

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/remove \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "list[]=$RULE_NAME"
```

### 重命名规则

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/rename \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "name=旧名称" \
  --data-urlencode "newName=新名称"
```

### 启用/禁用多规则模式

```bash
# 启用（允许同时选中多条规则）
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/allow-multiple-choice \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-0&allowMultipleChoice=1"

# 禁用
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/allow-multiple-choice \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-0&allowMultipleChoice=0"
```

### 禁用所有规则

```bash
# 禁用所有
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/disable-all-rules \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-1&disabledAllRules=1"

# 恢复所有
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/disable-all-rules \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-1&disabledAllRules=0"
```

---

## 分组管理（Groups）

> 🚨 **分组的核心：名称前缀 `\r` 必须是真正的回车符 (0x0D)，不是字面的反斜杠+r。**
> Whistle 完全依据"名称首字符是否为回车符 (0x0D)"来判断一个 Rule/Value 是否为分组。
>
> | 写法 | 实际传给 whistle 的名称 | 结果 |
> |------|------------------------|------|
> | `--data-urlencode $'name=\r分组名'` | `<0x0D>分组名` | ✅ 正确，识别为分组 |
> | `--data-urlencode "name=\r分组名"` | `\r分组名`（字面反斜杠+r） | ❌ 普通项，分组失效 |
> | `--data-urlencode 'name=\r分组名'` | `\r分组名`（字面反斜杠+r） | ❌ 普通项，分组失效 |
>
> **必须用 `$'...'`（ANSI-C quoting）**，bash/zsh/sh 原生支持，**无需 Python**。
> `$'...'` 只包裹参数值，`--data-urlencode` 是独立 token——不要写成 `$'--data-urlencode name=\r...'`（curl 会报 unknown option）。
> 服务名是变量时用拼接：`$'name=\r'"$SVC"`。

### 创建规则分组

分组名前加 `\r`（回车符）标记为分组：

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode $'name=\r分组名'
```

### 删除规则分组

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/remove \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode $'list[]=\r分组名'
```

### 重命名分组

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/rename \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode $'name=\r旧分组名' \
  --data-urlencode $'newName=\r新分组名'
```

### 移动规则到分组

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/rules/move-to \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "from=规则名" \
  --data-urlencode $'to=\r分组名' \
  --data-urlencode "group=false"
```

---

## Values 管理

### 获取 Values 列表（不含内容）

```bash
curl -s "http://127.0.0.1:{PORT}/cgi-bin/values/list"
```

> ⚠️ **注意**：`values/list` **不返回 value 内容**，只返回名称列表。要获取完整内容必须用 `/cgi-bin/init`。

### 获取所有 Values（含内容）

```bash
curl -s http://127.0.0.1:{PORT}/cgi-bin/init
```

**响应中的 values 部分：**
```json
{
  "values": {
    "list": [
      {"name": "mock.json", "data": "{\"code\":0}", "index": 0},
      {"name": "\r分组名", "data": "", "index": 1}
    ]
  }
}
```

> ⚠️ **关键**：Value 内容在响应中的字段名是 **`data`**（不是 `value`）。名称以 `\r` 开头的是 Value 分组。

### 创建 Value（仅创建，不写入内容）

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "name=mock-data.json"
```

### 创建并写入 Value 内容（推荐：一步完成）

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "name=mock-data.json" \
  --data-urlencode 'value={"retcode":"0","retmsg":"OK","data":{"list":[]}}'
```

> **重要**：
> - 创建和更新使用的是**同一个接口** `/cgi-bin/values/add`。如果 name 已存在则更新 value 内容；如果只传 name 不传 value 则仅创建空 Value。
> - **必须使用 `--data-urlencode`**，不能用 `-d`。用 `-d` 传 JSON 内容时特殊字符会导致内容丢失（写入为空）。

### 删除 Value

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/remove \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "list[]=mock-data.json"
```

### 重命名 Value

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/rename \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=旧名称" \
  --data-urlencode "newName=新名称"
```

### 创建 Value 分组

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode $'name=\r分组名'
```

### 移动 Value 到分组

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/values/move-to \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "from=value名称" \
  --data-urlencode $'to=\r分组名' \
  --data-urlencode "group=false"
```

---

## 代理控制

### 获取 Whistle 状态

```bash
curl -s http://127.0.0.1:{PORT}/cgi-bin/init
```

响应包含完整的 whistle 状态：`version`、`rules`、`values`、`interceptHttpsConnects` 等。

### HTTPS 拦截

```bash
# 启用 HTTPS 拦截
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/intercept-https-connects \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-0&interceptHttpsConnects=1"

# 禁用
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/intercept-https-connects \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-0&interceptHttpsConnects=0"
```

### HTTP/2

```bash
# 启用
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/enable-http2 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-0&enableHttp2=1"

# 禁用
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/enable-http2 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "clientId=1718600000000-0&enableHttp2=0"
```

---

## 抓包与重放

### 获取拦截的请求

```bash
TIMESTAMP=1718600000000
curl -s "http://127.0.0.1:{PORT}/cgi-bin/get-data?clientId=${TIMESTAMP}-0&startLogTime=-2&startSvrLogTime=-2&ids=&startTime=${TIMESTAMP}-000&dumpCount=0&lastRowId=${TIMESTAMP}-000&logId=&count=20&_=${TIMESTAMP}"
```

### 重放请求（Composer）

```bash
curl -s -X POST http://127.0.0.1:{PORT}/cgi-bin/composer \
  -H "Content-Type: application/x-www-form-urlencoded; charset=UTF-8" \
  --data-urlencode "useH2=" \
  --data-urlencode "url=https://example.com/api/test" \
  --data-urlencode "method=POST" \
  --data-urlencode "headers=Content-Type: application/json" \
  --data-urlencode 'body={"key":"value"}'
```

---

## `w2 add` 命令（CLI 写入规则）

除 HTTP API 外，`w2 add` 是另一种写入规则的方式（仅写入，不支持查看/删除）：

```bash
# 创建 .whistle.js
cat > /tmp/.whistle.js << 'EOF'
exports.name = 'mock-奖品详情';
exports.rules = `
/QueryMaterialsEquityInfo/ resBody://{QueryMaterialsEquityInfo.json} statusCode://200
`;
EOF

# 写入 whistle（-p 指定端口）
w2 add /tmp/.whistle.js -p 8088

# 强制覆盖已有同名规则
w2 add /tmp/.whistle.js -p 8088 --force
```

---

## 完整 Mock 写入示例

以下是一个完整的 mock 写入流程（等效于 whistle-mcp 的操作）：

```bash
PORT=8088

# 1. 创建 Value 分组
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/values/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode $'name=\rMyService'

# 2. 创建并写入 Value（mock 数据）
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/values/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "name=GetUserInfo.json" \
  --data-urlencode 'value={"retcode":"0","retmsg":"OK","data":{"name":"张三","age":28}}'

# 3. 将 Value 移入分组
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/values/move-to \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-1" \
  --data-urlencode "from=GetUserInfo.json" \
  --data-urlencode $'to=\rMyService' \
  --data-urlencode "group=false"

# 4. 创建规则
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/rules/add \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=mock-MyService"

# 5. 写入规则内容并启用
curl -s -X POST http://127.0.0.1:$PORT/cgi-bin/rules/select \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "clientId=1718600000000-0" \
  --data-urlencode "name=mock-MyService" \
  --data-urlencode "value=/UserService\.GetUserInfo/ resBody://{GetUserInfo.json} resType://json statusCode://200" \
  --data-urlencode "selected=true" \
  --data-urlencode "active=true" \
  --data-urlencode "key=w-reactkey-$(( RANDOM % 1000 ))" \
  --data-urlencode "hide=false" \
  --data-urlencode "changed=true"
```

---

## 注意事项

1. **`clientId` 格式**：`{timestamp}-{seq}`，如 `1718100000000-0`。用于标识客户端会话，任意生成即可。
2. **分组标记**：名称前加 `\r`（**真正的回车符 0x0D**）表示分组，无论是 Rules 还是 Values。
   - ⚠️ 必须用 ANSI-C quoting：`--data-urlencode $'name=\r分组名'`。双引号 `"...\r..."` 或单引号 `'...\r...'` 都只会得到**字面反斜杠+r**，分组不生效。纯 bash/curl 即可，无需 Python。
   - `move-to` 的 `to` 参数同理必须用 `$'to=\r分组名'`，否则返回 `ec=2`（目标分组未匹配），文件不会进组。
   - 验证分组：读 `~/.WhistleAppData/.whistle/values/properties` 的 `filesOrder`，分组名应满足 `ord(name[0])==13`；或 `curl .../values/list | grep -o $'\r[A-Za-z]*' | od -c` 看是否以 `\r`(0d) 开头。
3. **`/cgi-bin/rules/select` vs `/cgi-bin/rules/add`**：
   - `add` 仅创建空规则（不含内容）
   - `select` 写入规则内容 + 选中（启用）— **这是实际写入规则的接口**
   - 对 Default 规则用 `enable-default` / `disable-default`
4. **`/cgi-bin/values/add` 的双重用途**：只传 `name` 是创建空 Value；同时传 `name` + `value` 是写入内容。
5. **必须使用 `--data-urlencode`**：POST 请求中包含 JSON 或特殊字符时，**绝对不能**用简单的 `-d` 参数（会导致内容丢失/写入为空）。`--data-urlencode` 会自动处理编码。
6. **验证写入结果**：
   - 规则内容：通过 `GET /cgi-bin/rules/list` 查看 `data` 字段
   - Value 内容：通过 `GET /cgi-bin/init` 查看 `values.list[].data` 字段（注意：`values/list` 不返回内容！）
7. **字段名差异**：
   - 写入时用 `value` 参数（POST 请求体中）
   - 读取时在响应中是 `data` 字段（`rules/list` 和 `init` 返回的列表项中）
