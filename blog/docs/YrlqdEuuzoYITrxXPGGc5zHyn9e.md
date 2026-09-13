---
create_time: 1789222029
edit_time: 1789225625
title: 种红 Seedred 技术实现方案
categories:
  - product
---


<div class="callout callout-bg-5 callout-border-5">
<div class='callout-emoji'>📌</div>
<p> **对象：**种红 Seedred Web 工作台，版本 RFC-001，日期 2026-09-12。 **主模式：**Design RFC。 **实现范围：**MVP 做「选题 → 文案 → 封面 → Playwright 预览确认后发布」。不实现自动评论、打码过验、未授权全站爬取。</p>
<p>本文给实现者：先看目标与不变量，再按功能点表拆任务。架构图有文字等价，图不是唯一说明。父文档（可行性）只定方向，本页才是可验收的技术状态。</p>
</div>

# 1. 问题与证据

创作者要的是每天能发出去的笔记，不是又一个聊天框。卡点固定在三处：不知道发什么、写出来不像自己、封面在信息流里停不住。现成拼法是「新红看数据 + 大模型写字 + 稿定做图」，中间全靠复制粘贴。

平台侧约束已经核实：小红书主场在 App；官方开放平台偏商家交易；分享/小组件可以预填发布页；用户协议禁止未授权爬取和自动化访问。因此技术主路径必须是 **生成在云端、发布在用户手机上**。

成功标准（与可行性文档对齐，供验收）：

- 用户从空白到「可发布包」（标题+正文+话题+3:4 封面 PNG）≤ 8 分钟。
- 封面中文不乱码、不溢出安全区；标题 ≤ 20 字校验在服务端执行。
- 任何写向小红书的动作都停在用户确认之前。
- 单次成稿与封面都扣额度，免费用户打不爆模型账单。

# 2. 目标与非目标

<div class="flex gap-3 columns-2" column-size="2">
<div class="w-[50%]" width-ratio="50">
<div class="callout callout-bg-4 callout-border-4">
<div class='callout-emoji'>✅</div>
<p> <strong>目标（MVP）</strong></p>
<ul>
<li><p>人设驱动的选题、标题、正文、话题。</p>
</li>
<li><p>模板优先的 3:4 封面，支持实拍底图。</p>
</li>
<li><p>草稿日历与 Playwright 预览确认后发布。</p>
</li>
<li><p>额度、违禁词、AI 标识提示。</p>
</li>
</ul>
</div>
</div>
<div class="w-[50%]" width-ratio="50">
<div class="callout callout-bg-1 callout-border-1">
<div class='callout-emoji'>🚫</div>
<p> <strong>非目标（明确不做）</strong></p>
<ul>
<li><p>矩阵群控、云手机养号、自动打码、无人值守日更几十条。</p>
</li>
<li><p>自动点赞、评论、私信、养号。</p>
</li>
<li><p>自建全站达人库/热榜爬虫作为主数据。</p>
</li>
<li><p>聚光投放、蒲公英选号、多平台一键分发。</p>
</li>
</ul>
</div>
</div>
</div>

# 3. 约束与不变量

<table header_row="1">
<colgroup>
<col width="160"/>
<col width="660"/>
</colgroup>
<thead>
<tr><th><p>不变量</p></th><th><p>含义</p></th></tr>
</thead>
<tbody>
<tr><td><p>I-1 助发须确认</p></td><td><p>只操作用户本人扫码绑定的创作者后台；默认预览截图，用户确认后才点发布。验证码/风控立刻停。</p></td></tr>
<tr><td><p>I-2 无默认爬虫</p></td><td><p>选题语料默认来自用户粘贴/导入。第三方数据只能走合同化 API，代码里单独隔离。</p></td></tr>
<tr><td><p>I-3 额度先行</p></td><td><p>所有 LLM/生图调用先占额度再执行，失败回滚额度。</p></td></tr>
<tr><td><p>I-4 中文封面可排</p></td><td><p>封面主标题必须由排版引擎画字，禁止把「带汉字的整图」当作唯一路径。</p></td></tr>
<tr><td><p>I-5 内容归属用户</p></td><td><p>生成物版权归用户；服务端保留用于复现故障的短时日志，不拿用户笔记训练外部基础模型。</p></td></tr>
<tr><td><p>I-6 国内可达</p></td><td><p>登录、对象存储、推理默认走国内节点。海外模型只能当可选 Provider，不能当单点。</p></td></tr>
</tbody>
</table>

# 4. 部署形态：自建服务器 + Playwright 助发

