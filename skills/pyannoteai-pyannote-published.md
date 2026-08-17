---
name: Pyannote
description: Use when building speaker diarization and identification features into voice applications, meeting transcription systems, call center analytics, live captioning, or any audio processing workflow that needs to determine who spoke when and identify specific speakers using voiceprints.
metadata:
    mintlify-proj: pyannote
    version: "1.0"
---

# pyannote.ai Skill

## Product summary

pyannote.ai is a REST API for speaker diarization and identification in audio files and live streams. It answers "who spoke when?" by separating multi-speaker audio into speaker segments with labels (SPEAKER_00, SPEAKER_01, etc.), and "who is speaking?" by matching those segments to known voiceprints with specific identities. Use the API at `https://api.pyannote.ai/v1/` with Bearer token authentication. Key endpoints: `/diarize` (batch processing), `/identify` (with voiceprints), `/voiceprint` (create speaker profiles), `/live` (streaming WebSocket). Authenticate with API keys from the dashboard at `https://dashboard.pyannote.ai`. See the primary docs at https://docs.pyannote.ai.

## When to use

Reach for this skill when:

- **Building speaker diarization**: You need to separate multi-speaker audio into speaker segments with timestamps (e.g., meeting transcription, call analytics, video dubbing).
- **Identifying known speakers**: You have pre-enrolled voiceprints and need to recognize specific people in audio (e.g., contact center agent identification, speaker verification).
- **Processing live audio**: You're building real-time applications that need speaker labels as audio streams in (e.g., live meeting assistants, broadcast monitoring, contact center agent assist).
- **Combining with transcription**: You're merging diarization results with speech-to-text output to create speaker-attributed transcripts.
- **Analyzing confidence**: You need confidence scores to assess reliability of speaker segments or implement human-in-the-loop correction workflows.

Do not use this skill for: audio format conversion, speech recognition (use transcription endpoints if needed), voice cloning, or speaker verification without voiceprints.

## Quick reference

### API endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/diarize` | POST | Submit audio for speaker diarization (batch) |
| `/v1/identify` | POST | Diarize + match against voiceprints |
| `/v1/voiceprint` | POST | Create a voiceprint from single-speaker audio |
| `/v1/live` | POST | Create WebSocket stream for real-time diarization |
| `/v1/jobs/{jobId}` | GET | Poll job status and retrieve results |
| `/v1/jobs` | GET | List all jobs for your team |
| `/v1/test` | GET | Test API connection |

### Authentication

All requests require Bearer token in `Authorization` header:

```bash
Authorization: Bearer YOUR_API_KEY
```

Generate API keys in the dashboard at https://dashboard.pyannote.ai. Store securely; never expose in client code or public repos.

### Models

| Model | Use case | Features |
|-------|----------|----------|
| `precision-2` (default) | Production diarization, voiceprints, identification | 28% more accurate than Community-1; supports voiceprints, exclusive mode, flexible speaker counts |
| `live-1` | Real-time streaming | Sub-300ms latency; up to 8 speakers; 5-hour max duration per stream |
| `community-1` | Prototyping, low-volume | Open-source; no voiceprint support; lower accuracy |

Specify model in request: `"model": "precision-2"` or `"model": "community-1"`.

### Key parameters

**Diarization:**
- `url` (required): Direct, publicly accessible audio URL
- `model`: Which model to use (default: `precision-2`)
- `num_speakers`: Expected speaker count (omit for auto-detection)
- `min_speakers`, `max_speakers`: Range for auto-detection
- `exclusive`: Enable exclusive mode (one speaker at a time, easier STT reconciliation)
- `confidence`: Include confidence scores (boolean)
- `webhook`: HTTPS URL for async result delivery
- `webhookStatusOnly`: Send only status, not full output (boolean)

**Identification:**
- `url` (required): Audio file URL
- `voiceprints`: Array of `{label, voiceprint}` objects (up to 50)
- `matching.threshold`: Minimum confidence for match (0-100, default 0)
- `matching.exclusive`: Prevent multiple speakers matching same voiceprint (default true)

**Voiceprint:**
- `url` (required): Single-speaker audio (max 30 seconds)

### Limits and constraints

