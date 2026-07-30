---
name: cloud-print
description: 云平台打印 — 漫游打印、直接打印到云绑定设备、打印参数更新。适用于用户已在 any.webprinter.cn 绑定打印机的场景。
env_vars:
  - WEBPRINTER_ACCESS_TOKEN
---

# 云平台打印

用户在云平台已安装客户端并绑定打印机。用户发起打印后，云端渲染并通过客户端或直接发送到打印机完成作业。

## 认证

- 环境变量 `WEBPRINTER_ACCESS_TOKEN`
- Header：`Authorization: Bearer <token>`
- 获取令牌：用户需访问 `https://any.webprinter.cn/get-ai-server-token`

## 前置检查

首次执行打印操作前，检查用户环境是否就绪。

### checkInstallProgress — 检查安装与绑定状态

- 路径：`POST /openapi/platform/checkInstallProgressMCP`
- Content-Type：`application/json`
- 请求体：空 JSON 对象 `{}`
- 响应字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `hasClient` | boolean | 是否安装了智睦云打印客户端 |
| `hasDevice` | boolean | 是否已绑定/共享了打印机 |

- `hasClient: false` → 停止，提示用户去 `https://any.webprinter.cn` 下载客户端并绑定
- `hasDevice: false` → 停止，提示用户先在客户端共享或绑定一台打印机

---

## 核心流程

### 流程一：漫游打印（默认）

触发条件：用户说"打印"且未指定具体打印机名称。

步骤：
1. 如果是本地文件，调用 `uploadFileMCP` 上传，获取文件 URL
2. 如果是用户提供的 HTTPS 链接，直接使用该链接（不要下载到本地再上传）
3. 调用 `createRoamingTask` 创建任务
4. 用户持任务到打印机端扫码释放

### createRoamingTask — 创建漫游打印任务

- 路径：`POST /openapi/task/createRoamingTask`
- Content-Type：`application/json`
- 响应解析：返回的是纯文本任务 ID（非 JSON），需从响应体中提取

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `fileName` | string | 是 | 文档文件名（含扩展名） |
| `url` | string | 是 | 文件的可访问 HTTPS URL |
| `mediaFormat` | string | 是 | 文件格式，见"支持的文件格式"章节 |

响应：返回纯文本 taskId（如 `TASK_20240324_001`）

---

### 流程二：直接打印到指定设备

触发条件：用户明确说"用 XX 打印机打印""直接打印到 XX"。

步骤：
1. 调用 `queryPrinters` 获取打印机列表
2. 如果用户指定的打印机名称模糊或不在列表中，将可用打印机列给用户确认（过滤 `hidden: true` 的条目）
3. 如果是本地文件，先上传
4. 调用 `printDocument` 下发到打印机

### queryPrinters — 查询云平台可用打印机

- 路径：`POST /openapi/control/queryPrinters`
- Content-Type：`application/json`
- 请求体：空 JSON 对象 `{}`
- 响应：打印机列表（数组），每个打印机对象含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `deviceName` | string | 打印机名称 |
| `shareSn` / `controlSn` / `sn` | string | 服务端标识（三种名称为同一含义，用于定位打印机） |
| `hidden` | boolean | 是否隐藏，显示给用户时应过滤为 true 的条目 |
| `printerName` | string | 打印机名称（与 deviceName 相同或类似） |

### printDocument — 直接打印到指定设备

- 路径：`POST /openapi/task/directPrintDocumentMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `fileName` | string | 是 | 文档文件名（含扩展名） |
| `url` | string | 是 | 文件的可访问 HTTPS URL |
| `mediaFormat` | string | 是 | 文件格式 |
| `deviceName` | string | 是 | 打印机名称（来自 queryPrinters 返回值） |
| `controlSn` | string | 是 | 服务端标识（来自 queryPrinters 返回值） |

---

### 流程三：查询打印机能力

触发条件：用户明确提出"查询打印机能力""这台打印机支持什么"时才调用，不要主动调用。

### queryPrinterDetail — 查询打印机能力

- 路径：`POST /openapi/control/queryPrinterDetail`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `printerName` | string | 否 | 打印机名称 |
| `shareSn` | string | 否 | 服务端标识 |
| `deviceType` | string | 否 | 设备类型：`printer` / `scanner` / `camera` |

响应含打印机支持的颜色模式、单双面选项、纸张规格等能力信息。

---

### 流程四：更新打印参数

触发条件：用户已创建打印任务后，明确提出修改参数时才调用。四个参数各自使用独立 API，不要合并。

### updatePrinterSide — 更新单双面

- 路径：`POST /openapi/task/config/updatePrinterSideMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `side` | string | 是 | `ONESIDE` / `DUPLEX` / `TUMBLE` |

### updatePrinterColor — 更新颜色

- 路径：`POST /openapi/task/config/updatePrinterColorMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `color` | string | 是 | `COLOR` / `MONOCHROME` |

### updatePrinterCopies — 更新份数

- 路径：`POST /openapi/task/config/updatePrinterCopiesMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `copies` | integer | 是 | 份数，范围 1-99 |

### updatePrinterPaper — 更新纸张

- 路径：`POST /openapi/task/config/updatePrinterPaperMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `paper` | object | 是 | 纸张对象，含 `width`(mm) 和 `height`(mm)，二者均为 float |

用户常说"改成 A4""纸张设成 A3"——Agent 需要将纸型名称转换为毫米后传入。不能把纸型名称原样传给后端。

---

## 纸型映射表

| 纸型名称 | width (mm) | height (mm) |
|----------|-----------|-------------|
| A3 | 297.0 | 420.0 |
| A4 | 210.0 | 297.0 |
| A5 | 148.0 | 210.0 |
| A6 | 105.0 | 148.0 |
| B4 | 250.0 | 353.0 |
| B5 | 176.0 | 250.0 |
| LETTER | 215.9 | 279.4 |
| LEGAL | 215.9 | 355.6 |
| TABLOID | 279.4 | 431.8 |

别名：`a4` → `A4`，`letter` / `usletter` → `LETTER`，`legal` / `uslegal` → `LEGAL`，`tabloid` / `ledger` → `TABLOID`

---

## 安全边界

- 只接受两类文件来源：用户指定的本地文件、用户明确提供的 `https://` 文档链接
- 拒绝 `localhost`、`.local`、私网 IP 地址的 URL
- 如果 URL 指向内网，提醒用户该地址必须能被 `any.webprinter.cn` 服务器访问到
- 不要为了"验证链接"主动下载、抓取远程文档内容——直接把原始 URL 传给 API
- 首选路径：本地文件先上传，用返回的 `https://any.webprinter.cn/...` URL 创建任务
- 用户提供的远程链接域名可疑时，先提醒风险再继续

---

## 支持的文件格式

`HTML` `PNG` `JPG` `PDF` `BMP` `WEBP` `WORD` `EXCEL` `PPT` `TEXT` `WPS` `ODF` `ODT` `ODS` `ODP` `ODG` `XPS` `PWG`

---

## 核心规则

1. 用户未明确指定打印机时，默认走漫游打印
2. 用户提供 HTTPS 链接就直接用，不要下载到本地再上传
3. `queryPrinterDetail` 和所有 `update*` API 仅在用户明确要求时调用，不要主动触发
4. 每个 update 动作只修改一个参数（单双面/颜色/份数/纸张分别调用独立 API）
5. 纸型名称（A3、A4 等）必须转成毫米单位的 `width` 和 `height` 后再调用 API
6. 打印机定位以"打印机名称 + 服务端标识"为准，服务端标识字段名可能是 `sn` / `shareSn` / `controlSn`
