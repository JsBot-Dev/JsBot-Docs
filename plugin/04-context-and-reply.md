# 04 · 事件上下文与回复

处理器第二个参数 `ctx` 是事件上下文(`SnowLumaEventContext`),它把「当前事件」和「回复能力」打包在一起。

## ctx 成员一览

| 成员              | 类型                    | 说明                               |
| ----------------- | ----------------------- | ---------------------------------- |
| `ctx.event`       | `SnowLumaEvent`         | 当前事件对象(与处理器第一参数相同) |
| `ctx.client`      | `SnowLumaApiClient`     | SDK 客户端,同 `this.bot.client`    |
| `ctx.stopped`     | `boolean`               | 是否已被 `stopPropagation()` 停止   |
| `ctx.stopPropagation()` | `() => void`       | 停止后续处理器继续执行              |
| `ctx.reply(msg, options?)` | `(...) => Promise` | 按事件场景回复(群/私聊)         |
| `ctx.approve(options?)` | `(...) => Promise` | 同意请求事件                        |
| `ctx.reject(reason?, options?)` | `(...) => Promise` | 拒绝请求事件                 |
| `ctx.quickOperation(op, options?)` | `(...) => Promise` | 快捷操作(OneBot 快速操作)    |

## `ctx.reply()`

`reply` 自动判断场景:群消息回复到群里,私聊回复到私聊。无需手动传群号/QQ 号。

```ts
import { chain, text } from '@snowluma/sdk';
import type { CommandContext, OneBotMessageEvent } from '../core';

@Command('ping')
ping(event: OneBotMessageEvent, ctx: CommandContext) {
    ctx.reply('pong');          // 纯文本
    ctx.reply(text('你好'));     // text() 构建文本段
    ctx.reply(chain().text('多').face(1).text('段消息'));
}
```

`message` 参数类型是 `OutgoingMessage`:字符串、文本段、消息段数组或 `MessageChain` 均可。

## 消息链构建

SDK 提供两类构建方式,推荐**流式链**:

### 流式链 `chain()`

```ts
import { chain } from '@snowluma/sdk';

chain()
    .at(123456)          // @某人
    .text(' 你好')
    .face(178)           // 表情
    .image('https://example.com/a.png')
    .build();            // 得到消息段数组
```

### 单段函数

`text()` / `at()` / `image()` 等直接返回包含单段的链,可再链式追加:

```ts
import { at, image, reply } from '@snowluma/sdk';

at(123456).text(' 早上好');        // @123456 早上好
reply(10086).text('收到');         // 引用消息 10086 并回复
```

## 消息段参考

| 段函数         | 用法示例                                                        | 说明                 |
| -------------- | --------------------------------------------------------------- | -------------------- |
| `.text(str)`   | `.text('hello')`                                                | 纯文本               |
| `.at(qq, opts?)` | `.at(123456)` / `.at('all')`                                   | @某人或 @全体        |
| `.atAll()`     | `.atAll()`                                                      | @全体成员            |
| `.face(id)`    | `.face(178)`                                                    | QQ 表情              |
| `.reply(id)`   | `.reply(1234)`                                                  | 引用回复(单用)       |
| `.image(file, opts?)` | `.image('https://x/y.png')` / `.image('./local.png')`    | 图片(URL/路径/base64)|
| `.record(file)`| `.record('https://x/a.mp3')`                                    | 语音                 |
| `.video(file)` | `.video('https://x/a.mp4')`                                     | 视频                 |
| `.poke(type, id?)` | `.poke(1)`                                                 | 戳一戳               |
| `.json(data)`  | `.json({ app: 'com.tencent.miniapp', view: 'detail' })`         | JSON 卡片            |
| `.xml(data)`   | `.xml('<msg>...</msg>')`                                        | XML 卡片             |
| `.node(userId, nickname, content)` | `.node(123, '小明', '内容')`           | 转发消息节点         |
| `.forward(id)` | `.forward('resid')`                                             | 转发消息             |
| `.contact(type, id)` | `.contact('group', 123456)`                               | 推荐群/好友          |
| `.share({...})`| `.share({ url, title, image })`                                 | 链接分享             |
| `.music({...})`| `.music({ type: 'qq', id: '001' })`                             | 音乐分享             |
| `.location({...})` | `.location({ lat, lon, title })`                            | 位置                 |
| `.br()`        | `.br()`                                                        | 换行                 |

### 实用示例

```ts
import { chain, at, image, reply } from '@snowluma/sdk';
import type { CommandContext, OneBotGroupMessageEvent, OneBotMessageEvent, SnowLumaEventContext } from '../core';

// 复读并 @ 发送者
@OnGroupMessage()
echo(event: OneBotGroupMessageEvent, ctx: SnowLumaEventContext) {
    ctx.reply(chain().reply(event.message_id).at(event.user_id).text(` ${event.raw_message}`));
}

// 发送图片卡片
@Command('cat', { prefixes: '/' })
cat(event: OneBotMessageEvent, ctx: CommandContext) {
    ctx.reply(
        chain()
            .text('今日猫猫:')
            .image('https://cataas.com/cat')
            .br()
            .text('via cataas'),
    );
}

// @全体 + 通知
@Command('announce')
announce(event: OneBotMessageEvent, ctx: CommandContext) {
    ctx.reply(chain().atAll().text(' 全体注意!'));
}
```

## 发送到指定场景

`ctx.reply` 只回复当前会话。需要主动发送(定时任务、跨群通知)时用 `client`:

```ts
// 主动给指定群发消息
await this.bot.client.sendGroupMessage(123456, chain().text('通知:'));
// 主动给某人发私聊
await this.bot.client.sendPrivateMessage(10001, '你好');
```

更完整的动作调用见 [05-actions](05-actions.md)。

## 请求事件决策

好友/加群申请事件用 `approve` / `reject` 决策:

```ts
import type { OneBotRequestEvent, SnowLumaEventContext } from '../core';

@OnRequest('friend')
onFriend(event: OneBotRequestEvent, ctx: SnowLumaEventContext) {
    if (event.comment?.includes('jsbot')) {
        ctx.approve();
    } else {
        ctx.reject('需要备注来源');
    }
}
```

## 停止传播

`stopPropagation()` 让当前事件不再进入后续处理器:

```ts
import type { OneBotMessageEvent, SnowLumaEventContext } from '../core';

@OnMessage()
onlyPing(event: OneBotMessageEvent, ctx: SnowLumaEventContext) {
    if (event.raw_message !== 'ping') return;
    ctx.reply('pong');
    ctx.stopPropagation(); // 后续处理器不再收到这条消息
}
```