| Constraint | Value |
|-----------|-------|
| Max audio duration (diarization/identify) | 24 hours |
| Max audio duration (voiceprint) | 30 seconds |
| Max file size (diarization/identify) | 1 GiB |
| Max file size (voiceprint) | 100 MiB |
| Max voiceprints per identify request | 50 |
| Max concurrent streams per team | 10 |
| Max stream duration | 5 hours |
| Idle timeout (streaming) | 5 seconds |
| Rate limit (job submission) | 100 requests/minute |
| Rate limit (job polling) | 300 requests/minute |
| Job result retention | 24 hours |

### Output format

Diarization output (array of segments):

```json
[
  {"speaker": "SPEAKER_00", "start": 10.0, "end": 15.0},
  {"speaker": "SPEAKER_01", "start": 12.5, "end": 14.0}
]
```

Identification output includes diarization + identification + voiceprint confidence:

```json
{
  "diarization": [...],
  "identification": [
    {"speaker": "John Doe", "start": 3.0, "end": 5.9, "diarizationSpeaker": "SPEAKER_00", "match": "John Doe"}
  ],
  "voiceprints": [
    {"speaker": "SPEAKER_00", "match": "John Doe", "confidence": {"John Doe": 86}}
  ]
}
```

## Decision guidance

### When to use diarization vs. identification

| Need | Use | Why |
|------|-----|-----|
| Separate speakers, don't know who they are | `/diarize` | Faster, cheaper; returns generic labels (SPEAKER_00, SPEAKER_01) |
| Identify specific known people | `/identify` + voiceprints | Matches segments to pre-enrolled speaker profiles |
| Both separation and names | `/identify` | Single request returns both diarization and identification |

### When to use batch vs. streaming

| Scenario | Use | Why |
|----------|-----|-----|
| Processing recorded audio files | `/diarize` (batch) | Simpler; results available after processing |
| Live audio, need labels in real-time | `/live` (streaming) | WebSocket; sub-300ms latency; speaker events as they happen |
| Hybrid: record + process later | `/diarize` (batch) | No need for streaming overhead |

### When to poll vs. webhook

| Scenario | Use | Why |
|----------|-----|-----|
| Quick test, single request | Polling | Simple; no server setup needed |
| Production, many concurrent jobs | Webhooks | Avoids rate limit issues; async; scales better |
| Large payloads (>1MB) | `webhookStatusOnly: true` | Reduces payload size; get status only |

### When to use exclusive mode

| Scenario | Use exclusive | Why |
|----------|---------------|-----|
| Merging with STT/ASR output | `"exclusive": true` | One speaker per time window; easier reconciliation |
| Analyzing overlapping speech | `"exclusive": false` (default) | Preserves overlaps; shows when speakers talk over each other |

## Workflow

### Batch diarization (most common)

1. **Prepare audio**: Ensure audio URL is direct, publicly accessible, and ends with file extension (`.wav`, `.mp3`, etc.). Use signed S3 URLs or pyannote's upload endpoint.

2. **Submit diarization job**: POST to `/v1/diarize` with audio URL and optional parameters (model, speaker count, confidence, webhook).

3. **Get job ID**: Response includes `jobId` and `status: "created"`. Save the job ID.

4. **Retrieve results**: Either poll `/v1/jobs/{jobId}` every 10 seconds until status is `succeeded`/`failed`, or set up webhook to receive results automatically. **Results auto-delete after 24 hours** — save to your database immediately.

5. **Parse output**: Extract speaker segments with timestamps. If `confidence: true`, use confidence scores to filter unreliable segments.

6. **Integrate downstream**: Pass diarization to transcription system, analytics pipeline, or UI for display.

### Speaker identification with voiceprints

1. **Create voiceprints** (one-time per speaker):
   - For each person, submit 5-30 seconds of clear, single-speaker audio to `/v1/voiceprint`
   - Poll or webhook to get voiceprint output
   - **Save voiceprints to your database** (API deletes after 24 hours)

2. **Identify speakers in new audio**:
   - POST to `/v1/identify` with audio URL and array of voiceprints (label + voiceprint data)
   - Include `matching.threshold` (50-70 recommended for strict matching)
   - Set `matching.exclusive: true` to prevent multiple speakers matching same voiceprint

3. **Retrieve results**: Poll or webhook. Output includes diarization + identification + confidence scores per voiceprint.

4. **Filter by confidence**: Use confidence scores to decide which matches are reliable. Set threshold in request or post-process results.

