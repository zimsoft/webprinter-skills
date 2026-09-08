---
name: cloud-print
description: 云打印主流程 — 帮助已开通智睦云打印的用户完成漫游打印、直接打印到指定云打印机。适用于企业已完成初始化、有共享打印机的场景。
env_vars:
  - WEBPRINTER_ACCESS_TOKEN
---

# 云打印

当你所在企业已开通智睦云打印（服务器已装好、打印机已共享），就可以用本流程打印文件。整个过程由 AI 引导，你只需要提供文件、在必要时选一下打印方式。

## 认证

- 环境变量 `WEBPRINTER_ACCESS_TOKEN`（没有就先引导用户去 `https://any.webprinter.cn/get-ai-server-token` 获取）
- Header：`Authorization: Bearer ***`

---

## 前置检查：判断你的企业是否已开通云打印

打印之前，先确认企业状态，避免你白等。

### checkInstallProgress — 查询企业初始化状态

- 路径：`POST /openapi/platform/checkInstallProgressMCP`
- Content-Type：`application/json`
- 请求体：空 JSON 对象 `{}`
- 响应字段（用于判断当前用户的企业是否已完成云打印初始化、是否有共享打印机）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `hasClient` | boolean | 是否有绑定的服务器/客户端（打印服务器）；false → 需先下载并安装打印服务器 |
| `hasDevice` | boolean | 是否有共享的打印机，或企业已登记 UniMan 打印服务器（≠ 设备当前在线） |
| `hasUser` | boolean | 是否已给其他用户授予应用权限（企业是否有多名用户） |

按响应结果分流（对应主技能引导流程）。**初始化完成判据：`hasClient` 与 `hasDevice` 均为 true**（UniMan 内网模式除外）：

| 情况 | 处理 |
|------|------|
| `hasClient: false`（企业还没有绑定的打印服务器） | 告知用户企业尚未开通云打印；如用户急着打印 → 提示临时用 [[lan-print]]；否则引导用户联系管理员，管理员按 [[init-server]] 安装打印服务器 |
| `hasClient: true` 且 `hasDevice: false`（服务器已装但没有共享打印机） | 引导用户联系管理员共享打印机（[[init-server]]），开通前无法云打印 |
| `hasClient: true` 且 `hasDevice: true`（已就绪） | 企业已就绪，继续下面的打印流程 |

---

## 打印流程：你想打印一个文件

### 第 1 步：上传/准备文件

- 本地文件 → 调用 `uploadFileMCP`（见主技能公共 API）上传，拿到文件 URL
- 用户提供 HTTPS 链接 → 直接用该链接，不要下载到本地再上传

### 第 2 步：查你有没有可直打的打印机

调用 `queryPrinters`（见下）获取可用打印机列表，按下面规则决策（不要跳步）：

| 查询结果 | 处理方式 |
|---------|---------|
| **有在线的直打打印机** | 询问用户：要漫游打印，还是直接打到某台打印机？仅列出可见的在线设备供选择（`hidden: true` 的设备已静默排除，视同不存在，不要向用户提及） |
| **没有直打打印机** | 不询问，直接创建漫游打印任务（第 3 步方案 A） |
| **有直打打印机但都不在线** | 先告知用户当前没有设备在线，再直接创建漫游打印任务（第 3 步方案 A） |

### 第 3 步：按选择执行

#### 方案 A：漫游打印（默认）

触发条件：用户未指定打印机 / 没有在线直打设备 / 用户选择漫游。

1. 调用 `createRoamingTask` 创建任务
2. 用户持任务到打印机端扫码释放（或由客户端自动打印）

#### 方案 B：直接打印到指定设备

触发条件：用户明确说"用 XX 打印机打印""直接打印到 XX"，且该设备在线。

1. 调用 `printDocument` 下发到用户选定的打印机

---

## 接口明细

### queryPrinters — 查询你可用的打印机

