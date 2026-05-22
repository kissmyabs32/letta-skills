# Channel testing checklist

## Table of contents

- Basic commands
- Targeted test map
- User plugin smoke test
- First-party smoke test
- Discord smoke cases
- Debugging symptoms
- PR verification comment

## Basic commands

From the relevant `letta-code` worktree:

```bash
bun install
letta channels status
letta channels install <channel>
letta server --channels <channel> --debug
```

Run targeted tests first, then broad checks:

```bash
bun test src/channels/<channel-or-area>.test.ts
bun run typecheck
bun run lint
bun run build
```

`letta-code` uses `bun:test`, not a package `test` script.

## Targeted test map

- Account/config codec: `src/tests/<channel>-account-config.test.ts`.
- Protocol validators: `src/channels/<channel>-protocol.test.ts` and `src/websocket/listen-client-channel-accounts.test.ts`.
- Service snapshots/create/update: `src/channels/<channel>-service.test.ts` or shared `service.test.ts`.
- Adapter helpers: `src/channels/<channel>-adapter.test.ts` or focused helper tests.
- Registry routing: `src/channels/<channel>-registry.test.ts` and shared `registry.test.ts`.
- MessageChannel: `src/channels/<channel>-message-channel.test.ts`, `message-channel.test.ts`, `message-tool-schema.test.ts`.
- Media/transcription: `src/channels/<channel>-media.test.ts`, `transcription.test.ts`.
- Runtime deps/plugins: `runtimeDeps.test.ts`, `plugin-registry.test.ts`.

## User plugin smoke test

1. Create `~/.letta/channels/<id>/channel.json`, `plugin.mjs`, and `accounts.json`.
2. Install runtime deps:

   ```bash
   letta channels install <id>
   ```

3. Start listener:

   ```bash
   letta server --channels <id> --debug
   ```

4. Confirm plugin-specific startup log appears. If import fails, check `runtime/node_modules` and the user plugin `node_modules` symlink.
5. Send a platform message.
6. If pairing is enabled, redeem the code:

   ```bash
   letta channels pair --channel <id> --code <code> --agent <agent-id> --conversation <conversation-id>
   ```

7. Send another platform message. The conversation should receive a `<channel-notification>`.
8. Reply with `MessageChannel` using the channel id and `chat_id` from the notification.

## First-party smoke test

For first-party channels:

1. Configure account with `letta channels configure <channel>` or edit account JSON if testing a power-user field.
2. Start `letta server --channels <channel> --debug`.
3. Trigger inbound from the platform.
4. Verify route/pairing state in `~/.letta/channels/<channel>/routing.yaml` or `pairing.yaml`.
5. Verify the agent receives `<channel-notification>` with correct text, sender, chat id, thread id, and attachment XML.
6. Reply through `MessageChannel`.
7. Verify platform reply location: DM, channel, thread, or reply target.

## Discord smoke cases

For Discord changes, cover:

- DM pairing default.
- DM `open` account-bound routing.
- DM `allowlist` rejection.
- Guild top-level @mention.
- Auto-thread on mention true and false.
- Existing thread follow-up without new mention.
- `allowedChannels` parent-channel gating.
- `channelPolicy: "open"` in an allowed test channel.
- Unbound `channelPolicy: "open"` ambient message should not spam.
- Reactions in tracked threads.
- Audio/image attachments if media changed.

## Debugging symptoms

- Plugin not discovered: id mismatch between directory and `channel.json`, invalid id chars, or id shadows first-party channel.
- Install says already installed but imports fail: runtime resolver counted dev `node_modules`; ensure runtime deps actually exist under `~/.letta/channels/<id>/runtime/node_modules` and symlink `~/.letta/channels/<id>/node_modules` to it.
- Inbound receives pairing code forever: pairing was not redeemed for the same `accountId`/`senderId`, or listener cannot reload `pairing.yaml`.
- Inbound reaches agent but replies no-op: missing `plugin.messageActions` or route lookup mismatch in `MessageChannel` args.
- Route lookup misses: wrong `chatId`, wrong `accountId`, or unstable `threadId` choice.
- Discord mentions work but follow-ups do not: adapter mention gate may be running before route-aware continuation, or follow-up is in parent channel with no thread and no mention.
- Cron lease error: usually unrelated to channel runtime; another Letta Code process owns scheduler lease.

## PR verification comment

When commenting on a GitHub PR, include concise verification and follow the requested project or user footer convention:

```markdown
Verification:
- `bun test ...` -> pass.
- `bun run typecheck` -> clean.
- `bun run lint` -> clean.
```

Do not include `Generated with Letta Code` or co-author attribution.
