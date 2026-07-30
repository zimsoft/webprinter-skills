---
name: webprinter-print
description: WebPrinter 云打印服务技能集合。支持云平台漫游打印和局域网直连打印两大场景，共用 any.webprinter.cn 后端。
version: 2.0.0
env_vars:
  - WEBPRINTER_ACCESS_TOKEN
  - CDF_PRINT_API_KEY
---

# WebPrinter 打印服务

本技能是打印服务的总入口。根据用户意图自动路由到对应的子技能：

- 用户在云平台已绑定打印机 → 加载 [[cloud-print]]
- 用户有局域网物理打印机 → 加载 [[lan-print]]

## 服务信息

- 域名：`https://any.webprinter.cn`
- 所有接口要求 `Accept: application/json`
- 上传类接口 (`uploadFileMCP`) 使用 `multipart/form-data`
- 其他 POST 接口统一使用 `application/json`（`application/x-www-form-urlencoded` 会返回 404）

## 路由决策

```
用户说"打印"
  ├─ 提到"漫游""云打印""扫码取件""webprinter 客户端" → cloud-print
  ├─ 用户指定了云平台上的打印机名称 → cloud-print
  ├─ 用户提供了局域网 IP + 打印机型号 → lan-print
  ├─ 用户说"局域网打印机""网络打印机""9100""直接连打印机" → lan-print
  └─ 无法判断 → 询问用户：打印机是云平台绑定的还是本地的
```

## 公共 API

### uploadFileMCP — 上传文件

- 路径：`POST /openapi/mcpClient/uploadFileMCP`
- Content-Type：`multipart/form-data`
- 超时：120s

请求体（multipart）：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | binary | 是 | 文件二进制内容，附带原始文件名 |

响应字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `url` | string | 上传后的文件访问 URL |
| `repoId` | string | 存储仓库标识 |
| `path` | string | 存储路径 |

### 公共 Headers

云平台场景（cloud-print）：
```
Authorization: Bearer <WEBPRINTER_ACCESS_TOKEN>
```

局域网场景（lan-print）：
```
tid: cdf_ai_terminal
ttp: AI
Authorization: Bearer <CDF_PRINT_API_KEY>   （可选）
```

## 错误码速查

| 状态码 | 含义 |
|--------|------|
| 401 | 令牌缺失或失效 |
| 403 | 无访问权限 |
| 404 | 路径不存在（检查 Content-Type 是否为 JSON） |
| 5xx | 服务端异常 |

## 子技能

- [[cloud-print]] — 云平台漫游打印、直接打印到云绑定设备
- [[lan-print]] — 局域网打印机发现、驱动匹配、云端渲染、9100 直连下发
