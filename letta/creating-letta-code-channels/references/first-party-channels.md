# First-party Letta Code channels

## Table of contents

- When to use first-party
- New channel file cascade
- Per-channel field cascade
- Codec and protocol rules
- Desktop UI cascade
- Existing channel templates
- Safety checks

## When to use first-party

Use first-party channel work when the channel needs Desktop UI, rich typed snapshots, compatibility shims, SDK/runtime dependency install, special routing, or custom media/reaction behavior.

Use dynamic plugins instead for headless/community integrations where generic account config and generic pairing/routes are enough.

## New channel file cascade

Adding a first-party channel usually touches:

- `src/channels/types.ts` — channel id union, account/config types, type guards.
- `src/channels/plugin-registry.ts` — metadata and dynamic import.
- `src/channels/<channel>/plugin.ts` — `ChannelPlugin` export.
- `src/channels/<channel>/adapter.ts` — start/stop/inbound/send behavior.
- `src/channels/<channel>/message-actions.ts` — `MessageChannel` action surface.
- `src/channels/<channel>/setup.ts` — CLI setup wizard if needed.
- `src/channels/<channel>/runtime.ts` — runtime dependency helper if needed.
- `src/channels/<channel>/account-config.ts` — plugin-owned config codec for websocket/desktop config payloads.
- `src/channels/accounts.ts` — clone/default/legacy handling.
- `src/channels/config.ts` — legacy config codec if legacy config exists.
- `src/channels/service.ts` — snapshots, create/update, config APIs.
- `src/websocket/listener/client.ts` — wire-to-service mappings if protocol fields are top-level.
- `src/websocket/listener/protocol-inbound.ts` — validators for accepted protocol payloads.
- `src/types/protocol_v2.ts` — wire protocol account snapshots and create/update payloads if typed there.
- `src/channels/registry.ts` — only if routing differs from generic pairing/route flow.
- `src/channels/xml.ts` — channel-specific message formatting hints when needed.
- `src/channels/*.test.ts` and `src/websocket/*.test.ts` — targeted coverage.

## Per-channel field cascade

Adding a per-channel field, such as Discord `allowedChannels`, `channelPolicy`, `autoThreadOnMention`, or `transcribeVoice`, usually touches:

1. `src/channels/types.ts`.
   - Add field to `<Channel>ChannelAccount`.
   - Add field to `<Channel>ChannelConfig` if legacy/config snapshots need it.
   - Add comments documenting defaults and whether DMs are affected.
2. `src/channels/<channel>/account-config.ts`.
   - Accept snake_case config field.
   - Validate type strictly.
   - Map to camelCase account patch.
   - Serialize effective defaults in `toAccountConfig` and `toConfigSnapshotConfig`.
3. `src/channels/plugin-types.ts`.
   - Add to `ChannelPluginAccountPatch` if service create/update uses the generic patch type.
4. `src/channels/service.ts`.
   - Add to `ChannelConfigSnapshot` / `ChannelAccountSnapshot` if UI or protocol reads top-level field.
   - Add to `toAccountSnapshot` and `getChannelConfigSnapshot`.
   - Add to `createAccountFromPatch` and `mergeAccountPatch`.
   - Add to `setChannelConfigLive` existing/create paths.
   - Preserve explicit `false` with `??`, not `||`.
5. `src/channels/accounts.ts`.
   - Deep-copy arrays/objects in `cloneAccount`.
   - Add legacy/default migration when applicable.
6. `src/channels/config.ts`.
   - Parse legacy snake_case config fields when applicable.
7. Protocol/listener files.
   - For first-party typed payloads, update `src/types/protocol_v2.ts`, `src/websocket/listener/client.ts`, and `src/websocket/listener/protocol-inbound.ts`.
   - For nested plugin config payloads, account config adapters may handle validation, but still add protocol tests.
8. Adapter behavior.
   - Actually consume the field.
   - Keep default behavior identical for accounts missing the new field.
9. Tests.
   - Codec tests.
   - Service snapshot/create/update tests.
   - Protocol validator/round-trip tests.
   - Adapter or registry behavior tests.

## Codec and protocol rules

- Wire/config fields are snake_case (`allowed_channels`, `transcribe_voice`).
- In-memory TypeScript fields are camelCase (`allowedChannels`, `transcribeVoice`).
- Strictly reject unknown first-party config keys unless the channel intentionally accepts arbitrary plugin config.
- Serialize effective defaults in snapshots. Do not let UI-facing top-level fields drift as `undefined` when `config` says the default.
- Partial updates must preserve unspecified fields. `false` and empty arrays can be meaningful.

## Desktop UI cascade

Desktop channel UI lives in `letta-cloud`, not `letta-code`. For channel account field UI:

- `apps/code-desktop/ui/src/hooks/useChannels.ts` has channel snapshot unions and create/update payload unions.
- Channel manage dialogs live under `apps/code-desktop/ui/src/app/pages/Channels/_components/{Channel}ManageDialog/`.
- Hook companions live under `_hooks/use{Channel}ManageState.ts`.
- Translations live at `apps/code-desktop/ui/src/translations/en.json` under `channels.{channel}SetupDialog`.
- Multiline fields should use `RawTextArea` from `@letta-cloud/ui-component-library`.
- Typecheck with `pnpm exec nx run code-desktop:type-check`.

A `letta-code` backend field without desktop UI is acceptable for power-user JSON knobs, but document that explicitly.

## Existing channel templates

- Telegram: simplest first-party adapter, pairing-first, voice transcription, simple `messageActions`.
- Slack: Socket Mode, account-bound auto-routing, debouncer, thread hydration, default permission mode.
- Discord: gateway client, DMs + guild/thread routing, allowed channel gating, reactions, media downloads, auto-threading, delivery error replies.

## Safety checks

- Public channels should not expose approval/control prompts to arbitrary users.
- `open` policies should not create noisy unbound replies in every visible channel.
- Mention gates should not block already-routed conversations unless explicit.
- Gating should apply before expensive media downloads.
- Attachments/reactions should not bypass allowed-channel or sender policy.
