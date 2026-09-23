---
name: Pyannote
description: Use when building speaker diarization, speaker identification, or real-time voice analysis features. Reach for this skill when you need to identify who spoke when in audio, create voiceprints for speaker recognition, transcribe audio with speaker attribution, or stream live audio for real-time speaker detection.
metadata:
    mintlify-proj: pyannote
    version: "1.0"
---

# pyannote.ai Skill

## Product summary

pyannote.ai is an AI platform for speaker diarization and identification. It answers "who spoke when?" by separating multi-speaker audio into speaker segments with generic labels (SPEAKER_00, SPEAKER_01, etc.), and "who is speaking?" by matching speakers to known voiceprints. The API supports batch diarization jobs, real-time streaming diarization over WebSocket, speaker identification with voiceprints, and speaker-attributed transcription (STT orchestration).

**Key endpoints:**
- `POST /v1/diarize` — Submit audio for speaker diarization
- `POST /v1/live` — Create a WebSocket stream for real-time diarization
- `POST /v1/voiceprint` — Create a voiceprint from single-speaker audio
- `POST /v1/identify` — Match speakers in audio against known voiceprints
- `GET /v1/jobs/{jobId}` — Poll job status and retrieve results

**Primary docs:** https://docs.pyannote.ai

**Authentication:** All requests use Bearer token authentication with an API key from the dashboard at https://dashboard.pyannote.ai.

---

## When to use

Use pyannote.ai when:

- **Diarizing audio files**: You need to identify speaker segments and timestamps in recorded audio (meetings, calls, interviews, podcasts).
- **Real-time speaker tracking**: You're building live applications that need speaker labels as audio streams in (contact centers, live captioning, meeting assistants).
- **Speaker identification**: You have known speakers (enrolled via voiceprints) and need to recognize them in new audio.
- **Speaker-attributed transcription**: You need both who spoke and what they said, with word-level or turn-level timestamps.
- **Overlapping speech detection**: You need to detect and attribute speech when multiple speakers talk simultaneously.

Do not use pyannote.ai for:
- General speech-to-text without speaker attribution (use a dedicated STT service).
- Audio classification or emotion detection (outside scope).
- Voiceprint storage or management (you must store voiceprints in your own database).

---

## Quick reference

### Models

| Model | Use case | Key features |
|-------|----------|--------------|
| **precision-2** | Batch diarization, identification, transcription | 28% more accurate than Community-1; supports voiceprints, exclusive diarization, confidence scores |
| **live-1** | Real-time streaming diarization | Sub-300ms latency; up to 8 speakers; 100ms chunks over WebSocket |
| **community-1** | Prototyping, low-volume workloads | Open-source; lower accuracy; no voiceprints or identification |

### Diarization request parameters

```json
{
  "url": "https://example.com/audio.wav",
  "model": "precision-2",
  "numSpeakers": 2,
  "minSpeakers": 1,
  "maxSpeakers": 5,
  "confidence": false,
  "exclusive": false,
  "transcription": false,
  "webhook": "https://your-server.com/webhook"
}
```

| Parameter | Type | Purpose |
|-----------|------|---------|
| `url` | string | Direct, publicly accessible audio URL (required) |
| `model` | enum | `precision-2` (default) or `community-1` |
| `numSpeakers` | number | Exact speaker count (overrides min/max) |
| `minSpeakers` / `maxSpeakers` | number | Range for auto-detection |
| `confidence` | boolean | Include confidence scores (larger output) |
| `exclusive` | boolean | Diarization without overlapping speech |
| `transcription` | boolean | Enable STT orchestration (precision-2 only) |
| `webhook` | string | HTTPS URL for async job completion notification |

### Streaming audio format

| Property | Value |
|----------|-------|
| Format | PCM float 32-bit little-endian (`pcm_f32le`) |
| Sample rate | 16 kHz |
| Channels | Mono |
| Chunk duration | 100 ms (1600 samples) |
| Max buffer | 5 seconds (enforce real-time pace) |

### Job status values

- `pending` — Job queued
- `created` — Job created, not yet running
- `running` — Job in progress
- `succeeded` — Job completed successfully
- `failed` — Job failed
- `canceled` — Job canceled

### Limits

