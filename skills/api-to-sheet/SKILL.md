---
name: api-to-sheet
description: 接口文档写入飞书表格。当用户提供接口说明、API文档、接口定义，并要求生成飞书表格、写入飞书电子表格、导出接口文档到飞书时触发。使用 lark-cli 将接口信息（地址、参数、示例）自动写入飞书电子表格。
---

# 飞书API文档生成器（lark-cli 版）

根据接口说明，使用 lark-cli 将接口文档写入飞书电子表格。

> **前置条件**：如已安装 `lark-shared` skill，先阅读其中的认证和权限处理说明。

## lark-cli 安装检测与引导

**每次执行前，按以下顺序检测 lark-cli 并确定可用路径：**

### 第1步：定位 lark-cli 可执行文件

```bash
# Windows：用 where 查找所有 lark-cli 路径
where lark-cli 2>/dev/null

# macOS/Linux：用 which 查找
which lark-cli 2>/dev/null
```

从输出中**优先选择 `.cmd` 路径**（如 `C:\Users\xxx\.nvmd\bin\lark-cli.cmd`），避免选择有模块解析问题的裸路径（如 `D:\Program\Nodes\xxx\lark-cli`）。

**选择优先级**：
1. `$HOME/.nvmd/bin/lark-cli.cmd`（nvmd 管理的版本，最可靠）
2. 其他 `.cmd` 包装脚本路径
3. 最后才考虑裸 `lark-cli` 可执行文件

### 第2步：验证找到的路径

```bash
# 用第1步找到的完整路径验证（Windows 示例）
"C:\Users\xxx\.nvmd\bin\lark-cli.cmd" auth status

# macOS/Linux 示例
/usr/local/bin/lark-cli auth status
```

- **命令成功** → 将该路径设为本次会话的 `LARK_CLI` 变量，后续所有 lark-cli 命令都用这个完整路径执行
- **命令失败** → 换下一个 `.cmd` 路径重试

### 第3步：所有路径都找不到时的备用排查

```bash
# Windows 常见安装位置
ls ~/AppData/Roaming/npm/lark-cli* 2>/dev/null
ls "$HOME/.nvmd/bin/lark-cli"* 2>/dev/null

# macOS/Linux 常见安装位置
ls /usr/local/bin/lark-cli 2>/dev/null

# 查看全局 npm 包，确认是否已安装
npm list -g @larksuite/cli 2>/dev/null
npm root -g  # 查看 node_modules 位置
```

**只有在以上所有方法都找不到 lark-cli 时，才判定未安装，引导用户安装。**

> **关键提醒**：后续所有 lark-cli 命令都必须使用第2步验证通过的完整路径，**不要**直接用裸命令 `lark-cli`，因为裸命令可能因 node 模块路径问题报错。

### 安装步骤

向用户说明需要安装 lark-cli，然后依次执行：

```bash
# 1. 安装 lark-cli
npm install -g @larksuite/cli

# 2. 安装飞书 skills（lark-sheets 等）
npx skills add https://github.com/larksuite/cli -y -g

# 3. 初始化应用配置（会弹出授权链接，用户在浏览器中完成）
lark-cli config init --new
```

> **重要**：配置完成后需重启 AI Agent 工具（Claude Code / Trae / Cursor 等），确保 skills 完整加载。

详细安装指南参考：https://www.feishu.cn/content/article/7623291503305083853

## 工作流程

根据用户是否提供了接口文档，分两种模式：

---

### 模式A：用户提供了接口文档

用户直接贴出接口说明（文本/Markdown/JSON），AI 解析后写入飞书表格。

**第1步**：AI 按下方「输入解析规则」从用户输入中提取接口信息，整理为标准 JSON 格式。

**第2步** → 跳到「通用步骤」。

---

### 模式B：用户未提供接口文档，自动扫描项目

用户只说"生成接口文档"或"扫描接口"但没给具体内容，AI 自动扫描当前项目代码提取接口。

**第1步：扫描项目结构**

用 Glob 快速定位 API 相关文件：

