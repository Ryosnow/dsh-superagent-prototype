# 008 手册

这本手册说明怎样扩展 dsh-superagent。它使用 [001 领域模型](001-domain_zh.md) 和 [002 授权](002-authorization_zh.md) 里的术语，以及 [007 架构](007-architecture_zh.md) 里的组件。

## 增加一个来源

这份做法实现一个叫做 `webhook` 的来源种类。它通过虚构出来的命令行 `hookq`，读取一条想象中的消息队列，并把消息转换成触发。其余接口都属于 dsh-superagent。

实现有四项职责：翻译事件，定义接收规则和授权规则，提供回复和上下文，以及维持一个轮询循环。下面的编号步骤按这个顺序把这些部分建起来。

这个例子假定命令行遵守下面的约定：

- `hookq pull --after <n> --json` 打印位置 `n` 之后的消息，最旧的在前：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。一条消息含有 `id`、`conversation`、`thread`（开启它所在线程的那条消息的标识；那条消息本身没有这个字段）、带有 `id` 和 `bot` 的 `sender`，以及 `mentions_me`、`text`、`at` 和 `url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在一条消息下面回复，每个键只回复一次。对话已经被删除时，它失败，并把 `{"error": {"code": 410, "message": "…"}}` 写到标准错误。
- `hookq thread --conversation <id> --root <message id> --json` 打印一页线程，最旧的在前；更后面的页加上 `--after <cursor>`。打印出来的是 `{"items": […], "next": "<cursor>"}`，最后一页没有 `next`。

把这个种类和它的插件入口放在 `src/sources/webhook.ts`，写法跟随 `src/sources/github.ts`。来源变大以后，把翻译、渠道和轮询拆成各自的模块，像飞书来源那样。

### 1. 把事件映射到领域上

先选定稳定的映射，再动手写代码。触发的标识、线程的标识和渠道地址都会持久保存，事后改动会让已经存在的记录对不上。

| 概念 | 对本例的队列 |
| --- | --- |
| 触发 | 一条消息 |
| 线程 | 同一场对话里的一条消息，加上对它的回复：`<conversation>:<id of the first message>` |
| 行动者 | 发送者在队列上的标识，写法与允许名单一致 |
| 渠道地址 | 回复所挂靠的那场对话，以及那条消息 |
| 接收范围 | 一场对话，在准入表里单独占一行 |

### 2. 配置：模式、字段、事实与健康

这个模块导入下面各步要用的东西：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是稳定的来源标识。它出现在触发的 `source`、线程键的前缀、`source:webhook` 设置块，以及来源策略里。
- `schema` 校验已经存下的配置，并给出默认值。操作者保存实例之前，注册表会用这些默认值建出一个已经启用的实例。默认值应当什么都不接收；这个例子默认没有任何对话。
- `fields` 生成来源设置的表单，不必另写面板代码。`rows` 字段按第一列当作键来编辑各条；`advanced` 把较少用到的设置藏起来。来源能把标识解析成显示名时，实现 `refreshLabels`。把这些名字存在 `KV.sources.labels(kind)` 下面。带有 `choose` 和 `refresh` 的列，随后就能把这些名字提供出来。
- 可选的 `facts(deployment)` 展示部署检查推导出来的值，例如一条命令行背后的账号。部署检查目前只核对 `SETUP_FIELDS` 里列出的 GitHub 字段和飞书字段，所以这个例子不提供事实。
- `health` 声明来源的健康项，一开始是 `starting`。注册时占住每一个名字，并拒绝与 `runtime`、`reply`、`worker` 或各种效果种类撞名。用种类自己的名字，并把 `sync:<id>` 留给策略同步。出现 `error` 时，状态脚本以退出码 2 结束，每小时的检查也会通知操作者，所以只在可能需要人来处理时才这样报告。

### 3. 把一条消息译成触发

先校验命令行的响应，再把它转换成触发：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定的标识。** 用 `key` 从队列标识得出触发标识。`Engine.receive` 对已经存下的标识返回 `duplicate`，重复的接单只建立一张工单。因此把消息再读一遍是安全的。要把编辑当成新事件，就在标识里放上编辑版本，像 GitHub 对评论使用 `updated_at` 那样。
- **线程。** 第一条线程是主线程，评估归它。多出来的线程只补充关联上的上下文。来源自己选定标识；完整的键是 `webhook:<id>`。
- **发送者。** 在 `actor` 里用来源自己的标识。机器人和持续集成要设 `human: false`。列在名单里的机器人只能成为推断的候选。行动者的标识为空时，像持续集成那样，走名单之外、又没有点名的发送者所对应的那一格准入。只有消息确实在对智能体说话时，才把 `addressed` 设为 `mention` 或 `direct`；这样，允许名单上的人发出的消息才具有明确强度。绝不要为智能体自己的消息产生触发。
- **回复去处和元数据。** `channel` 标明回复发往哪里；`url` 链到源消息。`channel.thread` 把消息分组，以便按顺序送达：同一张工单里较晚的消息要排在较早的消息后面，直到较早的那条已经失败三次。别的工单不用等。来源专有的字段放在 `meta` 里，例如正在接收的对话和线程的根。`meta` 和 `text` 都是不可信的数据，不能当作授权。

智能体可以通过 `orchestrator_read trigger/<id>` 和 `thread/<key>` 读到这个新种类。它们给出消息正文、去向，以及邻近的触发。新种类不会自动得到内置技能，也不会自动得到原始服务记录所用的资源，或拉取请求的绑定。

文本搜索不用额外代码就能工作。人们用来源专有的指称，例如消息标识、网址或 `#12` 时，再实现 `find(query)`。返回这些指称所对应的线程键和搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