<div class="callout callout-bg-2 callout-border-2">
<div class='callout-emoji'>💡</div>
<p> **2026-09-12 产品决定：**放弃「纯 EdgeOne 无主机」。改用一台（或一小台）常驻服务器跑 API、队列和 Playwright。原因：要在创作者后台替用户点发布，必须有带图形栈的浏览器进程，Edge 函数做不到。</p>
<p>这与小红书用户协议中「禁止自动化访问」冲突，账号有限流/封号风险。产品默认 <strong>预览确认后再发</strong>，不做矩阵群控、不自动过验证码、不自动评论私信。</p>
</div>

<table header_row="1">
<colgroup>
<col width="160"/>
<col width="220"/>
<col width="440"/>
</colgroup>
<thead>
<tr><th><p>组件</p></th><th><p>放哪</p></th><th><p>说明</p></th></tr>
</thead>
<tbody>
<tr><td><p>Web 工作台</p></td><td><p>同一台机 Nginx + Node，或前面仍可挂 EdgeOne CDN</p></td><td><p>静态资源可走 CDN；API 必须回源到有状态的机器。</p></td></tr>
<tr><td><p>API / 额度 / 草稿</p></td><td><p>Rust axum + PostgreSQL（jobs 表当队列）</p></td><td><p>生成任务进队列，避免 HTTP 超时。</p></td></tr>
<tr><td><p>封面渲染</p></td><td><p>Rust worker：resvg 出 PNG；Playwright 只用于发布</p></td><td><p>主路径仍是模板画字。Playwright 主要用于发布，不是为了画封面。</p></td></tr>
<tr><td><p>发布 Worker</p></td><td><p>Playwright + Chromium，headed 或 Xvfb</p></td><td><p>每用户独立 user-data-dir。一台 4c8g 先跑 1–2 个并发浏览器。</p></td></tr>
<tr><td><p>对象存储</p></td><td><p>COS / S3</p></td><td><p>封面、底图、发布失败截图。</p></td></tr>
</tbody>
</table>

## 4.1 Playwright 助发（F-PUB-03）

入口用官方创作者 PC 后台，不破解 App、不走非官方 HTTP 签名：

- 图文：https://creator.xiaohongshu.com/publish/publish （选上传图文）
- 或：https://creator.xiaohongshu.com/publish/imgNote

状态机：

1.  **绑定：**用户点「连接小红书」→ Worker 开独立 Chromium → 打开创作者后台 → 页面展示二维码（截图或推流）→ 用户用 App 扫码。成功后只把该用户的 profile 目录留在服务器，加密盘权限仅 Worker 可读。
2.  **预览（默认）：**填标题、正文、上传封面/配图、话题，停在「发布」按钮前截图，回传工作台。用户点确认才进入下一步。
3.  **发布：**点击发布，等待成功页或笔记 id。失败则保存截图 + HTML dump，任务标 failed，不重试死磕。
4.  **中断：**出现验证码、二次扫码、风控提示、选择器找不到：立即停，通知用户手动处理。系统不接打码平台。

<table header_row="1">
<colgroup>
<col width="200"/>
<col width="620"/>
</colgroup>
<thead>
<tr><th><p>规则</p></th><th><p>必须遵守</p></th></tr>
</thead>
<tbody>
<tr><td><p>账号归属</p></td><td><p>只发用户自己扫码绑定的号。禁止运营代登、禁止导入别人的 Cookie 文件做卖号。</p></td></tr>
<tr><td><p>频率</p></td><td><p>单账号默认每天最多 3 条自动发，间隔 ≥ 30 分钟。可配置，但封顶要写死。</p></td></tr>
<tr><td><p>并发</p></td><td><p>同一账号同一时刻只能有 1 个浏览器。禁止一个 profile 多开。</p></td></tr>
<tr><td><p>选择器</p></td><td><p>选择器与流程版本化。页面改版导致失败时降级为「导出包 + 人工发」，不要随机乱点。</p></td></tr>
<tr><td><p>日志</p></td><td><p>不把 Cookie 明文打进日志。截图保留 7 天用于排障后删除。</p></td></tr>
<tr><td><p>告知</p></td><td><p>绑定页必须展示：自动化可能违反平台规则并导致限流或封号，损失由用户承担。</p></td></tr>
</tbody>
</table>

机器规格（起步）：1 台 4 核 8G，Ubuntu，装 Chromium 依赖与中文字体，Redis + Postgres 可同机或托管。Playwright 任务用队列，不要在 API 进程里 launch 浏览器。

仍可用 EdgeOne 只做 CDN 和静态站，但发布 Worker 不能放到边缘函数。

# 5. 方案取舍