| Limit | Value |
|-------|-------|
| Max audio duration (diarization/identification) | 24 hours |
| Max audio duration (voiceprint) | 30 seconds |
| Max file size (diarization/identification) | 1 GiB |
| Max file size (voiceprint) | 100 MiB |
| Max voiceprints per identification job | 50 |
| Max concurrent streams per team | 10 |
| Max stream duration | 5 hours |
| Rate limit (job submission) | 100 requests/minute |
| Rate limit (job polling) | 300 requests/minute |
| Job output retention | 24 hours after completion |
| Uploaded file retention | 48 hours |

---

## Decision guidance

### When to use batch diarization vs. streaming

| Scenario | Use batch (`/v1/diarize`) | Use streaming (`/v1/live`) |
|----------|---------------------------|---------------------------|
| **Recorded audio** | ✓ | — |
| **Live/real-time audio** | — | ✓ |
| **Need results before recording ends** | — | ✓ |
| **Sub-300ms latency required** | — | ✓ |
| **Full conversation available** | ✓ | — |
| **Transcription needed** | ✓ (precision-2 only) | — |

### When to use polling vs. webhooks

| Scenario | Use polling | Use webhooks |
|----------|------------|--------------|
| **Testing/prototyping** | ✓ | — |
| **Production systems** | — | ✓ |
| **Avoid rate limiting** | — | ✓ |
| **Simple scripts** | ✓ | — |
| **Async job handling** | — | ✓ |

### When to use diarization vs. identification

| Need | Use diarization | Use identification |
|------|-----------------|-------------------|
| **Separate speakers into segments** | ✓ | ✓ (includes diarization) |
| **Generic speaker labels (SPEAKER_00)** | ✓ | — |
| **Recognize specific known speakers** | — | ✓ |
| **Match against voiceprints** | — | ✓ |

### When to enable transcription

| Scenario | Enable `transcription: true` | Use separate STT |
|----------|------------------------------|------------------|
| **Need speaker-attributed text** | ✓ | — |
| **Want word-level timestamps** | ✓ | — |
| **Using precision-2 model** | ✓ | — |
| **Using community-1 model** | — | ✓ |
| **Already have transcripts** | — | ✓ (merge with diarization) |

---

## Workflow

### Batch diarization workflow

1. **Prepare audio**: Ensure audio is in a supported format (mp3, wav, m4a, ogg, flac, etc.) and accessible via a direct, publicly accessible URL. For private files, use a presigned URL (e.g., AWS S3 with 1-hour expiry).

2. **Create diarization job**: POST to `/v1/diarize` with the audio URL and optional parameters (model, speaker count, transcription, webhook).

3. **Track job**: Either poll `/v1/jobs/{jobId}` every 10 seconds, or specify a webhook URL to receive async notification when the job completes.

4. **Retrieve results**: When status is `succeeded`, extract the `output` field containing diarization segments (speaker, start, end timestamps). If transcription was enabled, also extract `wordLevelTranscription` and `turnLevelTranscription`.

5. **Store results**: Job output is deleted after 24 hours. Save diarization and transcription results to your own database immediately.

6. **Process output**: Parse segments to calculate speaker statistics, detect overlaps, format transcripts, or feed into downstream systems.

### Real-time streaming workflow

1. **Create stream session**: POST to `/v1/live` with your API key. Receive a single-use WebSocket URL.

2. **Connect WebSocket**: Open a WebSocket connection to the returned URL. Wait for the `open` event before sending audio.

3. **Stream audio**: Send 100ms chunks of PCM float32 mono 16kHz audio every 100ms. Do not rush ahead of real-time (max 5-second buffer).

4. **Receive events**: Listen for `diarization_speaker_start` and `diarization_speaker_end` JSON events with timestamps and speaker labels.

5. **Signal end**: Send `{"type": "end_of_stream"}` when done. The server will close the connection with code 1000.

6. **Handle errors**: Listen for `error` events if chunk format or size is invalid.

### Speaker identification workflow

1. **Create voiceprints**: For each known speaker, POST single-speaker audio (≤30 seconds) to `/v1/voiceprint`. Poll or use webhook to retrieve the voiceprint from the job output.

2. **Store voiceprints**: Save voiceprints to your database (they are deleted 24 hours after job completion).

3. **Identify speakers**: POST to `/v1/identify` with the audio URL and an array of voiceprints (label + voiceprint data). Optionally set `matching.threshold` (0-100) and `matching.exclusive` (true/false).