查询里出现队列消息的标识，例如 `m42`，就会到触发里去搜索它。

### 4. 来源接收什么：范围

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

引擎经 `SourceRegistry` 对每一个到来的触发调用 `scope`。超出范围的消息连同原因一起记成已过滤，并且不建立线程。把消息交给 `receive`，不要在轮询循环里默默丢掉，这样面板才能解释为什么被过滤。

`observed: true` 表示来源在这个范围里接收没有点名的消息，从而准入表可以对它们做决定。`row` 标明对应的那一行准入。

重新校验时，用授权里存下的元数据调用 `scope`。让它只依赖 `meta` 和配置。恢复看的是 `listened`，不是 `observed`。没有 `scope` 时，一个种类会接收自己产生的每一条触发，但一条都不观察。

### 5. 回复：渠道

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **幂等发送。** 发件箱调用 `SourceRegistry.send`，再由它调用 `channel.send`。来源插件卸掉的时候，消息会等着。一次发送可能在远端服务已经收到之后才失败，所以重试不能把消息发出两份。用 `message.id` 作为幂等键：`hookq` 支持 `--key`；GitHub 存一个藏起来的评论标记；飞书接受一个幂等键。
- **永久失败和可以重试的失败。** 成功时返回 `sent`，只在送达永远不可能时才返回 `gone`，例如本例的 410。一条已经无法送达的消息保留为已放弃，并且再也不入队；它出现在 `GET sources` 和来源页上。其他失败要抛出，这样发件箱会在 5 秒后重试，间隔倍增，直到 300 秒。`CliError.code` 是命令行输出里报告的那个代码。
- **允许的目的地。** `allowed(address, config)` 只检查已经存下的地址和当前配置。把每项必需的事实放进地址里；例如，飞书记下这场聊天是不是私聊。不再监听的目的地会被放弃，而来源被停用时，消息继续等待。没有 `allowed` 时，来源处于启用期间，所有地址都放行。
- **链接。** `link(address, links)` 返回面板经 `SourceRegistry.link` 所显示的网址和标签。

### 6. 给研判用的线程上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动的研判需要一条触发周围的那场对话。`orchestrator_eval` 先读开初的一窗；`orchestrator_eval_read` 再展开后续的页、根消息或父消息。实现可选的 `threadContext` 接口，把这些读取提供出来：

- `read(commands, trigger, part, cursor, config)` 返回一个 `ContextPart`。它的 `messages` 使用 `ContextMessage`：`{id, from, at, text}`，再加上用得上的 `bot`、`deleted`、`reply_to`、`url`、`path` 和 `line` 字段。最后一页不带 `next`。`body` 是展示给正在研判的那场会话的内容，这里就是同样这些消息。可选的 `status` 可以描述特殊结果，例如飞书的 `chat_neighbours`。`part` 取 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 在这些消息里找出这条触发自己的那一条。
- `images(text)` 和 `image(…)` 用于消息里带有图片的种类，飞书就是这样；这个例子不实现它们。

