# WebPrinter Skills

智睦云打印技能集合 — 让 AI 助手（Trae、QClaw、WorkBuddy 等）学会执行打印操作。

支持两大场景：
- **云平台打印** — 漫游打印、直接打印到云绑定设备
- **局域网打印** — 发现网络打印机、匹配驱动、云端渲染、通过 9100 端口下发

## 安装

在 AI 助手中直接说：

```
帮我安装这个技能 https://github.com/zimsoft/webprinter-skills
```

## 配置

| 环境变量 | 场景 | 必填 |
|----------|------|------|
| `WEBPRINTER_ACCESS_TOKEN` | 云平台打印 | 是 |

Token 获取地址：`https://any.webprinter.cn/get-ai-server-token`

| 环境变量 | 场景 | 必填 |
|----------|------|------|
| `CDF_PRINT_API_KEY` | 局域网打印 | 否 |
| `CDF_PRINT_CONFIG_JSON` | 局域网打印 | 否（JSON，如 `{"copies":2}`） |

局域网打印的 DOCX/PPT 转换需要 LibreOffice（仅在 `_cvturl` 超时降级时用到）。

## 使用

安装后直接在 AI 助手中说：

```
帮我打印这个文件：/path/to/document.pdf
```

AI 会自动加载技能并选择对应的打印方式。

## 目录结构

```
webprinter-skills/
├── README.md
├── SKILL.md                  # 主入口 — 路由决策 + 公共信息
├── cloud-print/
│   └── SKILL.md              # 云平台打印子技能
└── lan-print/
    └── SKILL.md              # 局域网打印子技能
```

## 更新

在 AI 助手中说：

```
帮我更新 webprinter-skills 技能
```

---

[智睦云打印](https://any.webprinter.cn)