<table header_row="1">
<colgroup>
<col width="140"/>
<col width="200"/>
<col width="200"/>
<col width="280"/>
</colgroup>
<thead>
<tr><th><p>决策</p></th><th><p>采用</p></th><th><p>否决</p></th><th><p>原因与后果</p></th></tr>
</thead>
<tbody>
<tr><td><p>形态</p></td><td><p>Web 工作台（桌面优先）</p></td><td><p>先做 App / 先做 Chrome 插件</p></td><td><p>封面编辑和日历需要大屏。插件等 P1，且只能读用户当前打开的页面。</p></td></tr>
<tr><td><p>后端</p></td><td><p>CVM/Docker：API + Redis 队列 + Playwright Worker</p></td><td><p>纯 EdgeOne 无主机（没有浏览器就发不了帖）</p></td><td><p>Playwright 需要常驻 Chromium，必须有服务器。生成和发布都走 Redis 队列。</p></td></tr>
<tr><td><p>封面</p></td><td><p>JSON 模板 → SVG → PNG（Rust resvg）</p></td><td><p>纯文生图当主路径</p></td><td><p>模型画汉字不稳定。模板可锁定账号风格。后果：设计师要先做 8 套模板，而不是堆 prompt。</p></td></tr>
<tr><td><p>选题数据</p></td><td><p>用户导入爆款样本 + 人设 + 搜索词词库</p></td><td><p>上线即全站爬取</p></td><td><p>合规。后果：冷启动要引导用户贴 5–10 条对标笔记，否则选题会「像编的」。</p></td></tr>
<tr><td><p>发布</p></td><td><p>Playwright 打开创作者后台，默认预览确认后点发布；失败则导出包兜底</p></td><td><p>非官方签名发帖、Cookie 买卖、自动打码过验</p></td><td><p>I-1。后果：体验比 SuperX 差一截，但这是能上线的代价。</p></td></tr>
<tr><td><p>模型</p></td><td><p>OpenAI 兼容网关，默认可切换 DeepSeek / 豆包 / 通义</p></td><td><p>绑死单一厂商</p></td><td><p>标题用强模型，正文用便宜模型。网关统一计费、熔断、审计。</p></td></tr>
</tbody>
</table>

# 6. 技术栈（前后端）

2026-09-12 选型已改： **后端 Rust + PostgreSQL，前端 Vue 3 工作台。** 封面不再共用 React 组件：约定一份 JSON 模板协议，Vue 负责预览，Rust 用 resvg 出 PNG。Playwright 仍是独立 Node Worker（创作者后台自动化生态在 Node 上，不把 Chromium 嵌进 API 进程）。

<table header_row="1">
<colgroup>
<col width="140"/>
<col width="280"/>
<col width="400"/>
</colgroup>
<thead>
<tr><th><p>层</p></th><th><p>选型</p></th><th><p>说明</p></th></tr>
</thead>
<tbody>
<tr><td><p>前端</p></td><td><p>Vue 3 + Vite + TypeScript + Pinia + Ant Design Vue + Tailwind CSS</p></td><td><p>关掉 Tailwind preflight，避免和 Ant Design 全局样式互打。组件用 Ant Design Vue，间距/栅格用 Tailwind。</p></td></tr>
<tr><td><p>前端路由/请求</p></td><td><p>Vue Router 4 + ofetch/axios；生成进度 SSE</p></td><td><p>Pinia 存工作台状态（当前人设、草稿、额度）。不靠狂轮询。</p></td></tr>
<tr><td><p>API</p></td><td><p>Rust：tokio + axum + sqlx + PostgreSQL 16</p></td><td><p>鉴权、额度事务、草稿 CRUD、投递任务。OpenAPI 用 utoipa。迁移用 sqlx migrate。</p></td></tr>
<tr><td><p>队列</p></td><td><p>同一 Postgres：jobs 表 + FOR UPDATE SKIP LOCKED</p></td><td><p>少引入 Redis。生成队列和发布队列用 job_kind 分开。API 入队，Worker 拉任务。</p></td></tr>
<tr><td><p>生成 Worker</p></td><td><p>同一 Rust workspace 的 seedred-worker 二进制</p></td><td><p>调 OpenAI 兼容 HTTP（豆包/DeepSeek/通义），resvg 出封面 PNG，写 COS。</p></td></tr>
<tr><td><p>发布 Worker</p></td><td><p>Node 20 + Playwright + Chromium</p></td><td><p>只做扫码、预览截图、点发布。Rust 用 HTTP/队列把任务交给它。不在 Rust 里重写一套浏览器自动化。</p></td></tr>
<tr><td><p>封面协议</p></td><td><p>cover.schema.json（标题、色板、底图、模板 id）</p></td><td><p>前端 Vue 按 schema 画预览；后端把 schema 编成 SVG 再 resvg。禁止纯文生图当主路径。</p></td></tr>
<tr><td><p>对象存储</p></td><td><p>腾讯云 COS，预签名上传</p></td><td><p>底图不经 API 内存中转。</p></td></tr>
<tr><td><p>登录支付</p></td><td><p>手机号短信 + 可选微信扫码；微信支付</p></td><td><p>Rust 侧用 reqwest 调微信/短信网关。</p></td></tr>
<tr><td><p>部署</p></td><td><p>Docker Compose：api / worker-gen / worker-publish / postgres / nginx</p></td><td><p>api 与 worker-gen 是 Rust 镜像；worker-publish 是 Node+Playwright 镜像。</p></td></tr>
</tbody>
</table>

