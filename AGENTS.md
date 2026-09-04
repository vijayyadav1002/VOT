# AGENTS.md

Guidance for AI coding assistants (Claude, Copilot, Cursor, etc.) working in this repo.

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
- **`CleanupService`** — `.cleanup(text, tone)` calls the selected AI provider with a `tone=<value>` prefix on the user message. Uses `CLEANUP_SYSTEM_PROMPT` for English and `CLEANUP_SYSTEM_PROMPT_HI` for Hinglish (`LANGUAGE_MODE === 'hi-en'`). Both prompts instruct the model to act as a text editor only — never answering questions or acting on the content, only cleaning it up. Hinglish cleanup is mixed-script (Hindi in Devanagari, English in Latin), never Urdu/Arabic script, and must not translate. A retry fires if English Latin collapses into Devanagari or if Arabic/Urdu letters appear. Whisper calls in `hi-en` send `language=hi` plus a mixed-script `prompt` so auto-detect does not pick Urdu.

### Recording flow

1. `startRecording()` acquires a mic stream, then starts either `SpeechRecognition` (browser) or `MediaRecorder` (API providers). A session mutex (`sessionPhase` + `takeGen`) ignores overlapping mic / Done / Start over while acquiring, recording, transcribing, or cleaning.
2. Stop **releases the mic** via `releaseAudio()` — not `MediaRecorder.pause()`. There is **no** AnalyserNode silence auto-stop.
3. After a non-empty take the session is **paused**. Concatenated transcript **text** (never audio blobs) is the source of truth in the textarea. The user chooses:
   - **Mic tap while paused**: re-acquire the mic and append the next take with `appendTranscript`.
   - **Done**: run `processCleanup()` once on the full concatenated raw.
   - **Start over**: discard the in-progress draft and start a new conversation.
4. Stop path:
   - **browser**: accumulated `state.rawTranscript` goes to `enterPaused()` (no cleanup yet).
   - **API providers**: `handleBlobStop()` transcribes that take’s blob, appends text, then `enterPaused()`.
5. `processCleanup(rawText)` runs **only on Done**: shows a skeleton, then the AI-cleaned version, then persists `{ id, ts, raw, cleaned, tone, language }` to `localStorage` key `voiceclip_history` (newest-first, FIFO cap 10).

### State

Single `state` object in module scope — no framework. `state.rawTranscript` is the unmodified transcript preserved for Re-clean. `state.sessionPhase` is `idle` | `acquiring` | `recording` | `paused` | `cleaning` | `done`. Finished clips live in `localStorage['voiceclip_history']`; paused drafts are memory-only.

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

`service-worker.js` caches the app shell (all 7 local files) on install and serves cache-first for same-origin GET requests. It does not intercept cross-origin API calls. Cache is versioned by `CACHE_NAME = 'voiceclip-v12'` — bump this string when deploying changes that must invalidate cached assets.

## Contribution guidelines

- No build step, no bundler, no npm packages — keep it that way.
- All JS stays in `app.js`, all CSS in `style.css`, all markup in `index.html`.
- Adding a new transcription provider: add a branch in `TranscriptionService.transcribeBlob()`.
- Adding a new cleanup provider: add a branch in `CleanupService.cleanup()`.
- Test on both desktop Chrome and iOS Safari before submitting a PR.
