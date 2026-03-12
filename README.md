# Social Monitor Plugin for Claude Code

每月自动监控小红书和微博账号近3个月动态，并生成格式化 Word 报告。

## 包含的 Skills

| Skill | 触发方式 | 功能 |
|-------|---------|------|
| `xhs-ck` | 输入"xhs-ck"或"查看小红书动态" | 抓取小红书账号近3个月笔记，生成 Word 报告 |
| `weibo-ck` | 输入"weibo-ck"或"查看微博动态" | 抓取微博账号近3个月帖子，生成 Word 报告 |

## 依赖

- [小红书 MCP Server](https://github.com/your-xhs-mcp-repo)（xhs-ck 需要）
- [Chrome DevTools MCP](https://github.com/your-chrome-mcp-repo)（weibo-ck 需要）
- Node.js + `docx` npm 包（生成 Word 文件）

```bash
npm install docx
```

## 安装

### 方法一：克隆后手动复制

```bash
git clone https://github.com/your-username/social-monitor-plugin.git

# macOS / Linux
cp -r social-monitor-plugin ~/.claude/plugins/social-monitor

# Windows（PowerShell）
Copy-Item -Recurse social-monitor-plugin $env:USERPROFILE\.claude\plugins\social-monitor
```

### 方法二：直接在 plugins 目录克隆

```bash
# macOS / Linux
cd ~/.claude/plugins
git clone https://github.com/your-username/social-monitor-plugin.git social-monitor

# Windows（PowerShell）
cd $env:USERPROFILE\.claude\plugins
git clone https://github.com/your-username/social-monitor-plugin.git social-monitor
```

## 配置

安装后必须编辑两个 SKILL.md 文件，填入你自己的账号信息：

### 1. 配置小红书监控账号

编辑 `~/.claude/plugins/social-monitor/skills/xhs-ck/SKILL.md`：

```markdown
## 监控账号列表

| 昵称 | userId | 搜索关键词 |
|------|--------|------------|
| 你的账号1 | `5655ecbe50c4b41526f41339` | 账号名称 |
| 你的账号2 | `604db41b00000000010031e0` | 账号名称 |

## 输出目录

所有 Word 文件保存到：`C:/Users/yourname/Documents/reports/`
```

> **如何找 userId**：在小红书 App 或网页版打开目标账号主页，URL 中 `/user/profile/` 后面的字符串即为 userId。

### 2. 配置微博监控账号

编辑 `~/.claude/plugins/social-monitor/skills/weibo-ck/SKILL.md`：

```markdown
## 监控账号列表

| 昵称 | userId（微博uid） | 简介 |
|------|-----------------|------|
| 你的账号1 | `1548718464` | 备注 |
| 你的账号2 | `1881976852` | 备注 |

## 输出目录

所有 Word 文件保存到：`C:/Users/yourname/Documents/reports/`
```

> **如何找微博uid**：打开目标账号微博主页，URL 格式为 `weibo.com/u/{uid}`，或通过第三方工具查询。

## 使用

配置完成后，在 Claude Code 中直接输入触发词即可：

```
xhs-ck
```

```
weibo-ck
```

Claude 会自动按顺序处理所有监控账号，生成 Word 文件到指定目录。

## 输出示例

- `某账号近3个月动态.docx`（小红书）
- `某账号_微博近3个月动态报告.docx`（微博）

每份报告包含：
- 账号基本信息（粉丝数、简介等）
- 近3个月发布内容列表
- 综合分析（发布节奏、内容主题、互动表现）