```
apps/web                 Vue 3 + Vite 工作台
crates/seedred-api       axum HTTP
crates/seedred-worker    选题 / 文案 / resvg 封面
crates/seedred-core      schema、额度、违禁词
apps/worker-publish      Playwright 扫码与发布
packages/cover-schema    cover.schema.json 前后端共用
```

起步机器仍是 Ubuntu 4 核 8G。nginx 反代 web 静态和 /api。worker-publish 并发先限 1–2 个 Chromium。

# 7. 系统分层

文字等价：用户用工作台做选题文案封面。API 把生成和发布写入 Postgres jobs 表。生成 Worker 调模型和出 PNG。发布 Worker 用该用户自己的 Chromium 目录打开创作者后台：先截预览，用户确认后再点发布。验证码则停。没有边缘函数代发。

# 8. 生成管线

文字等价：每一步都可单独重跑。选题不自动变成已发布笔记。封面在文案标题锁定后再渲染，避免先画图再改标题导致废图。发布永远是用户在小红书 App 内完成。

# 9. 功能点清单

状态：MVP 必须在第 0–3 个月上线；P1 第 4–9 个月；P2 第 10–18 个月；不做 除非本 RFC 被修订。ID 用于开任务，不要改语义只改编号。

## 9.1 账号、工作台、计费

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-AUTH-01</p></td><td><p>MVP</p></td><td><p>手机号验证码登录；可选微信扫码</p></td><td><p>未登录不能生成。验证码 60 秒限频。会话 30 天可续。</p></td></tr>
<tr><td><p>F-AUTH-02</p></td><td><p>MVP</p></td><td><p>个人工作空间，显示套餐与剩余额度</p></td><td><p>顶栏永久可见「成稿 x/80、封面 y/80」。</p></td></tr>
<tr><td><p>F-BILL-01</p></td><td><p>MVP</p></td><td><p>Free / Creator / Pro 额度：成稿次数、封面张数、人设数</p></td><td><p>Free：8 成稿 / 3 封面 / 1 人设。Creator：80 / 80 / 1。超限返回 402，不扣模型。</p></td></tr>
<tr><td><p>F-BILL-02</p></td><td><p>MVP</p></td><td><p>用量台账：每次生成写 ledger（占用、成功、回滚）</p></td><td><p>失败任务 100% 回滚额度。可按日导出。</p></td></tr>
<tr><td><p>F-BILL-03</p></td><td><p>P1</p></td><td><p>微信/支付宝订阅与年付</p></td><td><p>支付成功 1 分钟内额度生效。退款走人工。</p></td></tr>
<tr><td><p>F-TEAM-01</p></td><td><p>P1</p></td><td><p>Team 席位、10 个账号文件夹、成员角色 owner/editor/viewer</p></td><td><p>viewer 不能消耗额度。账号素材不串。</p></td></tr>
</tbody>
</table>

## 9.2 人设仓

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-PER-01</p></td><td><p>MVP</p></td><td><p>导入历史笔记：粘贴文本，或每行一条；上限 30 条，单条 2000 字</p></td><td><p>超限拒绝。导入后可删除单条。不主动去抓用户主页。</p></td></tr>
<tr><td><p>F-PER-02</p></td><td><p>MVP</p></td><td><p>抽取风格卡：常用词、句长、称呼、主题、绝不说的话</p></td><td><p>30 秒内出卡。用户可编辑每一项。后续成稿必须注入此卡。</p></td></tr>
<tr><td><p>F-PER-03</p></td><td><p>MVP</p></td><td><p>人设切换：MVP 1 个默认人设，Pro 可存 3 个</p></td><td><p>生成请求必须带 persona_id，缺省用默认。</p></td></tr>
<tr><td><p>F-PER-04</p></td><td><p>P1</p></td><td><p>从「已发布笔记复盘」回流更新人设（只使用用户勾选的成功笔记）</p></td><td><p>不自动覆盖用户手改字段。</p></td></tr>
</tbody>
</table>

