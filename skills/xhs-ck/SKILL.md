---
name: xhs-ck
description: 每月查看小红书监控账号的近3个月动态并生成Word报告。当用户输入"xhs-ck"、"查看小红书动态"、"生成小红书月报"、或提到要查看监控账号动态时，必须立即触发此 skill，按照标准流程依次处理所有账号。
version: 1.0.0
---

# 小红书月度动态监控 (xhs-ck)

## 配置说明

> ⚠️ 使用前必须修改以下两处配置：
> 1. **监控账号列表** — 替换为你自己要监控的小红书账号
> 2. **输出目录** — 替换为你本地的文件保存路径

---

## 监控账号列表

> 修改此表格，填入你要监控的账号。userId 在小红书 App 个人主页 URL 中可找到。

| 昵称 | userId | 搜索关键词 |
|------|--------|------------|
| 账号昵称1 | `请填入小红书userId` | 搜索用关键词 |
| 账号昵称2 | `请填入小红书userId` | 搜索用关键词 |

## 输出目录

所有 Word 文件保存到：`YOUR_OUTPUT_DIR`

> 示例：`C:/Users/yourname/Documents/reports/`

文件命名格式：`{昵称}近3个月动态.docx`

---

## 完整工作流程

### 第一步：检查登录状态

调用 `check_login_status`。

- ✅ 已登录 → 直接进入第二步
- ❌ 未登录 → 告知用户需要重新登录，调用 `get_login_qrcode`，等待用户确认登录成功后继续

> 登录失效是常见情况，不要跳过检查。

---

### 第二步：逐账号处理

对每个账号依次执行以下操作：

#### 2a. 获取 xsec_token

调用 `search_feeds`，keyword 填该账号的搜索关键词，从返回结果中找到匹配该 userId 的条目，提取其 `xsecToken` 字段。

> xsec_token 是 session 级别的访问令牌，每次都需要从搜索结果中新获取，不能复用旧值。

如果搜索结果中找不到目标账号：
- 换别的关键词再搜一次（如用小红书号、全名等）
- 仍然找不到则跳过该账号并告知用户，继续处理下一个

#### 2b. 获取用户资料

调用 `user_profile`，传入 `user_id`（用监控列表中存储的值）和上一步获取的 `xsec_token`。

返回数据可能很大（超出预览），此时完整数据会自动保存到临时 JSON 文件，用 `node -e` 脚本解析。

#### 2c. 解析近3个月笔记

**3个月截止时间计算：**

```javascript
// 今天往前推3个月
const cutoff = new Date();
cutoff.setMonth(cutoff.getMonth() - 3);
const cutoffTs = cutoff.getTime() / 1000;
```

**从笔记 ID 提取发布时间戳（关键技巧）：**

```javascript
// 笔记 ID 的前8位十六进制字符 = Unix 时间戳（秒）
const ts = parseInt(noteId.substring(0, 8), 16);
const date = new Date(ts * 1000);
```

筛选出时间戳 ≥ cutoff 的笔记，提取：日期、标题、类型（视频/图文）、点赞数、评论数、收藏数。

#### 2d. 生成 Word 文件

根据收集到的数据，在输出目录下创建 `gen_{昵称}.js`，运行后删除。

---

## Word 文档格式规范

### 页面设置

```javascript
// A4 竖向，四周 1 inch 边距
page: {
  size: { width: 11906, height: 16838 },
  margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 }
}
// 内容宽度 = 11906 - 1440*2 = 9026，但习惯用 9360（可微调）
```

### 颜色方案

| 用途 | 色值 |
|------|------|
| 表头背景 | `2C5F8A`（深蓝） |
| 表头文字 | `FFFFFF`（白） |
| 奇数行 | `FFFFFF`（白） |
| 偶数行 | `F0F5FA`（浅蓝灰） |
| 正文文字 | `333333` |
| 警告文字 | `C0392B`（红） |
| 章节标题 | `2C5F8A` |
| 副标题 | `888888` |

### 字体

全文使用 `微软雅黑`。

### 文档结构

1. **主标题**：`{昵称}小红书近3个月动态报告`（居中，40pt，深蓝 `1A3A5C`，粗体）
2. **副标题**：`博主：{昵称}  |  统计区间：{起始日期} ~ {今日}`（居中，22pt，灰色）
3. **第一节：用户基本信息**（section 标题 + 信息表格）
   - 信息表格：2列，左列为粗体标签（宽2000），右列为值（宽7360）
   - 包含：昵称、小红书号、所在地、简介/内容方向、关注数、粉丝数、获赞与收藏、历史总笔记
