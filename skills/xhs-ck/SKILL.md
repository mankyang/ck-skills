---
name: xhs-ck
description: 每月查看小红书监控账号的近3个月动态并生成Word报告。当用户输入"xhs-ck"、"查看小红书动态"、"生成小红书月报"、或提到要查看监控账号动态时，必须立即触发此 skill，按照标准流程依次处理所有账号。
version: 2.0.0
---

# 小红书动态监控 (xhs-ck)

## 监控账号列表

| 昵称 | userId |
|------|--------|
| 漂漂酱 | `5655ecbe50c4b41526f41339` |
| 胡盛呢 | `604db41b00000000010031e0` |
| M赵路 | `5c5af96b000000001100ef0f` |
| 魔都小蔚 | `59618ebf5e87e712999c195b` |
| 青青baby | `59ba17895e87e77c900d850d` |

## 输出目录

`C:/Users/yangc/Documents/Agent001/`

文件命名格式：`小红书动态_{查询日期}.xlsx`

---

## 完整工作流程

### 第一步：确认查询日期

用户会告知要查看的日期（如"3月10日"），记录为 `targetDate`（格式 `YYYY-MM-DD`）。

如果用户没有指定日期，询问用户要查看哪一天的内容。

---

### 第二步：确认浏览器登录状态

调用 `list_pages` 检查是否有小红书页面。

若无，调用 `new_page` 打开 `https://www.xiaohongshu.com`，调用 `take_screenshot` 确认：
- ✅ 已登录（显示首页内容）→ 继续
- ❌ 未登录（显示登录页）→ 提示用户在浏览器中手动登录，等待确认后继续

---

### 第三步：逐账号抓取数据

对每个账号依次执行：

#### 3a. 打开账号主页

```
navigate_page → https://www.xiaohongshu.com/user/profile/{userId}
```

等待页面加载完成。

#### 3b. 提取笔记列表

调用 `evaluate_script` 提取页面中所有笔记链接及日期：

```javascript
() => {
  const links = Array.from(document.querySelectorAll('a[href*="/explore/"]'));
  const notes = [];
  const seen = new Set();
  links.forEach(el => {
    const href = el.getAttribute('href') || '';
    const match = href.match(/\/explore\/([a-f0-9]{24})/);
    if (match && !seen.has(match[1])) {
      seen.add(match[1]);
      const noteId = match[1];
      const ts = parseInt(noteId.substring(0, 8), 16);
      const date = new Date(ts * 1000).toISOString().slice(0, 10);
      const title = el.querySelector('span')?.innerText?.trim() || '';
      notes.push({ noteId, date, title });
    }
  });
  return notes;
}
```

> **原理**：小红书笔记 ID 前8位十六进制 = Unix 时间戳（秒），直接换算为日期，无需额外接口。

#### 3c. 筛选目标日期

从结果中筛选 `date === targetDate` 的笔记。

如果页面加载的笔记不够（最新笔记日期仍早于 targetDate），说明该账号在目标日期无发布，直接跳过。

#### 3c-2. 提取互动内容（评论区）

在笔记列表页后，还需检查账号的**近期互动**——即该账号在他人笔记下发表的评论。

> 小红书个人主页不直接展示评论记录，需通过以下方式补充：
> - 若账号有"近期评论"入口（部分版本可见），点击进入提取
> - 若无法直接获取评论记录，在备注列标注"评论记录不可见"

#### 3d. 记录结果

将该账号筛选到的数据记录入汇总列表，区分两类：

**发布内容（原创笔记）：**
- 昵称、发布日期、内容类型（`原创笔记`）、笔记标题、点赞数、评论数、收藏数、笔记链接

**互动内容（评论/回复）：**
- 昵称、互动日期、内容类型（`评论`）、评论内容摘要、被评论笔记标题、被评论笔记链接

> 若评论记录不可见，该账号互动内容行填"评论记录不可见"。

---

### 第三步补充：提取笔记互动数据

进入各笔记详情页可获取互动数（点赞/评论/收藏），但为避免逐篇打开页面耗时过长，优先从列表页提取可见数据。

若列表页已返回互动数，直接使用：

