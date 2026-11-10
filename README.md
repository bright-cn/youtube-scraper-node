# 适用于 Node.js 和 JavaScript 的 YouTube 爬虫工具

使用 Node.js 和 JavaScript 将 YouTube 视频、字幕、Shorts、评论和频道抓取为 JSON。无需 YouTube Data API 密钥，没有配额限制。基于 [Bright Data YouTube 爬虫工具 API](https://www.bright.cn/products/web-scraper/youtube?utm_source=github&utm_medium=referral&utm_content=youtube-scraper-node) 构建。新账户每月可获得 5,000 个免费额度。

[![实时检查](https://github.com/bright-cn/youtube-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/youtube-scraper-node/actions/workflows/live.yml) [![最后验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/youtube-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-nodejs-抓取-youtube-视频) · [无需 API 密钥](#不使用-youtube-data-api-抓取-youtube) · [CLI](#批量-youtube-搜索爬虫-cli) · [端点](#抓取-youtube-视频字幕shorts频道和评论) · [字幕](#抓取-youtube-视频字幕) · [数据](#youtube-视频数据字段json) · [错误](#youtube-爬虫错误) · [MCP 和 Claude Code](#youtube-mcp-服务器和-claude-code-技能) · [免费额度](#这个-youtube-爬虫工具免费吗) · [文档](https://docs.brightdata.com/products/scrapers/youtube/introduction) · [支持](#支持)

使用 JavaScript 获取 JSON 格式的 YouTube 数据：包括带观看次数、点赞数、字幕和章节的视频与 Shorts、搜索结果、话题标签、频道、评论和播客单集。覆盖全部三个 Bright Data YouTube 抓取工具及其全部十种方法，每种方法对应一个函数。无需 YouTube Data API 密钥，没有配额限制，无需浏览器，也没有依赖项。基于 [Bright Data YouTube 爬虫工具 API](https://www.bright.cn/products/web-scraper/youtube?utm_source=github&utm_medium=referral&utm_content=youtube-scraper-node) 构建。

此外还提供用于关键词搜索的单命令 CLI，以及完全不需要 JavaScript 的 [Bright Data CLI](#youtube-mcp-服务器和-claude-code-技能)。

## 如何使用 Node.js 抓取 YouTube 视频

需要 Node 20 或更高版本，无需安装任何内容：这里使用内置的 `fetch`。由于代码使用顶层 `await`，请将其保存为 `.mjs` 文件。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户可[每月获得 5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```javascript
const API = "https://api.brightdata.com/datasets/v3";
const HEADERS = {
  Authorization: `Bearer ${process.env.BRIGHTDATA_API_TOKEN}`,
  "Content-Type": "application/json",
};

async function call(path, body) {
  const init = body === undefined ? { headers: HEADERS } : { method: "POST", headers: HEADERS, body: JSON.stringify(body) };
  const response = await fetch(API + path, init);
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
  return response.json();
}

const query = new URLSearchParams({
  dataset_id: "gd_lk56epmy2i5g7lzu0k",
  type: "discover_new",
  discover_by: "keyword",
  limit_per_input: "3",
  include_errors: "true",
});
const job = await call(`/trigger?${query}`, [{ keyword: "python tutorial", num_of_posts: "3" }]);
while (!["ready", "failed"].includes((await call(`/progress/${job.snapshot_id}`)).status)) {
  await new Promise((done) => setTimeout(done, 10_000));
}
for (const video of await call(`/snapshot/${job.snapshot_id}?format=json`)) {
  if (video.error) {
    console.log("error row:", video.error);
    continue;
  }
  console.log(video.views, video.likes, video.title.slice(0, 50));
}
```

```
error row: Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
332620 5802 Python Tutorial for AI: Everything You Need in 2 H
26081206 474384 Python Tutorial For Beginners in Hindi | Complete 
```

预计需要两到十一分钟：API 会运行搜索任务，循环会等待任务完成。按[每个视频](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中的三项设置不是可选项。

`limit_per_input` 用于限制运行规模和费用。`num_of_posts` 请求相同数量的结果；它在此爬虫工具中是文本字段，因此以 `"3"` 的形式发送。

`include_errors=true` 会让 API 将无法采集的视频作为带有 `error` 键的数据行返回。API 默认关闭此功能，因此失败的搜索看起来会与没有结果的搜索相同。

之所以保留 `error row` 分支，是因为在 2026 年 11 月 9 日，某次搜索中的部分视频返回了 "Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined"，而其他视频则完整返回。请参阅 [YouTube 爬虫错误](#youtube-爬虫错误)。

## 不使用 YouTube Data API 抓取 YouTube

YouTube 爬虫工具 API 不需要 Google Cloud 项目、YouTube Data API 密钥或 OAuth，也没有每日配额限制。唯一的凭据是 Bright Data API 密钥，通过 `BRIGHTDATA_API_TOKEN` 设置。本软件包及其命令也可以找到由 `bdata login` 存储的密钥。它会采集未登录访客可见的内容，包括字幕。

## 批量 YouTube 搜索爬虫 CLI

此仓库中的命令可同时针对多个关键词运行 Node.js 快速入门，并写入一个 JSON 文件。

```bash
npm install -g github:bright-cn/youtube-scraper-node
youtube-scraper "python tutorial" "home workout" --limit 2
```

```
Finding up to 2 videos per keyword, for: python tutorial, home workout
Usually two to ten minutes each. Billed per video, 5,000 free credits a month.
asking  "python tutorial"...
got     "python tutorial": 1 videos, 1 error rows: Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
asking  "home workout"...
got     "home workout": 1 videos, 1 error rows: Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined

Saved 2 videos as JSON to youtube.json (51 fields per video)
```

在终端中，`asking` 行会被原地更新的加载动画和运行计时器取代，因此你可以看到任务正在运行以及已经运行了多长时间。

```
--limit N      videos per keyword, default 5, minimum 1
--out PATH     output file, default youtube.json
```

该命令会从 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你曾运行过一次 `npx -p @brightdata/cli bdata login`，则会从已保存的登录信息中查找，供你以及该终端中的任何编程智能体使用。它不会读取 `.env` 文件；Node 可通过 `node --env-file=.env` 为其加载该文件。

如果希望每个关键词获得 `ok`、`note` 和 `error`，而不是原始数据行，请导入它而非直接运行。此软件包使用 ESM，因此请使用 `import`，不要使用 `require`。`scrape` 不会因为某个关键词失败而拒绝；读取 `videos` 前请检查 `ok`：

```javascript
import { scrape } from "@brightdata/youtube-scraper-node";

for (const outcome of await scrape(["python tutorial", "home workout"], { limit: 1 })) {
  if (outcome.ok) {
    const [video] = outcome.videos;
    console.log(`${outcome.keyword}: ${video.title.slice(0, 45)} (${video.views} views)`);
  } else {
    console.log(`${outcome.keyword} failed: ${outcome.error}`);
  }
}
```

```
python tutorial: Python Full Course for free 🐍 (13401278 views)
home workout: 25 MIN FULL BODY HIIT for Beginners - No Equi (34828112 views)
```

只有当某个关键词返回的每一行都是错误行时，该关键词才算失败。如果搜索返回了一些视频和一些错误行，视频会被保留，错误则会放入 `note`。

## 抓取 YouTube 视频、字幕、Shorts、频道和评论

Bright Data 提供三个 YouTube 爬虫工具，分别用于视频、频道和评论，共有十种调用方式，每种方式均记录在 [YouTube 爬虫工具 API 文档](https://docs.brightdata.com/api-reference/scrapers/social-media-apis/youtube-videos-collect-by-url)中。本软件包为每种方式提供一个函数。下面的每个代码片段都是完整的：在上面的 `npm install` 之后原样粘贴即可。每个片段都会在每周一的 Actions 中运行，规模较小的检查每隔一天运行一次，顶部徽章显示最新结果。

| 你已有 | 你想要 | 方法 |
| --- | --- | --- |
| 视频或 Shorts URL | 观看次数、点赞数、字幕、章节 | `videos(urls)` |
| 搜索词，例如 "python tutorial" | 匹配的视频 | `search("python tutorial", { limit: 5 })` |
| 搜索词和 YouTube 筛选条件 | 筛选后的视频 | `searchFilters("python tutorial", { limit: 5, upload_date: "This month" })` |
| 话题标签 | 相关视频和 Shorts | `hashtag("baking", { limit: 5 })` |
| 视频 URL | 相关评论 | `comments(urls, { limit: 10 })`，按每条评论计费 |
| 频道 URL | 订阅人数、观看次数、视频数量 | `channels(urls)` |
| 关键词 | 匹配的频道 | `channelsSearch("home cooking", { limit: 5 })` |
| 播客播放列表 URL | 其中的单集 | `podcast(url, { limit: 5 })` |
| 探索页面 URL | 其中的视频 | `explore("https://www.youtube.com/gaming/games", { limit: 5 })` |
| 频道的 `/videos` URL | 其近期视频 | `channelVideos(url, { limit: 5 })`，2026 年 11 月 9 日上游出现故障 |

每个函数都接受单个 URL 或 URL 数组。数组会作为一个任务，而不是每个 URL 一个任务。每个函数都会解析为数据行数组，API 无法采集的数据行会带有 `error` 键，而不是正常记录。接受限制参数的函数还会在同一选项对象中传递 API 接受的任何其他输入，并使用 API 中的字段名称：`search("python tutorial", { limit: 5, start_date: "10-01-2026" })`。

### 按 URL 抓取 YouTube 视频

```javascript
import { videos } from "@brightdata/youtube-scraper-node";

const [video] = await videos("https://www.youtube.com/watch?v=jNQXAC9IVRw");
console.log(video.title, "|", video.youtuber, "|", video.date_posted.slice(0, 10));
console.log(video.views, "views,", video.likes, "likes,", video.num_comments, "comments");
console.log(video.video_length, "seconds,", video.category, "| tags:", video.tags.join(", "));
```

```
Me at the zoo | @jawed | 2005-04-24
448085592 views, 20132396 likes, 10646130 comments
19 seconds, Film & Animation | tags: me at the zoo, jawed karim, first youtube video
```

按 URL 采集视频耗时 12 至 178 秒。Shorts URL 的工作方式相同。

### 抓取 YouTube 视频字幕

```javascript
import { videos } from "@brightdata/youtube-scraper-node";

const url = "https://www.youtube.com/watch?v=jNQXAC9IVRw";
const [video] = await videos(url, { transcription_language: "English" });
for (const line of video.formatted_transcript.slice(0, 3)) {
  const text = line.text.split(/\s+/).join(" ");
  console.log(`${(line.start_time / 1000).toFixed(1).padStart(5)}s  ${text}`);
}
console.log(video.transcript.length, "characters in all");
```

```
  1.2s  All right, so here we are, in front of the elephants
  5.3s  the cool thing about these guys is that they have really...
  8.0s  really really long trunks
217 characters in all
```

每条视频记录会包含两份字幕：`transcript` 是纯文本，`formatted_transcript` 是带时间信息的分行内容，`start_time` 和 `end_time` 以毫秒为单位。`transcript_language` 列出 YouTube 提供的语言，`chapters` 列出视频章节。文本会保留 YouTube 的换行符。

### 使用筛选条件抓取 YouTube 搜索结果

```javascript
import { searchFilters } from "@brightdata/youtube-scraper-node";

const found = await searchFilters("python tutorial", {
  limit: 2,
  upload_date: "This month",
  type: "Video",
  duration: "Under 4 minutes",
});
for (const video of found) {
  if (video.error) {
    console.log("error row:", video.error);
    continue;
  }
  console.log(video.date_posted.slice(0, 10), video.video_length, "s |", video.title.slice(0, 50));
}
```

```
error row: Parse error: Cannot read properties of undefined (reading 'runs')
2026-10-21 130 s | Python Tutorial for Beginners 🐍 | Install Python 
```

筛选条件采用 YouTube 自身的标签，并根据固定列表进行检查。"This month"、"Video" 和 "Under 4 minutes" 可被接受；"4-20 minutes" 会返回 HTTP 400：`["duration", "This value is not allowed"]`。

### 按话题标签抓取 YouTube Shorts

```javascript
import { hashtag } from "@brightdata/youtube-scraper-node";

for (const video of await hashtag("baking", { limit: 2 })) {
  if (video.error) {
    console.log("error row:", video.error);
    continue;
  }
  console.log(video.post_type, video.views, video.url);
}
```

```
error row: Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
post 361876 https://www.youtube.com/watch?v=bxVrq1cbBmA
```

传入标签时不要包含 `#`。`start_date`、`end_date` 和 `MM-DD-YYYY` 可将范围缩小到特定时间窗口。

### 抓取 YouTube 视频评论

```javascript
import { comments } from "@brightdata/youtube-scraper-node";

for (const comment of await comments("https://www.youtube.com/watch?v=jNQXAC9IVRw", { limit: 3 })) {
  console.log(comment.likes, "|", comment.username, "|", comment.comment_text.slice(0, 50));
}
```

```
4800000 | @SanDiegoZoo | We're so honored that the first ever YouTube video
0 | @boxifed | here before viral ⬇⬇
1 | @roshac995 | 10M comment
```

限制数量使用 `num_of_comments` 输入：请求 3 条，43 秒后返回 3 条。`sort_by` 和 `load_replies` 放在选项对象中。

### 抓取 YouTube 频道数据

```javascript
import { channels } from "@brightdata/youtube-scraper-node";

const [channel] = await channels("https://www.youtube.com/@NASA/about");
console.log(channel.name, channel.handle);
console.log(channel.subscribers, "subscribers,", channel.videos_count, "videos,", channel.views, "views");
```

```
NASA @NASA
15100000 subscribers, 6201 videos, 1134859779 views
```

速度最快的调用：12 秒。

### 按关键词查找 YouTube 频道

```javascript
import { channelsSearch } from "@brightdata/youtube-scraper-node";

for (const channel of await channelsSearch("home cooking", { limit: 2 })) {
  console.log(channel.subscribers, channel.name, channel.url);
}
```

```
1380000 집밥요리 Home Cooking https://www.youtube.com/@jibbab_yori
3300000 HomeCookingShow https://www.youtube.com/@homecookingshow
```

### 抓取 YouTube 播客单集

```javascript
import { podcast } from "@brightdata/youtube-scraper-node";

const playlist = "https://www.youtube.com/playlist?list=PLbnMTcZEga8QCKJvxsJVQV4uHD6bENdnP";
for (const episode of await podcast(playlist, { limit: 2 })) {
  console.log(episode.date_posted.slice(0, 10), episode.views, "|", episode.title.slice(0, 60));
}
```

```
2026-11-09 7455 | Are you risking your career if you post about work on social
2026-11-02 16724 | She gave all her employees flexible work. Here’s how she kep
```

传入播客播放列表 URL 或频道的 `/podcasts` URL。两个 SDK 都没有针对此方法的接口。

### 抓取 YouTube 探索和游戏页面

```javascript
import { explore } from "@brightdata/youtube-scraper-node";

for (const video of await explore("https://www.youtube.com/gaming/games", { limit: 2 })) {
  console.log(video.live_status, video.views, "|", video.title.slice(0, 50));
}
```

```
is_live 671 | コントロールネメシス 【シャドバWB/Shadowverse:Worlds Beyond】
is_live 25077 | [Shadowverse]BEYONDふやすぞ～！！！！
```

游戏页面主要是直播，因此 `live_status` 为 `is_live`，`views` 表示截至当前的直播计数。

### 抓取 YouTube 频道的视频

```javascript
import { channelVideos } from "@brightdata/youtube-scraper-node";

for (const row of await channelVideos("https://www.youtube.com/@NASA/videos", { limit: 2 })) {
  if (row.error) console.log("error:", row.error_code, row.error);
  else console.log(row.date_posted.slice(0, 10), row.title.slice(0, 60));
}
```

```
error: parse_error Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
error: parse_error Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
```

这是实际输出。2026 年 11 月 9 日，针对 @NASA 和 @MrBeast 尝试的所有频道共运行五次，均在约九分钟后只返回此错误。调用和输入都符合实时架构，每周检查会运行此代码块，因此功能恢复后徽章会显示结果。在此之前，可以使用带频道名称的 `search`，或对已知 URL 使用 `videos`，以获取相同记录。

### 立即触发，稍后获取

对于规模大于少量搜索的任务，不要让进程持续等待一小时。触发任务、保存快照 ID，并在准备就绪后获取。快照可下载 30 天。

```javascript
import { CHANNELS, fetchSnapshot, status, trigger } from "@brightdata/youtube-scraper-node";

const snapshot = await trigger(CHANNELS, [{ url: "https://www.youtube.com/@NASA/about" }]);
console.log("snapshot:", snapshot);
let state;
while (!["ready", "failed"].includes((state = await status(snapshot)))) {
  await new Promise((done) => setTimeout(done, 10_000));
}
console.log("status:", state);
console.log("fetched:", (await fetchSnapshot(snapshot))[0].name);
```

```
snapshot: sd_mw95unbndfba3pqds
status: ready
fetched: NASA
```

`trigger` 接受任意数据集、`discoverBy` 和 `limitPerInput`，因此可调用表中的每种方法。以下每项输入均已于 2026-11-09 通过实时 API 验证：

| 方法 | dataset_id | discover_by | 输入 |
| --- | --- | --- | --- |
| 按 URL 获取视频 | `gd_lk56epmy2i5g7lzu0k` | 无 | `{"url"}`，可选 `{"country", "transcription_language"}` |
| 搜索 | `gd_lk56epmy2i5g7lzu0k` | `keyword` | `{"keyword", "num_of_posts"}`，`num_of_posts` 为文本 |
| 带筛选条件的搜索 | `gd_lk56epmy2i5g7lzu0k` | `search_filters` | `{"keyword_search"}` 加筛选标签 |
| 话题标签 | `gd_lk56epmy2i5g7lzu0k` | `hashtag` | `{"hashtag", "num_of_posts"}` |
| 频道视频 | `gd_lk56epmy2i5g7lzu0k` | `url` | `{"url", "num_of_posts"}`，一个 `/videos` URL |
| 探索页面 | `gd_lk56epmy2i5g7lzu0k` | `explore` | `{"url"}`，可选 `{"all_tabs"}` |
| 播客 | `gd_lk56epmy2i5g7lzu0k` | `podcast_url` | `{"url"}` |
| 按 URL 获取频道 | `gd_lk538t2k2p1k3oos71` | 无 | `{"url"}` |
| 按关键词获取频道 | `gd_lk538t2k2p1k3oos71` | `keyword` | `{"keyword"}` |
| 评论 | `gd_lk9q0ew71spt1mxywf` | 无 | `{"url", "num_of_comments"}` |

`bdata pipelines info <dataset_id> <method>` 会输出 API 验证的输入字段。

## YouTube 视频数据字段（JSON）

视频中大多数人需要的字段：

```
url  title  youtuber  views  likes  num_comments  date_posted  video_length  transcript  tags
```

代码不会硬编码任何字段列表。API 返回的所有内容都会写入每条数据行和命令生成的文件中。

<!-- fields:start -->
每天根据已发布的架构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。记录包含适用于它的字段，以及 `timestamp` 和 `input`；示例文件中的第一个视频有 51 个键。

<details>
<summary>全部 49 个视频字段，<code>gd_lk56epmy2i5g7lzu0k</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `url` | url | 所采集视频的完整 YouTube 观看 URL |
| `title` | text | YouTube 视频的标题 |
| `youtuber` | text | 从频道浏览端点提取的频道 handle |
| `youtuber_md5` | text | youtuber 字段的 MD5 哈希值 |
| `video_url` | url | 从 YouTube 自适应格式中提取或从 signatureCipher 解码得到的视频文件直接流式传输 URL |
| `video_length` | number | 视频时长，以秒为单位 |
| `likes` | number | 视频获得的点赞数 |
| `views` | number | 视频获得的观看次数 |
| `date_posted` | date | 视频发布日期 |
| `description` | text | 视频随附的文字描述 |
| `num_comments` | number | YouTube 显示的视频评论数（YouTube 的计数包括顶层评论和回复） |
| `subscribers` | number | YouTube 频道的订阅人数 |
| `music` | object | 视频所用音乐的信息。仅当 YouTube 的 Content ID 系统识别出视频中的音乐时才会出现 |
| `video_id` | text | YouTube 视频的唯一标识符 |
| `channel_url` | url | 根据频道规范浏览端点生成的 URL 编码 YouTube 频道 URL |
| `preview_image` | image | 视频预览图的 URL |
| `shortcode` | text | 视频的唯一短标识符，与 video_id 相同，适用于常规视频和 Shorts 等所有 YouTube 视频类型 |
| `verified` | boolean | 表示频道是否已通过验证 |
| `handle_name` | text | 视频页面上显示的 YouTube 频道名称 |
| `avatar_img_channel` | url | YouTube 频道头像图片的 URL |
| `is_sponsored` | boolean | 表示视频是否包含 YouTube 所声明的付费内容叠加层 |
| `related_videos` | array | 当前视频旁边侧栏中显示的视频 URL 列表 |
| `license` | url | YouTube 视频元数据中声明的视频许可证 URL |
| `viewport_frames` | text | 根据 YouTube 流式传输数据得出的主要视频流格式分辨率 |
| `current_optimal_res` | text | 来自 YouTube 流式传输格式元数据的当前和最佳视频分辨率及帧率 |
| `codecs` | text | 从 YouTube 自适应流式传输格式中提取的视频和音频编解码器字符串，格式为视频编解码器和音频编解码器 |
| `color` | text | 从 YouTube 自适应格式颜色信息中提取的视频流色彩原色 |
| `quality` | text | YouTube 报告的第一个自适应流式传输格式质量标识符 |
| `quality_label` | text | YouTube 报告的易读视频质量标签 |
| `post_type` | text | 根据 URL 结构推断的 YouTube 内容类型。可能的值：标准视频为 post，YouTube Short 为 short，或 N/A |
| `youtuber_id` | text | YouTube 内部频道 ID，与频道 handle 不同 |
| `transcript` | text | 根据 YouTube 字幕轨道编译的视频完整纯文本字幕。优先使用英语字幕；如果英语不可用，则回退到第一个可用语言 |
| `formatted_transcript` | array | 来自 YouTube 字幕轨道的带时间戳字幕片段，每段包含开始时间、结束时间、持续时间（均以毫秒为单位）以及字幕文本 |
| `hashtags` | array | 视频标题或描述中出现的话题标签列表，每项包含标签文本及其 YouTube 搜索链接 |
| `tags` | array | 视频标签 |
| `next_recommended_videos` | array | 视频最后几秒内作为片尾卡片显示的视频，包括以毫秒为单位的时间信息和显示样式 |
| `recommended_videos` | array | YouTube 在当前视频旁边侧栏中推荐的视频 |
| `transcript_language` | array | YouTube 上此视频可用的字幕和文字稿语言列表，包括各轨道是否为自动生成 |
| `chapters` | array | 创作者定义的章节标记列表，每项包含标题、开始时间（以毫秒为单位）和预览图 |
| `transcription_language` | text | 调用方请求用于获取字幕的语言代码，在采集时作为输入参数传入 |
| `is_age_restricted` | boolean | 视频是否有年龄限制，根据非 OK 的可播放状态以及 YouTube 提供的年龄相关原因推断 |
| `channel_url_decoded` | text | 频道 URL 经百分号解码后的纯文本版本 |
| `category` | text | YouTube 根据其预定义分类体系分配给视频的内容类别 |
| `audio_tracks` | array | audio Tracks |
| `made_for_kids` | boolean | 表示视频是否被指定为面向儿童，根据 YouTube 迷你播放器的播放模式推断 |
| `player_notification_text` | text | 视频播放器内显示的面向用户的通知文本 |
| `ai_content_label` | boolean | 表示视频是否被标记为由 AI 生成或经 AI 大幅修改，无论该标记由创作者添加还是由 YouTube 自动检测。 |
| `live_status` | text | 视频的直播状态，等同于 yt-dlp 的 live_status。取值为 "is_live"、"was_live" 或 "not_live"；根据 YouTube 的信息字符串（例如 "Streamed live 29 minutes ago"）解析，普通视频默认为 "not_live" |
| `comments` | array | 视频最相关的 10 条评论 |

</details>

<details>
<summary>全部 21 个频道字段，<code>gd_lk538t2k2p1k3oos71</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `url` | url | 直接链接到 YouTube 个人资料的 URL |
| `handle` | text | 每个用户的唯一标识（例如 @name） |
| `handle_md5` | text | 此字段存储 handle 字段的 MD5 哈希值。 |
| `banner_img` | url | 链接到横幅图片的 URL |
| `profile_image` | url | 链接到头像图片的 URL |
| `name` | text | 个人资料名称 |
| `subscribers` | number | 个人资料的订阅人数 |
| `Description` | text | 用户在 YouTube 上用于自我介绍或分享兴趣与热爱的简短描述或陈述 |
| `videos_count` | number | 个人资料拥有的视频数量 |
| `created_date` | date | 个人资料创建日期 |
| `views` | number | 个人资料获得的观看次数 |
| `Details` | object | channel Details |
| `Links` | array | 用户可能从其 YouTube 个人资料链接的外部网站 |
| `identifier` | text | identifier |
| `discovery_input` | object | 发现输入值 |
| `id` | text | channel id |
| `has_podcast` | boolean | 是否有播客的指示标记 |
| `top_videos` | array | 热门视频 |
| `featured_video` | url | 精选视频 |
| `featured_channels` | array | 精选频道 |
| `collaborations` | array | 合作视频 |

</details>

<details>
<summary>全部 16 个评论字段，<code>gd_lk9q0ew71spt1mxywf</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `comment_id` | text | 每条评论的唯一标识符 |
| `comment_text` | text | 评论的文本内容 |
| `likes` | number | 评论获得的点赞数 |
| `replies` | number | 评论的回复数量 |
| `username` | text | 评论者的用户名 |
| `username_md5` | text | 此字段存储 username 字段的 MD5 哈希值。 |
| `user_channel` | text | 评论者 YouTube 频道的 URL |
| `date` | text | 评论发布日期 |
| `url` | url | 发布该评论的 YouTube 视频网址 |
| `video_id` | text | YouTube 视频的唯一标识符 |
| `replies_value` | array | 与此评论关联的回复对象数组 |
| `replies_without_names` | array | 移除个人身份信息（用户名）后的回复对象数组，仅包含经过 MD5 哈希处理的用户标识符 |
| `user_id` | text | 评论者 YouTube 频道的唯一标识符 |
| `mentioned_timestamps_minutes` | array | 评论中提及的时间戳数组，以分钟格式表示（例如 '2:35'） |
| `mentioned_timestamps_seconds` | array | 评论中提及的时间戳数组，转换为总秒数 |
| `date_iso` | date | ISO 格式的日期 |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>youtube-scraper "python tutorial" --limit 1</code></summary>

```json
{
  "generated_at": "2026-11-09T11:12:32.635Z",
  "keywords": [
    {
      "keyword": "python tutorial",
      "videos": [
        {
          "url": "https://www.youtube.com/watch?v=K5KVEU3aaeQ&t=56s&pp=ygUPcHl0aG9uIHR1dG9yaWFs",
          "title": "Python Full Course for Beginners",
          "youtuber": "@programmingwithmosh",
          "youtuber_md5": "be0da1195f186399397e909dff13d1b6",
          "video_url": "https://rr6---sn-ojq4f5-51.googlevideo.com/videoplayback?expire=1794244289&ei=YarxapufJu2a2_gPk-6TIA&ip=156.248.45.42&id=o-ANdxxaGl38S-b4LNNhkkoAlKjRx-D5QGnn_fMhnUlFF1&source=youtube&requiressl=yes&xpc=EgVo2aDSNQ%3D%3D&cps=617&met=1794222689%2C&mh=g7&mm=18%2C29&mn=sn-ojq4f5-51%2Csn-ojq4f5-cv&ms=aub%2Crdu&mv=u&mvi=6&pl=19&rms=aub%2Caub&pcm2=no&spc=mpRQ__TFYsZSkarIFqog5CLxWX_8YjDkZaBjvQrPfo2ycFX696L_2drt3MZ-uRwGpotVDWA&svpuc=1&ns=8UjBRsyTbGxdj0oYIgVcVW4Z&sabr=1&rqh=1&mt=1794221869&fvip=1&epbp=1&keepalive=yes&fexp=51565115%2C52250119%2C52323638&c=WEB&n=SD9umA7w9UBLw1G-qm&sparams=expire%2Cei%2Cip%2Cid%2Csource%2Crequiressl%2Cxpc%2Cpcm2%2Cspc%2Csvpuc%2Cns%2Csabr%2Crqh&sig=AE0s2JYwRgIhANX5D7K3n3sW41JZ1tgY0iHbP_JZ3lnnuGqxO-x4xItVAiEAvk9FnONA9AniflIAz7ZoVfLMEwYoWi8koXNjX9Gbabk%3D&lsparams=cps%2Cmet%2Cmh%2Cmm%2Cmn%2Cms%2Cmv%2Cmvi%2Cpl%2Crms&lsig=APaTxxMwRAIgfoSm9MRvnEC8uZS3-0Z743lU8ZGTDj10wIhrE6LqR5MCIDGykQLzAy1jzgrofXGX2SZUDruURRCTzC1Rx9fNfb-Y",
          "video_length": 7340,
          "likes": 172884,
          "views": 8099704,
          "date_posted": "2025-02-12T12:30:31.000Z",
          "description": "Master Python from scratch 🚀 No fluff—just clear, practical coding skills to kickstart your journey! \n\n🚀 Want to dive deeper?\n- Check out my complete Python mastery course: https://mosh.link/python-course\n- Subscribe for more awesome Python content: https://goo.gl/6PYaGF\n\n🔗 Related videos \nhttps://youtu.be/yVl_G-F7m8c?si=aoIiT6CyaRRfBsRM\n\n✋ Stay connected:\n- Full Courses: https://codewithmosh.com\n- Twitter: https://twitter.com/moshhamedani\n- Facebook: https://www.facebook.com/programmingwithmosh/\n- Instagram: https://www.instagram.com/codewithmosh.official/\n- LinkedIn: https://www.linkedin.com/school/codewithmosh/\n\n📖 Table of Content\n0:00:00 Introduction\n0:00:56 What is Python?\n0:04:11 Installing Python\n0:05:36 Python Interpreter\n0:07:30 Code Editors\n0:08:49 Your First Python Program\n0:12:25 Python Extension\n0:14:26 Linting Python Code\n0:18:40 Formatting Python Code\n0:22:51 Running Python Code\n0:24:30 Python Implementations\n0:26:59 How Python Code is Executed\n0:29:45 Quiz\n0:31:17 Python Mastery Course\n0:31:44 Variables\n0:34:48 Variable Names\n0:37:51 Strings\n0:43:20 Escape Sequences\n0:46:01 Formatted Strings\n0:48:09 String Methods\n0:54:03 Numbers\n0:56:50 Working With Numbers\n0:58:59 Type Conversion\n1:04:03 Quiz\n1:06:43 Comparison Operators\n1:08:46 Conditional Statements\n1:12:56 Ternary Operator\n1:15:04 Logical Operators\n1:19:07 Short-circuit Evaluations\n1:21:13 Chaining Comparison Operators\n1:22:35 Quiz\n1:24:18 For Loops\n1:27:56 For..Else\n1:30:42 Nested Loops\n1:33:26 Iterables\n1:36:34 While Loops\n1:41:33 Infinite Loops\n1:43:10 Exercise\n1:45:13 Defining Functions\n1:47:37 Arguments\n1:49:57 Types of Functions\n1:53:59 Keyword Arguments\n1:55:59 Default Arguments\n1:57:34 xargs\n\n#Python #AI #MachineLearning #WebDevelopment",
          "num_comments": 4503,
          "subscribers": 5180000,
          "video_id": "K5KVEU3aaeQ",
  ...
```

包含一个视频及其全部字段的完整文件位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## YouTube 爬虫错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在任何请求前以状态码 2 退出。请设置令牌或运行 `bdata login`。 |
| `failed  "keyword": Parse error: ...` | 以状态码 1 退出。该搜索中的每个视频都触发了爬虫的解析错误。2026 年 11 月 9 日，它显示为 `ORIGINAL_LANGUAGE_TO_COUNTRY is not defined`，有一次显示为 `Cannot read properties of undefined (reading 'runs')`。请重新运行，或使用其他关键词。 |
| `got     "keyword": 1 videos, 1 error rows: Parse error: ...` | 以状态码 0 退出。相同错误影响了部分视频，其余视频位于文件中。 |
| `failed  "keyword": timeout: snapshot ... not ready after 900s` | 以状态码 1 退出。请重新运行，或稍后通过 `fetchSnapshot()` 获取快照。 |
| `failed  "keyword": HTTP 401: ...` | 令牌已设置但不正确。 |

任何失败都会以状态码 1 退出，因此可以安全地使用运行结果控制脚本是否继续。

通过函数调用时，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入，例如不在其列表中的筛选标签。消息中会指出相应字段。 |
| 带有 `error` 键的数据行 | API 对某个视频或输入的响应：`parse_error`，或者已失效的页面。其他数据行不受影响。 |
| `ApiError: timeout: ...` | 任务超过了 900 秒的等待时间。向 `run` 传入 `{ timeoutMs }`，或使用 `trigger` 并稍后获取。 |

## YouTube MCP 服务器和 Claude Code 技能

无需 JavaScript，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，或者在 SSH 和 CI 中使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_lk538t2k2p1k3oos71 collect_by_url "https://www.youtube.com/@NASA/about" --pretty
```

2026 年 11 月 9 日运行结果：1 条记录、20 个字段、35 秒。CLI 会省略值为 null 的字段。

CLI 可通过数据集 ID 运行爬虫工具，包括所有 YouTube 爬虫。`bdata pipelines search youtube` 列出它们的 ID 和方法，`--limit-per-input N` 用于限制发现数量，默认为 10。

`npx skills add brightdata/skills` 会向 Claude Code、Cursor 和 Codex 提供这些命令及文档，因此之后可以使用自然语言操作。完整指南：[适用于编程智能体的 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

对于无需终端的托管式助手，可以使用 [Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)。它可通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper` 运行包括 YouTube 在内的任意预构建爬虫工具：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 支持的其他所有连接方式，从 LangChain 到 Zapier 和 n8n，请参阅[集成](https://docs.brightdata.com/integrations/introduction)。

<details>
<summary>CLI 的 YouTube 命令，均于 2026 年 11 月 9 日运行</summary>

在每条命令前加上 `npx -p @brightdata/cli`，或使用 `npm i -g @brightdata/cli` 安装一次。每条命令都会输出 JSON，并按记录计费。

三个爬虫工具，均使用 `bdata pipelines run <dataset_id> <method>` 运行：视频 `gd_lk56epmy2i5g7lzu0k`、频道 `gd_lk538t2k2p1k3oos71` 和评论 `gd_lk9q0ew71spt1mxywf`。

```bash
bdata pipelines run gd_lk9q0ew71spt1mxywf collect_by_url --url "https://www.youtube.com/watch?v=jNQXAC9IVRw" --num_of_comments 3 --format csv -o comments.csv
```

3 条评论、24 列、37 秒。

```bash
bdata pipelines run gd_lk56epmy2i5g7lzu0k discover_by_keyword --keyword "python tutorial" --num_of_posts 2 --limit-per-input 2 --timeout 900 --pretty
```

599 秒内返回 2 行：一个视频，以及一行包含 [YouTube 爬虫错误](#youtube-爬虫错误)中所述解析错误的数据。发现操作可能超过 CLI 默认的 600 秒，因此请保留 `--timeout 900`，或添加 `--async`，然后通过 `bdata pipelines results <snapshot_id>` 稍后获取。

`bdata pipelines info <dataset_id> <method>` 显示爬虫工具的输入字段。`bdata pipelines run --help` 显示 `--format json|csv|ndjson|jsonl`、`-o FILE`、`--timeout` 和 `--async`。

</details>

## 这个 YouTube 爬虫工具免费吗？

新的 Bright Data 账户可[每月获得 5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。每个 YouTube 爬虫工具都按记录计费：按视频、频道或评论计费。[定价](https://www.bright.cn/pricing/web-scraper)列出了费率。

## 支持

此仓库中的错误：[提交 issue](https://github.com/bright-cn/youtube-scraper-node/issues)，[CONTRIBUTING.md](CONTRIBUTING.md) 说明了需要提供哪些信息。任何与 API、你的账户或额度有关的问题，请联系 [Bright Data 支持](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## 适用于 Python 的 YouTube 爬虫工具

同一款 YouTube 爬虫工具也提供 Python 版本：[brightdata/youtube-scraper](https://github.com/brightdata/youtube-scraper)。

## 更多 Bright Data 开源爬虫工具

其他以相同方式构建并包含实时验证示例的 Node.js 爬虫工具：

- [适用于 Node.js 的 TikTok 爬虫工具](https://github.com/brightdata/tiktok-scraper-node)
- [适用于 Node.js 的 Instagram 爬虫工具](https://github.com/bright-cn/instagram-scraper-node)
- [适用于 Node.js 的 ChatGPT 爬虫工具](https://github.com/brightdata/chatgpt-scraper-node)
