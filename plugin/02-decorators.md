# 02 · 装饰器

装饰器是插件处理器的「声明式注册」。所有装饰器从 `src/core`(即 `../core`)导出。

## 总览

| 装饰器              | 触发条件           | 处理器签名              | 底层注册                 |
| ------------------- | ------------------ | ----------------------- | ------------------------ |
| `@Command(...)`     | 匹配命令文本       | `(event, ctx, match)`   | `client.command`         |
| `@AdminCommand()`   | 标记管理员指令     | `(event, ctx, match)`   | 与 `@Command` 共用,内核自动附带 `isAdmin` |
| `@OnMessage()`      | 任意消息           | `(event, ctx)`          | `client.onMessage`       |
| `@OnGroupMessage()` | 群聊消息           | `(event, ctx)`          | `client.onGroupMessage`  |
| `@OnPrivateMessage()`| 私聊消息          | `(event, ctx)`          | `client.onPrivateMessage`|
| `@OnNotice(type?)`  | 通知事件           | `(event, ctx)`          | `client.onNotice`        |
| `@OnRequest(type?)` | 请求事件           | `(event, ctx)`          | `client.onRequest`       |
| `@OnMiddleware()`   | 全部事件(拦截层)   | `(event, ctx, next)`    | `client.use`             |

一个方法可以同时挂多个不同种类的装饰器(会分别注册),但通常一个方法只挂一种,保持语义清晰。

## `@Command(command, options?)`

注册命令处理器。底层走 SDK 的 `client.command()`,匹配流程:

1. 取 `event.raw_message`,默认先去首尾空白(`trim`)
2. 依次尝试每个前缀(`prefixes`):命中则剥离前缀,再与 `command` 比较
3. 匹配成功 → 处理器收到第三个参数 `match`,参数从 `match` 里取

> **注意:默认前缀是 `/`。** 即 `@Command('ping')` 实际匹配消息 `/ping`。
> 想匹配裸命令 `ping`,需传 `{ prefixes: [''] }`(空字符串前缀)。

### 字符串命令(带参指令的首选)

字符串命令**天然支持参数**:消息按空白切分,第一个词是命令,其余作为参数。

```ts
import type { CommandContext, OneBotMessageEvent } from '../core';

// 裸命令:发送 ping 即命中
@Command('ping', { prefixes: [''] })
ping(event: OneBotMessageEvent, ctx: CommandContext) {
    ctx.reply('pong');
}
```

带参示例:发送 `/set mode json` 时

```ts
import type { CommandContext, CommandMatch, OneBotMessageEvent } from '../core';

@Command('set', { prefixes: ['/'] })
set(event: OneBotMessageEvent, ctx: CommandContext, match: CommandMatch) {
    const key = match.args[0];   // 'mode'
    const rest = match.rest;     // 'mode json'
    ctx.reply(`key=${key}, value=${rest}`);
}
```

| 字段            | 值                    |
| --------------- | --------------------- |
| `match.command` | `'set'`               |
| `match.args`    | `['mode', 'json']`    |
| `match.rest`    | `'mode json'`         |

- `args`:按空白切分,适合**定长参数**
- `rest`:保留原文剩余部分,适合把整段文本作为参数(复读、搜索、AI 提问等)

### 正则命令

正则用于需要**捕获组**或复杂结构的匹配。注意:**正则同样先剥离前缀再匹配**,正则里不要再写前缀斜杠。

```ts
import type { CommandContext, CommandMatch, OneBotMessageEvent } from '../core';

// 发送 /roll 100:剥离前缀后文本为 "roll 100"
@Command(/^roll (\d+)$/)
roll(event: OneBotMessageEvent, ctx: CommandContext, match: CommandMatch) {
    const sides = Number(match.match![1]); // 捕获组取数字 100
    ctx.reply(`${Math.floor(Math.random() * sides) + 1}`);
}
```

正则模式下:

- 捕获组在 `match.match`(`RegExpMatchArray`,与 `String.match` 结构一致)
- `match.args` / `match.rest` 是**正则匹配之外**的剩余文本,整段被正则匹配时为空

如果只是「命令词 + 空白参数」,没必要用正则,用字符串命令即可:

```ts
import type { CommandContext, CommandMatch, OneBotMessageEvent } from '../core';

@Command('roll', { prefixes: [''] })
roll(event: OneBotMessageEvent, ctx: CommandContext, match: CommandMatch) {
    const sides = Number(match.args[0] ?? 6);
    ctx.reply(`${Math.floor(Math.random() * sides) + 1}`);
}
```

### 选项 `options`

| 选项           | 类型               | 默认值      | 说明                                     |
| -------------- | ------------------ | ----------- | ---------------------------------------- |
| `prefixes`     | `string \| string[]` | `['/']`   | 命令前缀,命中时被剥离;裸命令传 `['']`     |
| `trim`         | `boolean`          | `true`      | 匹配前先去掉首尾空白                     |
| `caseSensitive`| `boolean`          | `false`     | 是否区分大小写                           |

