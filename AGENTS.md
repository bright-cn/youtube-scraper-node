# 供在此仓库中工作的编程智能体使用

更改任何内容前，请先阅读本文。以下每一行都已于 2026-11-09 对照实时 API、`@brightdata/sdk` 1.2.0 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，本文以实时 API 为准。

## 身份验证

- `token()` 会读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的 `bdata login` 密钥（SDK 读取的也是同一个文件）。不支持 `.env`；Node 20 可通过 `node --env-file=.env` 加载该文件。设置此变量或运行一次登录，是必须由人工完成的唯一一步；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅读取 `BRIGHTDATA_API_TOKEN`。它无法读取 CLI 登录信息。

## API 的工作方式

- 共有三个数据集：视频 `gd_lk56epmy2i5g7lzu0k`（按 URL 采集，可通过 `keyword`、`search_filters`、`hashtag`、`url`、`explore`、`podcast_url` 发现）、频道 `gd_lk538t2k2p1k3oos71`（按 URL 采集，可通过 `keyword` 发现）和评论 `gd_lk9q0ew71spt1mxywf`（按 URL 采集）。
- JavaScript SDK 可按 URL 采集视频、频道和评论，但不支持这七种发现方法。本软件包使用内置的 `fetch` 调用 `https://api.brightdata.com/datasets/v3`，且没有依赖项，因此全部十种方法的工作方式都相同。请保持这一点。测试通过 `transport` 模拟网络。
- 每次调用都是异步任务：`/trigger`、轮询 `/progress/<id>`，然后执行 `/snapshot/<id>?format=json`。运行时间从 12 秒（频道）到 250 秒（播客播放列表）不等。`TIMEOUT_MS` 为 900000。
- 如果状态轮询因网络错误失败，会持续重试直至截止时间；2026-10-24，一次 `fetch failed` 在五分钟后丢失了一个付费任务。HTTP 错误仍会导致拒绝。
- 始终发送 `include_errors=true`。API 默认会丢弃错误行，因此失败的输入看起来会像空结果。
- 每次发现都会发送 `limit_per_input`，因此不会按所有匹配项计费。`num_of_posts` 和 `num_of_comments` 也都按要求生效：请求 2 条，返回 2 条；请求 3 条评论，返回 3 条。
- API 验证的输入字段名称如下：
  - 关键词搜索：`keyword`；`num_of_posts` 是文本字段（使用 "3"，而不是 3）。
  - 搜索筛选条件：`keyword_search`，以及根据固定列表检查的 YouTube 筛选标签。"This month"、"Video" 和 "Under 4 minutes" 可通过；`duration` "4-20 minutes" 会返回 400，提示 "This value is not allowed"。
  - 话题标签：`hashtag` 不含 #，`num_of_posts` 为数字。
  - 评论：`num_of_comments`，可选 `sort_by`、`load_replies`。
  - `bdata pipelines info <dataset_id> <method>` 会输出实时输入架构。
- **频道视频（通过 `url` 发现）目前在上游存在故障。** 2026-11-09，针对 @MrBeast 和 @NASA 的三次 `/videos` 运行在约九分钟后，六行中的六行都返回 `parse_error`："Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined"。README 展示了该输出。恢复返回数据行后，请重新运行 README 代码块并替换其中展示的输出。
- 视频记录的 `description` 可能属于其他视频："Me at the zoo" 返回的描述内容与微塑料有关。在此问题修复前，切勿在 README 输出中显示 `description`。
- 视频记录包含 `transcript`、`formatted_transcript`（开始和结束时间，以毫秒为单位）以及 `chapters`。字幕文本包含换行符。
- 架构可能在不另行通知的情况下发生变化。切勿硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此仓库中工作

- `npm test` 可离线运行，无需令牌。`npm run lint` 必须通过。
- CI 会在空白机器上执行 README 自身的命令，并通过准确的 H2 文本查找各部分。如果重命名标题，也要同步修改 ci.yml。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则运行较小规模的实时检查。代码片段必须完整且可独立粘贴运行，展示的输出也必须是真实结果。
- README 中的字段表位于 `<!-- fields:start -->` 和 `<!-- fields:end -->` 之间，每天由 `.github/scripts/field_tables.mjs` 根据 https://docs.brightdata.com/scrapers-full.json 重新生成。请勿手动编辑。
- README 顶部的“最后验证”徽章行由每日运行自动重写。请勿手动编辑。
- 标题针对搜索进行了优化："YouTube 爬虫工具 for Node.js and JavaScript"、"How to scrape YouTube ..."。添加章节时请保持这一模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发。
