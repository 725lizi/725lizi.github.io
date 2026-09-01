---
layout: post
title: "FocusTimer3 接大模型实战（三）：让 AI 周报在断网、错 Key、超时下都不崩——流式输出与优雅降级的工程实践"
date: 2026-09-01
description: "HarmonyOS NEXT / ArkTS 接入 DeepSeek 的完整工程实践：SSE 流式输出与半包容错、先备兜底再请求的弱网降级、HTTP 错误分类、API Key 沙箱存储、Markdown 轻量渲染，以及纯逻辑/设备层的可测性取舍（108 个单测）"
categories: [HarmonyOS, 移动开发]
tags: [鸿蒙NEXT, ArkTS, 大模型接入, DeepSeek, SSE流式, 优雅降级, 单元测试, 混合架构]
author: 725lizi
lang: zh
---

# FocusTimer3 接大模型实战（三）：让 AI 周报在断网、错 Key、超时下都不崩

> 这是 FocusTimer3 系列的第三篇。第一篇《[项目复盘](./2026‑08‑23‑focustimer‑review.html)》讲怎么从零做出一个**能跑**的 App；第二篇《[重构实战（二）](./2026-08-30-focustimer-refactor-engineering.html)》讲怎么把它重构成**结构清晰、测得全**的版本，并特意留了一个「可替换的接口」。这一篇就来兑现那个伏笔——**把真正的大模型接进来**，而且重点不是「能调通」，而是「在任何糟糕情况下都不给用户白屏」。
>
> 项目源码：[https://github.com/725lizi/FocusTimer](https://github.com/725lizi/FocusTimer)
> English Version: [Integrating an LLM into a HarmonyOS App (III)](./2026-09-01-focustimer-llm-streaming-fallback-en.html)

## 〇、先说结论：这一阶段做了什么

我给一个本地的专注计时 App 接上了 DeepSeek 大模型，用来生成「本周学习周报」，并刻意按**生产可用**而不是「教程 demo」的标准来做：

| 能力 | 具体做法 | 靠什么保证不出错 |
|---|---|---|
| 流式逐字输出 | 用鸿蒙 `requestInStream` 逐段接收 SSE，界面像打字一样蹦字 | 纯逻辑 `SseChunkParser` 处理「半包/坏 JSON」，单测锁死 |
| 无网也有周报 | 三条件决策（开关+Key+网络），不满足就走本地规则 | 纯逻辑 `ReportStrategy`，枚举所有组合离线测 |
| 出错自动降级 | 断网/超时/错 Key/限流/服务端崩，统一分类后回退规则版 | `HttpErrorMapper` 状态码映射，单测锁死 |
| Key 安全 | 只存应用沙箱 `preferences`、内存缓存、绝不写死或上传 | 集中配置 `AiConfig`，代码里零硬编码密钥 |
| 文本整洁显示 | 自写 40 行轻量 Markdown 渲染，去掉大模型返回的 `**` 星号 | 纯函数 + 8 个单测 |
| 整体质量 | 全项目 **108 个单元测试**，小步提交、CI 保持绿色 | 纯逻辑全部可离线测，设备层走真机 |

下面把「为什么这么设计、坑在哪、怎么填」讲清楚。

## 一、先想清楚：接大模型最难的不是「发请求」

很多教程的「AI 接入」是这样的：点按钮 → 发一个 HTTP 请求 → 等几秒 → 把返回的字符串塞进文本框。**在 demo 里这就够了，但在一个我敢写进简历、敢被面试官深挖的项目里远远不够**，因为真实世界有这些破事：

- **网络是不可靠的**：地铁里断网、弱网丢包、服务器超时、运营商劫持；
- **大模型会失败**：Key 填错、欠费、触发限流（429）、服务端 5xx、返回空内容；
- **流式有陷阱**：响应不是「一整句一次给你」，而是一个字节块一个字节块地到，块的边界和句子、甚至和一个汉字的边界都不对齐；
- **密钥会泄露**：图省事把 Key 写死在代码里，push 到公开仓库等于把钱包贴门上；
- **大模型返回的是 Markdown 原文**：它用 `**加粗**` 做排版，而原生文本组件不认 Markdown，会把星号原样显示。

所以我给自己定的目标是：**无论上面哪一种情况发生，用户点下去都一定能拿到一份周报**——能用上大模型时更智能，用不上时本地规则版无缝顶上，而且界面要诚实标注「这份到底是谁写的」。这就是第二篇埋的伏笔：**规则引擎 + 大模型的混合架构，靠一个稳定的接口对上层屏蔽差异**。

## 二、总体设计：把一次 AI 周报拆成七个各司其职的零件

我没有写一个 `AiManager` 把所有事揉在一起（那只会造出第二个「巨石文件」），而是延续第二篇的分层，按「**能不能离线测试**」这条线切开：

```
用户点「AI 生成」
        │
        ▼
┌─────────────────────────────────────────────┐
│ AiReportOrchestrator（编排器，碰设备）        │
│  1. 先用规则引擎算好「兜底周报」攥在手里       │
│  2. ReportStrategy.decideSource 三条件决策    │
│     ├─ 不满足 ───────────────► 直接给规则版   │
│     └─ 满足 ─► DeepSeekClient 发起流式请求    │
│                    │                          │
│        成功逐字回调│        任何错误           │
│                    ▼              ▼           │
│              累积 AI 文本     shouldFallback? │
│                    │          是►交出兜底周报 │
│                    ▼                          │
│                 回传界面（带来源标签）          │
└─────────────────────────────────────────────┘

纯逻辑层（不碰网络/系统，可离线单测）
  ReportPromptBuilder  组装提示词与请求体、聚合本周数据
  SseChunkParser       切 SSE 分片、处理半包/坏 JSON
  ReportStrategy       决策走 AI 还是规则、出错是否降级
  HttpErrorMapper      状态码/错误码 → 统一错误类别
设备层（依赖真实网络与系统，走真机验证）
  DeepSeekClient       requestInStream 收发流、解码
  ApiKeyStore          沙箱读写 Key
  NetState             查询当前是否在线
```

这张图里藏着一个我最在意的设计原则：**把所有「给定输入就能算出确定输出」的判断，全部从设备代码里抽成纯函数**。原因很实在——纯函数不用联网、不用真机、毫秒级跑完，能轻松写出几十上百个边界用例；而真正碰网络的类，薄到几乎只剩「搬运」，想写错都难。

## 三、流式输出：逐字蹦出来的效果，以及「半包」这个大坑

### 3.1 为什么要流式

如果用普通请求，用户点下去要对着转圈干等 5~10 秒才一次性看到全部文字，体验很僵。流式（streaming）是大模型一边想一边吐字，服务端用 **SSE（Server-Sent Events，服务器持续往连接里推事件文本）** 协议，每吐一点就发一个 `data: {...}` 的文本块，客户端收到就显示，于是有了「打字机」效果。

DeepSeek 提供 **OpenAI 兼容**接口（请求/响应格式和 OpenAI 一样，换个端点和 Key 就能用），请求体里把 `stream` 设为 `true`：

```typescript
static buildStreamRequest(summary: WeeklyDataSummary, model: string): ChatCompletionRequest {
  const request: ChatCompletionRequest = {
    model: model,              // deepseek-chat
    messages: ReportPromptBuilder.buildMessages(summary),
    stream: true,              // 关键：开启流式
    temperature: 0.7,
    max_tokens: 600
  };
  return request;
}
```

鸿蒙网络栈 `@kit.NetworkKit` 里，普通请求用 `request()`（一次性给完整响应），**流式必须用 `requestInStream()`**，正文通过 `on('dataReceive')` 一段段回调。这里有个官方文档很容易看漏的点：`requestInStream` 的结束回调第二个参数**只是 HTTP 状态码（number），不是响应体**，正文全在 `dataReceive` 里——我第一次就差点按普通请求的习惯去结束回调里取正文。

### 3.2 第一个坑：一个汉字可能被网络切成两半

`dataReceive` 给的是 `ArrayBuffer`（原始字节），要先解码成字符串。中文 UTF-8 一个汉字占 3 个字节，网络分包完全可能把这 3 个字节切到两个包里。如果每包各自 `TextDecoder.decode`，接缝处就会出现乱码字符 `�`。

解法是用**有状态的流式解码器** `decodeWithStream`，它内部会记住「这次收到半个字符」，等下一包到了拼起来再解：

```typescript
private decoder: util.TextDecoder = util.TextDecoder.create('utf-8');

httpRequest.on('dataReceive', (data: ArrayBuffer) => {
  // decodeWithStream 会跨包保留「半个汉字」的残缺字节，避免接缝乱码
  const chunkText: string = this.decoder.decodeWithStream(new Uint8Array(data));
  const deltas: string[] = parser.feed(chunkText);
  // ...逐段追加显示
});
```

### 3.3 第二个坑：一行 JSON 也可能被切开（半包）

解码出文本后，问题没完。SSE 靠换行分隔事件，理想情况下一次回调正好是若干整行：

```
data: {"choices":[{"delta":{"content":"总体"}}]}

data: {"choices":[{"delta":{"content":"情况"}}]}

```

但网络不保证「一次回调 = 整数行」。某一次 `dataReceive` 拿到的可能是 `data: {"choices":[{"delta":{"content":"半`——**半行、半段 JSON**。如果这时直接 `JSON.parse`，必然抛异常。

我的做法是写一个纯逻辑的 `SseChunkParser`，内部维护一个 `buffer`：每次喂入新文本先和上次的残留拼起来，按换行切；**最后一段如果不以换行结尾，就说明它是「半包」，存进 buffer 等下一次**，其余整行才解析：

```typescript
feed(rawChunk: string): string[] {
  const deltas: string[] = [];
  this.buffer += rawChunk;                       // 和上次残留拼接
  const lines = this.buffer.split('\n');
  if (this.buffer.endsWith('\n')) {
    this.buffer = '';                            // 都完整，清空残留
  } else {
    this.buffer = (lines.pop() as string);       // 最后一段是半包，留下次
  }
  for (let line of lines) {
    line = line.trim();
    if (!line.startsWith('data:')) continue;     // 忽略空行/event 行
    const payload = line.substring('data:'.length).trim();
    if (payload === '[DONE]') { this.doneFlag = true; continue; } // 结束标记
    const text = SseChunkParser.extractDelta(payload);            // 安全取字
    if (text.length > 0) deltas.push(text);
  }
  return deltas;
}
```

取字那一步再套一层「绝不崩」防护：哪怕真的拿到一段残缺/非 JSON 文本，`try/catch` 后返回空串跳过，而不是让整个流式挂掉：

```typescript
private static extractDelta(jsonLine: string): string {
  try {
    const parsed = JSON.parse(jsonLine) as StreamChunk;
    return parsed.choices?.[0]?.delta?.content ?? '';
  } catch (err) {
    return '';   // 半段 JSON：这次先跳过，等后续包拼全
  }
}
```

> 工程原则：**网络边界上的一切输入都是不可信的**。解码要防「半个汉字」，解析要防「半行 / 坏 JSON」。因为这些拼接逻辑和网络无关，我把它整个抽成纯类，用单测喂各种「故意切碎」的输入来证明它正确——这比在真机上靠运气复现靠谱得多。

## 四、核心：一套「先备兜底、再请求」的弱网降级设计

流式解决「好看」，降级解决「不崩」，后者才是这个项目区别于 demo 的关键。

### 4.1 决策：三个条件同时满足才请求大模型

并不是每次都该去请求 AI。用户可能关了 AI 开关、可能还没填 Key、可能当前没网。这个判断抽成纯函数，三个条件**与**起来，缺一就走本地规则：

```typescript
static decideSource(input: ReportStrategyInput): ReportSource {
  if (input.llmEnabled && input.online && input.hasApiKey) {
    return 'llm';
  }
  return 'rule';
}
```

而且走规则版时，要给用户一句**看得懂、且诚实**的原因（`ruleReason`）：没开开关就说「AI 周报未开启」，没 Key 就说「未配置 API Key」，没网就说「当前无网络，已自动使用本地规则生成」。界面上永远标注来源，绝不把规则版冒充成 AI 写的——这和第二篇「诚实命名」是同一个价值观。

### 4.2 把五花八门的错误收敛成 8 类

真实失败的来源非常杂：HTTP 状态码（401/403/429/500……）、鸿蒙网络栈自己的数字错误码（如 `2300089` 表示请求被取消、`2300004/2300047` 表示超时）。如果上层直接面对这一堆原始码，根本没法写判断。于是用一个纯逻辑 `HttpErrorMapper` 把它们**归一化**成 8 种语义类别：

| 类别 `LlmErrorKind` | 含义 | HTTP 状态码 | 框架错误码 |
|---|---|---|---|
| `network` | 断网/连不上 | 其他 | 未识别码保守归此 |
| `timeout` | 超时 | 408 | 2300004 / 2300047 / 2300999 |
| `unauthorized` | Key 错/欠费/无权限 | 401 / 403 | —— |
| `rate_limited` | 触发限流 | 429 | —— |
| `bad_request` | 请求体有误 | 400 / 404 / 422 | —— |
| `server` | 服务端故障 | ≥500 | —— |
| `empty` | 返回为空 | 应用层判断 | —— |
| `cancelled` | 用户主动取消 | —— | 2300089 |

归一化之后，既能给每类配一句人话提示（`userMessage`，比如 401 →「API Key 无效或已欠费，请检查设置」），也让「要不要降级」的判断变得极其简单。

### 4.3 只有一种情况不兜底：用户自己取消

```typescript
// 除「用户主动取消」外，任何错误都回退规则版，保证一定有产出
static shouldFallback(kind: LlmErrorKind): boolean {
  return kind !== 'cancelled';
}
```

为什么唯独取消不兜底？因为取消代表「用户不要了」（比如等不及直接返回了），这时再默默生成一份反而浪费。**除此之外的一切异常——断网、超时、错 Key、限流、服务端崩、返回空——都兜底**，因为用户的核心诉求是「看到周报」，至于它是 AI 写的还是规则算的，是第二位。

### 4.4 编排器：为什么要「先把规则版算好攥在手里」

这是整个降级设计里我最满意的一个细节。编排器 `AiReportOrchestrator` 的顺序是**刻意**的：

1. **一进来，先不等网络，立刻用本地规则引擎把周报算好**，作为「备胎」；
2. 然后才做三条件决策，决定要不要请求 AI；
3. AI 流式成功，就用 AI 文本；过程中任何一步失败，因为备胎早就算好了，**瞬间**就能交出去，不用临时再算、更不会让用户对着错误弹窗发呆。

对比另一种写法——「先请求 AI，失败了再临时去算规则版」——我的做法把失败路径的耗时和不确定性降到了零。这就是「**Graceful Degradation（优雅降级）**」：系统在部分能力缺失时，不是整体崩溃，而是平滑退到一个稍弱但完整可用的形态。

### 4.5 收尾必须幂等，请求必须销毁

流式有多个结束路径（收到 `[DONE]`、`dataEnd`、出错、用户取消），稍不注意就会重复回调、界面状态错乱。我用一个 `settled` 布尔标志保证 `onComplete/onError` **只可能触发一次**；结束/取消时统一 `off()` 掉监听并 `destroy()` 掉请求对象，避免连接在后台空转——这正是第二篇修「定时器泄漏」时养成的纪律：**谁申请的资源，谁负责在离开时释放**。

## 五、API Key 的安全：绝不写死，只住沙箱

新手最容易犯、也最致命的错，是把 API Key 当字符串写在代码里。只要仓库是公开的，推上去的那一刻密钥就泄露了，别人能用你的额度刷到欠费。我的约束是：

- **代码里零硬编码 Key**，端点、模型名、超时、存储键名这些非密信息放 `AiConfig`，唯独 Key 不碰；
- Key 由用户在应用内设置弹层填写，通过 `@kit.ArkData` 的 `preferences` 存在**应用沙箱私有目录**（别的应用读不到），启动时读一次进内存缓存；
- 输入框用密文类型（显示圆点），防止旁边人偷看；
- 请求时才拼成 `Authorization: Bearer <key>` 头发出去，全程不上报到任何第三方。

```typescript
// AiConfig 里只有非密配置；Key 绝不在这里
static readonly BASE_URL = 'https://api.deepseek.com';
static readonly DEFAULT_MODEL = 'deepseek-chat';
static readonly PREFS_NAME = 'ai_settings';   // Key 存在这个私有 preferences
```

## 六、一个 UI 层的小坑：大模型返回的 Markdown 星号

真机第一次跑通时，周报正文出现了 `**总体情况**` 这种星号——大模型按提示词分四段，习惯用 Markdown 的 `**文字**` 表示加粗，而 ArkUI 的 `Text` 组件不认 Markdown，把星号原样画了出来。

我没有简单地「把星号删掉」（那样小标题就不突出了），也没有为这点需求引入整个 Markdown 解析库。按 **YAGNI（You Aren't Gonna Need It，用不到就先别做）**原则，周报只需要「加粗」一种语法，于是写了约 40 行的 `MarkdownLite`：一个纯函数按 `**` 把文本切成带 `bold` 标记的片段，界面用 `Text` 套多个 `Span`，加粗片段用粗体深色、普通片段用常规灰，**星号消失、层次保留**：

```typescript
Text() {
  ForEach(MarkdownLite.parseBoldSpans(this.weeklyReport), (span: TextSpan) => {
    Span(span.text)
      .fontWeight(span.bold ? FontWeight.Bold : FontWeight.Normal)
      .fontColor(span.bold ? '#2D3436' : Constants.COLOR_TEXT_SUB)
  })
}
```

它还顺手处理了「只写了半个 `**` 没闭合」的异常输入。更妙的是，因为**规则版和 AI 版共用同一个显示组件**，这一处修复同时治好了两个来源；连「分享出去的纯文本」也在出口统一 `stripBoldMarkers` 去掉星号（纯文本没法加粗，就干净地去掉）。这个纯函数同样配了 8 个单测。

## 七、可测性取舍：108 个单测不是平均用力

到这一阶段，全项目一共 108 个单元测试，但我**不是**给每个类都硬写测试，而是做了明确取舍：

| 模块 | 依赖网络/系统？ | 怎么验证 |
|---|---|---|
| `ReportPromptBuilder` | 否，纯逻辑 | 单测：本周/上周切分、环比、空数据、请求体 |
| `SseChunkParser` | 否，纯逻辑 | 单测：整包、半包跨包、多行、`[DONE]`、坏 JSON |
| `ReportStrategy` | 否，纯逻辑 | 单测：三条件 8 种组合、各类错误是否降级 |
| `HttpErrorMapper` | 否，纯逻辑 | 单测：每个状态码/框架码的归类 |
| `MarkdownLite` | 否，纯逻辑 | 单测：成对/行内/跨行/未闭合/纯去标记 |
| `DeepSeekClient` | 是，真实网络 | 不硬造假对象，靠真机走查（逻辑都已下沉到上面的纯类） |
| `AiReportOrchestrator` | 是，调度系统能力 | 真机走查；它的每个判断分支都复用已测纯函数 |

这背后是一个可以拿去和面试官聊的观点：**测试要投在「纯逻辑、易出错、可离线复现」的地方，回报率最高**；对于只做「调用系统 API + 转发已测结果」的薄设备层，硬用 mock（造假对象）去测，只会让测试又脆又没信息量。这不是「漏测」，是有意识的测试策略——而且我能清楚说出每一层为什么这么选。

另外还踩过一个和测试本身有关的坑：有个用例失败，期望值是我**照着当时的实现抄的**，结果实现里「一周起始时间」算错了，测试反而一起错。这件事的教训是**测试的期望值必须来自需求/独立计算，绝不能照着被测实现抄**，否则测试永远绿、却什么都没守住。

## 八、面试视角：这部分可能被怎么追问

**Q1：端上接大模型，弱网/无网/接口挂了你怎么保证体验？**
A：「先备兜底再请求」——进流程先算好本地规则版，再用「开关+Key+在线」三条件决定是否请求；流式中任何错误经 `HttpErrorMapper` 归一成 8 类，除用户主动取消外一律回退到已备好的规则版，并诚实标注来源与原因，任何路径用户都拿得到周报。

**Q2：流式输出怎么处理半包和乱码？**
A：两层。字节层用有状态的 `TextDecoder.decodeWithStream` 跨包拼回「半个汉字」防乱码；文本层用 `SseChunkParser` 维护 buffer，非整行留到下次，`data:` 取载荷、`[DONE]` 收尾、`JSON.parse` 套保护不让坏包崩掉流程。这些都是纯逻辑、有单测。

**Q3：为什么错误要先「分类」而不是直接 catch？**
A：原始错误来自 HTTP 状态码和系统数字错误码两套体系，直接面对它们没法写策略；归一化成有限的 8 类语义后，「是否降级、给用户什么提示」才能统一、可测、可维护。

**Q4：API Key 怎么保证不泄露？**
A：零硬编码，用户在应用内填写、存应用沙箱私有 preferences、内存缓存、密文输入框、仅请求时放 Authorization 头、不向第三方上报。

**Q5：为什么不直接用现成 Markdown 库？**
A：按 YAGNI，周报只需加粗一种语法，自写 40 行零依赖、可单测可控；引入整库会增加包体积和维护面，等真有标题/列表需求再渐进扩展。

**Q6：为什么有的类你写很多单测，网络类却不写？**
A：按可测性分层，把判断全部下沉为纯逻辑并高覆盖；设备薄层依赖真实网络、无法离线复现，硬 mock 价值低，靠真机走查 + 复用已测纯函数保证。这是测试投入的取舍，不是漏测。

## 九、下一步

阶段 2（AI MVP）到此，「规则 + 大模型」的混合智能已经在端上闭环：有网时 AI 流式逐字写周报，任何异常都平滑退回规则版。下一阶段（阶段 3）我会在 RAG 个人学习知识库和学习教练 Agent 之间选定一条做深，让 AI 不只「会说话」，而是**基于我自己的专注数据和笔记**给出有依据的建议——那才是真正区别于教程 demo 的壁垒。系列第四篇见。
