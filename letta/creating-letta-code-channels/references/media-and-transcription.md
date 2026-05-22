# Channel media and transcription

## Table of contents

- Attachment model
- Download rules
- Images and vision
- Audio transcription
- Telegram reference
- Discord reference
- Tests

## Attachment model

Inbound attachments use `ChannelMessageAttachment` in `src/channels/types.ts`:

```ts
{
  id: string;
  name?: string;
  mimeType?: string;
  sizeBytes?: number;
  kind: "image" | "file" | "audio" | "video";
  localPath?: string;
  imageDataBase64?: string;
  transcription?: string;
}
```

XML formatting in `src/channels/xml.ts` includes attachment metadata and transcription when present.

## Download rules

Adapters should:

- Enforce platform/file size limits before and after download.
- Use timeouts/abort controllers.
- Sanitize local filenames.
- Store in temp directories, not repo paths.
- Continue processing other attachments if one download fails.
- Avoid downloading after a message fails gating/allowlist checks.

## Images and vision

Images should include `imageDataBase64` when small enough and supported. Telegram and Discord both inline image bytes for vision paths. Keep existing message content types from the API; do not mutate API message types.

## Audio transcription

Transcription uses `src/channels/transcription/index.ts` and OpenAI Whisper:

- Requires `OPENAI_API_KEY`.
- Uses `transcribeAudioFile(localPath)`.
- Never throws; returns `{ success, text?, error? }`.
- Should be opt-in per account via `transcribeVoice` / `transcribe_voice`.

Do not call Whisper unless the account opt-in is enabled and `isTranscriptionConfigured()` is true.

## Telegram reference

Telegram has a distinct `voice` payload and `audio` payload. Existing behavior transcribes **voice memos only** when `transcribeVoice` is enabled. See:

- `src/channels/telegram/media.ts`.
- `src/channels/telegram/account-config.ts`.
- `src/channels/telegram-adapter.test.ts`.
- `src/channels/transcription.test.ts`.

## Discord reference

Discord voice messages arrive as ordinary audio attachments. There is no separate Telegram-style `voice` object. If supporting Discord transcription, apply opt-in transcription to inbound `audio/*` attachments.

Wire the field through:

- `DiscordChannelAccount` / `DiscordChannelConfig`.
- `discord/account-config.ts` as `transcribe_voice`.
- `config.ts` and `accounts.ts` for legacy/default parsing if needed.
- `service.ts` snapshots and create/update merging.
- Adapter attachment resolver params.
- Media tests mocking both CDN download and Whisper endpoint.

## Tests

Useful tests:

```bash
bun test src/channels/transcription.test.ts
bun test src/channels/telegram-adapter.test.ts
bun test src/channels/discord-media.test.ts
bun test src/channels/discord-channel-gating.test.ts
bun test src/channels/discord-service.test.ts
bun test src/websocket/listen-client-channel-accounts.test.ts
```

Mock `globalThis.fetch` for both platform download URL and `https://api.openai.com/v1/audio/transcriptions`. Restore `OPENAI_API_KEY` and `fetch` in `afterEach`.
