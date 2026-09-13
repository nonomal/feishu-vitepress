---
create_time: 1789222498
edit_time: 1789222932
title: 火候键盘：功能点与技术实现方案
categories:
  - product
---


<div class="callout callout-bg-2 callout-border-2">
<div class='callout-emoji'>📌</div>
<p>本文是<a href="KHEpdIyFwoqfuuxDlCWcuXhonZg">对标 Lovekey 做 AI 聊天回复键盘的可行性、收益与方案</a>的实现稿。品牌已定为 <strong>火候</strong>（英文 Finesse）。读者是产品和研发。范围以  <strong>P0：iOS 容器 App + 键盘扩展</strong> 为准；安卓、截图记忆、文风学习放 P1。未写进验收标准的，不算做完。</p>
</div>

资料时点：2026-09-12。技术约束按当前 iOS 自定义键盘规则理解，上架前须用真机和 App Store 审核再核一次。

# 1. 要做成什么样

一句话：用户在微信里卡住时，切到「火候」键盘，贴上对方的话，选出关系火候，得到 3 条像自己、不过火的回复，点一下写进输入框。

成功标准沿用立项稿，本文件只负责「P0 怎样算做完」：

- 国区 App Store 可上架；无完全访问时键盘能打字、能切走；有完全访问后「帮你回」可用。
- 从打开 App 到第一次成功插入回复，新手引导可独立走完，不依赖客服。
- 免费额度用尽后走容器 App 内购，键盘扩展里不出现商品页或广告。
- 不记录普通击键；密码类输入框不展示「帮你回」。

## 1.1 范围与非目标

<table header_row="1">
<colgroup>
<col width="328"/>
<col width="328"/>
</colgroup>
<thead>
<tr><th><p>做（P0）</p></th><th><p>明确不做</p></th></tr>
</thead>
<tbody>
<tr><td><p>iOS 16+ 容器 App、键盘扩展、帮你回、火候档、预置人设、恋爱/职场场景、免费额度、IAP、安全过滤、基础引导</p></td><td><p>自研拼音引擎、皮肤商店、自动读微信聊天、无障碍爬屏、代发消息、虚拟伴侣、键盘内广告/内购、山寨 Lovekey 素材</p></td></tr>
</tbody>
</table>

## 1.2 术语

<table header_row="1">
<colgroup>
<col width="115"/>
<col width="531"/>
</colgroup>
<thead>
<tr><th><p>词</p></th><th><p>含义</p></th></tr>
</thead>
<tbody>
<tr><td><p>火候</p></td><td><p>关系远近档，0–4。决定回复进取程度，不是人格标签。</p></td></tr>
<tr><td><p>人设</p></td><td><p>说话口气（温和、直接、清楚…）。与火候独立，可组合。</p></td></tr>
<tr><td><p>帮你回</p></td><td><p>用对方原文（可加自己上一句）生成 3 条候选并插入输入框。</p></td></tr>
<tr><td><p>完全访问</p></td><td><p>系统「允许完全访问」。没有它，键盘不能联网，也不能读剪贴板。</p></td></tr>
<tr><td><p>上屏</p></td><td><p>调用 <code>textDocumentProxy.insertText</code> 把候选写入宿主 App 输入框。</p></td></tr>
</tbody>
</table>

# 2. 谁在什么情况下用

<table header_row="1">
<colgroup>
<col width="152"/>
<col width="293"/>
<col width="293"/>
</colgroup>
<thead>
<tr><th><p>角色</p></th><th><p>情境</p></th><th><p>想要的结果</p></th></tr>
</thead>
<tbody>
<tr><td><p>卡壳的聊天者</p></td><td><p>微信收到一句，删了又打，3 分钟发不出去</p></td><td><p>30 秒内有 3 条能发出去的、不过火的回复</p></td></tr>
<tr><td><p>怕得罪人的职场用户</p></td><td><p>要回领导/客户，怕太硬或太软</p></td><td><p>切到职场场景，生成清楚、客气的回复</p></td></tr>
<tr><td><p>第一次装的人</p></td><td><p>不知道第三方键盘怎么开</p></td><td><p>按 App 里的步骤打开键盘和完全访问，第一次生成成功</p></td></tr>
</tbody>
</table>