## 9.3 爆款选题

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-TOP-01</p></td><td><p>MVP</p></td><td><p>赛道配置：品类、目标人群、禁写主题、目标（涨粉/种草/商单）</p></td><td><p>无赛道不能生成每日选题。</p></td></tr>
<tr><td><p>F-TOP-02</p></td><td><p>MVP</p></td><td><p>爆款拆解：用户粘贴笔记链接或全文；抽出标题公式、封面类型、钩子、评论区问法</p></td><td><p>链接若无法公开抓取，降级为「请粘贴正文」，不得循环重试破解。</p></td></tr>
<tr><td><p>F-TOP-03</p></td><td><p>MVP</p></td><td><p>对标样本库：手动保存至少 5 条拆解结果</p></td><td><p>样本不足 5 条时，选题页提示完整度，仍允许生成但打「低置信」标签。</p></td></tr>
<tr><td><p>F-TOP-04</p></td><td><p>MVP</p></td><td><p>生成 12 条选题：含搜索向 / 人设向 / 结构复用三类标签，每条有角度一句话</p></td><td><p>去重：与近 14 天已用选题相似度 &gt; 0.85 的丢弃。用户可「换一批」（扣 0.5 次成稿额度）。</p></td></tr>
<tr><td><p>F-TOP-05</p></td><td><p>MVP</p></td><td><p>选题操作：收藏、丢弃、一键去写</p></td><td><p>「去写」创建草稿并带上 topic_id、结构标签。</p></td></tr>
<tr><td><p>F-TOP-06</p></td><td><p>P1</p></td><td><p>搜索词辅助：内置品类词库 + 用户自定义词，选题必须带 1 个可搜词</p></td><td><p>词库可运营后台更新，不依赖爬热搜。</p></td></tr>
<tr><td><p>F-TOP-07</p></td><td><p>P2</p></td><td><p>合同化数据源适配器（新红/灰豚类 API 或官方灵感）</p></td><td><p>适配器开关默认关。无合同密钥则功能对用户隐藏。</p></td></tr>
<tr><td><p>F-TOP-X</p></td><td><p>不做</p></td><td><p>未授权全站热榜/达人库爬取</p></td><td><p>代码审查禁止出现默认爬虫任务。</p></td></tr>
</tbody>
</table>

## 9.4 文案生成

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-CPY-01</p></td><td><p>MVP</p></td><td><p>一次出 8 个标题。服务端硬校验：去空白后 ≤ 20 字（中文 1 字）</p></td><td><p>超长标题不得入库。带数字/痛点/反差的比例不作为硬指标，只做提示。</p></td></tr>
<tr><td><p>F-CPY-02</p></td><td><p>MVP</p></td><td><p>正文：短段、空行、可选 emoji 密度低/中/关；300–800 字可调</p></td><td><p>必须注入人设卡。禁止出现「作为 AI」类元话语。</p></td></tr>
<tr><td><p>F-CPY-03</p></td><td><p>MVP</p></td><td><p>话题 5 个以内、地点可选、评论区置顶 3 条</p></td><td><p>话题可一键复制。置顶话术不含微信号/外链。</p></td></tr>
<tr><td><p>F-CPY-04</p></td><td><p>MVP</p></td><td><p>体裁模板：清单、对比、避雷、测评、教程，用户必选其一</p></td><td><p>模板决定段落骨架，模型只填内容。</p></td></tr>
<tr><td><p>F-CPY-05</p></td><td><p>MVP</p></td><td><p>局部重写：只改标题 / 只改开头 / 更口语 / 更干货</p></td><td><p>局部重写扣 0.3 次额度。原文保留版本。</p></td></tr>
<tr><td><p>F-CPY-06</p></td><td><p>MVP</p></td><td><p>形态：图文笔记或短视频口播脚本（镜头提示，不生成视频）</p></td><td><p>脚本含 15 秒内钩子。不接视频渲染。</p></td></tr>
<tr><td><p>F-CPY-07</p></td><td><p>P1</p></td><td><p>「像我」打分：相对人设样本的用词重叠与句长差，仅提示</p></td><td><p>不把分数展示成「必爆概率」。</p></td></tr>
</tbody>
</table>

