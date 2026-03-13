# Social Monitor Plugin for Claude Code

每月自动监控小红书和微博账号近3个月动态，并生成格式化 Word 报告。

数据采集由 [MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) 完成，Claude 负责读取数据、分析内容、生成 Word 报告。

## 包含的 Skills

| Skill | 触发方式 | 功能 |
|-------|---------|------|
| `xhs-ck` | 输入"xhs-ck"或"查看小红书动态" | 抓取小红书账号近3个月笔记，生成 Word 报告 |
| `weibo-ck` | 输入"weibo-ck"或"查看微博动态" | 抓取微博账号近3个月帖子，生成 Word 报告 |
| `douyin-ck` | 输入"douyin-ck"或"查看抖音动态" | 抓取抖音账号近3个月视频，生成 Word 报告 |

## 前置依赖

### 1. MediaCrawler

```bash
git clone https://github.com/NanmiCoder/MediaCrawler.git
cd MediaCrawler
uv sync
uv run playwright install
```

> 需要 Python >= 3.11，[uv](https://github.com/astral-sh/uv) 包管理工具

### 2. Node.js + docx

```bash
npm install docx
```

## 安装本插件

### macOS / Linux

```bash
cd ~/.claude/plugins
git clone https://github.com/mankyang/ck-skills.git social-monitor
```

### Windows（PowerShell）

```powershell
cd $env:USERPROFILE\.claude\plugins
git clone https://github.com/mankyang/ck-skills.git social-monitor
```

## 配置

安装后编辑两个 SKILL.md，填入你的账号和路径：

### 小红书（xhs-ck）

编辑 `~/.claude/plugins/social-monitor/skills/xhs-ck/SKILL.md`：

```markdown
### 监控账号列表

| 昵称 | userId |
|------|--------|
| 漂漂酱 | `5655ecbe50c4b41526f41339` |
| 胡盛呢 | `604db41b00000000010031e0` |

### 路径配置

MEDIACRAWLER_PATH = C:/path/to/MediaCrawler
OUTPUT_DIR        = C:/path/to/output/
```

### 抖音（douyin-ck）

编辑 `~/.claude/plugins/social-monitor/skills/douyin-ck/SKILL.md`：

```markdown
### 监控账号列表

| 昵称 | sec_uid 或完整主页 URL |
|------|----------------------|
| 英超解说郭灿亮 | `MS4wLjABAAAA...` |

### 路径配置

MEDIACRAWLER_PATH = C:/path/to/MediaCrawler
OUTPUT_DIR        = C:/path/to/output/
```

> sec_uid 获取方式：浏览器打开目标账号抖音主页，复制 URL 中 `/user/` 后面的字符串，或直接填完整主页 URL。

### 微博（weibo-ck）

编辑 `~/.claude/plugins/social-monitor/skills/weibo-ck/SKILL.md`：

```markdown
### 监控账号列表

| 昵称 | userId（微博uid） | 简介 |
|------|-----------------|------|
| 郭灿亮 | `1548718464` | 体育解说 |
| M赵路 | `1881976852` | 主持人 |

### 路径配置

MEDIACRAWLER_PATH = C:/path/to/MediaCrawler
OUTPUT_DIR        = C:/path/to/output/
```

## 使用

配置完成后，在 Claude Code 中直接输入触发词：

```
xhs-ck
```
```
weibo-ck
```

Claude 会自动：
1. 将账号列表写入 MediaCrawler 配置
2. 运行 MediaCrawler 抓取数据
3. 解析近3个月内容
4. 为每个账号生成 Word 报告

## 输出示例

- `某账号近3个月动态.docx`（小红书）
- `某账号_微博近3个月动态报告.docx`（微博）

每份报告包含：
- 账号基本信息（粉丝数、简介、IP属地等）
- 近3个月发布内容列表（含互动数据）
- 综合分析（发布节奏、内容主题、互动表现）

## 更新日志

### v2.1.0
- 新增 `douyin-ck` skill，支持抖音创作者近3个月视频监控
- 配置方式：填入 sec_uid 或完整主页 URL

### v2.0.0
- 数据采集改用 MediaCrawler，稳定性大幅提升
- 不再依赖 Chrome DevTools MCP 实时操控浏览器
- 新增 creator_info 字段（粉丝数、IP属地等更完整）

### v1.0.0
- 初始版本，基于 XHS MCP + Chrome DevTools MCP 实时抓取
