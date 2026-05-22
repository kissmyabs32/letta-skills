# Letta Code channel architecture

## Table of contents

- Core objects
- Lifecycle
- Account stores
- Inbound flow
- Outbound flow
- Route keys
- Listener/runtime gotchas

## Core objects

Channels are platform adapters that bridge external messages into the agent turn queue and expose outbound actions through the shared `MessageChannel` tool.

Important files:

- `src/channels/types.ts` — shared account/config/message/adapter types.
- `src/channels/plugin-types.ts` — `ChannelPlugin`, account patch, config patch, `messageActions` contracts.
- `src/channels/plugin-registry.ts` — first-party and dynamic plugin discovery.
- `src/channels/registry.ts` — adapter registry, pairing, auto-routing, route lookup, delivery buffering.
- `src/channels/accounts.ts` — `accounts.json` load/save, clone, legacy migration.
- `src/channels/routing.ts` — `routing.yaml` route store.
- `src/channels/pairing.ts` — pairing code store.
- `src/channels/service.ts` — desktop/websocket-facing account/config APIs and snapshots.
- `src/tools/impl/message-channel.ts` — shared agent tool dispatching outbound actions.

## Lifecycle

`initializeChannels(channelNames)` loads channel accounts, creates adapters, registers them in `ChannelRegistry`, and starts them. The listener later installs the message handler and marks the registry ready. Before ready, inbound deliveries buffer.

Adapters must expose:

```ts
{
  id: `${channelId}:${accountId}`,
  channelId,
  accountId,
  start(),
  stop(),
  isRunning(),
  sendMessage(msg),
  sendDirectReply(chatId, text, options?),
  onMessage?: (msg) => Promise<void>,
  prepareInboundMessage?(msg, { isFirstRouteTurn }),
  handleControlRequestEvent?(event)
}
```

`onMessage` is assigned by the registry. Platform SDK handlers should call it with an `InboundChannelMessage`.

## Account stores

First-party accounts live in `~/.letta/channels/<channel>/accounts.json` via `src/channels/accounts.ts`. Dynamic plugin accounts use the same store shape but put plugin-owned fields under `config`.

Legacy configs are still parsed from older `config.json`-style files for some first-party channels. If adding a field, check both `accounts.ts` and `config.ts` for legacy/default handling.

## Inbound flow

1. Platform event arrives in adapter.
2. Adapter filters bot/self messages, allowlists, mention policy, etc.
3. Adapter downloads/normalizes media if needed.
4. Adapter calls `adapter.onMessage(inbound)`.
5. `ChannelRegistry.handleInboundMessage` checks pending control requests.
6. Registry resolves account config.
7. Registry either auto-routes (Slack/Discord style) or falls through to pairing/generic route lookup.
8. Registry optionally calls `adapter.prepareInboundMessage` for first-route context hydration.
9. Registry formats XML with `formatChannelNotification` and delivers/buffers it for the agent turn queue.

## Outbound flow

1. Agent calls `MessageChannel`.
2. Tool resolves route/account from the current channel turn source or explicit args.
3. Tool loads the channel plugin and calls `messageActions.handleAction`.
4. `handleAction` validates the action and calls `adapter.sendMessage` or another adapter method.
5. Adapter posts to platform and returns `{ messageId }` when possible.

If `messageActions` is missing, `MessageChannel` does not throw. It returns a string result saying the channel exposes no actions. Treat this as a serious wiring bug.

## Route keys

A route is keyed by channel, account, `chatId`, and optional `threadId`.

Choose these deliberately:

- Direct chats: `chatId` is the platform DM/chat id, `threadId` null.
- Slack threads: usually `chatId` channel id, `threadId` Slack thread timestamp.
- Discord auto-created threads: `chatId` often becomes thread channel id, `threadId` thread channel id.
- Discord parent-channel inline replies: `chatId` parent channel id, `threadId` null.

Inbound and outbound must agree. If inbound emits one key but `MessageChannel` uses another, replies look like no-ops or route misses.

## Listener/runtime gotchas

- Running listeners reload `pairing.yaml` and `routing.yaml` on the next inbound miss; restart only when adapter/account config itself changed.
- Listener/server mode differs from TUI for some hooks and secrets. Do not assume TUI-only initialization runs in listener mode.
- Remote/self-hosted channel listener behavior may skip Cloud environment registration depending on `LETTA_BASE_URL`.