```ts
import type { CommandContext, OneBotMessageEvent } from '../core';

// 支持 /ping 和 !ping,大小写不敏感
@Command('ping', { prefixes: ['/', '!'] })
ping(event: OneBotMessageEvent, ctx: CommandContext) {
    ctx.reply('pong');
}

// 裸命令 ping(不带任何前缀)
@Command('ping', { prefixes: [''] })
ping(event: OneBotMessageEvent, ctx: CommandContext) {
    ctx.reply('pong');
}
```

### 处理器参数

`match` 类型为 `CommandMatch`,字段如下:

| 字段      | 说明                                              |
| --------- | ------------------------------------------------- |
| `command` | 匹配到的命令(字符串命令=第一个词;正则=整个匹配文本) |
| `text`    | 剥离前缀后的消息文本                              |
| `prefix`  | 命中的前缀(裸命令时为空字符串)                    |
| `args`    | 匹配后剩余文本按空白切分的参数数组(正则:匹配之外的剩余) |
| `rest`    | 与 `args` 同源但保留原文(正则:匹配之外的剩余)      |
| `match`   | `RegExpMatchArray \| null`(仅正则时非 null,捕获组在此) |

## `@AdminCommand()` — 管理员指令

标记方法为管理员指令,**必须与 `@Command` 搭配使用**。命中该指令的消息事件会被内核中间件自动附带 `isAdmin: true`,用于后续鉴权:

```ts
import type { CommandContext, OneBotMessageEvent } from '../core';

@AdminCommand()
@Command('ban')
ban(event: OneBotMessageEvent, ctx: CommandContext) {
    // 命中该指令的消息会带 event.isAdmin === true
}
```

要点:

- `@AdminCommand()` 不改变处理器签名,仍按 `@Command` 的 `(event, ctx, match)` 编写
- 事件类型统一用 `OneBotMessageEvent`(内核已通过类型增强为其注入 `isAdmin?: boolean`),直接读 `event.isAdmin` 即可,无需额外类型
- 内核只负责**打标**,拦截/放行逻辑由你在中间件里读 `event.isAdmin` 自行实现(示例见 [06-middleware](06-middleware.md))
- 未命中管理员指令的普通消息,事件**不会**带 `isAdmin` 字段(值为 `undefined`)

## 事件装饰器

事件装饰器不传 `command`,而是监听某类事件。处理器第一个参数是**事件对象** `event`,类型随装饰器不同而收窄。

### `@OnMessage()` — 任意消息

```ts
import type { OneBotMessageEvent, SnowLumaEventContext } from '../core';

@OnMessage()
onMessage(event: OneBotMessageEvent, ctx: SnowLumaEventContext) {
    // event 可能是群消息或私聊消息,常用 event.raw_message 取文本
    if (event.raw_message.includes('早上好')) {
        ctx.reply('早上好呀');
    }
}
```

### `@OnGroupMessage()` — 群聊消息

```ts
import type { OneBotGroupMessageEvent, SnowLumaEventContext } from '../core';

@OnGroupMessage()
onGroup(event: OneBotGroupMessageEvent, ctx: SnowLumaEventContext) {
    const { group_id, user_id, raw_message, sender } = event;
    ctx.reply(`群 ${group_id} 里 ${sender.nickname} 说: ${raw_message}`);
}
```

`event` 类型为 `OneBotGroupMessageEvent`,特有字段:

| 字段      | 说明                          |
| --------- | ----------------------------- |
| `group_id`| 群号                          |
| `user_id` | 发送者 QQ 号                  |
| `sender`  | `{ user_id, nickname, card, role, ... }` |
| `message` | 消息段数组(结构化)            |
| `raw_message` | 纯文本消息                |
| `message_id` | 消息 ID(可用于回复)      |

### `@OnPrivateMessage()` — 私聊消息

```ts
import type { OneBotPrivateMessageEvent, SnowLumaEventContext } from '../core';

@OnPrivateMessage()
onPrivate(event: OneBotPrivateMessageEvent, ctx: SnowLumaEventContext) {
    const { user_id } = event;
    ctx.reply(`你好, ${user_id},这里是私聊`);
}
```

`event` 类型为 `OneBotPrivateMessageEvent`,有 `user_id` 等字段,无 `group_id`。

### `@OnNotice(type?)` — 通知事件

`type` 可选,填写则只处理该子类型,不填处理所有通知。

