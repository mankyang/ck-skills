---
name: weibo-ck
description: 每月查看微博监控账号的近3个月动态并生成Word报告。当用户输入"weibo-ck"、"查看微博动态"、"生成微博月报"、或提到要查看微博监控账号动态时，必须立即触发此 skill，按照标准流程依次处理所有账号。
version: 2.0.0
---

# 微博动态监控 (weibo-ck)

## 监控账号列表

| 昵称 | userId（微博uid） |
|------|-----------------|
| 主持人郭灿亮 | `1548718464` |
| M赵路 | `1881976852` |
| 胡盛呢 | `1680206441` |
| 小泠是ling不是leng | `5997250222` |
| 青鸢小妞子 | `1865587582` |
| 杨越VJ | `1000891302` |
| 小青青88 | `1877781754` |
| 言午日月 | `1463765784` |
| 门芯羽 | `1840892682` |
| 咬咬ayao | `1818134777` |
| 童鑫Aka_B-Rabbit | `1989026297` |
| 王燕华EVA | `1887126080` |
| 赵菁candyjj | `1720896651` |
| 范鸣迅 | `2815750065` |
| Jessie杨柳 | `1805029353` |
| 魔都小蔚99 | `1833890324` |

## 输出目录

`C:/Users/yangc/Documents/Agent001/`

文件命名格式：`微博动态_{查询日期}.xlsx`

---

## 完整工作流程

### 第一步：确认查询日期

用户会告知要查看的日期（如"3月10日"），记录为 `targetDate`（格式 `YYYY-MM-DD`）。

如果用户没有指定日期，询问用户要查看哪一天的内容。

---

### 第二步：确认浏览器登录状态

调用 `list_pages` 检查是否有微博页面。

若无，调用 `new_page` 打开 `https://weibo.com`，调用 `take_screenshot` 确认：
- ✅ 已登录（显示首页 feed、头像）→ 继续
- ❌ 未登录（显示登录页）→ 提示用户在浏览器中手动登录，等待确认后继续

---

### 第三步：逐账号抓取数据

对每个账号依次执行：

#### 3a. 打开账号主页

```
navigate_page → https://weibo.com/u/{userId}
```

等待页面加载（调用 `take_screenshot` 确认已显示账号内容）。

#### 3b. 滚动提取帖子

调用 `evaluate_script` 提取当前可见帖子：

```javascript
() => {
  const posts = [];
  const seen = new Set();
  document.querySelectorAll('article, [class*="Feed_body"], [class*="wbpro-feed-content"]').forEach(el => {
    const text = el.innerText?.trim().slice(0, 200) || '';
    if (!text || seen.has(text.slice(0, 50))) return;
    seen.add(text.slice(0, 50));
    const timeEl = el.querySelector('time, a[class*="time"], [class*="time"] a');
    const datetime = timeEl?.getAttribute('datetime') || timeEl?.innerText?.trim() || '';
    const link = el.querySelector('a[href*="/detail/"]')?.getAttribute('href') || '';
    posts.push({ text, datetime, link });
  });
  return posts;
}
```

**滚动策略：** 每次滚动 2000px，最多滚动 30 次，满足以下任一条件停止：
- 找到目标日期的帖子
- 最早帖子日期已早于 targetDate 超过 3 天（确认无遗漏后停止）
- 连续 3 轮无新帖子出现

滚动脚本：
```javascript
() => { window.scrollBy(0, 2000); return window.scrollY; }
```

#### 3c. 解析帖子日期

微博时间格式多样，统一转换为 `YYYY-MM-DD`：

| 显示格式 | 转换方法 |
|---------|---------|
| `2026-03-10 22:08` | 直接截取前10位 |
| `3月10日 22:08` | 结合当前年份拼接 |
| `昨天 18:00` | 今日日期 -1 天 |
| `X小时前` | 当前时间 -X 小时后取日期 |
| `X天前` | 当前日期 -X 天 |
| `刚刚` | 当前日期 |

#### 3d. 筛选目标日期并区分内容类型

从提取结果中筛选 `date === targetDate` 的帖子，按类型区分：

**原创微博：** 正文无转发标记（不以"//[@]"或"转发微博"开头）
- 昵称、日期、内容类型（`原创`）、内容摘要、转发数、评论数、点赞数、帖子链接

