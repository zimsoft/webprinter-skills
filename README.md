# 智睦云打印技能集合

智睦云打印技能集合 — 让 AI 助手（Trae、QClaw、WorkBuddy 等）学会执行打印操作。

支持两大场景：
- **云平台打印** — 漫游打印、直接打印到云绑定设备
- **局域网云驱直连** — 企业未开通云打印时的临时应急方案

## 安装

在 AI 助手中直接说：

```
帮我安装这个技能 https://github.com/zimsoft/webprinter-skills
```

## 配置

| 环境变量 | 场景 | 必填 |
|----------|------|------|
| `WEBPRINTER_ACCESS_TOKEN` | 云平台打印 | 是 |

Token 获取地址：[https://any.webprinter.cn/get-ai-server-token](https://any.webprinter.cn/get-ai-server-token)

| 环境变量 | 场景 | 必填 |
|----------|------|------|
| `WEBPRINTER_ACCESS_TOKEN` | 局域网云驱直连 | 否 |
| `CDF_PRINT_CONFIG_JSON` | 局域网云驱直连 | 否（JSON，如 `{"copies":2}`） |

局域网云驱直连的 DOCX/PPT 转换需要 LibreOffice（仅在 `_cvturl` 超时降级时用到）。

## 使用

安装后直接在 AI 助手中说：

```
帮我打印这个文件：/path/to/document.pdf
```

AI 会自动加载技能，按引导流程选择对应的打印方式（云打印优先；未开通云打印时引导联系管理员，仅紧急情况走局域网云驱直连）。

## 目录结构

```
webprinter-skills/
├── README.md
├── SKILL.md                  # 主入口 — 引导流程（token → 云打印优先 → 无共享打印机找管理员）+ 公共信息
├── cloud-print/
│   └── SKILL.md              # 云打印主流程子技能（漫游/直打/参数更新）
├── init-server/
│   └── SKILL.md              # 管理员安装打印服务器并共享打印机
└── lan-print/
    └── SKILL.md              # 局域网云驱直连（仅未开通云打印时的临时应急方案）
```

## 更新

在 AI 助手中说：

```
帮我更新 webprinter-skills 技能
```

---

[智睦云打印](https://any.webprinter.cn)