证据来自立项稿对 Lovekey 的拆解：付费来自「卡壳时立刻能发」，不是来自输入法本身。本产品用火候和人设降低油腻，而不是堆话术市场。

# 3. 功能地图

<table header_row="1">
<colgroup>
<col width="96"/>
<col width="156"/>
<col width="328"/>
<col width="80"/>
</colgroup>
<thead>
<tr><th><p>优先级</p></th><th><p>功能</p></th><th><p>用户可观察到什么</p></th><th><p>P0 是否必须</p></th></tr>
</thead>
<tbody>
<tr><td><p>P0</p></td><td><p>新手引导</p></td><td><p>分步教：添加键盘 → 完全访问 → 微信里切到火候</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>无权限键盘</p></td><td><p>未开完全访问时仍能打字、切键盘；帮你回入口说明缺权限</p></td><td><p>是（审核）</p></td></tr>
<tr><td><p>P0</p></td><td><p>帮你回</p></td><td><p>粘贴对方原文，生成 3 条，点选上屏</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>火候档</p></td><td><p>0–4 五档，生成结果明显不同档</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>场景</p></td><td><p>恋爱 / 职场一键切，默认恋爱</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>预置人设</p></td><td><p>8 个口气，键盘与 App 同步</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>额度与会员</p></td><td><p>免费 20 次；月 38 / 年 128 / 终身 298；键盘内只提示去 App 开通</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>安全阀</p></td><td><p>骚扰、威胁、未成年相关被拒；火候 0–1 不出口味过重的句子</p></td><td><p>是</p></td></tr>
<tr><td><p>P0</p></td><td><p>隐私底线</p></td><td><p>不上传击键；安全输入框无帮你回</p></td><td><p>是</p></td></tr>
<tr><td><p>P1</p></td><td><p>截图补上下文</p></td><td><p>App 里选聊天截图，勾选几轮后供键盘使用</p></td><td><p>否</p></td></tr>
<tr><td><p>P1</p></td><td><p>会话记忆</p></td><td><p>按「这个人」记住火候和最近几轮</p></td><td><p>否</p></td></tr>
<tr><td><p>P1</p></td><td><p>文风</p></td><td><p>用户贴自己发过的话，之后更像他</p></td><td><p>否</p></td></tr>
<tr><td><p>P1</p></td><td><p>开场白</p></td><td><p>无对方原文时，按场景生成第一句</p></td><td><p>否</p></td></tr>
<tr><td><p>P1</p></td><td><p>安卓</p></td><td><p>同一套 API 的 IME</p></td><td><p>否</p></td></tr>
<tr><td><p>P2</p></td><td><p>职场包 / 客服口吻</p></td><td><p>更深模板与年付主推</p></td><td><p>否</p></td></tr>
</tbody>
</table>

# 4. P0 行为与验收

## 4.1 新手引导

 **前置：**用户刚安装，尚未添加键盘。

 **行为：**容器 App 首页按 4 步走，每步有系统设置跳转或示意图：打开键盘列表 → 添加「火候」→ 打开「允许完全访问」→ 在任意输入框长按地球键切到火候。App 能检测「键盘是否已添加」「是否完全访问」（扩展侧 `hasFullAccess` 写入 App Group，App 读取）。

 **验收：**

- 未添加键盘时，不能把「开始聊天」画成已就绪。
- 已添加但未完全访问时，文案明确说：现在可以打字，帮你回需要打开完全访问；提供跳转。
- 两状态都满足后，首页出现「去微信试一次」和一张 10 秒内能看完的示意图。
- 不要求用户注册才能做完引导。