```
Glob: **/router*/**/*.{js,ts,py,java,go}
Glob: **/controller*/**/*.{js,ts,py,java,go}
Glob: **/route*/**/*.{js,ts,py,java,go}
Glob: **/api*/**/*.{js,ts,py,java,go}
Glob: **/*Controller*.{js,ts,py,java,go}
Glob: **/*Route*.{js,ts,py,java,go}
```

如果没匹配到，扩大范围扫描所有源码文件。

**第2步：识别接口定义**

用 Grep 搜索常见框架的路由/API 定义模式：

```
Grep: (router\.(get|post|put|delete|patch)|@Get|@Post|@Put|@Delete|app\.(get|post)|@RequestMapping|@GetMapping|@PostMapping|func.*Handler|HandleFunc)
```

**第3步：读取关键文件**

用 Read 读取匹配到的文件，从中提取：
- HTTP 方法（GET/POST/PUT/DELETE）
- 路由路径
- 请求参数（query/body/path 参数）
- 返回结构（从 response struct / DTO / 类型定义推断）
- 注释中的接口描述

**第4步：整理为标准格式**

将扫描结果整理为标准 JSON 格式（见下方定义），按模块/控制器分组。

**第5步** → 进入「通用步骤」。

---

### 通用步骤

**G1：确定目标表格**

- 用户提供飞书表格链接 → 用 `lark-cli sheets +info` 获取表格信息，追加到该表格底部（隔一行）
- 没有提供链接 → 用 `lark-cli sheets +create` 创建新表格

**G2：用 lark-cli 写入数据**

AI 将解析好的接口数据转换为二维数组（见「接口文档 → 二维数组映射」），通过 `lark-cli` 写入飞书表格。多个接口时一次性拼好整个二维数组写入，接口之间隔一行。

**G3：返回结果**

返回表格链接。

## lark-cli 操作命令

> **注意**：以下命令中的 `lark-cli` 应替换为检测阶段确认的完整路径（如 `"C:\Users\xxx\.nvmd\bin\lark-cli.cmd"`），不要直接用裸命令。

### 创建新表格

```bash
# 创建空表格
lark-cli sheets +create --title "接口文档_项目名"

# 创建到指定文件夹
lark-cli sheets +create --title "接口文档_项目名" --folder-token "fldcnxxxxx"

# 仅预览
lark-cli sheets +create --title "接口文档_项目名" --dry-run
```

创建后记录返回的 `spreadsheet_token`、`sheet_id` 和 `url`。

### 查看已有表格信息

```bash
# 通过 URL
lark-cli sheets +info --url "https://xxx.feishu.cn/sheets/shtcnxxxxx"

# 通过 token
lark-cli sheets +info --spreadsheet-token "shtcnxxxxx"
```

### 读取表格内容（查找末尾行）

追加前需要先读取 A 列确定最后一行位置：

```bash
lark-cli sheets +read --spreadsheet-token "shtcnxxxxx" \
  --sheet-id "<sheetId>" --range "A:A"
```

### 写入数据

```bash
# 覆盖写入（新表格或指定位置）
lark-cli sheets +write --spreadsheet-token "shtcnxxxxx" \
  --sheet-id "<sheetId>" --range "A1" \
  --values '<二维数组JSON>'

# 追加写入（已有表格末尾追加）
lark-cli sheets +append --spreadsheet-token "shtcnxxxxx" \
  --sheet-id "<sheetId>" --range "A1" \
  --values '<二维数组JSON>'

# 预览
lark-cli sheets +write --spreadsheet-token "shtcnxxxxx" \
  --sheet-id "<sheetId>" --range "A1" \
  --values '<二维数组JSON>' --dry-run
```

## 接口文档 → 二维数组映射

AI 将解析好的接口数据按以下布局转换为二维数组，一次性写入：

### 单个接口的数据布局（6列 A-F）

