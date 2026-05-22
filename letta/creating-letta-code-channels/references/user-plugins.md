# Dynamic user channel plugins

## Table of contents

- Layout
- channel.json
- accounts.json
- Minimal plugin contract
- messageActions
- Runtime dependencies
- Headless pairing
- Security
- Debugging

## Layout

```text
~/.letta/channels/<id>/
  channel.json
  plugin.mjs
  accounts.json
  pairing.yaml
  routing.yaml
  runtime/
    package.json
    node_modules/
```

Use dynamic plugins for community/headless channels and fast experiments. Do not use first-party ids (`telegram`, `slack`, `discord`); the registry skips user plugins that shadow them.

## channel.json

```json
{
  "id": "whatsapp-community",
  "displayName": "WhatsApp Community",
  "entry": "./plugin.mjs",
  "runtimePackages": ["some-sdk@1.0.0"],
  "runtimeModules": ["some-sdk"]
}
```

Rules:

- `id` must match the directory name.
- `entry` is relative to the channel directory and must not escape it.
- `runtimePackages` install under `runtime/` via `letta channels install <id>`.
- `runtimeModules` are what the plugin imports/checks.
- Runtime resolution should not count parent/dev repo `node_modules`.

## accounts.json

Use `account.config` for plugin-owned settings:

```json
{
  "accounts": [
    {
      "channel": "whatsapp-community",
      "accountId": "main",
      "displayName": "WhatsApp Community",
      "enabled": true,
      "dmPolicy": "pairing",
      "allowedUsers": [],
      "config": {
        "token": "...",
        "phoneNumberId": "..."
      },
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-01-01T00:00:00.000Z"
    }
  ]
}
```

Custom accounts have no generic `binding` field. Routing lives in `routing.yaml`, created by pairing, `letta channels route add`, or direct file management.

## Minimal plugin contract

`plugin.mjs` exports `channelPlugin` or `default`:

```js
export const channelPlugin = {
  metadata: {
    id: "whatsapp-community",
    displayName: "WhatsApp Community",
    runtimePackages: ["some-sdk@1.0.0"],
    runtimeModules: ["some-sdk"]
  },

  async createAdapter(account) {
    let onMessageHandler = null;
    let running = false;

    return {
      id: `whatsapp-community:${account.accountId}`,
      channelId: "whatsapp-community",
      accountId: account.accountId,
      name: account.displayName ?? "WhatsApp Community",
      async start() { running = true; },
      async stop() { running = false; },
      isRunning() { return running; },
      async sendMessage(msg) { return { messageId: crypto.randomUUID() }; },
      async sendDirectReply(chatId, text, options) {},
      get onMessage() { return onMessageHandler; },
      set onMessage(handler) { onMessageHandler = handler; }
    };
  },

  messageActions: {
    describeMessageTool() { return { actions: ["send"] }; },
    async handleAction({ adapter, request, formatText }) {
      if (request.action !== "send") return `Error: unsupported action ${request.action}`;
      const formatted = formatText(request.message ?? "");
      const result = await adapter.sendMessage({
        channel: request.channel,
        chatId: request.chatId,
        text: formatted.text,
        parseMode: formatted.parseMode,
        replyToMessageId: request.replyToMessageId,
        threadId: request.threadId
      });
      return `Message sent to ${request.channel} (message_id: ${result.messageId})`;
    }
  }
};
```

Inbound messages must call `adapter.onMessage(msg)` with:

```ts
{
  channel: string;
  accountId?: string;
  chatId: string;
  senderId: string;
  senderName?: string;
  chatLabel?: string;
  text: string;
  timestamp: number;
  messageId?: string;
  threadId?: string | null;
  chatType?: "direct" | "channel";
  isMention?: boolean;
  attachments?: ChannelMessageAttachment[];
  reaction?: ChannelReactionEvent;
  raw?: unknown;
}
```

## messageActions

Every reply-capable plugin needs `messageActions`. See `references/message-actions.md` for details. Without it, inbound can work while outbound silently fails.

## Runtime dependencies

Install runtime deps:

```bash
letta channels install <id>
```

Check:

- Dependencies are under `~/.letta/channels/<id>/runtime/node_modules`.
- User plugin import can resolve them, often through a `node_modules` symlink beside `plugin.mjs`.
- The runtime resolver does not falsely pass because the monorepo has the dependency installed.

## Headless pairing

Pair from CLI:

```bash
letta channels pair \
  --channel <id> \
  --code <code> \
  --agent <agent-id> \
  --conversation <conversation-id>
```

Or route statically:

```bash
letta channels route add \
  --channel <id> \
  --chat-id <platform-chat-id> \
  --agent <agent-id> \
  --conversation <conversation-id>
```

`dmPolicy` behavior:

- `pairing`: unknown senders get a pairing code. Good for manual tests.
- `allowlist`: only `allowedUsers` sender IDs pass. Good for known headless users.
- `open`: everyone can reach routing lookup. Use with explicit routes or safe public channels.

## Security

- Do not expose tool approval/control prompts publicly.
- Treat platform user IDs and channel IDs as untrusted input.
- Avoid logging secrets from `account.config`.
- Avoid dynamic imports outside the plugin directory and runtime dependency paths.

## Debugging

- Plugin not discovered: id mismatch, invalid id chars, bad `channel.json`, first-party id shadowing.
- Install passes but import fails: runtime deps are not actually in the user plugin runtime path.
- Pairing code repeats: pairing was redeemed for different `accountId`/sender or listener did not reload store.
- Reply no-ops: missing `messageActions` or wrong route keys.