## 4.2 无完全访问时的键盘

 **行为：**扩展必须带地球键。未完全访问时提供可插入字符的基础 QWERTY（含常用标点、删除、换行），帮你回区域展示锁态说明，不发起网络请求。

 **验收：**关闭完全访问后，在备忘录里能打出「你好」，能切到其他键盘。Instruments 下扩展内存峰值 &lt; 50 MB。审核用机按此路径走一遍不崩溃。

## 4.3 帮你回

 **前置：**完全访问已开；剪贴板里是用户刚复制的对方一句话，或用户在键盘粘贴框里改过。

 **行为：**

1. 打开键盘时，若剪贴板是纯文本且长度 1–500 字，预填到「对方说了」。超过则截断并提示。
2. 可选填「我上一句」，可空。
3. 点「按火候生成」。生成中按钮不可重复点。流式展示或完成后一次出示 3 条，标签固定为「稳妥 / 合适 / 更进一步」。
4. 点某条即上屏。上屏后保留 3 条，允许改点另一条（先删已插入内容再插入新内容；若删不干净则改为追加，并 toast「已插入，请在对话框里检查」）。

 **验收：**

- 微信、备忘录、iMessage 各插入一次成功。
- 从点生成到 3 条齐全，Wi-Fi 下 P50 &lt; 2.5 s，P95 &lt; 6 s（国内机房、短句）。超时 8 s 给出重试，不白屏。
- 空输入点生成：提示「先贴上对方的话」，不扣次数。
- 3 条互不相同；火候 0 与火候 4 对同一句的「更进一步」明显不同（抽检 10 条，人工过）。

## 4.4 火候档与场景、人设

<table header_row="1">
<colgroup>
<col width="97"/>
<col width="306"/>
<col width="328"/>
</colgroup>
<thead>
<tr><th><p>火候</p></th><th><p>恋爱场景含义</p></th><th><p>职场场景含义</p></th></tr>
</thead>
<tbody>
<tr><td><p>0 生</p></td><td><p>刚加好友 / 相亲第一句</p></td><td><p>不熟的客户或跨部门</p></td></tr>
<tr><td><p>1 淡</p></td><td><p>说过几句</p></td><td><p>普通同事</p></td></tr>
<tr><td><p>2 常</p></td><td><p>经常聊，未明确关系</p></td><td><p>合作中的同事</p></td></tr>
<tr><td><p>3 近</p></td><td><p>暧昧或已在交往</p></td><td><p>熟络、可略轻松</p></td></tr>
<tr><td><p>4 熟</p></td><td><p>伴侣或很熟，仍禁止油腻连发</p></td><td><p>内部很熟，仍禁止情绪发泄式回复</p></td></tr>
</tbody>
</table>

预置人设 P0 共 8 个：恋爱侧温和、直接、轻松、认真、克制；职场侧清楚、客气、坚定。人设只改口气，不改火候。

 **验收：**键盘改火候/场景/人设后，杀键盘进程再开，状态仍在（App Group）。默认：场景=恋爱，火候=2，人设=温和。首页不做「网红话术市场」瀑布流。

## 4.5 额度与会员

 **行为：**每个 Apple ID（拿不到则用 Keychain 设备号）免费 20 次成功生成。空输入、安全拦截、服务端 5xx 不扣次。用尽后键盘展示剩余 0，主按钮变为「去火候开通」，点击用 URL Scheme / App Group 打开容器 App 会员页。会员页提供：连续包月 38 元、包年 128 元、终身 298 元。终身不是首屏主按钮，主按钮是包年。

 **验收：**

- 键盘扩展 UI 内无 SKPayment、无价格表、无广告。
- 恢复购买能把同一 Apple ID 的年费/终身拉回来。
- 沙盒账号买月费后，扩展在 10 s 内额度变为无限（或日上限 200 次防刷）。