4. **第二节：近3个月发布笔记（共N篇）**（section 标题 + 笔记表格）
   - 笔记表格列：发布日期、笔记标题、类型、点赞、评论
   - 如有特殊情况（如长期停更、笔记数很少）用红色警告文字标注
5. **第三节：综合分析**（section 标题 + 分析内容）
   - 分析子项：发布节奏、内容主题、互动表现
   - 每个子项用 `▌ 发布节奏` 格式的小标题 + 要点列表（bullet numbering）

### 页眉/页脚

```javascript
// 页眉：右对齐，带下边线
`小红书动态报告 · {昵称}`

// 页脚：居中
`第 {页码} 页  ·  数据采集时间：{今日}`
```

### 辅助函数模板

```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
  AlignmentType, BorderStyle, WidthType, ShadingType, VerticalAlign,
  LevelFormat, Header, Footer, PageNumber } = require('docx');
const fs = require('fs');

const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };
const headerBorder = { style: BorderStyle.SINGLE, size: 1, color: "2C5F8A" };
const headerBorders = { top: headerBorder, bottom: headerBorder, left: headerBorder, right: headerBorder };

function cell(text, options = {}) {
  return new TableCell({
    borders: options.header ? headerBorders : borders,
    width: { size: options.width || 2000, type: WidthType.DXA },
    shading: options.header
      ? { fill: "2C5F8A", type: ShadingType.CLEAR }
      : options.alt
      ? { fill: "F0F5FA", type: ShadingType.CLEAR }
      : { fill: "FFFFFF", type: ShadingType.CLEAR },
    margins: { top: 100, bottom: 100, left: 120, right: 120 },
    verticalAlign: VerticalAlign.CENTER,
    children: [new Paragraph({
      alignment: options.center ? AlignmentType.CENTER : AlignmentType.LEFT,
      children: [new TextRun({
        text,
        bold: options.header || options.bold,
        color: options.header ? "FFFFFF" : options.warn ? "C0392B" : "333333",
        size: options.header ? 22 : 20,
        font: "微软雅黑"
      })]
    })]
  });
}

function sectionTitle(text) {
  return new Paragraph({
    spacing: { before: 320, after: 160 },
    border: { bottom: { style: BorderStyle.SINGLE, size: 4, color: "2C5F8A", space: 4 } },
    children: [new TextRun({ text, bold: true, size: 28, color: "2C5F8A", font: "微软雅黑" })]
  });
}

function infoRow(label, value) {
  return new TableRow({
    children: [
      cell(label, { width: 2000, bold: true }),
      cell(value, { width: 7360 }),
    ]
  });
}
```

---

## 重要注意事项

### 中文引号陷阱

标题、文案中如含有中文弯引号 `"` `"`，**必须**把该字符串改用单引号包裹：

```javascript
// ❌ 错误（中文引号会破坏 JS 字符串解析）
"东方人认为的"高级感""

// ✅ 正确
'东方人认为的"高级感"'
```

### 笔记数量极少的处理

如果近3个月笔记数 < 3 篇，或存在长期断更（>30天无内容），需要：
- 在笔记表格上方添加红色警告说明
- 在分析节"发布节奏"中明确指出停更时长

### 大文件解析示例

当 user_profile 返回数据过大，用以下脚本解析：

```javascript
// node -e "..."
const data = JSON.parse(require('fs').readFileSync('PATH_TO_JSON', 'utf8'));
const notes = data.notes || data.user?.notes || [];
const cutoff = Math.floor(new Date(new Date().setMonth(new Date().getMonth()-3)).getTime()/1000);
const recent = notes.filter(n => parseInt((n.id||n.note_id||'').substring(0,8),16) >= cutoff);
console.log(JSON.stringify(recent.map(n => ({
  id: n.id||n.note_id,
  title: n.title||n.display_title,
  type: n.type,
  likes: n.interact_info?.liked_count || n.liked_count || 0,
  comments: n.interact_info?.comment_count || n.comment_count || 0,
  date: new Date(parseInt((n.id||n.note_id).substring(0,8),16)*1000).toISOString().slice(0,10)
})), null, 2));
```

---

## 处理完成后

所有账号处理完毕后，汇总告知用户：
- 每个账号生成的文件路径
- 各账号近3个月笔记数量
- 如有跳过的账号，说明原因
