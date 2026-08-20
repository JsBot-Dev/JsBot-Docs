# 07 · 类型与守卫

JsBot 全链路有类型推导,但事件对象是 OneBot 协议包,不同场景字段不同,需要用**类型守卫**收窄后再访问特有字段。

## 事件类型层级

```text
SnowLumaEvent
├── OneBotMessageEvent
│   ├── OneBotGroupMessageEvent   message_type: 'group'
│   └── OneBotPrivateMessageEvent message_type: 'private'
├── OneBotNoticeEvent
├── OneBotRequestEvent
└── OneBotMetaEvent
```

所有事件基类 `OneBotBaseEvent` 都含:

| 字段      | 类型     | 说明          |
| --------- | -------- | ------------- |
| `time`    | `number` | 事件时间戳    |
| `self_id` | `number` | 机器人自身 QQ |
| `post_type` | `string` | 事件大类     |

## 类型守卫

`@snowluma/sdk` 提供一组类型守卫,用于把 `SnowLumaEvent` 收窄为具体类型:

```ts
import {
    isMessageEvent,
    isGroupMessageEvent,
    isPrivateMessageEvent,
    isNoticeEvent,
    isRequestEvent,
    isMetaEvent,
} from '@snowluma/sdk';
```

用法:

```ts
import { Plugin, OnMessage } from '../core';
import { isGroupMessageEvent } from '@snowluma/sdk';

@OnMessage()
onMessage(event, ctx) {
    if (isGroupMessageEvent(event)) {
        // 收窄为 OneBotGroupMessageEvent,可安全访问 group_id
        const { group_id, sender } = event;
        ctx.reply(`来自群 ${group_id},昵称 ${sender.nickname}`);
    } else if (isPrivateMessageEvent(event)) {
        ctx.reply('来自私聊');
    }
}
```

### 自定义守卫

可用 `when` 式谓词或自写 `type is` 函数:

```ts
function isPing(event: { raw_message?: string }): boolean {
    return event.raw_message === 'ping';
}
```

## 常用类型导入

```ts
import type {
    SnowLumaEventContext,      // ctx
    CommandMatch,              // @Command 的 match
    OneBotMessageEvent,
    OneBotGroupMessageEvent,
    OneBotPrivateMessageEvent,
    OneBotNoticeEvent,
    OneBotRequestEvent,
    OneBotSender,
    OutgoingMessage,           // reply 参数
    SnowLumaAction,            // 动作名联合类型
    ActionParams,
    ActionResult,
} from '@snowluma/sdk';
```

## 处理器参数类型写法

### 事件处理器

```ts
import type { SnowLumaEventContext, OneBotGroupMessageEvent } from '@snowluma/sdk';

@OnGroupMessage()
onGroup(event: OneBotGroupMessageEvent, ctx: SnowLumaEventContext) {
    ctx.reply(`群 ${event.group_id}`);
}
```

### 命令处理器

`ctx` 附带 `command` 字段,`match` 是第三个参数:

```ts
import type { CommandContext, CommandMatch, OneBotMessageEvent } from '../core';

@Command('ping')
ping(event: OneBotMessageEvent, ctx: CommandContext, match: CommandMatch) {
    ctx.reply(`命中命令: ${match.command}`);
}
```

> `CommandContext = SnowLumaEventContext & { command: CommandMatch }`,由 `../core` 导出,无需手动拼。

### 中间件

```ts
import type { EventNext, SnowLumaEventContext } from '../core';

@OnMiddleware()
async mw(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
    await next();
}
```

## 事件字段速查

### 群消息特有

```ts
event.group_id;      // 群号
event.sender;        // { user_id, nickname, card, role }
event.anonymous;     // 匿名信息(可能为 undefined)
event.message;       // 结构化消息段数组
event.raw_message;   // 纯文本
event.message_id;    // 消息 ID
```

### 私聊消息特有

```ts
event.user_id;
event.sub_type;      // friend / group / other
```

### 通知事件

```ts
event.notice_type;   // 如 'group_recall'
```

### 请求事件

```ts
event.request_type;  // 'friend' | 'group'
event.flag;          // 处理请求需要的标记
event.comment;       // 申请备注
```

## 事件对象与消息段

`event.message` 是结构化消息段数组(`AnyMessageSegment[]`),每段形如:

```ts
{ type: 'text', data: { text: 'hello' } }
{ type: 'at', data: { qq: '123456' } }
{ type: 'image', data: { file: 'https://...' } }
```

可用 SDK 的 `toCQString` / `fromCQString` 转换格式:

```ts
import { toCQString, fromCQString } from '@snowluma/sdk';

const cq = toCQString(event.message); // 转 CQ 码字符串
const segs = fromCQString(cq);        // CQ 码 → 消息段
```

## 收窄惯用法

在 `@OnMessage` 里想同时拿到群号和文本,又不愿写死类型:

```ts
@OnMessage()
onMessage(event, ctx) {
    if (!isMessageEvent(event)) return;
    const text = event.raw_message;
    const scene = isGroupMessageEvent(event)
        ? `群 ${event.group_id}`
        : `与 ${event.user_id} 的私聊`;
    ctx.reply(`${scene}: ${text}`);
}
```

## 下一步

- 完整动作返回类型 → [05-actions](05-actions.md)
- 类型安全的最佳实践 → [08-best-practices](08-best-practices.md)