# 插件开发指南 · 导读

本目录是 JsBot 的插件开发文档,按主题拆分为多个文件。建议按编号顺序阅读。

## 文档目录

| 文档                                        | 内容                                       |
| ------------------------------------------- | ------------------------------------------ |
| [01-quickstart](01-quickstart.md)               | 插件本质、目录约定、最小插件、登记方式      |
| [02-decorators](02-decorators.md)                   | `@Command` 与全部事件装饰器详解             |
| [03-lifecycle](03-lifecycle.md)               | `onLoad` / `onUnload` 钩子                 |
| [04-context-and-reply](04-context-and-reply.md) | `ctx` API、消息链构建、各类消息段           |
| [05-actions](05-actions.md)               | 调用 OneBot 动作、分类示例、错误处理        |
| [06-middleware](06-middleware.md)                   | 事件中间件拦截/放行/改写                    |
| [07-types-and-guards](07-types-and-guards.md)           | 事件类型层级、类型守卫、断言模式            |
| [08-best-practices](08-best-practices.md)               | 约定、错误处理、并发、状态管理、测试        |

## 基础概念

一个插件 = **一个继承 `Plugin` 的类** + **若干装饰器标注的处理器方法**。

```ts
import { Plugin, Command } from '../core';

export class PingPlugin extends Plugin {
    @Command('ping')
    ping(event, ctx) {
        ctx.reply('pong');
    }
}
```

核心框架(`src/core`)在启动时自动完成:

1. 实例化插件(`new PluginCtor(bot)`)
2. 扫描类上的装饰器元数据
3. 把每个处理器注册到 SDK 客户端(`client.command()` / `client.onGroupMessage()` 等)
4. 退出时按相反顺序注销并调用 `onUnload`

## 常用导入路径

- 框架能力:`import { Plugin, Command, OnGroupMessage, ... } from '../core'`(即 `src/core/index.ts`)
- SDK 类型与工具:`import { chain, at, image, ... } from '@snowluma/sdk'`
- SDK 类型守卫:`import { isGroupMessageEvent } from '@snowluma/sdk'`

## 术语

| 术语       | 含义                                           |
| ---------- | ---------------------------------------------- |
| 事件 Event | OneBot 协议推送的消息/通知/请求等数据包         |
| 处理器     | 插件类中由装饰器标注的方法,事件命中时被调用     |
| 上下文 ctx | `SnowLumaEventContext`,处理器第二个参数,用于回复 |
| 动作 Action| 机器人主动调用的 OneBot 接口,如发送消息、禁言    |

## 下一步

[01-quickstart →](01-quickstart.md)
