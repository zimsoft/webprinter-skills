---
name: init-server
description: 指导管理员安装智睦云打印服务器并共享打印机，为企业开通云打印。当用户所在企业还没有共享打印机（未初始化）时，由管理员按本技能完成服务器安装、打印机共享、用户体系配置。
env_vars:
  - WEBPRINTER_ACCESS_TOKEN
---

# 云打印服务器安装与打印机共享（管理员操作指南）

本技能面向**打印管理员**（IT/行政）。当同事反馈"公司还不能云打印/找不到共享打印机"时，由你按本指南完成服务器安装，之后全公司同事就能用云打印了。

> 若你不是管理员，只是普通用户：请把本文档转给管理员处理即可，无需自己操作。
> 管理员操作手册（详细图文版）：《云打印服务器安装指南》https://liqdopokb8.feishu.cn/docx/A0audnVIaoolXqxWUHPcvYlUnFd

## 一、安装前准备

### 1. 准备一台 Windows 服务器电脑（或虚拟机）

| 项目 | 要求 |
|------|------|
| 系统 | Windows 10 / 11，或 Windows Server（**不建议 Windows 7**） |
| 内存 | 8G 以上 |
| CPU | Intel i5 及以上 |

这台电脑需要**长期开机**（它是企业的打印服务器），请：

- [ ] **关闭 Windows 自动休眠** — 防止电脑休眠导致云打印不可用。操作步骤见：《关闭 Windows 自动休眠》https://liqdopokb8.feishu.cn/docx/JBAQdgSf6oE0unxpG4ScZtSxn2b

### 2. 装好要共享的打印机

- 在服务器上安装需要共享给同事的打印机驱动，并确保**打印测试页能正常出纸**
- 具体安装方法见：《服务器如何添加待管控的打印机》https://liqdopokb8.feishu.cn/docx/VUv9drG8voOzkcxQpuVc3PtRnzb

### 3. 安装办公软件

- 安装 **WPS Office 或 Microsoft Office**（云端转换文档格式时依赖）

## 二、确定用户体系

智睦云支持多种企业接入方式，请根据公司实际情况选择一种：

| 接入方式 | 适用场景 |
|---------|---------|
| 飞书 / 企业微信 / 钉钉 | 公司使用这些办公平台 |
| 域控（Active Directory） | 公司有 Windows 域环境 |
| 自建用户 | 无第三方平台，智睦云自建账号体系 |
| 定制接入 | 需要对接公司自有系统 |

选择指南见：《服务器的组织架构和用户体系》https://liqdopokb8.feishu.cn/docx/QeuAdid3loDeNLxFSYLcAKtOnsd
各平台应用开通见：《如何开通企业微信、飞书、钉钉应用》https://liqdopokb8.feishu.cn/docx/WEbvdfD79o6ytTxmGYycdocdnrf
定制平台对接见：《定制平台对接》https://liqdopokb8.feishu.cn/docx/RNnudqv4coHF8mxsoc7cOsoEnGf

## 三、下载并安装服务器

按《云打印服务器安装指南》主文档的"下载并安装服务器"章节操作：
https://liqdopokb8.feishu.cn/docx/A0audnVIaoolXqxWUHPcvYlUnFd

## 四、安装完成后的验证

安装完成后，请确认：

1. **服务器端**：打印服务器服务正常运行，已共享的打印机出现在管理后台
2. **用户端**：让一位同事尝试云打印一个测试文件（走 [[cloud-print]] 流程）
   - 若同事的 `checkInstallProgress` 返回企业已初始化、有共享打印机 → 开通成功
   - 若仍提示未初始化 → 检查服务器是否在线（是否休眠/关机）、打印机是否已共享

## 常见问题

| 现象 | 处理 |
|------|------|
| 同事反馈"企业未初始化" | 服务器未安装或未运行，按本指南第三、四节检查 |
| 同事反馈"没有共享打印机" | 服务器上打印机未共享，见"服务器如何添加待管控的打印机"子文档 |
| 云打印偶发不可用 | 优先检查服务器是否自动休眠/关机（第一节已要求关闭休眠） |

## 开通后的路径

开通完成后，企业同事的打印流程回到正常路径：
- 普通打印 → [[cloud-print]]（漫游 / 直打）
- 局域网直连仅在企业未开通时作为临时应急 → [[lan-print]]