相对立项稿的收窄：P0 只做「20 次不限时」，不做「20 次或 24 小时取先到」。24 小时窗更像卡死试用。转化不够时再加时间窗，不作为上架门槛。

## 4.6 安全阀与隐私

 **行为：**服务端在模型前后都做规则+模型分类。直接拒绝：骚扰纠缠、威胁恐吓、性相关且涉及未成年、诈骗话术。火候 0–1 时，把露骨暧昧改写成中性。被拒时键盘展示「这条不适合代写」，不返回擦边改写充数。安全输入：当 `keyboardType` 为 `numberPad` / `phonePad` / `decimalPad`，或 `textContentType` 能判断为 password / oneTimeCode 时，隐藏帮你回。

 **验收：**测试集至少 30 条攻击句（纠缠、骂人、骗验证码、未成年）拦截率 ≥ 90%，误杀日常恋爱句 &lt; 10%（集合外置，上线前人工标）。完全访问说明里写清：只在用户点生成时上传「对方原文 / 我上一句 / 火候人设」，不上传击键。

# 5. 主路径与异常

正常路径：安装 → 引导完成 → 微信复制对方句子 → 切火候 → 生成 3 条 → 点「合适」上屏 → 用户可改字再发。

<table header_row="1">
<colgroup>
<col width="207"/>
<col width="165"/>
<col width="328"/>
</colgroup>
<thead>
<tr><th><p>异常</p></th><th><p>表现</p></th><th><p>处理</p></th></tr>
</thead>
<tbody>
<tr><td><p>无完全访问</p></td><td><p>帮你回锁态</p></td><td><p>说明 + 跳转设置；不请求网络</p></td></tr>
<tr><td><p>剪贴板空或非文本</p></td><td><p>粘贴框空</p></td><td><p>占位「先复制对方的话」；不扣次</p></td></tr>
<tr><td><p>网络失败 / 超时</p></td><td><p>生成失败</p></td><td><p>重试一次；仍失败则提示，不扣次</p></td></tr>
<tr><td><p>额度用尽</p></td><td><p>不能生成</p></td><td><p>去 App 开通；已生成的 3 条仍可上屏</p></td></tr>
<tr><td><p>安全拒绝</p></td><td><p>无 3 条</p></td><td><p>固定文案；不扣次</p></td></tr>
<tr><td><p>模型返回无法拆成 3 条</p></td><td><p>解析失败</p></td><td><p>把全文放进「合适」一条，稳妥/更进一步禁用；记错误日志</p></td></tr>
<tr><td><p>宿主不允许插入</p></td><td><p>insertText 无效</p></td><td><p>复制到剪贴板并提示「已复制，去对话框粘贴」</p></td></tr>
<tr><td><p>扩展内存压力</p></td><td><p>系统杀键盘</p></td><td><p>UI 不得加载大图/WebView；图片只在容器 App</p></td></tr>
</tbody>
</table>

# 6. 技术实现

P0 目标环境：iOS 16.0+，Swift 5.9，Xcode 当前稳定版。后端部署在国内区域（阿里云或华为云北京/上海），延迟和备案都按境内公众服务准备。模型不自训，只调已备案厂商 API。

## 6.1 架构取舍

备选 A：键盘扩展内直接调模型厂商。否决——密钥进扩展、额度无法服务端强制、换模型要发版。

备选 B：容器 App 做本地代理，扩展用 App Group 排队。否决——扩展不能依赖 App 在前台，用户切微信时 App 已在后台，生成会挂。

 **采用 C：扩展只做 UI 和 HTTPS，业务网关鉴权、计量、拼提示词、调模型、审核。** 扩展内存只保轻 UI；OCR、IAP、引导放容器 App。

不变量：密钥不出客户端；没点「生成」不上行聊天文本；键盘扩展不出现商店 UI；普通击键不上行。