- 路径：`POST /openapi/control/queryPrinters`
- Content-Type：`application/json`
- 请求体：空 JSON 对象 `{}`
- 响应：打印机列表（数组），每个打印机对象含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `deviceName` | string | 打印机名称 |
| `shareSn` / `controlSn` / `sn` | string | 服务端标识（三种名称为同一含义，用于定位打印机） |
| `printerName` | string | 打印机名称（与 deviceName 相同或类似） |
| `printerAlias` | string | 打印机别名 |
| `deviceType` | string | 设备类型：`printer` / `scanner` / `camera` |
| `onlineStatus` | string | 在线状态：`"1"`=在线、`"0"`=离线。服务端会对状态为空的设备实时探测（约 3s 超时），探测失败也按 `"1"` 返回，故 `"1"` 不代表必然可打，向用户确认时仍允许改走漫游 |
| `hidden` | boolean | 是否隐藏。`hidden: true` 的设备视同不存在：静默排除，不展示、不作为选项、不向用户提及 |
| `deletable` | boolean | 是否可删除 |

在线判断：以 `onlineStatus == "1"` 为准；若想复核某台设备当前是否真在线，可再调用 `checkDeviceStatus`（`POST /openapi/control/checkDeviceStatus`）。

### createRoamingTask — 创建漫游打印任务

- 路径：`POST /openapi/task/createRoamingTask`
- Content-Type：`application/json`
请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `fileName` | string | 是 | 文档文件名（含扩展名） |
| `url` | string | 是 | 文件的可访问 HTTPS URL |
| `mediaFormat` | string | 是 | 文件格式，见"支持的文件格式"章节 |

响应 JSON：

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | boolean | 是否成功 |
| `data` | string | 任务 ID（如 `TASK_20240324_001`） |

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

## 可选操作（仅在用户明确要求时调用）

### 查询打印机能力

触发条件：用户明确提出"查询打印机能力""这台打印机支持什么"时才调用，不要主动调用。

#### queryPrinterDetail — 查询打印机能力

- 路径：`POST /openapi/control/queryPrinterDetail`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `printerName` | string | 否 | 打印机名称 |
| `shareSn` | string | 否 | 服务端标识 |
| `deviceType` | string | 否 | 设备类型：`printer` / `scanner` / `camera` |

响应含打印机支持的颜色模式、单双面选项、纸张规格等能力信息。

### 修改已创建任务的打印参数

触发条件：用户已创建打印任务后，明确提出修改参数时才调用。四个参数各自使用独立 API，不要合并。

#### updatePrinterSide — 更新单双面

- 路径：`POST /openapi/task/config/updatePrinterSideMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `side` | string | 是 | `ONESIDE` / `DUPLEX` / `TUMBLE` |

#### updatePrinterColor — 更新颜色

- 路径：`POST /openapi/task/config/updatePrinterColorMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `color` | string | 是 | `COLOR` / `MONOCHROME` |

#### updatePrinterCopies — 更新份数

- 路径：`POST /openapi/task/config/updatePrinterCopiesMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `copies` | integer | 是 | 份数，范围 1-99 |

#### updatePrinterPaper — 更新纸张

- 路径：`POST /openapi/task/config/updatePrinterPaperMCP`
- Content-Type：`application/json`

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务 ID |
| `paper` | object | 是 | 纸张对象，含 `width`(mm) 和 `height`(mm)，二者均为 float |

用户常说"改成 A4""纸张设成 A3"——需要将纸型名称转换为毫米后传入，不能把纸型名称原样传给后端。

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

## 文件来源安全边界

- 只接受两类文件来源：你指定的本地文件、你明确提供的 `https://` 文档链接
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

1. **`hidden: true` 的打印机视同不存在**：查询后静默排除，不展示、不作为选项、不向用户提及（用户不应感知到有设备被排除）
2. 用户未明确指定打印机时，默认走漫游打印
3. 用户提供 HTTPS 链接就直接用，不要下载到本地再上传
4. `queryPrinterDetail` 和所有 `update*` API 仅在用户明确要求时调用，不要主动触发
5. 每个 update 动作只修改一个参数（单双面/颜色/份数/纸张分别调用独立 API）
6. 纸型名称（A3、A4 等）必须转成毫米单位的 `width` 和 `height` 后再调用 API
7. 打印机定位以"打印机名称 + 服务端标识"为准，服务端标识字段名可能是 `sn` / `shareSn` / `controlSn`
8. 能不问就不问：只有存在在线直打设备时才询问用户"漫游 or 直打"；没有在线设备时直接创建漫游任务