```javascript
() => {
  const links = Array.from(document.querySelectorAll('a[href*="/explore/"]'));
  const notes = [];
  const seen = new Set();
  links.forEach(el => {
    const href = el.getAttribute('href') || '';
    const match = href.match(/\/explore\/([a-f0-9]{24})/);
    if (match && !seen.has(match[1])) {
      seen.add(match[1]);
      const noteId = match[1];
      const ts = parseInt(noteId.substring(0, 8), 16);
      const date = new Date(ts * 1000).toISOString().slice(0, 10);
      const title = el.querySelector('span')?.innerText?.trim() || '';
      // 互动数（从卡片底部提取）
      const card = el.closest('section, [class*="note"], [class*="card"]');
      const likes = card?.querySelector('[class*="like"], [class*="heart"]')?.innerText?.trim() || '';
      const comments = card?.querySelector('[class*="comment"]')?.innerText?.trim() || '';
      notes.push({ noteId, date, title, likes, comments });
    }
  });
  return notes;
}
```

---

### 第四步：生成 Excel 表格

所有账号处理完毕后，创建 `C:/Users/yangc/Documents/Agent001/gen_xhs_{日期}.js`，运行后删除。

#### Excel 结构

- Sheet 名：`小红书动态`
- 列：`昵称` | `日期` | `内容类型` | `标题/内容` | `点赞` | `评论` | `收藏` | `链接`
- 如某账号当日无任何内容，添加一行：昵称填账号名，其余列填 `（当日无动态）`

#### 生成代码模板

```javascript
const ExcelJS = require('exceljs');
const path = require('path');

async function main() {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet('小红书动态');

  ws.columns = [
    { header: '昵称',     key: 'name',    width: 16 },
    { header: '日期',     key: 'date',    width: 14 },
    { header: '内容类型', key: 'type',    width: 12 },
    { header: '标题/内容',key: 'title',   width: 48 },
    { header: '点赞',     key: 'likes',   width: 8  },
    { header: '评论',     key: 'comments',width: 8  },
    { header: '收藏',     key: 'collects',width: 8  },
    { header: '链接',     key: 'url',     width: 55 },
  ];

  // 表头样式
  ws.getRow(1).eachCell(cell => {
    cell.fill = { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FF2C5F8A' } };
    cell.font = { bold: true, color: { argb: 'FFFFFFFF' }, name: '微软雅黑', size: 11 };
    cell.alignment = { vertical: 'middle', horizontal: 'center' };
  });
  ws.getRow(1).height = 22;

  // 数据行（用 rows 数组替换）
  const rows = [
    // 原创笔记示例：
    // { name: '漂漂酱', date: '2026-03-10', type: '原创笔记', title: '...', likes: '1.2万', comments: '320', collects: '890', url: 'https://...' },
    // 评论示例：
    // { name: '漂漂酱', date: '2026-03-10', type: '评论', title: '回复@某人：内容摘要...', likes: '', comments: '', collects: '', url: 'https://...' },
    // 无动态示例：
    // { name: '胡盛呢', date: '（当日无动态）', type: '', title: '', likes: '', comments: '', collects: '', url: '' },
  ];

  rows.forEach((r, i) => {
    const row = ws.addRow(r);
    row.height = 18;
    const fill = i % 2 === 0
      ? { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FFFFFFFF' } }
      : { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FFF0F5FA' } };
    row.eachCell(cell => {
      cell.fill = fill;
      cell.font = { name: '微软雅黑', size: 10 };
      cell.alignment = { vertical: 'middle', wrapText: true };
    });
    if (r.url && r.url.startsWith('http')) {
      const urlCell = row.getCell('url');
      urlCell.value = { text: r.url, hyperlink: r.url };
      urlCell.font = { name: '微软雅黑', size: 10, color: { argb: 'FF0563C1' }, underline: true };
    }
  });

  // 冻结首行
  ws.views = [{ state: 'frozen', ySplit: 1 }];

  const outPath = path.join('C:/Users/yangc/Documents/Agent001', `小红书动态_TARGET_DATE.xlsx`);
  await wb.xlsx.writeFile(outPath);
  console.log('已生成：' + outPath);
}

main().catch(console.error);
```

> 将 `rows` 数组替换为实际数据，`TARGET_DATE` 替换为目标日期（如 `2026-03-10`）。

---

## 重要注意事项

### 中文引号陷阱

笔记标题中如含中文弯引号 `"` `"`，生成 JS 脚本时**必须用反引号**包裹字符串：

```javascript
// ✅ 正确
title: `东方人认为的"高级感"`,
```

### 账号无内容的处理

如该账号在目标日期无发布内容，Excel 中该行记录为：

| 昵称 | 发布日期 | 笔记标题 | 笔记链接 |
|------|---------|---------|---------|
| 漂漂酱 | （当日无发布） | | |

---

## 处理完成后

汇总告知用户：
- 生成的文件路径
- 各账号在目标日期发布的笔记数量
- 如有账号跳过，说明原因