没有 `threadContext` 的种类不给研判提供线程上下文；触发自己的正文仍然可以读。

消息回复更早的消息时，加上 `repliesTo(trigger)`。它返回对应的触发标识，这样跟进通知和执行器的输入就能在那条消息已经存下时把它显示出来。例如，若 `hookq` 提供 `reply_to`，而翻译把它存成 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁可以发起：来源在策略里的那一部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 校验来源策略，其中包括 `allowed` 里的发送者标识。标识的比较不区分大小写。准入表另行校验，并且在各种类之间共用。这部分策略存在 `policy.sources.webhook`，并计入策略版本。
- `fields` 生成 **评估 → 在各来源上** 下面的表单，开头是 `allowed`。`explicit` 说明怎样向智能体开口。可选的 `unnamed: {person, bot}` 给看不出是谁的标识提供显示用的标签，例如飞书的 `ou_…`；这个例子省去它，因为队列标识本身就是能读的名字。
- `rows(config)` 列出各条准入范围，以及它们可用的格子。每一行的标识必须和 `scope` 返回的 `row` 一致。未点名消息的格子，只在来源会观察这些消息的地方才可用。没有 `rows` 的种类用默认格子。
- 可选的 `bots` 列出可以成为推断候选的机器人，条目的形状和 `allowed` 一样。没有它，任何机器人都不准入。
- 可选的 `sync: {label, ready(part), run(host, part, signal)}` 在启动时，以及在按下 **立即同步** 时，刷新成员名单。飞书用的是群里的成员。失败记在 `sync:<id>` 下面，一分钟后重试，间隔倍增到一小时，直到成功。

### 8. 给执行器用的变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

执行设置在准备命令旁边展示已经声明的 `variables`，即使来源已经停用也显示。启用之后，`values` 用部署、配置，以及发起这张工单的那条触发，提供这张工单专有的值。

准备命令把它们读成环境变量；执行器在自己的指令里读到这些值。使用 `DSH_SUPERAGENT_<KIND>_…` 这个前缀。工单不是从这一个来源发起时，返回空值，并且绝不要放进密钥。

### 9. 插件入口和它的循环

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