```ts
import type { OneBotNoticeEvent, SnowLumaEventContext } from '../core';

// 只处理「群消息撤回」
@OnNotice('group_recall')
onRecall(event: OneBotNoticeEvent, ctx: SnowLumaEventContext) {
    ctx.reply(`收到撤回通知(时间 ${event.time})`);
}

// 处理所有通知,自行判断类型
@OnNotice()
onNotice(event: OneBotNoticeEvent, ctx: SnowLumaEventContext) {
    this.bot.logger.info(`通知: ${event.notice_type}`);
}
```

常见通知子类型:

| notice_type         | 说明                 |
| ------------------- | -------------------- |
| `group_recall`      | 群消息撤回           |
| `friend_recall`     | 好友消息撤回         |
| `group_increase`    | 新人入群             |
| `group_decrease`    | 成员退群/被踢        |
| `group_admin`       | 管理员变动           |
| `group_upload`      | 群文件上传           |
| `friend_add`        | 好友添加             |

### `@OnRequest(type?)` — 请求事件

`type` 可选:`'friend'`(好友申请)或 `'group'`(加群/邀请)。未处理请求事件默认不自动放行。

```ts
import type { OneBotRequestEvent, SnowLumaEventContext } from '../core';

// 自动同意好友申请
@OnRequest('friend')
onFriendRequest(event: OneBotRequestEvent, ctx: SnowLumaEventContext) {
    ctx.approve();
}

// 加群申请按备注关键词决定
@OnRequest('group')
onGroupRequest(event: OneBotRequestEvent, ctx: SnowLumaEventContext) {
    if (event.comment?.includes('jsbot')) {
        ctx.approve();
    } else {
        ctx.reject('请注明来源');
    }
}
```

`ctx.approve()` / `ctx.reject(reason)` 见 [04-context-and-reply](04-context-and-reply.md)。

### `@OnMiddleware()` — 事件中间件

拦截所有事件,可放行、拦截或改写。详见 [06-middleware](06-middleware.md)。

```ts
import type { EventNext, SnowLumaEvent, SnowLumaEventContext } from '../core';

@OnMiddleware()
async filter(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
    if ('user_id' in event && event.user_id === 10001) return;
    await next();
}
```

## 组合示例:一个功能多个入口

同一逻辑同时支持命令与关键词触发:

```ts
import { Plugin, Command, OnGroupMessage, OnPrivateMessage } from '../core';
import type { OneBotMessageEvent, SnowLumaEventContext } from '../core';

export class GreetPlugin extends Plugin {
    @Command('hello')
    @OnGroupMessage()
    @OnPrivateMessage()
    greet(event: OneBotMessageEvent, ctx: SnowLumaEventContext) {
        ctx.reply('Hello!');
    }
}
```

> 注意:此时 `@Command` 分支的处理器会多收到第三个参数 `match`,其余分支忽略即可。

## 处理器参数与类型标注

### 参数顺序(重要)

**所有处理器的第一个参数都是事件对象 `event`,不是 `ctx`。**

| 装饰器           | 处理器签名                              |
| ---------------- | --------------------------------------- |
| `@Command`       | `(event, ctx, match)`                   |
| 事件装饰器       | `(event, ctx)`                          |
| `@OnMiddleware`  | `(event, ctx, next)`                    |

`ctx` 上才有 `reply()` / `approve()` 等方法;`event` 是事件数据(如 `event.group_id`)。把第一个参数命名为 `ctx` 会让 `ctx.reply()` 在运行时崩溃(事件对象没有 `reply`)。

### 为什么参数要手动标注类型

TypeScript 的装饰器**无法推断**处理器方法的参数类型:装饰器只对方法做单向类型检查,不会反向给方法参数提供类型。因此在 `strict` 下,未标注的参数会报隐式 `any`。需要手动标注。

### 推荐标注写法

框架在 `src/core` 导出了便捷类型,从 `../core` 一次导入即可:

```ts
import { Plugin, Command, OnGroupMessage } from '../core';
import type {
    CommandContext,       // = ctx(含 command 字段),@Command 专用
    CommandMatch,         // @Command 的 match
    OneBotGroupMessageEvent,
    OneBotMessageEvent,
    SnowLumaEventContext, // 事件装饰器的 ctx
} from '../core';

export class TypedPlugin extends Plugin {
    @Command('test')
    test(event: OneBotMessageEvent, ctx: CommandContext, match: CommandMatch) {
        ctx.reply(`命中: ${match.command}`);
    }

    @Command('simple')
    simple(event: OneBotMessageEvent, ctx: CommandContext) {
        ctx.reply('不用 match 时可以不写第三个参数');
    }

    @OnGroupMessage()
    onGroup(event: OneBotGroupMessageEvent, ctx: SnowLumaEventContext) {
        ctx.reply(`群 ${event.group_id}`);
    }
}
```

不习惯标注时可以省略类型(`strict` 下会报隐式 any,不影响运行),或在 `07-types-and-guards` 查看更多类型写法。
