# Social Monitor Plugin for Claude Code

监控小红书和微博账号的指定日期动态，通过 Chrome 浏览器抓取数据，生成 Excel 表格。

## 包含的 Skills

| Skill | 触发方式 | 功能 |
|-------|---------|------|
| `xhs-ck` | 输入"xhs-ck"或"查看小红书动态" | 抓取小红书账号指定日期的发布和互动内容，生成 Excel 表格 |
| `weibo-ck` | 输入"weibo-ck"或"查看微博动态" | 抓取微博账号指定日期的发布和互动内容，生成 Excel 表格 |

## 前置依赖

### Node.js + exceljs

```bash
npm install exceljs
```

### Chrome 浏览器

需要通过 Chrome DevTools MCP 控制浏览器。确保 Claude Code 已配置 `chrome-devtools` MCP server。

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

安装后编辑两个 SKILL.md，填入你的账号和输出路径：

### 小红书（xhs-ck）

编辑 `skills/xhs-ck/SKILL.md`，替换监控账号列表和输出目录：

```markdown
## 监控账号列表

| 昵称 | userId |
|------|--------|
| 账号昵称1 | `请填入小红书userId` |
| 账号昵称2 | `请填入小红书userId` |

## 输出目录

所有文件保存到：`YOUR_OUTPUT_DIR`
```

> userId 在小红书 App 个人主页 URL 中可找到。

### 微博（weibo-ck）

编辑 `skills/weibo-ck/SKILL.md`，替换监控账号列表和输出目录：

```markdown
## 监控账号列表

| 昵称 | userId（微博uid） |
|------|-----------------|
| 账号昵称1 | `请填入微博uid` |
| 账号昵称2 | `请填入微博uid` |

## 输出目录

所有文件保存到：`YOUR_OUTPUT_DIR`
```

> 微博 uid 可在账号主页 URL `weibo.com/u/{uid}` 中找到。

## 使用

配置完成后，在 Claude Code 中输入触发词并告知要查看的日期：

```
xhs-ck 查看3月10日
```
```
weibo-ck 查看3月10日
```

Claude 会自动：
1. 用 Chrome 浏览器逐账号访问主页
2. 提取指定日期的发布和互动内容
3. 生成 Excel 表格

## 输出

- `小红书动态_{日期}.xlsx`
- `微博动态_{日期}.xlsx`

每份表格包含：

| 列 | 说明 |
|----|------|
| 昵称 | 账号名称 |
| 日期 | 发布/互动日期 |
| 内容类型 | 原创笔记 / 评论（小红书）；原创 / 转发 / 评论（微博） |
| 标题/内容 | 笔记标题或内容摘要 |
| 互动数据 | 点赞、评论、收藏/转发数 |
| 链接 | 原帖链接（可点击） |

## 更新日志

### v3.0.0
- 数据采集改用 Chrome DevTools，直接操控浏览器，无需额外 MCP 或爬虫依赖
- 输出格式从 Word 报告改为 Excel 表格（exceljs）
- 支持按指定日期筛选内容
- 新增互动内容字段：区分原创、转发、评论三种类型
- 移除 douyin-ck skill

### v1.0.0
- 初始版本，基于 XHS MCP + Chrome DevTools MCP 实时抓取，输出 Word 报告
