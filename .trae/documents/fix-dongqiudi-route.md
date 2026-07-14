# 修复 RSSHub 懂球帝 (dongqiudi) 路由

## 摘要 (Summary)

懂球帝网站结构变更导致多个 dongqiudi RSS 路由失效。主要问题包括：桌面版文章页 `__NUXT__` 数据结构变更、球员新闻路由缺失 `return`、Unix 时间戳解析错误、球队页赛果数据结构变更。本计划在新建分支 `fix/dongqiudi-route` 上完成所有修复，并在本地通过 `pnpm dev` 验证。

## 现状分析 (Current State Analysis)

通过对仓库现有代码和线上目标站点的对比验证，确认了以下问题：

### 已验证可用的接口
- ✅ `https://api.dongqiudi.com/app/tabs/web/{id}.json` (头条新闻列表) — 返回正常，包含 `articles` 数组
- ✅ `https://www.dongqiudi.com/api/old/columns/{id}` (专题列表) — 返回正常
- ✅ `https://api.dongqiudi.com/v3/archive/app/channel/feeds` (球队/球员新闻列表) — 返回正常
- ✅ 移动版文章页 `https://m.dongqiudi.com/article/{aid}.html` — 仍有 `window.__INITIAL_STATE__`，`ProcessFeedType3` 正则仍能匹配
- ✅ `parseDate` 工具底层基于 `dayjs`，传入数字时按毫秒解析；传 `'X'` 才按 Unix 秒解析

### 已验证失效的部分

#### 1. 桌面版文章页 `__NUXT__` 结构变更（影响最严重）
- **旧结构**: `window.__NUXT__.data[0].newData.{body, writer, show_time}`
- **新结构**: `window.__NUXT__.data[0].article.{body, author, publishedAt}`
  - `newData.body` → `article.body` (HTML 内容)
  - `newData.writer` → `article.author` (作者名)
  - `newData.show_time` (Unix 秒, 数字) → `article.publishedAt` (字符串如 `"2024-01-28 14:51"`)
- **影响**: `utils.ts` 的 `ProcessFeedType2` 拿不到数据直接 `return`，导致 `description`、`author`、`pubDate` 全部缺失 → 被 `top-news.ts` 和 `utils.ts` 的 `ProcessFeed` (球队/球员新闻) 调用，影响三条路由

#### 2. `player-news.ts` 缺失 `return`
- 当前代码: `await utils.ProcessFeed(ctx, 'player', playerId);` (无 `return`)
- 对比 `team-news.ts`: `return await utils.ProcessFeed(ctx, 'team', teamId);`
- **影响**: handler 返回 `undefined`，RSSHub 报错"Route handler did not return valid data"

#### 3. Unix 时间戳解析错误
- 顶部新闻 API 返回的 `show_time` 是 Unix 秒时间戳（如 `1784042294`）
- `parseDate(item.show_time)` 会把数字当作毫秒，得到 1970-01-21
- 应改为 `parseDate(item.show_time, 'X')`（参考 `special.ts` 已有的正确写法）
- **影响**: `top-news.ts` 和 `utils.ts` 的 `ProcessFeed`

#### 4. 球队页赛果数据结构变更（影响 `result.ts`）
- **旧结构**: `data.teamScheduleData[]`，每项含 `match_title / team_A_name / team_B_name / fs_A / fs_B / match_id / scheme / start_time`
- **新结构**: `data.scheduleList[]`，每项含 `competition / home / away / homeScore / awayScore / id / startTime`
- 球队名: `data.teamDetail.base_info.team_name` → `data.teamInfo.name`
- 链接: 旧的 `scheme.replace('dongqiudi:///game/', 'https://www.dongqiudi.com/liveDetail/')` 不再可用，需要直接拼接 `https://www.dongqiudi.com/liveDetail/${id}`

## 提议变更 (Proposed Changes)

### Git 工作流
1. 基于当前 `master` 创建新分支 `fix/dongqiudi-route`
2. 在该分支上完成所有修复
3. **不提交 commit、不推送、不发 PR**（用户只要求"新建一个分支完成这个修复"，等待用户审查后再决定是否走 PR 流程）

### 文件变更清单

#### 1. `lib/routes/dongqiudi/utils.ts`

**`ProcessFeedType2` 函数** — 适配新的 `__NUXT__` 结构：
- `dom.window.__NUXT__.data[0].newData` → `dom.window.__NUXT__.data[0].article`
- `data.body` → `data.body` (字段名相同)
- `data.writer` → `data.author`
- `parseDate(data.show_time, 'X')` → `parseDate(data.publishedAt)` (已是字符串格式，dayjs 可直接解析)