**转发微博：** 正文包含转发标记（`//` 引用原文）
- 昵称、日期、内容类型（`转发`）、转发评论内容、被转发账号、原文摘要、帖子链接

**互动（评论）：** 若微博主页"评论"tab可见，提取当日评论记录
- 昵称、日期、内容类型（`评论`）、评论内容、被评论博主、被评论微博摘要、链接

> 微博个人主页通常只展示原创和转发，评论记录一般不公开可见。若无法获取，标注"评论记录不可见"。

---

### 第四步：生成 Excel 表格

所有账号处理完毕后，创建 `C:/Users/yangc/Documents/Agent001/gen_weibo_{日期}.js`，运行后删除。

#### Excel 结构

- Sheet 名：`微博动态`
- 列：`昵称` | `日期` | `内容类型` | `内容摘要` | `转发` | `评论` | `点赞` | `帖子链接`
- 如某账号当日无任何动态，添加一行：昵称填账号名，其余列填 `（当日无动态）`

#### 生成代码模板

```javascript
const ExcelJS = require('exceljs');
const path = require('path');

async function main() {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet('微博动态');

  ws.columns = [
    { header: '昵称',     key: 'name',    width: 18 },
    { header: '日期',     key: 'date',    width: 14 },
    { header: '内容类型', key: 'type',    width: 10 },
    { header: '内容摘要', key: 'content', width: 52 },
    { header: '转发',     key: 'repost',  width: 8  },
    { header: '评论',     key: 'comment', width: 8  },
    { header: '点赞',     key: 'like',    width: 8  },
    { header: '帖子链接', key: 'url',     width: 50 },
  ];

  // 表头样式
  ws.getRow(1).eachCell(cell => {
    cell.fill = { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FF1F4E79' } };
    cell.font = { bold: true, color: { argb: 'FFFFFFFF' }, name: '微软雅黑', size: 11 };
    cell.alignment = { vertical: 'middle', horizontal: 'center' };
  });
  ws.getRow(1).height = 22;

  // 数据行（用 rows 数组替换）
  const rows = [
    // 原创示例：
    // { name: '主持人郭灿亮', date: '2026-03-10', type: '原创', content: '...', repost: '12', comment: '34', like: '560', url: 'https://weibo.com/...' },
    // 转发示例：
    // { name: 'M赵路', date: '2026-03-10', type: '转发', content: '转发评论 // @博主名: 原文摘要...', repost: '3', comment: '8', like: '45', url: 'https://weibo.com/...' },
    // 无动态示例：
    // { name: '胡盛呢', date: '（当日无动态）', type: '', content: '', repost: '', comment: '', like: '', url: '' },
  ];

  rows.forEach((r, i) => {
    const row = ws.addRow(r);
    row.height = 18;
    const fill = i % 2 === 0
      ? { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FFFFFFFF' } }
      : { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FFF7FBFF' } };
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

  ws.views = [{ state: 'frozen', ySplit: 1 }];

  const outPath = path.join('C:/Users/yangc/Documents/Agent001', `微博动态_TARGET_DATE.xlsx`);
  await wb.xlsx.writeFile(outPath);
  console.log('已生成：' + outPath);
}

main().catch(console.error);
```

> 将 `rows` 数组替换为实际数据，`TARGET_DATE` 替换为目标日期（如 `2026-03-10`）。

---

## 重要注意事项

### 中文引号陷阱

帖子内容中如含中文弯引号 `"` `"`，生成 JS 脚本时**必须用反引号**包裹字符串：

```javascript
// ✅ 正确
content: `他说"这场比赛很精彩"`,
```

### 账号无内容的处理

如该账号在目标日期无发布内容，Excel 中记录为：

| 昵称 | 发布日期 | 内容摘要 | 帖子链接 |
|------|---------|---------|---------|
| 主持人郭灿亮 | （当日无发布） | | |

### 页面懒加载

微博主页懒加载较慢，每次滚动后等待页面稳定再提取。若帖子数量没有增加，可多滚动一次再试。

---

## 处理完成后

汇总告知用户：
- 生成的文件路径
- 各账号在目标日期发布的帖子数量
- 如有账号跳过，说明原因
