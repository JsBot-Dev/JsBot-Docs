# 08 · 最佳实践

## 1. 文件与组织

- 一个插件一个文件,放 `src/plugins/`;复杂插件拆成目录(入口 + 数据/工具模块)
- 插件类名即 `this.name`,命名用「功能名 + Plugin」后缀,如 `PingPlugin`
- 只在 `src/plugins/index.ts` 或 `main.ts` 注册,不要在插件内部互相 new

```ts
// 复杂插件目录结构
src/plugins/weather/
  index.ts        // WeatherPlugin(入口)
  service.ts      // 天气 API 封装
  types.ts        // 类型定义
```

## 2. 回复优先用 ctx

- 回复当前会话一律 `ctx.reply(...)`,不要手动 `sendGroupMessage`(还要判断场景)
- 需要主动/定时/跨会话发送时才用 `client.sendGroupMessage` 等

```ts
// 推荐
@Command('ping')
ping(ctx) {
    ctx.reply('pong');
}

// 不推荐:还得手动判断场景与事件类型
@Command('ping')
async ping(ctx) {
    const { event } = ctx;
    if (isGroupMessageEvent(event)) {
        await this.bot.client.sendGroupMessage(event.group_id, 'pong');
    } else if (isPrivateMessageEvent(event)) {
        await this.bot.client.sendPrivateMessage(event.user_id, 'pong');
    }
}
```

## 3. 错误处理

- 依赖外部服务(HTTP、DB)的处理器用 `try/catch`,避免异常导致分发中断
- 对失败动作给用户可读的提示
- `onLoad` 抛错会终止整个启动,校验失败应尽早抛、带清晰信息

```ts
@Command('weather', { prefixes: '/' })
async weather(event, ctx) {
    try {
        const data = await this.bot.client.getStrangerInfo(event.user_id);
        ctx.reply(JSON.stringify(data));
    } catch (err) {
        ctx.reply(`查询失败: ${err instanceof Error ? err.message : err}`);
    }
}
```

## 4. 并发与防抖

事件分发是并发的(每个处理器独立调用),共享状态要小心:

- 计数器/缓存用同步操作或 `Map` + 锁,不要假设同一时刻只有一个处理器在跑
- 高频命令(如复读、抽卡)加简单防抖/冷却,避免刷屏和 API 过载

```ts
export class CooldownPlugin extends Plugin {
    private lastUse = new Map<number, number>();

    @Command('cd')
    cd(event, ctx) {
        const now = Date.now();
        const last = this.lastUse.get(event.user_id) ?? 0;
        if (now - last < 3000) {
            ctx.reply('操作太频繁,请稍后再试');
            return;
        }
        this.lastUse.set(event.user_id, now);
        ctx.reply('ok');
    }
}
```

## 5. 状态管理

- 插件内状态存 `private` 字段,构造器/`onLoad` 里初始化
- 需要跨插件共享时,挂到 `bot` 上,如 `bot.store`(无内置,自行约定 key)
- 持久化数据在 `onUnload` 落盘,或在变更时即时写

```ts
export class CountPlugin extends Plugin {
    private count = 0;

    @OnMessage()
    bump() {
        this.count++;
    }

    async onUnload() {
        await writeFile('count.json', JSON.stringify({ count: this.count }));
    }
}
```

## 6. 配置

- 需要 token/URL 等参数时,优先用环境变量(加载进 `bot.config`)
- 不要在插件代码里硬编码敏感信息
- 从 `this.bot.config` 读取框架配置

```ts
export class ApiPlugin extends Plugin {
    private readonly token = process.env.API_TOKEN;

    async onLoad() {
        if (!this.token) {
            throw new Error('ApiPlugin: 缺少 API_TOKEN 环境变量');
        }
    }
}
```

## 7. 事件收窄

- `@OnMessage` 里用 `isGroupMessageEvent` 等守卫收窄,再访问特有字段
- `@OnGroupMessage` / `@OnPrivateMessage` 已自动收窄,无需再判断

```ts
@OnMessage()
onMessage(event, ctx) {
    if (!isGroupMessageEvent(event)) return;
    ctx.reply(`群 ${event.group_id}`);
}
```

## 8. 性能

- 同步计算直接做;异步任务(下载、请求)用 `Promise` 并行,不要串行 await
- 重复用到的消息链构建结果可缓存
- 长时间循环任务避免阻塞:用 `setImmediate` 分片或拆成小任务

## 9. 测试建议

框架未内置测试运行器,推荐 `vitest` + mock client:

```ts
// 用假 client 验证插件注册
const calls: string[] = [];
const fakeClient = {
    command: (s: any) => (calls.push(`command:${s}`), () => {}),
    onGroupMessage: () => (calls.push('group'), () => {}),
};
const reg = new PluginRegistry(fakeClient as any, new Logger('error'));
reg.register(new MyPlugin({} as any));
expect(calls).toContain('command:ping');
```

对纯逻辑(解析、计算)直接测函数;对消息回复,断言传入 `ctx.reply` 的内容。

## 10. 文档与维护

- 插件文件头部用 JSDoc 注明用途、命令示例
- 命令/事件改动同步更新本文档目录里的对应章节
- 保持 `npx tsc --noEmit` 通过后再提交