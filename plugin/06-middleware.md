# 06 · 中间件

中间件是所有事件进入处理器**之前**的一道拦截层,适合做全局性的横切逻辑:**鉴权、风控、日志、消息改写、放行控制**。

## 签名

```ts
import type { EventNext, SnowLumaEvent, SnowLumaEventContext } from '../core';

@OnMiddleware()
async filter(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
    // 事件分发前的逻辑
    await next();   // 放行:继续后续处理器
    // 事件分发后的逻辑(可选)
}
```

- `next()` 放行事件到下一层;不调用 `next()` 则吞掉事件,后续处理器不执行
- 同一插件可声明多个 `@OnMiddleware`,多个中间件按注册顺序串成管道

## 放行与拦截

```ts
import { Plugin, OnMiddleware } from '../core';
import type { EventNext, SnowLumaEvent, SnowLumaEventContext } from '../core';

// 全局黑名单:屏蔽指定用户/群
export class BlacklistPlugin extends Plugin {
    private blocked: Set<number>;

    async onLoad() {
        this.blocked = new Set([10001, 10002]);
    }

    @OnMiddleware()
    async block(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
        const userId = 'user_id' in event ? event.user_id : undefined;
        const groupId = 'group_id' in event ? event.group_id : undefined;

        if ((userId && this.blocked.has(userId)) || (groupId && this.blocked.has(groupId))) {
            this.bot.logger.info(`拦截来自 ${userId ?? groupId} 的事件`);
            return; // 不调用 next(),事件到此为止
        }
        await next();
    }
}
```

## 全局日志

```ts
import { Plugin, OnMiddleware } from '../core';
import type { EventNext, SnowLumaEvent, SnowLumaEventContext } from '../core';

export class LogPlugin extends Plugin {
    @OnMiddleware()
    async log(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
        const start = Date.now();
        await next();
        const kind = event.post_type;
        this.bot.logger.debug(`${kind} 事件处理耗时 ${Date.now() - start}ms`);
    }
}
```

## 消息改写

中间件可以修改 `event` 后再放行(对同一事件对象原地改):

```ts
import type { EventNext, SnowLumaEvent, SnowLumaEventContext } from '../core';

@OnMiddleware()
async normalize(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
    if ('raw_message' in event && typeof event.raw_message === 'string') {
        event.raw_message = event.raw_message.trim();
    }
    await next();
}
```

> 注意:事件对象由 SDK 分发,修改仅影响本进程内的后续处理器,不会回传。

## 按类型精筛

中间件收到的是 `SnowLumaEvent`,需要先用守卫收窄类型再访问字段:

```ts
import { isGroupMessageEvent } from '@snowluma/sdk';
import type { EventNext, SnowLumaEvent, SnowLumaEventContext } from '../core';

@OnMiddleware()
async onlyGroup(event: SnowLumaEvent, ctx: SnowLumaEventContext, next: EventNext) {
    if (!isGroupMessageEvent(event)) return;
    // 此时 event 是 OneBotGroupMessageEvent,可安全访问 group_id
    await next();
}
```

## 执行顺序

顺序 = 插件注册顺序(`builtinPlugins` 数组 / `bot.use()` 顺序),先注册的先执行:

```
BlacklistPlugin.mw → LogPlugin.mw → 各插件的命令/事件处理器
```

中间件里的 `next()` 是「继续分发」,所有处理器结束后返回。因此:

- 想在所有处理前做检查 → `next()` 之前写逻辑
- 想在所有处理后做收尾 → `next()` 之后写逻辑

## 与命令/事件处理器的关系

- 中间件按声明顺序运行,**早于**同插件的其它处理器
- 中间件拦截后,该事件不会再触发任何 `@Command` / `@OnMessage` 等
- 一个事件会依次经过:所有中间件 → 匹配的处理器

## 典型场景清单

| 场景       | 做法                               |
| ---------- | ---------------------------------- |
| 用户/群黑名单 | 拦截特定 `user_id` / `group_id`   |
| 关键词风控 | 命中敏感词直接拦截                  |
| 频率限制   | 按 `user_id` 计数,超频拦截          |
| 全局埋点   | 在 `next()` 前后计时、计数           |
| 统一预处理 | 规范化 `raw_message`,注入字段        |