### Real-time streaming diarization

1. **Create stream session**: POST to `/v1/live`, get WebSocket URL.

2. **Connect WebSocket**: Open connection to returned URL. Wait for `open` event before sending audio.

3. **Stream audio**: Send 100ms chunks of PCM float32 (16 kHz mono) as binary frames at real-time pace. Do not rush ahead; server enforces 5-second buffer.

4. **Receive events**: Server emits JSON events: `diarization_speaker_start` and `diarization_speaker_end` with timestamp and speaker label.

5. **End stream**: Send `{"type": "end_of_stream"}` JSON frame when done. Server closes connection with code 1000.

6. **Handle errors**: If server sends `error` event, stop sending audio and close connection.

## Common gotchas

- **Audio URL must be direct and public**: URLs requiring authentication, redirects, or confirmation steps will fail with "Could not load audio" error. Use signed S3 URLs or pyannote's upload endpoint.

- **Results auto-delete after 24 hours**: Save diarization/identification output to your database immediately. Polling after 24 hours returns empty results.

- **Voiceprints are reusable but must be stored**: Create voiceprint once, save the output, reuse in future identify requests. Don't recreate voiceprints repeatedly.

- **Voiceprints require single-speaker audio**: If voiceprint audio contains multiple speakers or background noise, identification accuracy suffers. Validate audio before creating voiceprints.

- **Overlapping speech is detected by timestamp overlap**: Two speakers with overlapping timestamps = overlapping speech. Use `exclusive: true` to avoid overlaps in output.

- **Confidence scores are per-voiceprint, not per-segment**: In identification output, `confidence` object shows how well each voiceprint matches each speaker. Low confidence doesn't mean the speaker isn't in the audio; it means the voiceprint didn't match well.

- **Rate limits are per-endpoint, per-team**: 100 req/min for job submission, 300 req/min for polling. Polling too aggressively triggers 429 errors. Use webhooks in production.

- **Streaming audio must be real-time pace**: Sending audio faster than real-time (e.g., 500ms chunks every 100ms) causes buffer overflow and connection closure. Respect the 100ms chunk duration.

- **Streaming WebSocket URLs are single-use**: Each `/v1/live` call returns a new URL. URLs expire after use or after idle timeout (5 seconds).

- **Webhook URLs must be HTTPS**: Plain HTTP rejected. Ensure your webhook endpoint has valid SSL certificate.

- **Community-1 model doesn't support voiceprints**: Voiceprint and identification features only work with `precision-2`. Attempting to use voiceprints with `community-1` fails.

- **Exclusive mode changes output semantics**: With `exclusive: true`, only one speaker per time window. Overlapping speech is resolved to the most likely speaker. Use for STT reconciliation only.

## Verification checklist

Before submitting work with pyannote.ai:

- [ ] Audio URL is direct, publicly accessible, and ends with file extension (`.wav`, `.mp3`, etc.)
- [ ] API key is valid and has sufficient credits/subscription
- [ ] Request includes required fields: `url` for diarization, `url` + `voiceprints` for identification
- [ ] Model specified (or default `precision-2` is acceptable)
- [ ] If using webhooks: URL is HTTPS, endpoint returns 200 OK, signature verification implemented (optional but recommended)
- [ ] If polling: rate limit headers checked; not polling faster than 10-second intervals
- [ ] Results saved to database before 24-hour auto-deletion window
- [ ] Voiceprints stored securely if reusing for future identification
- [ ] Confidence scores reviewed if `confidence: true` in request
- [ ] Streaming audio sent at real-time pace (100ms chunks, not faster)
- [ ] Error handling in place for failed jobs, rate limits, and network timeouts

## Resources

- **Comprehensive navigation**: https://docs.pyannote.ai/llms.txt — page-by-page listing of all documentation
- **API Reference**: https://docs.pyannote.ai/api-reference/diarize — complete endpoint documentation with request/response schemas
- **Tutorials**: https://docs.pyannote.ai/tutorials/how-to-diarize-audio — step-by-step guides for diarization, identification, streaming, and integration with transcription
- **Troubleshooting**: https://docs.pyannote.ai/support/troubleshooting — common errors and solutions

---

> For additional documentation and navigation, see: https://docs.pyannote.ai/llms.txt