## 9.5 爆款封面

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-COV-01</p></td><td><p>MVP</p></td><td><p>画布 3:4，导出 1242×1660 PNG，sRGB</p></td><td><p>文件 &lt; 5MB。下载文件名含草稿 id。</p></td></tr>
<tr><td><p>F-COV-02</p></td><td><p>MVP</p></td><td><p>模板 ≥ 8 套：大字报、清单 5 条、左右对比、实拍叠字、数字结论、前后对比、问答、纯色金句</p></td><td><p>每套有安全区。标题被截断则渲染失败，不交付缺字图。</p></td></tr>
<tr><td><p>F-COV-03</p></td><td><p>MVP</p></td><td><p>中文排版引擎画主标题/副标题，支持描边与对比色</p></td><td><p>抽样 100 条真实标题，溢出率 = 0。不用模型「画」汉字。</p></td></tr>
<tr><td><p>F-COV-04</p></td><td><p>MVP</p></td><td><p>上传 1 张底图（JPG/PNG/WebP ≤ 10MB），可选背景虚化</p></td><td><p>无底图时用模板纯色/纹理。不盗用他人笔记封面图作为默认素材。</p></td></tr>
<tr><td><p>F-COV-05</p></td><td><p>MVP</p></td><td><p>同稿 3 变体：不同模板或配色，一次扣 3 张封面额度</p></td><td><p>用户可单张重出（扣 1 张）。</p></td></tr>
<tr><td><p>F-COV-06</p></td><td><p>MVP</p></td><td><p>封面编辑：改字、换模板、换底图后实时预览（前端 SVG），点生成才出 PNG</p></td><td><p>预览不扣生图额度。出 PNG 才扣。</p></td></tr>
<tr><td><p>F-COV-07</p></td><td><p>MVP</p></td><td><p>账号封面风格：主色、辅色、字体（2 个中文字体）、贴纸密度</p></td><td><p>新封面默认继承。风格变更不改历史 PNG。</p></td></tr>
<tr><td><p>F-COV-08</p></td><td><p>P1</p></td><td><p>可选 AI 底图（知识/插画类），单独额度；默认关</p></td><td><p>生成底图后仍走排版引擎叠字。产品文案提示需 AI 标识。</p></td></tr>
<tr><td><p>F-COV-09</p></td><td><p>P1</p></td><td><p>主体抠图（人/产品）贴到模板前景</p></td><td><p>抠图失败则回退原图，不阻塞导出。</p></td></tr>
</tbody>
</table>

## 9.6 合规、草稿、发布

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-CMP-01</p></td><td><p>MVP</p></td><td><p>违禁词：标题、正文、封面文字、话题</p></td><td><p>命中则标红可发但强提示。词库可热更新。不承诺过审。</p></td></tr>
<tr><td><p>F-CMP-02</p></td><td><p>MVP</p></td><td><p>导流检测：微信、QQ、二维码话术、站外短链</p></td><td><p>评论区话术命中则禁止一键复制，必须用户改。</p></td></tr>
<tr><td><p>F-CMP-03</p></td><td><p>MVP</p></td><td><p>发布前 AI 标识检查清单（不可跳过勾选）</p></td><td><p>未勾选不能进入导出/唤起。</p></td></tr>
<tr><td><p>F-CMP-04</p></td><td><p>P1</p></td><td><p>近 7 天草稿文本相似度提示</p></td><td><p>仅提示，不拦截。禁止宣传「洗稿过检测」。</p></td></tr>
<tr><td><p>F-DRF-01</p></td><td><p>MVP</p></td><td><p>周视图日历；状态：选题 / 成稿 / 有封面 / 已导出 / 已发布</p></td><td><p>「已发布」只能用户手勾，系统不回调小红书。</p></td></tr>
<tr><td><p>F-DRF-02</p></td><td><p>MVP</p></td><td><p>草稿版本：每次成稿/封面生成追加 version</p></td><td><p>可回滚上一版文案，不回滚已删底图。</p></td></tr>
<tr><td><p>F-DRF-03</p></td><td><p>MVP</p></td><td><p>建议发布时间：无历史则用品类默认（工作日 12:00、19:30）</p></td><td><p>不展示伪精确「算法最佳秒」。</p></td></tr>
<tr><td><p>F-PUB-01</p></td><td><p>MVP</p></td><td><p>导出发布包：title.txt、body.txt、tags.txt、cover.png</p></td><td><p>一键复制正文。包内不含小红书 cookie。</p></td></tr>
<tr><td><p>F-PUB-02</p></td><td><p>MVP</p></td><td><p>分享开放平台 / 小组件唤起；未开通时隐藏按钮只留导出</p></td><td><p>功能开关 PUBLISH_SHARE_SDK。真机未通过则默认关。</p></td></tr>
<tr><td><p>F-PUB-03</p></td><td><p>MVP</p></td><td><p>Playwright 助发：扫码绑定创作者后台，默认预览确认后点发布</p></td><td><p>绑定二维码、预览截图、确认后发出；验证码立即停；单账号日限 3 条。群控仍不做。</p></td></tr>
</tbody>
</table>

