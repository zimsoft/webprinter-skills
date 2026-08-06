---
name: lan-print
description: 局域网直连打印 — 发现网络打印机、匹配驱动、云端渲染、9100 端口下发。适用于用户有物理打印机和 IP 的场景。
env_vars:
  - CDF_PRINT_API_KEY
---

# 局域网直连打印

用户在局域网内有一台网络打印机（支持 9100 端口），需要完成：发现打印机 → 搜索驱动 → 上传文件 → 云端渲染 → 下载打印数据 → 通过 TCP 9100 发送到打印机。

## 认证

- 环境变量 `CDF_PRINT_API_KEY`（可选 Bearer Token）
- 所有请求携带 Headers：
  - `tid: cdf_ai_terminal`
  - `ttp: AI`
  - 如果配置了 `CDF_PRINT_API_KEY`，附加 `Authorization: Bearer <token>`

---

## 三步流程

```
用户说"打印这个文件到局域网打印机"
  │
  ├─ 步骤一：确认打印机
  │   ├─ 已有记录？→ 直接使用
  │   ├─ 用户提供 IP？→ 登记 IP，尝试 mDNS 获取型号
  │   ├─ 都不清楚？→ mDNS 扫描局域网打印机，列出让用户选
  │   └─ 型号未知？→ IPP / SNMP / HTTP 兜底探测（见"型号探测"章节）
  │
  ├─ 步骤二：确认驱动
  │   ├─ 已有记录中有驱动？→ 直接使用
  │   └─ 没有？→ 调用 _sgdv2 搜索驱动候选，选第一个匹配度最高的
  │
  └─ 步骤三：打印
      ├─ 上传文件 → uploadFileMCP
      ├─ 非 PDF 文件 → _cvturl 转换（超时降级 LibreOffice 本地转换）
      ├─ PDF URL → _pfs 云端渲染
      ├─ 下载 fileUrl 二进制数据
      └─ TCP 9100 发送到打印机
```

---

## 步骤一：发现与登记打印机

### mDNS 扫描局域网打印机

使用 `zeroconf` 库（`python-zeroconf`）监听以下服务类型 3-10 秒：

- `_ipp._tcp.local.`
- `_printer._tcp.local.`
- `_pdl-datastream._tcp.local.`

从 mDNS 广播中提取：IP 地址、打印机友好名称、型号（优先从 TXT 记录的 `note`/`name`/`printer-name` 字段取，型号从 `usb_mdl`/`mdl`/`model`/`product`/`ty` 字段取）。

### 按 IP 登记单台打印机

用户提供 IP 时，先尝试 mDNS 查找该 IP 对应的名称和型号。如果 mDNS 无结果，以 IP 作为名称，型号留空。

### 打印机记录管理

打印机信息保存为 Markdown 文件，按 IP 去重更新。模板格式：

```markdown
# <打印机名称>

- Name: <名称>
- IP: <IP地址>
- Port: <9100或其他端口>
- Model: <型号>
- Driver: <驱动名称>
- Notes: <备注>
```

查找打印机时按 IP 匹配已有记录。驱动信息在步骤二中回写到记录文件中。

### 型号探测（型号未知时的兜底）

当 mDNS 无法获取型号时（型号显示为 `-`），按以下优先级探测：

| 优先级 | 方式 | 说明 |
|--------|------|------|
| ① IPP | `ipptool -tv http://<IP>:631/ipp/print get-printer-attributes.test` | 最可靠，即使 SNMP 禁用也通常可用 |
| ② HTTP(S) | `curl -sk --max-time 5 http://<IP>/` | 部分打印机 Web 管理页含型号，注意 HTTPS 重定向 |
| ③ SNMP | `snmpget -v2c -c public <IP> 1.3.6.1.2.1.25.3.2.1.3.1` | 成功率低，很多打印机关闭 SNMP |

实战发现的常见 Web UI 模式：

| 品牌 | 特征 | 型号提取方式 |
|------|------|-------------|
| EPSON | Server 头 `EPSON HTTP Server`，80 返回 307 → HTTPS | `<title>` 含型号（如 `L6270 Series`） |
| Canon | 8000 端口 RPS 登录页 | `<title>` 含型号（如 `iR-ADV 4525 III`） |
| Xerox | frames 架构，80 端口 | `<title>` 含系列名，具体子型号需用户确认 |