```
行1: [功能名称, <name>,       版本号, <version>,   功能描述, <description>]
行2: [接口地址, <url>,        "",     "",          请求方式, <method>]
行3: [输入参数, 参数名,        类型,   说明,         必须,     缺省值]
行4+:["",      <name>,        <type>, <desc>,      <required>,<default>]
...（如有请求头）
行N: [请求头,   参数名,        类型,   说明,         必须,     缺省值]
行N+:[ "",     <name>,        <type>, <desc>,      <required>,<default>]
...
行M: [输出参数, 字段名,        类型,   说明,         必返,     备注]
行M+:["",      <name>,        <type>, <desc>,      <required>,<remark>]
...
行X: [请求示例, <requestExample>, "",  "",          "",       ""]
行Y: [返回示例, <responseExample>, "",  "",          "",       ""]
```

### 示例：将一个接口写入

```bash
# 假设解析出的接口数据：
# name=用户登录, version=V1.0.0, description=用户登录接口
# url=/api/login, method=POST
# inputParams: [{name:username, type:string, desc:用户名, required:Y}]
# outputParams: [{name:code, type:int, desc:状态码, required:Y, remark:0成功}]

lark-cli sheets +write --spreadsheet-token "shtcnxxxxx" \
  --sheet-id "<sheetId>" --range "A1" \
  --values '[
    ["功能名称","用户登录","版本号","V1.0.0","功能描述","用户登录接口"],
    ["接口地址","/api/login","","","请求方式","POST"],
    ["输入参数","参数名","类型","说明","必须","缺省值"],
    ["","username","string","用户名","Y","--"],
    ["输出参数","字段名","类型","说明","必返","备注"],
    ["","code","int","状态码","Y","0成功"],
    ["请求示例","curl -X POST /api/login ...","","","",""],
    ["返回示例","{\"code\":0,\"data\":{...}}","","","",""]
  ]'
```

### 多个接口

多个接口拼接时，接口之间隔一行（空行），然后继续写下一组。可以分多次 `+append`，也可以一次性拼好整个二维数组。

## 输入解析规则

从用户描述中提取以下信息：

### 基本信息提取

| 关键词 | 字段 |
|--------|------|
| 接口名称/名称/功能名 | name |
| 接口地址/地址/URL/接口 | url |
| 请求方式/方法/GET/POST | method |
| 版本 | version |
| 描述/说明 | description |

### 参数提取

**Markdown表格**：识别表头包含`参数名`、`类型`、`说明`等关键词

**文本列表**：
```
- username: string, 用户名, 必填
- password: string, 密码
```

**CSV格式**：首行为表头，后续为数据

### 字段映射

根据上下文判断参数类型：
- "输入参数"/"请求参数" → inputParams
- "请求头"/"Headers" → headers
- "输出参数"/"返回参数"/"返回值" → outputParams

## 标准JSON格式（AI 内部中间格式）

AI 解析后整理为此格式，再转换为二维数组：

```json
{
  "name": "接口名称",
  "version": "V1.0.0",
  "description": "功能描述",
  "url": "接口地址",
  "method": "POST",
  "inputParams": [
    {"name": "参数名", "type": "类型", "desc": "说明", "required": "Y", "default": "默认值"}
  ],
  "headers": [
    {"name": "参数名", "type": "类型", "desc": "说明", "required": "Y", "default": "默认值"}
  ],
  "outputParams": [
    {"name": "字段名", "type": "类型", "desc": "说明", "required": "Y", "remark": "备注"}
  ],
  "requestExample": "请求示例",
  "responseExample": "返回示例"
}
```

## 样式规则（参考）

> 注：lark-cli sheets shortcuts 不直接支持样式设置。以下样式规则供后续通过 lark-cli 原生 API（如 `sheets spreadsheet.sheets.setStyle`、`sheets spreadsheet.sheets.mergeCells`）或手动调整时参考。

- **标签列**（功能名称、参数名等）：浅蓝背景 `#E7F3FF`，居中
- **数据列**：无背景色，左对齐
- **边框**：浅灰色 `#CCCCCC`，全边框
- **合并**：接口地址值(B:D)、示例值(B:F)

## 参考

- lark-cli 操作参考见 lark-sheets skill（`+create`、`+write`、`+append`、`+info`、`+read`、`+find`）
