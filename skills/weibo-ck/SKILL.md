---
name: weibo-ck
description: 每月查看微博监控账号的近3个月动态并生成Word报告。当用户输入"weibo-ck"、"查看微博动态"、"生成微博月报"、或提到要查看微博监控账号动态时，必须立即触发此 skill，按照标准流程依次处理所有账号。
version: 1.0.0
---

# 微博月度动态监控 (weibo-ck)

## 配置说明

> ⚠️ 使用前必须修改以下两处配置：
> 1. **监控账号列表** — 替换为你自己要监控的微博账号
> 2. **输出目录** — 替换为你本地的文件保存路径

---

## 监控账号列表

> 修改此表格，填入你要监控的微博账号。userId（微博uid）可在该账号主页 URL `weibo.com/u/{uid}` 中找到。

| 昵称 | userId（微博uid） | 简介 |
|------|-----------------|------|
| 账号昵称1 | `请填入微博uid` | 备注说明 |
| 账号昵称2 | `请填入微博uid` | 备注说明 |

## 输出目录

所有 Word 文件保存到：`YOUR_OUTPUT_DIR`

> 示例：`C:/Users/yourname/Documents/reports/`

文件命名格式：`{昵称}_微博近3个月动态报告.docx`

---

## 完整工作流程

### 第一步：确认浏览器与登录状态

1. 调用 `list_pages` 检查浏览器是否已打开
2. 如果没有微博页面，调用 `new_page` 打开 `https://weibo.com`
3. 等待页面加载，调用 `take_snapshot` 判断是否已登录：
   - ✅ 已登录（页面显示首页 feed、头像等）→ 直接进入第二步
   - ❌ 未登录（显示登录页或注册引导）→ 调用 `navigate_page` 前往 `https://passport.weibo.com/sso/signin`，提示用户扫码登录，等待用户确认后继续

> 登录状态会过期，不要跳过检查。

---

### 第二步：逐账号处理

对每个账号依次执行以下操作：

#### 2a. 前往账号主页

调用 `navigate_page`，URL 为：`https://weibo.com/u/{userId}`

等待页面加载（`wait_for` 等待账号昵称文字出现）。

#### 2b. 获取账号基本信息

调用 `take_snapshot` 或 `evaluate_script` 提取以下信息：
- 昵称、粉丝数、关注数、简介、IP属地
- 如有视频播放量、转评赞总计也一并记录

```javascript
// 提取账号基本信息
(() => {
  const name = document.querySelector('.ProfileHeader_name_1KbBs, [class*="name"]')?.innerText?.trim();
  const fans = document.querySelector('[class*="fans"] span, [class*="follower"] span')?.innerText?.trim();
  const follow = document.querySelector('[class*="follow"] span')?.innerText?.trim();
  const desc = document.querySelector('[class*="desc"], [class*="intro"]')?.innerText?.trim();
  return { name, fans, follow, desc };
})()
```

如果选择器失效，改用 `take_snapshot` 手动从快照文本中提取。

#### 2c. 滚动收集近3个月微博

**日期截止计算：**

```javascript
// 3个月前的日期
const cutoff = new Date();
cutoff.setMonth(cutoff.getMonth() - 3);
```

**滚动策略：**

每轮执行 `evaluate_script` 滚动页面并提取帖子，循环直到满足停止条件：

```javascript
// 滚动并提取帖子
(() => {
  window.scrollBy(0, 2000);
  const posts = [];
  document.querySelectorAll('article').forEach(el => {
    const text = el.innerText;
    const timeEl = el.querySelector('time, [class*="time"], a[href*="detail"]');
    posts.push({ text: text.slice(0, 300), time: timeEl?.getAttribute('datetime') || timeEl?.innerText });
  });
  return posts;
})()
```

**停止条件（满足任一即停）：**
- 连续滚动 3 轮都没有出现新帖子
- 最早帖子的日期已早于截止日期（3个月前）
- 总滚动次数超过 60 次（防止无限循环）

**帖子去重：** 按帖子文本前50字符去重，避免重复收录懒加载中的重复条目。

#### 2d. 解析帖子数据

对收集到的帖子文本，提取：
- **发布日期**：解析 `datetime` 属性或帖子内时间文字（如"3月2日 22:08"、"1小时前"等）
  - 若显示"X小时前"/"昨天"/"X天前"，换算为具体日期
- **内容摘要**：取正文前 100 字，去掉转发标记和多余换行
- **互动数据**：转发数、评论数、点赞数（从帖子底部操作栏提取，格式通常为"转发 12 评论 34 赞 56"）

筛选：只保留发布日期 ≥ 截止日期的帖子。

#### 2e. 生成 Word 文件

根据收集数据，使用 `docx` npm 库生成 Word 文件：

1. 创建临时 JS 脚本 `{YOUR_OUTPUT_DIR}/gen_{昵称}.js`
2. 用 `node gen_{昵称}.js` 运行
3. 验证文件生成成功后删除临时脚本

---

## Word 文档格式规范

### 页面设置