IPP 返回的关键字段：

| 字段 | 含义 |
|------|------|
| `printer-make-and-model` | 精确型号（如 `EPSON L6270 Series`） |
| `printer-device-id` | 含 MFG/MDL/CMD 完整标识 |
| `printer-info` | 打印机友好名称 |
| `marker-names` / `marker-levels` | 墨量/碳粉名称和余量百分比 |
| `pages-per-minute` / `pages-per-minute-color` | 打印速度 |
| `sides-supported` | 支持的单双面模式 |
| `print-color-mode-supported` | 支持的颜色模式 |

注意：部分打印机 IPP 监听在 80 端口而非 631，可尝试 `http://<IP>/ipp/print`。

### TCP 连通性测试

打印前测试 `IP:9100` 是否可达（TCP 连接，超时 3s）。不可达则报错退出。

---

## 步骤二：驱动搜索

### searchDriver — 搜索驱动候选

- 路径：`POST /openapi/cdf/_sgdv2`
- Content-Type：`application/json`
- 超时：30s

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `model` | string | 是 | 打印机型号字符串，用于模糊匹配驱动 |

响应字段：

- 无匹配驱动时返回 `null`
- 匹配成功时返回单个驱动对象（非数组包裹）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `driver` | string | 驱动名称（选择后回写到打印机记录） |
| `manufacturer` | string | 制造商 |
| `product` | string | 产品名 |
| `matchLevel` | string | 匹配度：`EXACT` / `LIKELY` / `NONE` |
| `installed` | boolean | 驱动是否已安装 |
| `desc` | string | 驱动描述 |

驱动选择策略：优先选 `matchLevel: EXACT`。`model` 参数需足够精确（如 `L6270 Series` 而非 `EPSON`）才能命中。

---

## 步骤三：打印

### 3.1 上传文件

调用公共接口 `uploadFileMCP`（详见主 SKILL.md），获取返回的 `url`、`repoId`、`path`。

### 3.2 格式转换（仅非 PDF 文件）

PDF 文件跳过此步骤。DOCX/EXCEL/PPT/图片等需要先转为 PDF。

#### 方式一（优先）：convertUrlToPdf — 云端转换

- 路径：`POST /openapi/cdf/_cvturl`
- Content-Type：`application/json`
- 超时：60s

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `repoId` | string | 是 | 来自 uploadFileMCP 返回 |
| `path` | string | 是 | 来自 uploadFileMCP 返回 |

响应：直接返回转换后的 PDF URL 字符串（非 JSON）。

#### 方式二（降级）：LibreOffice 本地转换

当 `_cvturl` 超时（60s）或失败时，自动降级为本地 LibreOffice 转换：

```bash
libreoffice --headless --convert-to pdf <源文件路径> --outdir <输出目录>
```

转换后的 PDF 重新调用 `uploadFileMCP` 上传，获得新的 PDF URL。

### 3.3 云端渲染 — printForSkill

- 路径：`POST /openapi/cdf/_pfs`
- Content-Type：`application/json`
- 超时：300s

请求体：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `printer` | object | 是 | 打印机信息对象 |
| `url` | string | 是 | 文件的 PDF URL |
| `fileName` | string | 否 | 原始文件名 |
| `config` | object | 否 | 打印配置 |

`printer` 对象字段：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `driver` | string | 是 | 驱动名称 |
| `name` | string | 是 | 打印机名称 |
| `portType` | string | 是 | 固定值 `NET` |
| `portAddr` | string | 是 | 打印机 IP 地址 |
| `portSerial` | string | 否 | 端口序列号 |
| `fromSn` | string | 否 | 来源服务器 SN |

`config` 对象字段（均为可选）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `copies` | integer | 份数 |
| `color` | string | `COLOR` / `MONOCHROME` |
| `side` | string | `ONESIDE` / `DUPLEX` / `TUMBLE` |
| `pageRanges` | string | 页码范围 |
| `orientation` | string | `PORTRAIT` / `LANDSCAPE` |
| `quality` | string | `DRAFT` / `LOW` / `NORMAL` / `HIGH` |
| `paperConfig` | object | 纸张配置，含 `name`(string)、`width`(mm, float)、`height`(mm, float) |

纸张预设：

