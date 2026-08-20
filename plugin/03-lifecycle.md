# 03 · 生命周期

`Plugin` 基类提供两个可选的生命周期钩子,用于在处理器注册前后做初始化和清理。

| 钩子       | 触发时机                       | 建议用途                         |
| ---------- | ------------------------------ | -------------------------------- |
| `onLoad`   | 处理器注册**之前**             | 加载配置、初始化数据、预取资源   |
| `onUnload` | 处理器注销**之后**、连接关闭前 | 释放资源、持久化状态、打点上报   |

## 完整示例

```ts
import { Plugin, Command, OnMessage } from '../core';
import type { CommandContext, OneBotMessageEvent } from '../core';
import { loadData, saveData } from './data';

export class StatsPlugin extends Plugin {
    private hits = 0;

    // 启动时:读取持久化数据,并记录登录信息
    async onLoad() {
        this.hits = await loadData('hits');
        const login = await this.bot.client.getLoginInfo();
        this.bot.logger.info(`账号 ${login.user_id} 已登录,历史计数 ${this.hits}`);
    }

    // 停止时:把计数落盘
    async onUnload() {
        await saveData('hits', this.hits);
        this.bot.logger.info(`本次运行共收到 ${this.hits} 条消息`);
    }

    @OnMessage()
    count() {
        this.hits++;
    }

    @Command('stats')
    stats(event: OneBotMessageEvent, ctx: CommandContext) {
        ctx.reply(`累计收到 ${this.hits} 条消息`);
    }
}
```

## 执行时序

`bot.start()` 的完整顺序:

```
1. 对每个插件(按注册顺序):
   a. await plugin.onLoad()
   b. 扫描装饰器,注册处理器
2. await client.connect()          ← 连接成功后事件才开始分发
3. 输出「Bot 已启动」
```

`bot.stop()`(收到 SIGINT/SIGTERM 时自动触发):

```
1. 对每个插件(按注册顺序):
   a. 注销处理器
   b. await plugin.onUnload()
2. 断开 client 连接
3. 输出「Bot 已停止」
```

## 关键点

- **onLoad 里不要注册处理器** — 处理器由框架统一扫描注册,无需(也无法)手动注册
- **onLoad 失败会中断启动** — 抛异常会阻止 `client.connect()`,整个 Bot 启动失败
- **onUnload 中的 await 会完成** — 优雅退出会等待钩子执行完毕,适合落盘等收尾工作
- **注册顺序 = `builtinPlugins` 数组顺序 = `bot.use()` 调用顺序**,中间件按此顺序生效

## 多次启动/停止

`start()` / `stop()` 是幂等的:重复调用无副作用。但同一进程反复 start/stop 时,`onLoad`/`onUnload` 会各执行一次,插件应按「可重复装载」来设计(避免 onLoad 里的单次性副作用)。
