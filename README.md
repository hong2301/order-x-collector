# X 平台账号数据采集与整理项目 (x_collector)

面向具体分析需求的 **X（Twitter）平台账号数据采集、整理与交付**项目。

## 项目概述

采集 **X 平台账号 `@iChongqing_CIMC`**（重庆国际传播中心官方账号）在时间范围
**2018.07.01 — 2026.08.31** 内的全部帖文，覆盖：

- 原创帖、纯转发、回复帖、引用帖四类帖文
- 完整保存原始正文（不清洗、不翻译、不删减）、发布时间、帖子链接、标签、提及账号、外链、媒介形式及平台互动数据

采集完成后经过多轮数据整理补齐（会话ID / 回复帖ID / 引用帖转发数 / 外链展开 / 图片OCR），
最后按统一字段规范导出标准数据表，并输出多维度分析报表，形成最终交付物。

## 目录结构

```
x_collector/
├── 输入关键词采集帖子/
│   ├── main.py            # 采集主程序（DrissionPage + Chrome 调试端口）
│   ├── input.csv          # 待采集链接清单（url, keyword 两列）
│   └── 启动命令.txt        # 分段并行采集的参考命令
├── 结果/                  # 各批次采集原始数据（回复贴、引用贴等，不入库）
├── 交付v1/                # 最终交付物（标准主数据 + 3 份分析报表）
└── 字段说明.docx          # 交付字段规范说明
```

## 数据采集

### 技术方案

- **DrissionPage** 连接本机 Chrome 调试端口（`--remote-debugging-port`）进行页面控制
- 抓取帖子流中的正文、发布者、时间、媒体、互动数据（浏览/点赞/转发/回复/收藏/引用）
- 短链展开走本地代理（127.0.0.1:7897）
- 支持多进程并行分段采集（按 input 行号切分）

### 命令行

```bash
# 单进程 / 并发分段（每个 Chrome 端口跑一段）
python main.py --port 5268 --start 1 --end 100
python main.py --port 5269 --start 101 --end 200
```

| 参数 | 说明 |
|------|------|
| `--port` | Chrome 调试端口，默认 5268 |
| `--start` | 起始索引（1-based，对应 input.csv 行号） |
| `--end` | 结束索引，0 表示采集到末尾 |

### 采集产出

- `postData/*.json`：每篇帖子的完整数据（含 `postStatus`，标记帖子是否已被作者删除/封禁）
- `imgData/*.json`：图片截图 base64

## 数据整理流程

1. **多批次合并去重**：早/晚多轮采集按"帖文链接 + 发布时间"双键去重，同帖保留数据最全的一条
2. **回复帖关系补全**：通过回复流序列推导
   - 会话ID = 序列第一篇帖文的 ID
   - 被回复帖ID = 主贴帖文前一篇的 ID（校验其发布者属于"被回复账号"）
   - 帖子被删除时记录为 `-1:delete`
3. **引用帖转发数**：按引用帖ID 匹配引用贴采集结果，回填引用转发量
4. **外链真实地址展开**：t.co 短链接返回 JS/meta 跳转页（非标准 302），需从响应 HTML 中提取真实目标地址；多重短链逐层展开
5. **图片文字识别（OCR）**：对图片进行文字识别，依据"是否有文字"标记，将与帖文关联的 OCR 文本写入 `media_alt_text` 字段
6. **时间统一**：发布时间统一转为北京时间（UTC+8），记录到秒

## 交付数据规范（31 字段）

一条帖子一行，`post_id` 唯一标识。字段分组：

| 分组 | 字段 |
|------|------|
| A. 样本识别 | post_id, post_url, account_id, account_handle, created_at, snapshot_datetime |
| B. 原始文本 | text_raw, lang, hashtags, mentions, urls, expanded_urls |
| C. 帖文关系 | is_retweet, is_reply, is_quote, conversation_id, retweeted_post_id, quoted_post_id, in_reply_to_post_id, in_reply_to_user_id |
| D. 媒介形式 | media_type, media_count, media_url, media_alt_text, video_duration |
| E. 平台传播数据 | view_count, like_count, retweet_count, reply_count, bookmark_count, quote_count |

完整字段定义见根目录 **`字段说明.docx`**（含字段作用与采集说明）。

## 交付物（交付v1/）

| 文件 | 说明 |
|------|------|
| `主数据.xlsx` | 标准格式全量主数据 |
| `分析1_媒体类型.xlsx` | 媒体类型分布（image/video/gif 及混合组合） |
| `分析2_图片替代文本.xlsx` | 图片文字识别率分析（含分年度） |
| `分析3_外链平台.xlsx` | 外链指向平台统计（官网/YouTube/CGTN 等） |

## 数据约定

- 正文原样保存，不做清洗、翻译、删减
- 互动数据保存原始整数；**无法抓取的数据留空，不得用 0 代替**
- 被删除/失效的帖子统一标记记录，不自行删除帖文行