## 9.7 P1 / P2 扩展

<table header_row="1">
<colgroup>
<col width="110"/>
<col width="70"/>
<col width="280"/>
<col width="360"/>
</colgroup>
<thead>
<tr><th><p>ID</p></th><th><p>优先级</p></th><th><p>功能</p></th><th><p>验收</p></th></tr>
</thead>
<tbody>
<tr><td><p>F-ANL-01</p></td><td><p>P1</p></td><td><p>本账号复盘：用户手动填阅读/点赞/收藏，或粘贴创作中心截图后 OCR 数字</p></td><td><p>不声称官方数据。OCR 失败则让用户手填。</p></td></tr>
<tr><td><p>F-CMT-01</p></td><td><p>P1</p></td><td><p>评论回复建议：用户粘贴一条评论，出 3 条回复，用户自己发</p></td><td><p>不提供批量回复、不连私信。</p></td></tr>
<tr><td><p>F-EXT-01</p></td><td><p>P1</p></td><td><p>浏览器插件：仅在 xiaohongshu.com 笔记页显示「拆到种红」</p></td><td><p>只读当前页用户可见内容。无自动翻页。</p></td></tr>
<tr><td><p>F-API-01</p></td><td><p>P2</p></td><td><p>用户 API / MCP：日历与生成，不含发布</p></td><td><p>鉴权用个人 token。速率限制写进网关。</p></td></tr>
</tbody>
</table>

# 10. 数据模型（MVP）

PostgreSQL。关键表：

<table header_row="1">
<colgroup>
<col width="160"/>
<col width="660"/>
</colgroup>
<thead>
<tr><th><p>表</p></th><th><p>关键字段</p></th></tr>
</thead>
<tbody>
<tr><td><p>users</p></td><td><p>id, phone, wechat_openid, plan, period_end</p></td></tr>
<tr><td><p>workspaces</p></td><td><p>id, owner_id, name</p></td></tr>
<tr><td><p>personas</p></td><td><p>id, workspace_id, style_json, samples_json, is_default</p></td></tr>
<tr><td><p>topic_samples</p></td><td><p>id, workspace_id, raw_text, source_url nullable, parse_json</p></td></tr>
<tr><td><p>topics</p></td><td><p>id, workspace_id, title, angle, tags[], status</p></td></tr>
<tr><td><p>drafts</p></td><td><p>id, workspace_id, topic_id, persona_id, genre, title, body, topics[], status, scheduled_at</p></td></tr>
<tr><td><p>draft_versions</p></td><td><p>id, draft_id, payload_json, created_at</p></td></tr>
<tr><td><p>covers</p></td><td><p>id, draft_id, template_id, overlay_json, png_key, width, height</p></td></tr>
<tr><td><p>jobs</p></td><td><p>id, type, status, input_json, error, cost_cents</p></td></tr>
<tr><td><p>usage_ledger</p></td><td><p>id, user_id, kind, delta, job_id, state occupied/committed/rolled_back</p></td></tr>
</tbody>
</table>

对象存储路径：ws/{workspaceId}/assets/{id} 用户底图；ws/{workspaceId}/covers/{coverId}.png 成品。URL 带 15 分钟签名。

# 11. 接口契约（MVP 摘要）

HTTP JSON，鉴权 Bearer session。幂等：写接口接受 Idempotency-Key。生成类全部 202 + job_id，客户端轮询 GET /v1/jobs/:id。

<table header_row="1">
<colgroup>
<col width="280"/>
<col width="540"/>
</colgroup>
<thead>
<tr><th><p>接口</p></th><th><p>行为</p></th></tr>
</thead>
<tbody>
<tr><td><p>POST /v1/personas/extract</p></td><td><p>从 samples 抽风格卡。扣 0 额度。超时 30s。</p></td></tr>
<tr><td><p>POST /v1/topics/parse</p></td><td><p>拆一条爆款。无爬虫权限时 body 必填。</p></td></tr>
<tr><td><p>POST /v1/topics/generate</p></td><td><p>出 12 条选题。占成稿额度 1。失败回滚。</p></td></tr>
<tr><td><p>POST /v1/drafts</p></td><td><p>从 topic 创建草稿。</p></td></tr>
<tr><td><p>POST /v1/drafts/:id/copy</p></td><td><p>生成标题+正文。占成稿 1。</p></td></tr>
<tr><td><p>POST /v1/drafts/:id/covers</p></td><td><p>渲染 3 变体。占封面 3。返回 PNG 签名 URL。</p></td></tr>
<tr><td><p>POST /v1/drafts/:id/export</p></td><td><p>校验 AI 标识勾选与导流项后出包。</p></td></tr>
<tr><td><p>POST /v1/drafts/:id/share-bridge</p></td><td><p>仅当 SDK 开关打开。返回唤起参数，不代发。</p></td></tr>
</tbody>
</table>

