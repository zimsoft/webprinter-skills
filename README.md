# WebPrinter Skills

智睦云打印技能集合 — 让 AI 助手（Trae、QClaw、WorkBuddy、Claude Code 等）学会执行打印操作。

支持两大场景：
- **云平台打印** — 漫游打印、直接打印到云绑定设备
- **局域网打印** — 发现网络打印机、匹配驱动、云端渲染、通过 9100 端口下发

## 安装

### 方式一：Claude Code

将仓库克隆到用户技能目录（或项目 `.claude/skills/` 下）：

```bash
# 克隆到用户技能目录
git clone https://github.com/zimsoft/webprinter-skills.git \
  ~/.claude/skills/webprinter-skills

# 或克隆到项目技能目录
git clone https://github.com/zimsoft/webprinter-skills.git \
  .claude/skills/webprinter-skills
```

### 方式二：其他 AI 编程助手

大多数 AI 编程助手（Trae、QClaw、WorkBuddy 等）支持将 Skill 目录放到项目或用户配置中。以 QClaw 为例：

```bash
git clone https://github.com/zimsoft/webprinter-skills.git \
  ~/.qclaw/skills/webprinter-skills
```

> 具体路径因工具而异，请参考对应工具的 Skill 安装文档。核心原则：让工具能扫描到 `SKILL.md` 文件即可。

### 前置依赖

- 云平台打印需要 `WEBPRINTER_ACCESS_TOKEN`（在 https://any.webprinter.cn/get-ai-server-token 获取）
- 局域网打印可选 `CDF_PRINT_API_KEY`
- 局域网打印的 DOCX/PPT 等格式转换需要 LibreOffice（仅 `_cvturl` 超时降级时需要）

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

- **SKILL.md**（根目录）是入口，AI 会根据用户意图自动加载对应的子技能
- **cloud-print/SKILL.md** 处理漫游打印、直接打印、参数更新
- **lan-print/SKILL.md** 处理打印机发现、驱动匹配、云端渲染、9100 下发

## 配置

| 环境变量 | 场景 | 必填 |
|----------|------|------|
| `WEBPRINTER_ACCESS_TOKEN` | 云平台打印 | 是 |
| `CDF_PRINT_API_KEY` | 局域网打印 | 否 |
| `CDF_PRINT_CONFIG_JSON` | 局域网打印 | 否（JSON，如 `{"copies":2}`） |

## 验证安装

安装后，在 AI 助手中尝试：

```
帮我打印这个文件：/path/to/document.pdf
```

AI 会自动加载技能并根据语境选择云平台打印或局域网打印。

## 更新

```bash
cd <skills-install-path>/webprinter-skills
git pull
```

---

[智睦云打印](https://any.webprinter.cn)
