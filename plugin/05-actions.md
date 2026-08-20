# 05 · 动作调用

机器人主动调用 OneBot 接口的能力来自 `this.bot.client`(`SnowLumaWebSocketClient`),它是 SDK 提供的**全类型化动作客户端**。所有动作都有类型签名、入参校验与返回类型推导。

`ctx.client` 与 `this.bot.client` 是同一个实例,处理事件时也可用 `ctx.client`。

## 访问方式

```ts
import { Plugin, Command } from '../core';
import type { CommandContext, OneBotMessageEvent } from '../core';

export class AdminPlugin extends Plugin {
    @Command('mute', { prefixes: '/' })
    async mute(event: OneBotMessageEvent, ctx: CommandContext) {
        if (event.message_type !== 'group') return;
        await this.bot.client.setGroupBan(event.group_id, event.user_id, 600);
        ctx.reply('已禁言 10 分钟');
    }
}
```

## 动作分类与示例

### 消息

| 动作                | 说明                     |
| ------------------- | ------------------------ |
| `sendMsg(params)`   | 通用发送                 |
| `sendGroupMessage(gid, msg)` | 群消息           |
| `sendPrivateMessage(uid, msg)` | 私聊消息        |
| `getMessage(id)`    | 取消息                   |
| `deleteMessage(id)` | 撤回                     |
| `markMessageAsRead(id)` | 标记已读             |

```ts
await this.bot.client.sendGroupMessage(123456, '自动群发');
await this.bot.client.deleteMessage(10086); // 撤回
```

### 好友

```ts
await this.bot.client.getFriendList();
await this.bot.client.getStrangerInfo(10001); // 陌生人资料
await this.bot.client.sendLike(10001);        // 点赞
await this.bot.client.friendPoke(10001);      // 戳一戳
await this.bot.client.setFriendRemark(10001, '备注名');
```

### 群管理

```ts
// 成员
await this.bot.client.setGroupKick(groupId, userId, { rejectAddRequest: true });
await this.bot.client.setGroupBan(groupId, userId, 600);        // 禁言
await this.bot.client.setGroupWholeBan(groupId, true);          // 全员禁言
await this.bot.client.setGroupAdmin(groupId, userId, true);     // 设管理员
await this.bot.client.setGroupCard(groupId, userId, '新名片');
await this.bot.client.setGroupName(groupId, '新群名');
await this.bot.client.setGroupSpecialTitle(groupId, userId, '头衔');

// 查询
const list = await this.bot.client.getGroupMemberList(groupId);
const info = await this.bot.client.getGroupMemberInfo(groupId, userId, { noCache: true });
const history = await this.bot.client.getGroupMessageHistory({ group_id: groupId, count: 20 });
```

`getGroupMemberList` 返回 `JsonObject[]`,成员常用字段:`user_id`、`nickname`、`card`、`role`、`title`。

### 文件

```ts
await this.bot.client.uploadGroupFile(groupId, './file.zip', { name: 'file.zip' });
const url = await this.bot.client.getGroupFileUrl(groupId, fileId);
const files = await this.bot.client.getGroupRootFiles(groupId);
await this.bot.client.downloadFile({ url: 'https://...', name: 'file.zip' });
```

### 精华与反应

```ts
await this.bot.client.setEssenceMessage(messageId);
const list = await this.bot.client.getEssenceMessageList(groupId);
await this.bot.client.setGroupReaction({ message_id: messageId, code: '144', is_set: true });
```

### 转发消息

```ts
import { chain } from '@snowluma/sdk';

const forward = chain()
    .node(123, '小明', '第一条')
    .node(456, '小红', '第二条');

await this.bot.client.sendGroupForwardMessage(groupId, forward);
```

### 请求决策(主动侧)

```ts
await this.bot.client.setFriendAddRequest(flag, true);                 // 同意好友申请
await this.bot.client.setGroupAddRequest(flag, { approve: true });     // 同意加群
```

## 通用调用

不确定使用哪个动作时,可以用通用接口:

```ts
// call:取 data,失败抛 SnowLumaApiError
const data = await this.bot.client.raw('get_group_member_info', {
    group_id: 1,
    user_id: 2,
    no_cache: true,
});

// 保留完整 OneBot 响应信封 { status, retcode, data, message }
const envelope = await this.bot.client.rawResponse('send_msg', { ... });
```

完整动作列表见 `node_modules/@snowluma/sdk/dist/client/api-client.d.ts`,或 `getLoginInfo`/`getStatus`/`getVersionInfo` 快速自检:

```ts
const v = await this.bot.client.getVersionInfo();
this.bot.logger.info(`OneBot 实现: ${JSON.stringify(v)}`);
```

## 错误处理

`client.call` / 类型化动作在接口返回失败时抛 `SnowLumaApiError`(含 `retcode`、`message`)。建议在需要可靠性的路径 catch:

```ts
import type { CommandContext, OneBotMessageEvent } from '../core';

@Command('ban', { prefixes: '/' })
async ban(event: OneBotMessageEvent, ctx: CommandContext) {
    if (event.message_type !== 'group') return;
    try {
        await this.bot.client.setGroupBan(event.group_id, event.user_id, 600);
        ctx.reply('已禁言');
    } catch (err) {
        ctx.reply(`禁言失败: ${err instanceof Error ? err.message : err}`);
    }
}
```

可用错误类型(从 `@snowluma/sdk` 导入):

```ts
import {
    SnowLumaApiError,
    SnowLumaAuthError,
    SnowLumaTimeoutError,
    SnowLumaConnectionError,
} from '@snowluma/sdk';
```

## 与 `ctx.reply` 的分工

| 场景                 | 推荐                       |
| -------------------- | -------------------------- |
| 回复当前会话         | `ctx.reply(...)`           |
| 主动发送/定时任务    | `client.sendGroupMessage`  |
| 需要操作别的会话     | `client.send*`             |

## 下一步

- 事件类型与守卫 → [07-types-and-guards](07-types-and-guards.md)
- 错误处理与并发约定 → [08-best-practices](08-best-practices.md)