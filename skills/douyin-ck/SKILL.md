---
name: douyin-ck
description: 每月查看抖音监控账号的近3个月动态并生成Word报告。当用户输入"douyin-ck"、"查看抖音动态"、"生成抖音月报"、或提到要查看抖音监控账号动态时，必须立即触发此 skill，按照标准流程依次处理所有账号。
version: 1.0.0
---

# 抖音月度动态监控 (douyin-ck)

数据采集使用 [MediaCrawler](https://github.com/NanmiCoder/MediaCrawler)，Claude 负责读取数据并生成 Word 报告。

## 配置说明

> ⚠️ 使用前必须修改以下配置项

### 监控账号列表

> `sec_uid` 获取方式：用浏览器打开目标账号的抖音主页，复制 URL 中 `/user/` 后面的字符串（`MS4wLjAB` 开头），或直接填完整主页 URL。

| 昵称 | 完整主页 URL |
|------|------------|
| 英超解说郭灿亮 | `https://www.douyin.com/user/MS4wLjABAAAA-QZ8M4FoCiGQAi5lz9lPJTxb_Klt4vuXYGajE1cA0DyUtsCMiRpTo98IaObz29SU?from_tab_name=main` |
| M赵路 | `https://www.douyin.com/user/MS4wLjABAAAAx8K39beYBOq8wv--gjPFPcrJnBSQhw0jvvqzjPKTsy8?from_tab_name=main` |
| 胡盛呢 | `https://www.douyin.com/user/MS4wLjABAAAAcVKUoNcPwZtqr8vZ7zyvQe1mRm5F4QpO9bM9M8JydhY?from_tab_name=main` |
| 漂漂酱 | `https://www.douyin.com/user/MS4wLjABAAAAxXu5l-9OOlHDifMASizvZyDAYjTIGf-jbZ9ZUqEgOsD_eHPPF6ybedKCUtyE4UAO?from_tab_name=main` |
| Miki青青 | `https://www.douyin.com/user/MS4wLjABAAAA-QBwmPS8onaw5ykG8YYGGCB0Iy9G5mvFrqtjVJFDY2cHD9nAcaPts2t5r0WIdI7T?from_tab_name=main` |
| 魔都小蔚 | `https://www.douyin.com/user/MS4wLjABAAAA7FplxZHfbH6QOpKkM2EpkbaXus5IM8UiydiV6undfOw?from_tab_name=main` |

### 路径配置

```
MEDIACRAWLER_PATH = YOUR_MEDIACRAWLER_DIR   # MediaCrawler 仓库根目录
OUTPUT_DIR        = YOUR_OUTPUT_DIR          # Word 文件保存目录
```

---

## 完整工作流程

### 第一步：配置 MediaCrawler

执行以下 Python 脚本，将监控账号写入 MediaCrawler 配置：

```python
import re

config_path = "{MEDIACRAWLER_PATH}/config/dy_config.py"
content = open(config_path, encoding="utf-8").read()

# 填入完整主页 URL（程序自动解析 sec_uid）
dy_creators = [
    "https://www.douyin.com/user/MS4wLjABAAAA-QZ8M4FoCiGQAi5lz9lPJTxb_Klt4vuXYGajE1cA0DyUtsCMiRpTo98IaObz29SU?from_tab_name=main",
    "https://www.douyin.com/user/MS4wLjABAAAAx8K39beYBOq8wv--gjPFPcrJnBSQhw0jvvqzjPKTsy8?from_tab_name=main",
    "https://www.douyin.com/user/MS4wLjABAAAAcVKUoNcPwZtqr8vZ7zyvQe1mRm5F4QpO9bM9M8JydhY?from_tab_name=main",
    "https://www.douyin.com/user/MS4wLjABAAAAxXu5l-9OOlHDifMASizvZyDAYjTIGf-jbZ9ZUqEgOsD_eHPPF6ybedKCUtyE4UAO?from_tab_name=main",
    "https://www.douyin.com/user/MS4wLjABAAAA-QBwmPS8onaw5ykG8YYGGCB0Iy9G5mvFrqtjVJFDY2cHD9nAcaPts2t5r0WIdI7T?from_tab_name=main",
    "https://www.douyin.com/user/MS4wLjABAAAA7FplxZHfbH6QOpKkM2EpkbaXus5IM8UiydiV6undfOw?from_tab_name=main",
]

new_list = "DY_CREATOR_ID_LIST = " + repr(dy_creators)
content = re.sub(r'DY_CREATOR_ID_LIST\s*=\s*\[.*?\]', new_list, content, flags=re.DOTALL)
open(config_path, "w", encoding="utf-8").write(content)
print("配置写入完成")
```

同时确认 `config/base_config.py` 中以下配置：

```python
SAVE_DATA_OPTION = "json"       # 输出 JSON 格式
CRAWLER_MAX_NOTES_COUNT = 50    # 足够覆盖3个月内容
ENABLE_GET_COMMENTS = False     # 月报不需要评论
HEADLESS = False                # 首次登录需要显示浏览器
SAVE_LOGIN_STATE = True         # 保存登录态
```

### 第二步：运行 MediaCrawler

```bash
cd {MEDIACRAWLER_PATH}
```

**首次运行**（需要扫码登录）：
```bash
uv run main.py --platform douyin --lt qrcode --type creator
```

**后续运行**（使用缓存登录态）：
```bash
uv run main.py --platform douyin --lt cache --type creator
```

等待爬取完成，输出文件位于：
```
{MEDIACRAWLER_PATH}/data/douyin/contents/creator_contents_{YYYYMMDD}.json
{MEDIACRAWLER_PATH}/data/douyin/creators/creator_creators_{YYYYMMDD}.json
```

### 第三步：读取数据并过滤近3个月

```python
import json
from datetime import datetime, timedelta
import glob
from collections import defaultdict

mc_path = "{MEDIACRAWLER_PATH}"

# 读取视频内容
content_files = sorted(glob.glob(f"{mc_path}/data/douyin/contents/creator_contents_*.json"))
if not content_files:
    print("未找到输出文件，请确认 MediaCrawler 是否运行成功")
    exit(1)

all_videos = json.loads(open(content_files[-1], encoding="utf-8").read())

# 读取创作者基本信息
creator_files = sorted(glob.glob(f"{mc_path}/data/douyin/creators/creator_creators_*.json"))
creator_info = {}
if creator_files:
    for item in json.loads(open(creator_files[-1], encoding="utf-8").read()):
        creator_info[item["user_id"]] = item
        # 字段：user_id(sec_uid), nickname, gender, desc, ip_location,
        #       follows, fans, interaction, videos_count

# 3个月截止时间
cutoff = datetime.now() - timedelta(days=90)

# 按 sec_uid 分组过滤
by_user = defaultdict(list)

for video in all_videos:
    # 字段：aweme_id, title, desc, create_time(时间戳秒),
    #       liked_count, collected_count, comment_count, share_count,
    #       ip_location, nickname, sec_uid, aweme_url
    ts = video.get("create_time", 0)
    if isinstance(ts, str):
        ts = int(ts)
    video_date = datetime.fromtimestamp(ts)
    if video_date >= cutoff:
        title = video.get("title") or video.get("desc", "")[:50]
        by_user[video["sec_uid"]].append({
            "date": video_date.strftime("%Y-%m-%d"),
            "title": title,
            "likes": video.get("liked_count", 0),
            "comments": video.get("comment_count", 0),
            "shares": video.get("share_count", 0),
            "collects": video.get("collected_count", 0),
            "ip": video.get("ip_location", ""),
            "url": video.get("aweme_url", ""),
        })

# 按日期倒序
for uid in by_user:
    by_user[uid].sort(key=lambda x: x["date"], reverse=True)

print(json.dumps({"videos": dict(by_user), "info": creator_info}, ensure_ascii=False, indent=2))
```

### 第四步：逐账号生成 Word 文件

对监控列表中每个账号，根据上一步得到的数据，在 `{OUTPUT_DIR}` 下创建并运行 `gen_{昵称}.js`，运行完成后删除临时脚本。

---

## Word 文档格式规范

### 页面设置

```javascript
page: {
  size: { width: 11906, height: 16838 },  // A4
  margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 }
}
```

### 颜色方案

| 用途 | 色值 |
|------|------|
| 主标题 | `1A1A2E`（深夜蓝） |
| 副标题/章节 | `E94560`（抖音红） |
| 表头背景 | `1A1A2E` |
| 表头文字 | `FFFFFF` |
| 标签列背景 | `F5E6E8`（浅粉） |
| 偶数行 | `FDF6F7` |
| 正文文字 | `1A1A1A` |
| 副文字 | `888888` |
| 警告文字 | `C0392B`（红） |

### 字体

全文使用 `Arial`。

### 文档结构

1. **主标题**：`{昵称}`（居中，52pt，深蓝 `1A1A2E`，粗体）
2. **副标题**：`抖音近3个月动态报告`（居中，34pt，抖音红 `E94560`）
3. **统计区间**：`统计周期：{3个月前日期} — {今日}`（居中，21pt，灰色 `888888`）
4. **分隔线**
5. **第一节：一、账号基本信息**（h1 + 2列信息表格）
   - 左列标签宽 2256，右列内容宽 6770
   - 包含：账号昵称、粉丝数、关注数、视频总数、IP属地、简介
6. **第二节：二、内容主题分布**（h1 + 3列主题表格）
   - 列：主题 | 内容描述 | 占比
   - 根据视频标题/描述归纳 3-5 个主题分类
7. **第三节：三、近3个月发布视频（共N条）**（h1 + 视频明细表格）
   - 表格列：发布日期 | 视频标题/描述 | 点赞 | 评论 | 转发 | 收藏
   - 视频数 < 5 条或断更 > 30 天，在表格上方加红色警告
8. **分隔线**
9. **第四节：四、总结**（h1 + 2-3 段分析 + 末行关键词灰色斜体）

### 页眉/页脚

```javascript
// 页眉：右对齐，带下边框
`{昵称} 抖音动态报告`
// 页脚：居中
`第 {页码} 页  |  生成日期：{今日}`
```

### 关键代码片段

```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
  HeadingLevel, AlignmentType, BorderStyle, WidthType, ShadingType,
  Header, Footer, PageNumber } = require('docx');
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
      children: [new TextRun({ text: String(text), bold, font: "Arial", size: 20 })]
    })]
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

生成 JS 脚本时，对所有正文文本**统一使用反引号**（模板字符串）：

```javascript
const title = `视频标题里的"特殊内容"`;
```

### MediaCrawler 运行失败时

- 登录缓存过期：改用 `--lt qrcode` 重新登录
- 检查 `DY_CREATOR_ID_LIST` 格式（sec_uid 或完整主页 URL）
- 查看 MediaCrawler 控制台日志

### 视频数量极少的处理

视频数 < 5 条或断更 > 30 天：
- 在第三节上方添加红色警告
- 在总结中明确指出更新频率低

---

## 处理完成后

所有账号处理完毕后，汇总告知用户：
- 每个账号生成的文件路径
- 各账号近3个月视频数量
- 如有跳过的账号，说明原因