错误：402 quota_exceeded；409 compliance_block（导流未改）；422 title_too_long；424 provider_unavailable（模型熔断，可重试）。提供 /v1/xhs/bind 与 /v1/drafts/:id/publish（排队，默认 preview）。

# 12. 封面渲染设计

模板输入是 JSON（与 cover.schema.json 一致），示例：

```json
{
  "templateId": "big-type-01",
  "title": "这 5 个坑我替你踩了",
  "subtitle": "新手做副业",
  "items": ["坑1", "坑2"],
  "palette": { "bg": "#111", "fg": "#FF2D55" },
  "baseImageKey": "optional"
}
```

流水线：前端 Vue 按 schema 预览 → 用户点生成 → Rust worker 编 SVG、resvg 出 PNG、写入 COS。中文字体随 worker 镜像打包（思源黑体/阿里巴巴普惠体），禁止运行时下载。

失败：标题超安全区、底图解码失败、字体缺字。缺字直接 fail，不要用方框交付。

# 13. 模型网关

- 统一 Chat Completions。每请求写入 job.cost_cents（按官方价 × 1.2 缓冲）。
- 路由：标题 F-CPY-01 用较强模型；选题与正文用便宜模型；人设抽取用中等。
- Prompt 版本化，表 prompt_versions，生成记录带 version，禁止把 prompt 写死在五处。
- 熔断：某 Provider 连续 5 次 5xx 或 p95 &gt; 20s，切备用。全部失败返回 424。

# 14. 安全、失败、兼容

- PII：手机号加密存储。笔记样本默认不进分析日志正文，只留 hash 与长度。
- 上传：MIME 白名单、图片解码后再编码，防伪装。
- Prompt 注入：用户样本放在明确的 XML 分隔区，系统指令禁止服从样本里的「忽略以上」。
- Rollback：额度占用 15 分钟未 committed 自动回滚。渲染失败不留半成品给用户。
- 无数据迁移负担（新系统）。SDK 开关默认 false，打开不改表结构。

# 15. 上线、观测、验收

MVP 上线门禁（同时满足）：

1. F-TOP-04、F-CPY-01/02、F-COV-02/03、F-PUB-01、F-BILL-02 自动化测试绿。
2. 封面中文溢出测试 100/100。
3. 额度失败回滚测试：杀掉 worker 后 ledger 全部 rolled_back。
4. 代码扫描：仓库无小红书 cookie 字段、无 puppeteer 登录小红书脚本。
5. 内部 5 人走通「导入 8 条样本 → 出选题 → 成稿 → 三封面 → 导出」≤ 8 分钟。

观测：job 成功率、p95 耗时、单次成本、402 率、渲染失败原因。告警：日模型成本超预算 20%、队列堆积 &gt; 5 分钟。

回滚：前端 feature flag 关闭生成按钮；Worker 停消费；已扣费额度不追回（财务例外再处理）。

# 16. 建议实现顺序（90 天）

1. 第 1–2 周：鉴权、额度 ledger、草稿表、空工作台。
2. 第 3–4 周：人设抽取 + 文案成稿（先可复制文本）。
3. 第 5–7 周：8 套封面模板 + 服务端渲染 + 三变体。
4. 第 8–9 周：选题生成（依赖样本库）+ 日历 + 导出包。
5. 第 10 周：违禁词、AI 标识、打磨耗时；并行申请分享 SDK。
6. 第 11–12 周：内测 20 人，修溢出与「不像我」，准备 Creator 付费开关。

# 17. 未决

- 分享 SDK 申请周期与能否预填多图： **blocked 到真机实验**。不影响导出包路径。
- 公开笔记链接解析是否稳定：默认当不稳定处理，UI 以粘贴正文为主。
- 微信支付主体与类目：上线付费前必须有。
- 字体商用授权具体合同：渲染上线前必须有。

本 RFC 不构成自动发帖或数据采集的授权。若要增加爬虫或代发，必须另开 RFC 并废止 I-1 / I-2。

