# CLAUDE.md

> **Note:** This file has been superseded by [`AGENTS.md`](./AGENTS.md), which contains the same guidance in a tool-agnostic format. Please refer to that file instead.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**VoiceClip** — a zero-dependency PWA that records voice, transcribes it, and uses an AI model to clean up the text. All code lives in `voiceclip/` as plain static files (no build step, no bundler, no npm).

## Running the app

```bash
# Any static file server works — the app must be served, not opened as file://
cd voiceclip
npx serve .          # or: python3 -m http.server 8080
# Then open http://localhost:3000 (or 8080)
```

For HTTPS (required for mic on iOS real devices), expose via ngrok in a second terminal:

```bash
ngrok http 3000
# Use the https://*.ngrok-free.app URL on the device
```

Microphone access on iOS requires HTTPS. For local dev, `localhost` is treated as secure by all major browsers. For real iOS devices, use the ngrok HTTPS URL.

## Architecture

All logic is in three files:

| File | Role |
|------|------|
| `app.js` | All JavaScript — state, services, event wiring |
| `style.css` | All styling — no utility classes, plain CSS custom properties |
| `index.html` | Shell markup — no JS inline, just DOM structure |

### CONFIG block (`app.js` top)

The single `CONFIG` object at the top of `app.js` controls all provider routing. Changes made via the Settings modal are persisted to `localStorage` and merged over `CONFIG` at startup via `applyStoredConfig()`.

### Service layer pattern

Two service objects route to provider implementations based on `CONFIG`:

- **`TranscriptionService`** — `.transcribeBlob(audioBlob)` for API providers; `.startBrowserRecognition(onInterim, onFinal, onError)` for the `"browser"` provider (webkitSpeechRecognition). Adding a new provider only requires touching these two objects.
- **`CleanupService`** — `.cleanup(text, tone)` calls the selected AI provider with the shared system prompt and a `tone=<value>` prefix on the user message.

### Recording flow

1. `startRecording()` acquires mic stream → creates `AudioContext` + `AnalyserNode` (for waveform) → starts either `SpeechRecognition` (browser) or `MediaRecorder` (API providers).
2. Silence detection polls `AnalyserNode` every 150 ms; auto-stops after 2.5 s below threshold.
3. Stop path:
   - **browser**: accumulated `state.rawTranscript` goes directly to `processCleanup()`.
   - **API providers**: `handleBlobStop()` uploads the blob, then calls `processCleanup()`.
4. `processCleanup(rawText)` shows raw text immediately, then replaces with AI-cleaned version.

### State

Single `state` object in module scope — no framework. `state.rawTranscript` is the unmodified transcript preserved for Re-clean.

## Provider reference

| Key | Transcription endpoint |
|-----|----------------------|
| `browser` | `webkitSpeechRecognition` (free, no key) |
| `openai-whisper` | `POST https://api.openai.com/v1/audio/transcriptions` |
| `groq-whisper` | `POST https://api.groq.com/openai/v1/audio/transcriptions` |
| `openai-gpt4o-mini` | same OpenAI endpoint, model `gpt-4o-mini-transcribe` |

| Key | Cleanup endpoint |
|-----|-----------------|
| `anthropic` | `POST https://api.anthropic.com/v1/messages` (needs `anthropic-dangerous-direct-browser-access: true` header for browser calls) |
| `openai-gpt4o-mini` | `POST https://api.openai.com/v1/chat/completions` — uses `TRANSCRIPTION_API_KEY` |
| `groq-llama` | `POST https://api.groq.com/openai/v1/chat/completions` — uses `TRANSCRIPTION_API_KEY` |

## PWA / service worker

`service-worker.js` caches the app shell (all 7 local files) on install and serves cache-first for same-origin GET requests. It does not intercept cross-origin API calls. Cache is versioned by `CACHE_NAME = 'voiceclip-v1'` — bump this string when deploying changes that must invalidate cached assets.
