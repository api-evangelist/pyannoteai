---
name: pyannoteai-identify-speakers
description: Enroll known speakers as voiceprints with pyannoteAI, then diarize a recording and match its speakers to those identities.
api: openapi/pyannoteai-api-api-openapi.yml
operations: [voiceprint, getJobById, identify]
generated: '2026-09-16'
method: generated
source: openapi/pyannoteai-api-api-openapi.yml, conventions/pyannoteai-conventions.yml, https://docs.pyannote.ai/tutorials/identification-with-voiceprints
---

# Identify speakers with voiceprints

Base URL `https://api.pyannote.ai`, `Authorization: Bearer <API key>`.

## Steps

1. **Enroll each known speaker.** `voiceprint` (`POST /v1/voiceprint`) with `url` pointing at single-speaker audio (up to 30 seconds). Keep the returned `jobId`.
2. **Collect the voiceprints.** Poll `getJobById` (`GET /v1/jobs/{jobId}`) or use a webhook; on `succeeded`, `output.voiceprint` is the voiceprint string. Store it yourself with a label — voiceprints are not addressable server-side resources.
3. **Run identification.** `identify` (`POST /v1/identify`) with the recording `url` and `voiceprints: [{label, voiceprint}, ...]`; optionally `matching` (`threshold`, `exclusive`) plus the same speaker-count and webhook options as diarization.
4. **Read the match.** Poll `getJobById` for the identify job; on `succeeded`, `output` holds the diarization plus identification segments and per-speaker `match`/`confidence`.

## Rules

- Each `voiceprint` and `identify` call is a separate billable job and has no idempotency key; do not blindly retry submissions.
- 100 submissions/min and 300 job reads/min per team, per endpoint; back off on `429`. `402` means credit/subscription is required.
- Validation failures return `400` with `errors[]` naming the offending field.
