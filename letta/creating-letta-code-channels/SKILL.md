---
name: creating-letta-code-channels
description: Builds, reviews, debugs, and tests Letta Code channels, including first-party adapters (Telegram, Slack, Discord, Custom) and dynamic user channel plugins under ~/.letta/channels. Use when adding channel support, changing account fields, channel routing, pairing, MessageChannel actions, media/transcription, runtime dependencies, listener behavior, desktop channel UI, or channel PR review.
license: MIT
---

# Creating Letta Code channels

Use this for work in a `letta-code` checkout involving channels: first-party adapters, dynamic user plugins, routing/pairing, `MessageChannel`, media attachments, channel account config, websocket protocol, and channel tests.

## First move

1. Work in the target `letta-code` checkout or a worktree. If reviewing a PR, fetch the PR branch and inspect the actual diff against current `origin/main`.
2. Identify the channel shape:
   - **Dynamic user plugin** for headless/community experiments under `~/.letta/channels/<id>/`.
   - **First-party channel** under `src/channels/<id>/` when it needs Desktop UI, rich typed snapshots, protocol fields, bespoke routing, runtime install support, or migration compatibility.
3. Check whether the work changes one of these high-risk surfaces:
   - Account/config fields.
   - Routing semantics and route keys.
   - `MessageChannel` outbound actions.
   - Public-channel safety and control requests.
   - Media download/transcription/image handling.
   - Runtime dependency resolution.
4. Read only the reference files needed for the surface:
   - `references/architecture.md` — core channel architecture, route flow, account stores, registry behavior.
   - `references/first-party-channels.md` — first-party file cascade and exact field-addition checklist.
   - `references/user-plugins.md` — dynamic plugin manifest/account/runtime/headless flow.
   - `references/message-actions.md` — `MessageChannel` wiring and common no-op failure mode.
   - `references/routing-and-policies.md` — DM policy, account-bound routing, Slack/Discord threading, allowed channels.
   - `references/media-and-transcription.md` — attachments, image base64, audio transcription, size limits.
   - `references/testing.md` — targeted tests, smoke tests, debug symptoms.

## Non-negotiables

- Always implement `messageActions` for any reply-capable channel. Without it, `MessageChannel` returns a string like `Channel "X" does not expose MessageChannel actions.` and agents often mistake that for success.
- Never modify API message types. Use the protocol shapes as-is and add typed fields deliberately through the full cascade.
- For public channels, do not post tool approval/control prompts publicly unless there is verified operator routing. A default `sendDirectReply` approval prompt can leak tool names, args, cwd, and approval instructions.
- Adding a per-channel field is not a one-file change. Follow the cascade in `references/first-party-channels.md`.
- Test the four legs of every channel: plugin discovery/import, inbound route/pairing, channel notification to the agent, outbound `MessageChannel` to the platform.
- Use `bun:test`; there is no package `test` script. Run targeted tests first, then `bun run typecheck`, `bun run lint`, and usually `bun run build` before finalizing.

## Channel type decision

Use a **dynamic user plugin** when:
- The channel is experimental, community-owned, or headless.
- Desktop UI is not required.
- Account state can live in `account.config`.
- You can tolerate generic pairing/routes and CLI-first setup.

Use a **first-party channel** when:
- It needs Desktop setup/manage UI or typed top-level snapshots.
- It needs special routing like Slack/Discord auto-routing or thread hydration.
- It needs SDK/runtime dependency installation from the channel registry.
- It needs custom media handling, voice transcription, reactions, uploads, or platform-specific reply formatting.
- It must be supported as `telegram`, `slack`, `discord`, or another blessed channel id.

First-party ids cannot be shadowed by user plugins: `telegram`, `slack`, `discord`, and `custom` user plugin directories are ignored.

## Core workflow for implementation

1. Read the relevant existing channel as the template:
   - Telegram: simplest first-party channel, pairing-first, useful `messageActions` reference.
   - Slack: account-bound routing, Socket Mode, debouncing, default permission mode.
   - Discord: account-bound routing, guild/thread behavior, allowed channel gating, media download.
2. Update types and account config before adapter behavior.
3. Wire behavior in the adapter and registry.
4. Wire `MessageChannel` action support.
5. Add tests close to the behavior:
   - Pure helper tests for gates, formatters, debounce keys, codecs.
   - Service/protocol tests for account fields.
   - Adapter tests for inbound/outbound behavior where mocking is practical.
   - Registry tests for route creation/pairing behavior.
6. Run targeted tests, typecheck, lint, and build.
7. If pushing to someone else's PR branch with maintainer permissions, add a normal commit. Do not force-push or squash their branch unless explicitly asked.

## Review workflow

When reviewing a channel PR:

1. Fetch the PR branch and diff against fresh `origin/main`, not stale local `main`.
2. Inspect code paths, not just PR prose. PR bodies frequently omit cascade gaps.
3. Check for these regressions:
   - Existing accounts without new optional fields still behave the same.
   - Defaulted fields are effective in snapshots, not `undefined` in UI-facing top-level fields.
   - `false` and empty arrays survive partial updates where intended.
   - Adapter gates do not prevent existing routed conversations from continuing unless that is explicit.
   - Public/open modes cannot spam every visible channel when unbound.
   - Reactions, attachments, and debounced messages do not bypass route/account safety.
   - `MessageChannel` has `messageActions` and route lookup uses the same `chatId`/`threadId` keys emitted inbound.
4. Run the new targeted tests. If touching TypeScript channel code, run `bun run typecheck` and `bun run lint`.
5. In GitHub comments, follow the repository or user-requested comment footer convention. Do not add AI attribution unless explicitly required.

## Scaffold helper for user plugins

Use the bundled scaffold for a minimal dynamic plugin skeleton:

```bash
npx tsx <SKILL_DIR>/scripts/scaffold-user-channel-plugin.ts \
  my-channel "My Channel" \
  --runtime-package some-sdk@1.0.0 \
  --runtime-module some-sdk
```

It creates `channel.json`, `plugin.mjs`, and `accounts.example.json`. Replace TODO inbound/outbound logic with real SDK calls. Then test discovery, install, pairing, inbound delivery, and `MessageChannel` replies.
