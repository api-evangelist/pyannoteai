---
name: pyannoteai-diarize-audio
description: Submit an audio file to pyannoteAI for speaker diarization (who spoke when), optionally with speaker-attributed transcription, and collect the result by polling or webhook.
api: openapi/pyannoteai-api-api-openapi.yml
operations: [getMediaUploadURL, diarize, getJobById]
generated: '2026-09-16'
method: generated
source: openapi/pyannoteai-api-api-openapi.yml, openapi/pyannoteai-media-api-openapi.yml, conventions/pyannoteai-conventions.yml, rate-limits/pyannoteai-rate-limits.yml, https://docs.pyannote.ai/tutorials/how-to-diarize-audio
---

# Diarize an audio file

Base URL `https://api.pyannote.ai`. Every request sends `Authorization: Bearer <API key>` (keys come from https://dashboard.pyannote.ai). Confirm the key first with `testKey` (`GET /v1/test`) if unsure.

## Steps

1. **Make the audio reachable.** `url` must be a publicly accessible (or presigned) audio URL. If the file is local, call `getMediaUploadURL` (`POST /v1/media/input`) with a `media://` url you choose, PUT the file to the returned presigned `url`, then use the `media://` url in step 2.
2. **Submit the job.** `diarize` (`POST /v1/diarize`) with `url` and optional `model`, `numSpeakers` or `minSpeakers`/`maxSpeakers`, `exclusive`, `confidence`, `transcription` + `transcriptionConfig`, and `webhook` (HTTPS) / `webhookStatusOnly`. The response is `JobCreated` with `jobId`.
3. **Wait for completion.** Either receive the webhook (verify `X-Signature` HMAC-SHA256 with `X-Request-Timestamp`, see asyncapi/pyannoteai-webhooks.yml) or poll `getJobById` (`GET /v1/jobs/{jobId}`) — around every 10 seconds, not faster.
4. **Read the output.** When `status` is `succeeded`, read `output` (diarization segments with speaker, start, end; transcription fields when requested). `failed` and `canceled` are also terminal.

## Rules

- **No idempotency.** There is no Idempotency-Key. A retried `diarize` call creates a second billable job — only retry a submission when you know the first did not create a job.
- **Rate limits** are per team, per endpoint, 60-second window: 100/min for submissions, 300/min for job reads. On `429`, back off before retrying.
- **Errors**: `400` returns `ValidationErrorResponse` (`message`, `errors[]` of `field`/`message`); `402` means a subscription/credit is required; `429` is rate limiting. `ApiError` carries a `requestId` to quote to support.
- Job outputs are deleted 24 hours after completion — persist what you need.