**`ProcessFeed` 函数** — 修复 Unix 时间戳解析：
- `pubDate: parseDate(article.show_time)` → `pubDate: parseDate(article.show_time, 'X')`

#### 2. `lib/routes/dongqiudi/player-news.ts`

修复缺失的 `return` 语句：
```ts
// 旧
await utils.ProcessFeed(ctx, 'player', playerId);
// 新
return await utils.ProcessFeed(ctx, 'player', playerId);
```

#### 3. `lib/routes/dongqiudi/top-news.ts`

修复 Unix 时间戳解析：
```ts
// 旧
pubDate: parseDate(item.show_time),
// 新
pubDate: parseDate(item.show_time, 'X'),
```

#### 4. `lib/routes/dongqiudi/result.ts`

适配新的球队页数据结构：
- `data.teamScheduleData.filter(...)` → `data.scheduleList.filter((m) => m.homeScore !== null && m.awayScore !== null)`
- `data.teamDetail.base_info.team_name` → `data.teamInfo.name`
- 每个条目字段映射:
  - `result.match_title` → `result.competition`
  - `result.team_A_name` → `result.home`
  - `result.team_B_name` → `result.away`
  - `result.fs_A` → `result.homeScore`
  - `result.fs_B` → `result.awayScore`
  - `result.match_id` → `result.id`
  - `result.start_time` → `result.startTime`
- 标题模板: `${result.match_title} ${result.team_A_name} ${result.fs_A}-${result.fs_B} ${result.team_B_name}` → `${result.competition} ${result.home} ${result.homeScore}-${result.awayScore} ${result.away}`
- 链接: `result.scheme.replace('dongqiudi:///game/', 'https://www.dongqiudi.com/liveDetail/')` → `https://www.dongqiudi.com/liveDetail/${result.id}`
- `pubDate: parseDate(result.start_time)` → `pubDate: parseDate(result.startTime)`
- 删除不再使用的 `guid: result.match_id`（`link` 已作为唯一标识，参考 AGENTS.md #27 避免重复 guid）

### 不变更的内容
- `namespace.ts` — 站点信息未变
- `special.ts` — 移动版文章页 `ProcessFeedType3` 已验证仍可用
- `daily.ts` — 仅做重定向
- 仍使用 `got`（`@/utils/got`）和 `JSDOM`，不做 `ofetch`/`cheerio` 重构（避免过度工程化）

## 假设与决策 (Assumptions & Decisions)

1. **假设**: 新 `article.publishedAt` 字段格式为 `"YYYY-MM-DD HH:mm"`，dayjs 默认可解析，无需指定格式字符串
2. **假设**: 球队页 `scheduleList` 中已结束的比赛 `homeScore`/`awayScore` 为数字，未开始为 `null`，按此过滤可得赛果
3. **决策**: 不重写为 `ofetch`/`cheerio` — 现有 `got`+`JSDOM` 可用且修复最小化，符合"避免过度工程化"原则
4. **决策**: 不修改 `daily.ts` 和 `special.ts` — 验证仍可用，无需变动
5. **决策**: 只创建分支不提交 commit — 用户只要求"新建一个分支完成这个修复"，验证通过后等待用户审查再决定是否走 PR 流程

## 验证步骤 (Verification Steps)

1. **拉取依赖**: `pnpm install`
2. **启动开发服务器**: `pnpm dev` (默认监听 http://localhost:1200)
3. **逐条测试路由**:
   - `http://localhost:1200/dongqiudi/top_news/1` — 头条新闻（验证 `ProcessFeedType2` 和 Unix 时间戳修复）
   - `http://localhost:1200/dongqiudi/team_news/50001755` — 皇家马德里新闻（验证 `ProcessFeed` 修复）
   - `http://localhost:1200/dongqiudi/player_news/50000339` — 球员新闻（验证 `return` 修复）
   - `http://localhost:1200/dongqiudi/result/50001755` — 皇马赛果（验证 `result.ts` 结构适配）
   - `http://localhost:1200/dongqiudi/special/41` — 新闻大爆炸（验证未受影响，回归测试）
   - `http://localhost:1200/dongqiudi/daily` — 早报重定向（验证未受影响）
4. **检查每个 feed**:
   - 返回的 XML/JSON 格式有效
   - 文章 `title`、`link`、`description`、`pubDate`、`author` 字段均填充正确
   - `pubDate` 不是 1970 年附近的时间
5. **Lint 检查**: `pnpm lint`（确认无新增告警，遵循 AGENTS.md 规范）
6. **二次请求**: 再次访问同一路由，验证 `cache.tryGet` 缓存命中正常