4. **Retrieve results**: Poll or use webhook. The output includes both `diarization` (generic speaker segments) and `identification` (matched speaker names with confidence scores).

5. **Interpret confidence**: Check confidence scores to decide whether to accept or reject matches. Higher threshold values (50-70) are stricter.

---

## Common gotchas

- **Audio URL must be direct and public**: URLs that require authentication, redirect multiple times, or expire will fail with "Could not load audio". Test the URL in an incognito browser window. Use presigned URLs with limited validity (e.g., 1 hour) for private cloud storage.

- **Job output expires after 24 hours**: Diarization, identification, voiceprints, and transcription results are automatically deleted. Retrieve and store them in your own database immediately upon job completion.

- **Voiceprints are not stored by pyannote**: You must manage voiceprint storage. Each voiceprint job output is deleted after 24 hours. Retrieve the voiceprint and save it to your database.

- **Streaming audio must be real-time pace**: Sending audio faster than real-time (rushing ahead) will cause the WebSocket to close. Enforce 100ms chunks at real-time pace.

- **Streaming audio format is raw PCM, not WAV**: Do not send WAV/RIFF headers. Send only raw PCM float32 bytes. Convert from int32 or other formats before sending.

- **Transcription only works with precision-2**: The `transcription: true` flag is ignored for community-1 model. Use precision-2 or a separate STT service.

- **Identification cannot use transcription**: You cannot enable both `transcription: true` and voiceprints in the same request. Use diarization + transcription, or identification without transcription.

- **Rate limits are per endpoint**: Polling `/v1/jobs/{jobId}` has a 300 req/min limit; submitting jobs has a 100 req/min limit. Excessive polling will trigger 429 errors. Use webhooks in production.

- **Confidence scores require explicit flag**: Set `confidence: true` to include confidence values in diarization output. This increases output size significantly.

- **Overlapping speech is included by default**: If you need exclusive diarization (one speaker at a time), set `exclusive: true`. This removes overlapping segments.

- **No speech detected is a job failure**: If audio contains no speech, the job will fail with status `failed`. Check the error message in the job output.

- **Voiceprint requires single speaker**: Audio for voiceprint creation must contain only one speaker with no overlaps. Multi-speaker audio will produce invalid voiceprints.

---

## Verification checklist

Before submitting diarization or identification work:

- [ ] Audio URL is a direct link (ends with .wav, .mp3, etc.) and publicly accessible
- [ ] Audio URL does not require authentication or redirect more than twice
- [ ] Audio file is within size limits (1 GiB for diarization/identification, 100 MiB for voiceprints)
- [ ] Audio duration is within limits (24 hours for diarization/identification, 30 seconds for voiceprints)
- [ ] API key is valid and has sufficient credits/subscription
- [ ] Webhook URL (if used) is HTTPS and responds with 2xx status
- [ ] Job output has been retrieved and stored in your database (within 24 hours)
- [ ] For identification: voiceprints are stored in your database before the 24-hour window expires
- [ ] For streaming: WebSocket connection is fully open before sending audio
- [ ] For streaming: audio is sent at real-time pace (100ms chunks every 100ms)
- [ ] For streaming: audio format is PCM float32 mono 16kHz (no WAV headers)
- [ ] For transcription: using precision-2 model (not community-1)
- [ ] For transcription: not combining with identification in the same request

---

## Resources

**Comprehensive page listing:** https://docs.pyannote.ai/llms.txt

**Critical documentation pages:**
- [API Reference: Diarize](https://docs.pyannote.ai/api-reference/diarize) — Full diarization endpoint documentation with all parameters
- [How to Diarize Audio](https://docs.pyannote.ai/tutorials/how-to-diarize-audio) — Complete batch diarization workflow with polling and webhook examples
- [Streaming Diarization](https://docs.pyannote.ai/tutorials/streaming-real-time) — Real-time WebSocket streaming with code examples
- [Speaker Identification with Voiceprints](https://docs.pyannote.ai/tutorials/identification-with-voiceprints) — Voiceprint creation and identification workflow
- [Data Retention](https://docs.pyannote.ai/data-retention) — Job output and file retention policies
- [Troubleshooting](https://docs.pyannote.ai/support/troubleshooting) — Common errors and solutions

---

> For additional documentation and navigation, see: https://docs.pyannote.ai/llms.txt