## 6.2 工程拆分

<table header_row="1">
<colgroup>
<col width="161"/>
<col width="288"/>
<col width="289"/>
</colgroup>
<thead>
<tr><th><p>模块</p></th><th><p>职责</p></th><th><p>约束</p></th></tr>
</thead>
<tbody>
<tr><td><p>HuohouApp</p></td><td><p>引导、会员、设置、隐私协议、检测键盘状态</p></td><td><p>可较重；Vision OCR 只放这里</p></td></tr>
<tr><td><p>HuohouKeyboard</p></td><td><p>QWERTY、帮你回、上屏</p></td><td><p>无 WebView、无大图、无 StoreKit；峰值内存 &lt; 50 MB</p></td></tr>
<tr><td><p>HuohouShared</p></td><td><p>App Group 模型、URL Scheme、常量</p></td><td><p>两端共用，结构体要 Codable 且向后兼容</p></td></tr>
<tr><td><p>API</p></td><td><p>鉴权、额度、生成、收据</p></td><td><p>国内 HTTPS；TLS 1.2+</p></td></tr>
</tbody>
</table>

建议 Bundle：`app.huohou.ios`，扩展 `app.huohou.ios.keyboard`，App Group `group.app.huohou.ios`。键盘 `PrimaryLanguage` 用 `zh-Hans`，`RequestsOpenAccess` = YES。展示名「火候」。

## 6.3 一次生成的时序

额度采用预扣 + 失败回滚：5xx、客户端超时、安全拒绝回滚；成功拆出至少 1 条才确认扣除。防止「失败也扣次」和「重试刷免费」。

## 6.4 客户端状态

App Group 只存小 JSON，不存聊天全文（P1 会话记忆再单开文件，按联系人别名加密）。

```json
{
  "scene": "dating",
  "heat": 2,
  "persona_id": "warm",
  "entitlement": "free",
  "quota_left": 17,
  "keyboard_added": true,
  "full_access": true,
  "session": "eyJ..."
}
```

会话 token 放 Keychain，扩展与 App 同 Access Group。键盘被杀再开时先读本地 entitlement 做 UI，再后台拉 `/v1/me` 校准。校准失败不把已购用户打回免费，只在连续失败 &gt; 24 h 时提示「打开火候 App 刷新会员」。

## 6.5 接口（P0）

鉴权：`Authorization: Bearer <session>`。设备首次 `POST /v1/device/bootstrap` 用 Keychain UUID + App Attest（若耗时过长可 P0 先做 UUID，P1 补 Attest）。IAP 成功后 `POST /v1/iap/verify` 传 signed transaction，服务端走 App Store Server API。

<table header_row="1">
<colgroup>
<col width="100"/>
<col width="244"/>
<col width="171"/>
<col width="223"/>
</colgroup>
<thead>
<tr><th><p>方法</p></th><th><p>路径</p></th><th><p>作用</p></th><th><p>失败</p></th></tr>
</thead>
<tbody>
<tr><td><p>POST</p></td><td><p>/v1/device/bootstrap</p></td><td><p>创建设备会话</p></td><td><p>409 设备锁定</p></td></tr>
<tr><td><p>GET</p></td><td><p>/v1/me</p></td><td><p>额度与会员</p></td><td><p>401 刷新 bootstrap</p></td></tr>
<tr><td><p>POST</p></td><td><p>/v1/iap/verify</p></td><td><p>校验苹果交易</p></td><td><p>402 收据无效</p></td></tr>
<tr><td><p>POST</p></td><td><p>/v1/replies/stream</p></td><td><p>生成 3 条</p></td><td><p>见下</p></td></tr>
</tbody>
</table>

生成请求体：

```json
{
  "scene": "dating",
  "heat": 2,
  "persona_id": "warm",
  "incoming": "今天下班好早",
  "outgoing_prev": "",
  "locale": "zh-CN",
  "client_request_id": "uuid"
}
```