```javascript
// A4 竖向，四周 1 inch 边距
page: {
  size: { width: 11906, height: 16838 },
  margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 }
}
// 内容宽度 = 9026 DXA
```

### 颜色方案

| 用途 | 色值 |
|------|------|
| 主标题 | `1F4E79`（深海蓝） |
| 副标题/章节 | `2E75B6`（中蓝） |
| 表头背景 | `1F4E79` |
| 表头文字 | `FFFFFF` |
| 标签列背景 | `D5E8F0`（浅蓝） |
| 偶数行 | `F7FBFF` |
| 正文文字 | `000000` |
| 副文字 | `888888` |
| 警告文字 | `C0392B`（红） |

### 字体

全文使用 `Arial`。

### 文档结构

1. **主标题**：`{昵称}` （居中，52pt，深蓝 `1F4E79`，粗体）
2. **副标题**：`微博近3个月动态报告`（居中，34pt，中蓝 `2E75B6`）
3. **统计区间**：`统计周期：{3个月前日期} — {今日}`（居中，21pt，灰色 `888888`）
4. **分隔线**（Paragraph 底部边框）
5. **第一节：一、账号基本信息**（h1 标题 + 2列信息表格）
   - 表格：左列标签宽 2256，右列内容宽 6770
   - 包含：账号昵称、账号类型、粉丝数、关注数、转评赞总计、IP属地、视频累计播放（如有）
6. **第二节：二、内容主题分布**（h1 标题 + 3列主题表格）
   - 列：主题 | 内容描述 | 占比
   - 根据帖子内容归纳 3-5 个主题分类
7. **第三节：三、近3个月详细动态**（h1 标题 + 按月 h2 小节 + bullet 列表）
   - 按月倒序排列（最新月份在前）
   - 每条 bullet 格式：`{月日 时间} — {内容摘要}`
8. **分隔线**
9. **第四节：四、总结**（h1 标题 + 段落正文）
   - 2-3 段综合分析
   - 末行关键词（灰色斜体）

### 页眉/页脚

```javascript
// 页眉：右对齐，带下边框线
`{昵称} 微博动态报告`

// 页脚：居中，页码 + 生成日期
`第 {页码} 页  |  生成日期：{今日}`
```

### 关键代码片段

```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
  HeadingLevel, AlignmentType, BorderStyle, WidthType, ShadingType,
  LevelFormat, Header, Footer, PageNumber } = require('docx');
const fs = require('fs');

const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

function cell(text, shading, bold = false, width = 4680) {
  return new TableCell({
    borders,
    width: { size: width, type: WidthType.DXA },
    shading: shading ? { fill: shading, type: ShadingType.CLEAR } : undefined,
    margins: { top: 80, bottom: 80, left: 150, right: 150 },
    children: [new Paragraph({
      children: [new TextRun({ text, bold, font: "Arial", size: 20 })]
    })]
  });
}

function bullet(text) {
  return new Paragraph({
    numbering: { reference: "bullets", level: 0 },
    spacing: { before: 40, after: 40 },
    children: [new TextRun({ text, font: "Arial", size: 21 })]
  });
}

function divider() {
  return new Paragraph({
    border: { bottom: { style: BorderStyle.SINGLE, size: 6, color: "CCCCCC", space: 1 } },
    children: [new TextRun("")]
  });
}
```

---

## 重要注意事项

### 中文引号陷阱

生成 JS 脚本时，如果帖子内容含有中文弯引号 `"` `"`，**必须**用单引号包裹该字符串，或用反引号（模板字符串），否则会破坏 JS 字符串语法：

```javascript
// 错误
"他说"这场比赛很精彩""

// 正确（单引号包裹）
'他说"这场比赛很精彩"'

// 正确（模板字符串）
`他说"这场比赛很精彩"`
```

实际操作：写 JS 脚本时，对所有 bullet/para 文本内容**统一使用反引号**（模板字符串），彻底规避此问题。

### 帖子时间解析

微博帖子的时间显示格式多样，需要处理：
- `2026-03-02 22:08` → 直接解析
- `3月2日 22:08` → 结合当前年份解析
- `昨天 18:00` → 今天日期 -1
- `X小时前` → 当前时间 -X 小时
- `X天前` → 当前日期 -X 天
- `刚刚` → 当前时间

### 帖子数量极少的处理

如果近3个月帖子数 < 5 条，或存在长期断更（> 30 天无内容），需要：
- 在第三节上方添加红色警告说明
- 在总结中明确指出更新频率低

### 页面懒加载

微博主页采用懒加载，滚动后需等待 1-1.5 秒再提取内容：

```javascript
// 每轮滚动后等待
await new Promise(r => setTimeout(r, 1200));
```

在 Claude 实际操作时，每次 `evaluate_script` 调用之间自然有等待时间，通常足够。但如果发现帖子列表没有增加，可以多滚动几次再试。

---

## 处理完成后

所有账号处理完毕后，汇总告知用户：
- 每个账号生成的文件路径
- 各账号近3个月帖子数量
- 如有跳过的账号，说明原因
