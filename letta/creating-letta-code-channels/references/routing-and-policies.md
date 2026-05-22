# Channel routing and policies

## Table of contents

- DM policies
- Pairing flow
- Auto-routing flow
- Slack patterns
- Discord patterns
- Allowed channel gating
- Open mode safety
- Thread hydration

## DM policies

Shared `dmPolicy` values:

- `pairing`: unknown users receive a pairing code. Operator approves via Desktop UI or CLI.
- `allowlist`: only configured sender IDs can message.
- `open`: anyone who can reach the bot can route, subject to account/channel behavior.

Do not assume `dmPolicy` applies to public/guild/channel messages. Slack and Discord use additional channel-specific rules for guild/channel traffic.

## Pairing flow

Generic pairing flow:

1. Unknown sender sends message.
2. Registry creates pending pairing code.
3. Adapter sends pairing instructions.
4. Operator runs `letta channels pair --channel <id> --code <code> --agent <agent-id> --conversation <conversation-id>` or uses Desktop UI for first-party channels.
5. Route is written to `routing.yaml`.
6. Next message uses the route.

User plugins are CLI-first. First-party channels may have Desktop UI.

## Auto-routing flow

Slack and Discord can bind an entire account/app/bot to an agent. Registry creates conversations automatically for DMs or channel interactions.

Risks:

- Missing `agentId` should not spam public channels.
- Account-bound routes need stable summaries for new conversations.
- Existing routes should continue even when the initial trigger condition is absent.

## Slack patterns

Slack channel behavior is `app_mention`-triggered by default. Replies are threaded using `thread_ts`. Slack has inbound debounce support and thread context hydration.

When changing Slack routing, check:

- `thread_ts` preservation.
- `defaultPermissionMode` defaults.
- Socket Mode event ack behavior.
- Debounce key: account + channel + thread + sender.

## Discord patterns

Discord has two separate surfaces:

- DMs: gated by `dmPolicy`.
- Guild channels: gated by `channelPolicy`, mentions, threads, allowed channels, and account binding.

Default Discord UX:

1. Ignore normal guild channel chatter.
2. User @mentions bot.
3. Bot strips its mention and auto-creates a thread by default.
4. Agent replies in that thread.
5. Follow-up messages inside that thread route without more mentions.

Important knobs:

- `allowedChannels`: parent channel allowlist. Threads match on `parentId` when available.
- `channelPolicy: "mention" | "open"`: whether non-mentioned guild messages are processed.
- `autoThreadOnMention`: whether a new thread is created for top-level mentions.
- `inboundDebounceMs`: optional trailing-edge message stacking.

`autoThreadOnMention: false` with `channelPolicy: "mention"` keeps the first response inline, but follow-up parent-channel messages still need another @mention. That is expected unless route-aware adapter gating is implemented.

## Allowed channel gating

Apply allowed-channel gates before route creation and before expensive work like media downloads. For threads, compare the parent channel id when possible; fall back to the thread id only if parent is unavailable.

Empty or undefined allowlist should preserve old behavior.

## Open mode safety

Open/public modes can be noisy. Guard these cases:

- Unbound open-mode bots should not reply to every ambient channel message.
- Public channels should not post internal control prompts.
- `open` should usually be paired with an allowed channel allowlist.
- Tests should cover unbound ambient traffic and explicit mentions separately.

## Thread hydration

Slack and Discord can hydrate first-route context for existing threads. This is context hydration, not replay:

- Prior messages appear in `<thread-context>` / `<thread-history>`.
- They are not delivered as separate turns.
- Limits are bounded and usually text-only.
- Hydration usually runs only on first route turn and only when `threadId` is present.