`incoming` 必填，1–500 字。`client_request_id` 幂等 10 分钟：同一 id 重试返回同一结果、不重复扣次。

SSE 事件：`delta`（拼 JSON）、`done`（完整 `replies`）、`error`。业务错误码：`quota_exceeded`、`filtered`、`invalid_input`、`upstream_timeout`、`unauthorized`。成功的 `done`：

```json
{
  "replies": [
    {"tag": "稳妥", "text": "下班早也挺好，回家歇会儿。"},
    {"tag": "合适", "text": "这么早？今天事情少？"},
    {"tag": "更进一步", "text": "那晚饭能从容点了。"}
  ],
  "quota_left": 16
}
```

限流：免费 10 次/分钟；会员 30 次/分钟。超限 429，`Retry-After` 秒。会员另有 200 次/自然日硬顶，防盗刷打爆模型账单。

## 6.6 提示词与模型

系统提示分层，顺序固定，后层不得覆盖安全层：

1. 安全：不写骚扰、威胁、诈骗、未成年相关；不教用户隐瞒已读或伪造身份。
2. 身份：你在给用户起草「他自己要发的下一句」，必须像日常微信，短句，少形容词，不用网红腔。
3. 场景 + 火候：用上表的档位说明「过火」边界。火候 4 仍禁止连珠炮和性暗示。
4. 人设：只调语气词和句长。
5. 输出：只输出 3 条 JSON，每条不超过 40 个汉字（职场可到 60）。不用 emoji，除非对方原句已用。

P0 选一家国内已备案、延迟低的文本模型（通义 / 豆包 / DeepSeek 按当时合同与备案名单定）。温度 0.7，max tokens 够 3 条短句即可。单次目标成本 &lt; 0.05 元。厂商密钥只在网关。

被否方案：端侧小模型——键盘 60–70 MB 内存放不下可用中文模型。被否方案：一次请求并发生成 3 遍——费用 ×3，P0 不划算。

## 6.7 IAP 与账号

P0 不强制登录。会员权益绑 Apple ID 收据。容器 App 用 StoreKit 2 监听 `Transaction.updates`，把 signed transaction 交给网关。网关校验后把 entitlement 写入设备会话。

商品 ID 建议：`huohou.plus.monthly`、`huohou.plus.yearly`、`huohou.lifetime`。年费为首推。家庭共享：P0 跟随 App Store 默认；若苹果把终身标成非消耗，需在审核备注里写清恢复购买路径。

安卓 P1 再用微信/支付宝，账号体系到 P1 才上「Apple 登录 / 手机号」做跨端恢复。P0 换机：打开 App → 恢复购买。

## 6.8 隐私、日志、合规

<table header_row="1">
<colgroup>
<col width="152"/>
<col width="531"/>
</colgroup>
<thead>
<tr><th><p>数据</p></th><th><p>处理</p></th></tr>
</thead>
<tbody>
<tr><td><p>击键</p></td><td><p>不上行、不落盘</p></td></tr>
<tr><td><p>生成请求文本</p></td><td><p>点生成才上传；服务端默认存 7 天用于安全抽检，到期删除或哈希；不做推荐训练对外卖</p></td></tr>
<tr><td><p>收据、设备号</p></td><td><p>用于会员；不与通讯录关联</p></td></tr>
<tr><td><p>崩溃日志</p></td><td><p>可用国内崩溃收集，关闭键盘扩展里的第三方广告 SDK</p></td></tr>
</tbody>
</table>

上架并行件（不在本文件展开流程，但是发布门）：软著、App 备案、生成合成类算法备案、隐私政策写明完全访问用途、生成内容按要求做 AI 标识。P0 可先上 App Store，安卓商店等备案截图。

## 6.9 可观测与测试