插件注入 `orchestrator`。第一个效果注册这个种类，释放的时候把它注销。第二个效果带着一个 `SourceContext` 跑轮询循环。释放会中止循环并等它跑完，应用停止或重启时也是如此。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```

- **部署检查和开关。** 先等 `ctx.ready`，并且在 `ctx.configured()` 为假的时候什么都不读。每一轮都读 `ctx.config(kind)`：来源关闭时，这个值是未定义的。
- **游标。** 读取位置存在 `ctx.state` 里，用的是 `KV`（`src/kv.ts`）中带类型的条目，例如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键字面量只出现在那份注册表中。外部包可以用导出的 `record` 和 `family` 定义键。设成 `undefined` 会删掉一条记录；`ctx.atomic` 把多次写入合成一个事务。

  只有被这一段覆盖的每条触发都已经存下，才把游标往前移。一轮被打断时，游标保持不动；稳定的标识让重放是安全的。按时间戳记的游标，用 `src/sources/poll.ts` 里的 `cursorAt(started)`，它减去五秒，给迟到的远端写入留出余地。
- **线程元数据。** 接收一条触发之后，调用 `ctx.threads.describe(key, {label, url, facts})`，用来填上展示用的信息。`ctx.threads.patch(key, meta)` 转交给 `threads.patchMeta`，设置来源拥有的键；值为 `undefined` 时则把这些键删掉。两者都忽略不存在的线程，包括已经被过滤的触发所在的线程。
- **逐条错误。** 计下格式错误的条目，然后继续，不要让整批失败。`src/infra/commands.ts` 里的 `validItems(item, list, tally)` 支持这种做法。线程是各自轮询的时候，用 `src/sources/poll.ts` 里的 `pollThreads`；它把每次失败记成 `pollError`。它的 `polled` 谓词保留仍然有效的订阅，以及从最近一条触发和工单结束这两者里较晚的那个时刻算起的两周。
- **健康。** 只通过 `ctx.report` 写已经声明的条目。`health(state, code, values)` 提供固定文案，例如 `setupPending`、`sourceOff` 和 `pollFailures`。连续失败要先累计次数，再报告 `error`。
- **中止。** 把信号传给 `ctx.commands.withSignal(signal)`，也传给 `delay`，并在条目之间检查它，于是停止或重启会立刻结束这个循环。

这个例子不发送由来源发起的通知。需要这种通知的来源调用 `ctx.notify`；发件箱像送达其他回复一样送达它们。

它也不把线程绑到工单是否已经完成。只有来源上线程的生命周期决定工作是否已经做完时，才使用 `ctx.tickets.bindThread` 和 `threadClosed`，就像 GitHub 的拉取请求那样。工单何时结束，仍由引擎决定。

### 10. 注册插件

在 `tsdown.config.ts` 里把这个模块建成它自己的入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 里导出构建出来的文件：把 `"./webhook-watch": "./lib/webhook-watch.js"` 加进 `exports`，并把这个插件加进包自带的 `cordis.patch.yml` 的 `insert` 列表，设为停用，和飞书一样：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在一份配置档的 `cordis.patch.yml` 里启用这个插件，写上 `id: dsh-superagent-webhook` 和 `disabled: false`。构建并重启。新来源出现在来源设置里，也出现在 **评估 → 在各来源上** 下面。

把它的模块和规格加进 [007 架构](007-architecture_zh.md) 的模块表和测试表。`tests/docs.spec.ts` 检查两份索引都是完整的。

### 11. 测试

按 `tests/sources.spec.ts` 的写法建立 `tests/webhook.spec.ts`：临时目录里放一份真实的存储，一个带着 `HealthBoard` 的 `SourceRegistry`，一台端口都换成替身的引擎，以及由 `sourceContextOf` 得到的 `SourceContext`。

通过监视 `Commands.run` 来假装命令行调用。也要监视 `withSignal`，因为它通常会返回一个新的 `Commands` 实例。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 用来构造一条队列里的消息。至少要覆盖：

- 翻译：稳定的标识、线程、行动者、`human` 和 `addressed`，以及不为智能体自己的消息产生触发；
- 一轮读取：触发只存储一次，范围之外的一条是 `filtered`，游标移到它们之后，允许名单里的发送者做出的提及被评成 `explicit`；
- 中途截断的一轮，游标保持原位；
- `send`：成功是 `sent`，只有 410 才是 `gone`，其他任何失败都抛出，命令行的错误由 `commandError` 构造；
- 线程上下文：一页上的消息，以及下一页的游标，做法和 `tests/registry.spec.ts` 经 `SourceRegistry.threadContext` 读取一个种类的线程一样；
- 插件：`apply` 注册这个种类，并在部署检查还在等待时报告 `setupPending`；它的释放函数把这个种类注销；逃出循环的失败变成来源的健康状态，而不是未处理的拒绝；
- 来源在策略中的那一部分，以及它的准入行，做法和 `tests/registry.spec.ts`、`tests/policy.spec.ts` 检查飞书来源一样。

### 清单

1. 映射触发、线程、行动者、渠道地址和接收范围。
2. 写这个种类：`schema`、`fields`、`health`，部署检查会推导出值时再加上 `facts`。
3. 把事件译成带有稳定标识的触发，并丢掉智能体自己的消息。
4. 在 `scope` 里只根据 `meta` 决定来源接收什么。
5. 给它一条渠道，其 `send` 把每条消息只发一次，并且只在这场对话永远无法接收时才回答 `gone`。
6. 它能读到线程时，把线程上下文交给研判。
7. 给它策略里的一部分，带上 `allowed` 和准入行。
8. 登记它在执行环境里的变量。
9. 写插件入口，并写一个循环：等待部署检查、跟随开关、在 `receive` 之后移动游标、按条目保住错误、报告健康，并在中止时停下；在 `KV` 里给游标命名。
10. 加上构建入口、导出项，以及 `cordis.patch.yml` 里的插入项，并在 007 的表里列出这个模块和它的规格。
11. 像 `tests/sources.spec.ts` 测试其他来源那样测试它。