| 名称 | width | height |
|------|-------|--------|
| A3 | 297.0 | 420.0 |
| A4 | 210.0 | 297.0 |
| A5 | 148.0 | 210.0 |
| LETTER | 215.9 | 279.4 |
| LEGAL | 215.9 | 355.6 |

环境变量 `CDF_PRINT_CONFIG_JSON` 可传入 JSON 格式的默认打印配置，如 `{"copies":2,"color":"MONOCHROME"}`。

`_pfs` 响应字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | boolean | 云端渲染是否成功 |
| `reason` | string | 成功/失败原因 |
| `fileUrl` | string | 渲染数据分片目录路径（末尾为 `/`） |
| `taskId` | string | 任务 ID |

### 3.4 下载打印数据并 9100 下发（拆包流式）

`_pfs` 返回的 `fileUrl` 是分片目录路径，打印数据被拆成多个固定大小（~1MB）的二进制分片。通过 `.info` 元数据获取分片数量后逐片下载并发送。

#### 3.4.1 读取 .info

```
GET {fileUrl}.info
```

响应字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `fileIndex` | integer | 分片总数 N，分片路径为 `{fileUrl}/0` ~ `/{N-1}` |
| `totalBytes` | integer | 原始合并后的总字节数 |
| `gzip` | boolean | 分片是否经 gzip 压缩 |
| `cdfStatus` | string | 渲染状态，`CLOSE` 表示完成 |
| `jobId` | string | 任务 ID |

#### 3.4.2 流式下发（方案 A — 边下边发）

开一个 TCP 连接到 `IP:9100`，逐片下载并立即 sendall，最后关闭连接。打印机不关心数据分几次到达，TCP 字节流对它是透明的。

```
解析 .info → fileIndex = N
TCP connect IP:9100
for i in 0..N-1:
    chunk = GET {fileUrl}/{i}
    sock.sendall(chunk)
sock.close()
```

此方案内存占用低，适合大文件（几十 MB 渲染输出）。单个 TCP 连接的发送间隙对打印机无影响。

---

## 硬性约束

**禁止将原始文件（PDF/DOCX/图片等）直接通过 TCP 9100 或其他端口发送到打印机。**

原始文件格式不被打印机理解，会导致：
- 打印机解析失败，打印出乱码或空白页
- 部分打印机（如 EPSON L6270）收到非预期数据后会短暂断网（ping 100% 丢包，数分钟后恢复）

所有打印数据必须经过云端 `_pfs` 渲染管道生成，客户端只负责从 `fileUrl` 下载二进制后转发到 9100 端口。

---

## 故障与降级

### _cvturl 云端转换失败

`_cvturl` 对部分文件格式偶发超时（60s），概率约 60%。降级步骤：
1. 本地调用 LibreOffice `--headless --convert-to pdf` 转换为 PDF
2. 对转换后的 PDF 重新调用 `uploadFileMCP` 上传
3. 使用新上传的 PDF URL 继续后续 `_pfs` 流程

此降级链路稳定，不影响最终打印结果。

### _pfs 返回 success 但分片数据为空（驱动冷启动）

**现象：** `_pfs` 返回 `success: true` + 有效 `fileUrl`，但 `GET {fileUrl}/{0}` 返回 HTTP 200 + 0 bytes。

**根因：** 云端驱动渲染管道冷启动。同一打印机首次调用 `_pfs` 时，渲染节点需要加载驱动/预热，前几次请求可能产出空分片。后续命中缓存后恢复正常。

**处理方案：** 重试最多 3 次 `_pfs` 请求（间隔 3-5 秒），等待驱动预热。检查方式：`.info` 的 `cdfStatus` 为 `CLOSE` 且 `totalBytes > 0` 时渲染有效。持续失败则反馈用户排查云端渲染驱动兼容性。**禁止跳过 _pfs 裸发原始文件。**

### _pfs 整体不可用（超时无响应）

极少数情况下 `_pfs` 完全超时（无 HTTP 响应）。此场景只能反馈用户稍后重试，无法绕过 `_pfs` 裸发。

### searchDriver 非交互环境需要自动选择

无 stdin 时用 `--pick 1` 自动选第一个候选。非交互环境下直接选 `matchLevel` 最高的驱动即可。

### 打印机型号无法识别

参见"型号探测"章节的三级兜底方案。IPP 是最可靠的识别手段，即使 SNMP 禁用、Web 界面不可爬取，IPP 通常都可用。
