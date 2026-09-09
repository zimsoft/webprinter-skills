---
name: webprinter-print
description: 帮助用户完成打印。先引导获取 Token，优先使用云打印（漫游或直打到指定打印机）；企业未开通云打印时引导联系管理员安装打印服务器并共享打印机（init-server）；仅在企业未初始化且暂时无法共享打印机时，才临时使用局域网云驱直连（lan-print）。
version: 2.1.0
env_vars:
  - WEBPRINTER_ACCESS_TOKEN
---

# 云打印服务

本技能帮你解决"怎么把文件打印出来"的问题。根据你的企业是否已开通智睦云打印，按下面的顺序一步步引导你，全程不需要你了解技术细节。

## 引导流程（严格按顺序执行，不要跳步）

### 第 1 步：先获取 Token

调用云平台接口前，需要先有访问令牌：

- 如果你还没有 `WEBPRINTER_ACCESS_TOKEN`，先引导你去 [https://any.webprinter.cn/get-ai-server-token](https://any.webprinter.cn/get-ai-server-token) 获取，拿到后再继续。
- 如果接口返回 401，同样先引导你重新获取 Token，再继续。

### 第 2 步：优先使用云打印（cloud-print）

默认优先引导你走云打印（漫游打印 / 直打云打印机）。先判断你的企业是否已开通：

- 调用 `checkInstallProgress`（见 [[cloud-print]] 前置检查）判断企业是否完成初始化：
  - **已初始化、企业有共享打印机** → 正常走 [[cloud-print]]
  - **企业还没有共享打印机** → 见第 3 步
  - **企业尚未初始化，且你暂时无法共享打印机** → 见第 5 步（临时方案）

### 第 3 步：企业没有共享打印机 → 找管理员开通

如果你所在的企业还没有共享打印机，普通用户无法自行开通，需要管理员在服务器上安装打印服务器并共享打印机：

- 引导你联系公司的打印管理员（IT/行政），把《云打印服务器安装指南》发给管理员：
  https://liqdopokb8.feishu.cn/docx/A0audnVIaoolXqxWUHPcvYlUnFd
- 如果你本人就是管理员 → 加载 [[init-server]]，按里面的步骤安装服务器并共享打印机
- 在管理员完成安装之前，你暂时无法使用云打印，告知用户耐心等待即可

### 第 4 步：云打印过程中，文件怎么打（漫游 or 直打）

当你（已开通云打印的用户）要打印某个文件时，按下面的顺序决策：

1. 先调用 `queryPrinters`（见 [[cloud-print]]）查你有没有可直打的打印机
   > `hidden: true` 的打印机视同不存在：静默排除，不展示、不给选项、不向用户提及
2. **有在线的直打打印机** → 询问你：要漫游打印，还是直接打到某台打印机？（仅列出可见的在线设备供选择）
3. **没有直打打印机** → 不需要询问，直接帮你创建漫游打印任务
4. **有直打打印机但都不在线** → 先告诉你当前设备都不在线，然后直接帮你创建漫游打印任务

> 原则：能不问就不问，只有"有在线直打设备可选"时才让你做选择。

### 第 5 步：临时应急方案（lan-print 云驱直连）

只有当你的企业还没完成初始化、且你暂时无法让管理员共享打印机、但又急着打印时，才提示你可以**临时**使用局域网云驱直连（[[lan-print]]），直接把文件打到局域网里的物理打印机。

- 明确告知：这是临时方案，等企业正式开通云打印后应回到云打印（漫游/直打）。

---

## 服务信息

- 域名：`https://any.webprinter.cn`
- 所有接口要求 `Accept: application/json`
- 上传类接口 (`uploadFileMCP`) 使用 `multipart/form-data`
- 其他 POST 接口统一使用 `application/json`（`application/x-www-form-urlencoded` 会返回 404）

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
Authorization: Bearer <WEBPRINTER_ACCESS_TOKEN>   （可选）
```

## 错误码速查

| 状态码 | 含义 |
|--------|------|
| 401 | 令牌缺失或失效（回到第 1 步重新获取 Token） |
| 403 | 无访问权限 |
| 404 | 路径不存在（检查 Content-Type 是否为 JSON） |
| 5xx | 服务端异常（稍后重试） |

## 子技能

- [[cloud-print]] — 云打印主流程：漫游打印、直打云打印机、企业初始化检查（推荐路径）
- [[init-server]] — 指导管理员安装打印服务器并共享打印机（企业开通云打印）
- [[lan-print]] — 局域网云驱直连打印（仅企业未初始化时的临时应急方案）
