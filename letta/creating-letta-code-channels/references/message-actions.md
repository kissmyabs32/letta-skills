# MessageChannel and channel message actions

## Table of contents

- Why messageActions matter
- Minimal shape
- Action design
- Route and target resolution
- Common failures
- Tests

## Why messageActions matter

`MessageChannel` is the shared agent tool for outbound channel replies. It delegates channel-specific behavior to `plugin.messageActions`.

If a plugin lacks `messageActions`, the tool returns a string result like:

```text
Channel "discord" does not expose MessageChannel actions.
```

Agents may not treat that as an exception. This is the most common reason inbound works but replies silently fail.

## Minimal shape

Use Telegram as the simple first-party reference. Dynamic plugins can use this shape:

```js
messageActions: {
  describeMessageTool() {
    return { actions: ["send"] };
  },

  async handleAction({ adapter, request, formatText }) {
    if (request.action !== "send") {
      return `Error: unsupported action ${request.action}`;
    }
    const formatted = formatText(request.message ?? "");
    const result = await adapter.sendMessage({
      channel: request.channel,
      chatId: request.chatId,
      threadId: request.threadId,
      replyToMessageId: request.replyToMessageId,
      text: formatted.text,
      parseMode: formatted.parseMode,
    });
    return `Message sent to ${request.channel} (message_id: ${result.messageId})`;
  },
}
```

First-party channels should usually implement actions in `src/channels/<channel>/message-actions.ts` and export them from `plugin.ts`.

## Action design

Common actions:

- `send` — text reply.
- `react` — add/remove reaction.
- `upload_file` / media action — upload attachment.
- Channel-specific actions only when they are platform concepts agents should use.

Keep action names stable. Tool descriptions should tell the agent which arguments matter for the current channel, especially `chatId`, `threadId`, `messageId`, and `replyToMessageId`.

## Route and target resolution

`MessageChannel` can infer the current channel/route from turn sources when the agent is replying to an inbound channel notification. Explicit args still need to match route keys.

Check:

- Inbound `chatId` matches outbound target `chatId`.
- Inbound `threadId` is preserved when replies should stay threaded.
- `accountId` is included for multi-account channels.
- Direct replies vs normal messages use correct adapter methods.

For Discord and Slack, thread handling is easy to break. Test top-level mentions, existing thread replies, and route-derived replies.

## Common failures

- Missing `messageActions`: tool returns no-action string.
- Wrong `chatId`: route lookup misses.
- Wrong `threadId`: reply posts to parent channel or cannot find route.
- Adapter `sendMessage` ignores `replyToMessageId` or `threadId`.
- Channel formatting escapes/mentions incorrectly.
- Bot lacks platform permissions for threads, reactions, or uploads.

## Tests

Useful tests:

```bash
bun test src/channels/message-channel.test.ts
bun test src/channels/<channel>-message-channel.test.ts
bun test src/channels/message-tool-schema.test.ts
bun test src/tests/tools/tool-execution-context.test.ts
```

Add route-specific tests for new channel actions, especially if adding custom schema contributions.
