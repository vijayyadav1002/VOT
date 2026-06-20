---
name: app_initialize
description: Build a PWA voice-to-text app called "VoiceClip" with the following requirements.
---
Build a PWA voice-to-text app called "VoiceClip" with the following requirements.

## Core User Flow
1. User opens the app (installed to home screen on iOS, or browser on desktop/Android)
2. Taps a large record button → starts recording audio
3. Speaks naturally
4. Taps stop (or silence is auto-detected after 2.5 seconds of quiet)
5. App transcribes the audio, then sends it to an AI model for cleanup
6. Cleaned text is displayed with a large "Copy" button
7. User copies and pastes wherever they want
8. Optional: tone selector (Casual / Formal / Bullet Points) before or after transcription

---

## Tech Stack
- Vanilla HTML + CSS + JavaScript (single index.html, app.js, style.css)
- No frameworks, no build tools — must run by opening index.html or deploying as static files
- PWA: include manifest.json and service-worker.js for offline shell + home screen install
- MediaRecorder API for audio capture
- webkitSpeechRecognition (free, no API key) as the DEFAULT transcription engine
- Whisper/other APIs as optional upgrade (see provider config below)
- Anthropic Claude API for AI text cleanup

---

## Provider Configuration — ENV-DRIVEN, FLEXIBLE
All API keys and provider selection must be read from a config object at the TOP of app.js 
so the user can swap providers by editing one block:

```javascript
const CONFIG = {
  // --- Transcription Provider ---
  // Options: "browser" | "openai-whisper" | "groq-whisper" | "openai-gpt4o-mini"
  TRANSCRIPTION_PROVIDER: "browser",
  TRANSCRIPTION_API_KEY: "", // only needed if not using "browser"

  // --- AI Cleanup Provider ---
  // Options: "anthropic" | "openai-gpt4o-mini" | "groq-llama"
  CLEANUP_PROVIDER: "anthropic",
  CLEANUP_API_KEY: "", // Anthropic API key

  // --- Model overrides (optional) ---
  TRANSCRIPTION_MODEL: "whisper-1",          // used for openai-whisper
  CLEANUP_MODEL: "claude-haiku-4-5-20251001", // cheapest fast Claude model
};


Implement a TranscriptionService and CleanupService that each check CONFIG and route to
the correct provider. Adding a new provider should require touching only those two service
functions, nowhere else.

Transcription Providers to Implement

	1.	browser (default) — uses webkitSpeechRecognition, free, no key needed
	•	Stream results in real time while user speaks
	•	Fall back gracefully if browser doesn’t support it
	2.	openai-whisper — POST to https://api.openai.com/v1/audio/transcriptions
	•	Model: CONFIG.TRANSCRIPTION_MODEL (default “whisper-1”)
	•	$0.006/min
	3.	groq-whisper — POST to https://api.groq.com/openai/v1/audio/transcriptions
	•	Same OpenAI-compatible format, just different base URL and key
	•	Much faster, cheaper at scale
	4.	openai-gpt4o-mini — POST to https://api.openai.com/v1/audio/transcriptions
	•	Model: “gpt-4o-mini-transcribe”
	•	$0.003/min (half the price of whisper-1)

AI Cleanup Providers to Implement

	1.	anthropic (default) — POST to https://api.anthropic.com/v1/messages
	•	Model: CONFIG.CLEANUP_MODEL (default “claude-haiku-4-5-20251001”)
	•	Header: x-api-key + anthropic-version: 2023-06-01
	2.	openai-gpt4o-mini — POST to https://api.openai.com/v1/chat/completions
	•	Model: “gpt-4o-mini”
	•	Uses TRANSCRIPTION_API_KEY (same OpenAI key)
	3.	groq-llama — POST to https://api.groq.com/openai/v1/chat/completions
	•	Model: “llama3-8b-8192”
	•	Uses TRANSCRIPTION_API_KEY (same Groq key)

AI Cleanup System Prompt

Use this exact system prompt for all cleanup providers:

“You are a voice transcription editor. You receive raw speech-to-text output and return
only the cleaned version — no commentary, no explanation, no quotation marks around it.

Rules:

	•	Remove filler words: um, uh, like, you know, so, basically, literally, right
	•	Fix run-on sentences with proper punctuation
	•	Correct obvious grammar errors
	•	Preserve the speaker’s original meaning and vocabulary exactly
	•	If tone=formal: use professional language, complete sentences
	•	If tone=casual: keep it conversational, contractions are fine
	•	If tone=bullets: convert to a clean markdown bullet list

Return ONLY the cleaned text. Nothing else.”

UI Design

	•	Dark background (#0f0f0f), single centered card (max-width 480px)
	•	Large circular record button (80px) — red pulse animation while recording
	•	Waveform visualization using Web Audio API AnalyserNode while recording
	•	Status text below button: “Tap to record” → “Recording…” → “Transcribing…” → “Done”
	•	Tone selector: 3 pill buttons (Casual | Formal | Bullets) — Casual selected by default
	•	Result text area: displays cleaned text, editable by user before copying
	•	Large “Copy” button — turns green with checkmark for 2 seconds after copy
	•	Small “Re-clean” button to re-run cleanup with a different tone without re-recording
	•	Settings gear icon (top right) → modal to edit CONFIG values without touching code
	•	Toast notifications for errors (mic denied, API failure, etc.)

PWA Requirements

	•	manifest.json with name, short_name, theme_color, background_color, display: standalone
	•	service-worker.js that caches the app shell (index.html, app.js, style.css, manifest.json)
	•	App must load and show UI when offline (transcription/cleanup will fail gracefully with a toast)
	•	Generate a simple SVG icon (192x192 and 512x512) — microphone icon on dark background

File Structure to Generate

voiceclip/
├── index.html
├── app.js
├── style.css
├── manifest.json
├── service-worker.js
├── icon-192.svg
└── icon-512.svg


Error Handling

	•	Mic permission denied → toast: “Microphone access is required. Please allow it in your browser settings.”
	•	webkitSpeechRecognition not supported → auto-switch to MediaRecorder + warn user they need an API key
	•	API call fails → toast with error message + “Try again” option, raw transcript still shown
	•	Empty recording → toast: “No audio detected. Try again.”

iOS PWA Note

Add this to index.html <head>:

<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<link rel="apple-touch-icon" href="icon-192.svg">


Mic access on iOS Safari requires HTTPS. Include a visible banner if the app detects it’s
running on HTTP: “For microphone access on iOS, this app must be served over HTTPS.”

Deliverable

Generate all 7 files completely. No placeholders, no TODOs. Every function must be
implemented. The app should work end-to-end with CONFIG.TRANSCRIPTION_PROVIDER=“browser”
and a valid CONFIG.CLEANUP_API_KEY (Anthropic) with zero additional code changes.


---

**How to use it:** paste this into Claude Code in your terminal (`claude` command), and it'll generate all 7 files in one shot. The only thing you'll need to add manually is your Anthropic API key in the `CONFIG` block at the top of `app.js`.

To swap providers later, it's literally one line — change `TRANSCRIPTION_PROVIDER: "browser"` to `"groq-whisper"` and add the key. That's it.