客户端埋点只记状态，不记原文：引导每步完成、完全访问开关、生成开始/成功/失败原因码、上屏、付费页曝光/成交。网关指标：QPS、TTFT、P95、扣次成功率、filtered 比例、上游错误、单次成本。

测试最低集：

- 扩展 UI 测试：无完全访问打字；有权限生成；额度 0。
- 真机：微信、备忘录、系统短信、Safari 输入框。
- 内存：反复切键盘 50 次，无泄漏到被杀。
- 安全回归：固定 30 条攻击句。
- IAP 沙盒：购买、恢复、过期（月费）。

# 7. 90 天怎么切开

<table header_row="1">
<colgroup>
<col width="113"/>
<col width="328"/>
<col width="162"/>
</colgroup>
<thead>
<tr><th><p>周</p></th><th><p>交付</p></th><th><p>人</p></th></tr>
</thead>
<tbody>
<tr><td><p>1–2</p></td><td><p>工程骨架：App + 扩展 + App Group + 假数据生成 3 条上屏；引导能跳系统设置</p></td><td><p>iOS</p></td></tr>
<tr><td><p>3–5</p></td><td><p>网关 bootstrap / me / stream；接一家模型；安全预检；额度预扣</p></td><td><p>后端 + iOS</p></td></tr>
<tr><td><p>6–7</p></td><td><p>火候/场景/人设调通；提示词抽检；超时与失败回滚</p></td><td><p>后端 + 产品</p></td></tr>
<tr><td><p>8–9</p></td><td><p>StoreKit 2、会员页、键盘锁态跳转、隐私文案</p></td><td><p>iOS</p></td></tr>
<tr><td><p>10–11</p></td><td><p>真机打磨、内存、审核材料、备案材料并行</p></td><td><p>全员</p></td></tr>
<tr><td><p>12</p></td><td><p>提审；修拒审；TestFlight 给 10 个目标用户走完第一次上屏</p></td><td><p>全员</p></td></tr>
</tbody>
</table>

P1 不穿插进这 90 天，除非 P0 提审被 4.4.1 卡住需要补「无网络也能用」的打字体验。

# 8. 开放问题

<table header_row="1">
<colgroup>
<col width="272"/>
<col width="273"/>
<col width="193"/>
</colgroup>
<thead>
<tr><th><p>问题</p></th><th><p>影响</p></th><th><p>谁在何时定</p></th></tr>
</thead>
<tbody>
<tr><td><p>最低系统用 iOS 16 还是 15</p></td><td><p>15 多覆盖、16 少坑（StoreKit 2 / 并发）</p></td><td><p>开工第 1 周，iOS 负责人</p></td></tr>
<tr><td><p>密码框检测 API 在目标系统是否够用</p></td><td><p>不够则只能按 keyboardType 启发式隐藏帮你回</p></td><td><p>第 2 周真机验证</p></td></tr>
<tr><td><p>模型厂商与备案号</p></td><td><p>决定网关合同和上架材料</p></td><td><p>第 3 周前</p></td></tr>
<tr><td><p>App Attest 是否进 P0</p></td><td><p>防刷额度；实现成本</p></td><td><p>第 3 周，可降级为设备 UUID</p></td></tr>
<tr><td><p>微信对自定义键盘 insertText 的机型差异</p></td><td><p>失败则走复制粘贴兜底，体验下降</p></td><td><p>第 6 周真机矩阵</p></td></tr>
<tr><td><p>算法备案周期</p></td><td><p>安卓商店；iOS 先上的合规口径</p></td><td><p>与开发并行，法务/运营</p></td></tr>
</tbody>
</table>

<div class="callout callout-bg-2 callout-border-2">
<div class='callout-emoji'>📎</div>
<p>本文件不批准预算，也不替代算法备案材料。P0 验收以「真机在微信里完成一次上屏 + 沙盒买通年费」为发布门。截图 OCR 和安卓在 P0 发布后再开子页。</p>
</div>

