---
name: pyannoteai-stream-live-diarization
description: Open a pyannoteAI live stream and send audio over its WebSocket to receive real-time speaker diarization.
api: openapi/pyannoteai-streaming-api-openapi.yml
operations: [createStream, getStream]
generated: '2026-09-16'
method: generated
source: openapi/pyannoteai-streaming-api-openapi.yml, asyncapi/pyannoteai-streaming-asyncapi.yml, https://docs.pyannote.ai/tutorials/streaming-real-time
---

# Stream live audio for real-time diarization

Base URL `https://api.pyannote.ai`, `Authorization: Bearer <API key>`.

## Steps

1. **Create the stream.** `createStream` (`POST /v1/live`). The `StreamCreated` response returns the stream `id` and the WebSocket `url`.
2. **Stream audio.** Connect to the returned WebSocket `url` and send audio / receive diarization messages as described in the provider's AsyncAPI document (asyncapi/pyannoteai-streaming-asyncapi.yml, published at https://docs.pyannote.ai/asyncapi.yaml).
3. **Check status.** `getStream` (`GET /v1/live/{id}`) returns `status`, `startedAt` and `completedAt` for the stream.

## Rules

- `createStream` counts against the 100 requests/min per-team submission budget; back off on `429`.
- There is no idempotency key; a retried `createStream` opens another stream.
- `402` means a subscription/credit is required; `400` returns `ValidationErrorResponse`.
