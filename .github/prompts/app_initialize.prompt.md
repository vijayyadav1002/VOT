---
name: app_initialize
description: Build VoiceClip — a zero-dependency PWA that records voice, transcribes it, and cleans up the text with AI.
---

Build a PWA voice-to-text app called **VoiceClip** that generates all 7 files completely — no placeholders, no TODOs. Every function must be implemented. The app must work end-to-end with `TRANSCRIPTION_PROVIDER="browser"` and a valid `CLEANUP_API_KEY` (Anthropic) with zero additional code changes.

---

## Core User Flow

1. User opens the app (installed to home screen on iOS, or in browser on desktop/Android)
2. Taps a large record button → starts recording
3. Speaks naturally
4. Taps stop (or silence is auto-detected after 2.5 s)
5. App transcribes the audio, then sends it to an AI model for cleanup
6. Cleaned text is displayed with a "Copy" button and a "Re-clean" button
7. User copies and pastes wherever they want
8. Tone selector (Casual / Formal / Bullets) and Language selector (EN / Hinglish) are always visible in a controls dock at the bottom

---

## Tech Stack

- Vanilla HTML + CSS + JavaScript — `index.html`, `app.js`, `style.css`
- No frameworks, no build tools, no npm — plain static files served as-is
- PWA: `manifest.json` + `service-worker.js` for offline shell and home-screen install
- `webkitSpeechRecognition` as the default transcription engine (free, no key)
- API providers (Whisper, etc.) as optional upgrades
- Anthropic Claude as default AI cleanup provider

---

## Provider Configuration

Single `CONFIG` object at the top of `app.js`. All other code routes through it. Settings modal merges into it via `localStorage`.

```js
const CONFIG = {
  // "browser" | "openai-whisper" | "groq-whisper" | "openai-gpt4o-mini"
  TRANSCRIPTION_PROVIDER: 'browser',
  TRANSCRIPTION_API_KEY: '',

  // "anthropic" | "openai-gpt4o-mini" | "groq-llama"
  CLEANUP_PROVIDER: 'anthropic',
  CLEANUP_API_KEY: '',

  TRANSCRIPTION_MODEL: 'whisper-1',
  CLEANUP_MODEL: 'claude-haiku-4-5-20251001',
  LANGUAGE_MODE: 'en',   // 'en' | 'hi-en'
};

// Merge persisted settings over defaults at startup
(function applyStoredConfig() {
  try {
    const saved = localStorage.getItem('voiceclip_config');
    if (saved) Object.assign(CONFIG, JSON.parse(saved));
  } catch (_) {}
}());
```

Implement a `TranscriptionService` and `CleanupService`. Adding a new provider requires touching only those two objects.

---

## Transcription Providers

**`browser`** (default) — `webkitSpeechRecognition` / `SpeechRecognition`
- `continuous: true`, `interimResults: true`
- Lang: `'hi-IN'` when `LANGUAGE_MODE === 'hi-en'`, otherwise `navigator.language || 'en-US'`
- Stream interims to the result textarea in real time via an `onInterim` callback
- Accumulate finals in `state.rawTranscript` via an `onFinal` callback
- On `onerror`, ignore `'no-speech'`; call `onError` for everything else
- Return the recognition instance so it can be stopped

**`openai-whisper`** — `POST https://api.openai.com/v1/audio/transcriptions`
- Model: `CONFIG.TRANSCRIPTION_MODEL` (default `whisper-1`)
- Append `language: 'en'` unless `LANGUAGE_MODE === 'hi-en'`

**`groq-whisper`** — `POST https://api.groq.com/openai/v1/audio/transcriptions`
- Model: `whisper-large-v3`
- Same language logic as above

**`openai-gpt4o-mini`** — `POST https://api.openai.com/v1/audio/transcriptions`
- Model: `gpt-4o-mini-transcribe`
- Same language logic

All three API providers use `FormData` with `file: audioBlob` named `recording.webm` and `Authorization: Bearer <key>`.

---

## AI Cleanup Providers

User message format: `tone=<casual|formal|bullets>\n\n<raw text>`

**`anthropic`** (default) — `POST https://api.anthropic.com/v1/messages`
- Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `anthropic-dangerous-direct-browser-access: true`
- Model: `CONFIG.CLEANUP_MODEL`; `max_tokens: 2048`
- Response: `.content[0].text.trim()`

**`openai-gpt4o-mini`** — `POST https://api.openai.com/v1/chat/completions`
- Model: `gpt-4o-mini`; uses `CONFIG.TRANSCRIPTION_API_KEY`
- Response: `.choices[0].message.content.trim()`

**`groq-llama`** — `POST https://api.groq.com/openai/v1/chat/completions`
- Model: `llama3-8b-8192`; uses `CONFIG.TRANSCRIPTION_API_KEY`
- Response: `.choices[0].message.content.trim()`

---

## System Prompts

Use `CLEANUP_SYSTEM_PROMPT` for English; `CLEANUP_SYSTEM_PROMPT_HI` for Hinglish (when `LANGUAGE_MODE === 'hi-en'`).

```
CLEANUP_SYSTEM_PROMPT:
"You are a voice transcription editor. You receive raw speech-to-text output and return
only the cleaned version — no commentary, no explanation, no quotation marks around it.

Rules:
• Remove filler words: um, uh, like, you know, so, basically, literally, right
• Fix run-on sentences with proper punctuation
• Correct obvious grammar errors
• Preserve the speaker's original meaning and vocabulary exactly
• If tone=formal: use professional language, complete sentences
• If tone=casual: keep it conversational, contractions are fine
• If tone=bullets: convert to a clean markdown bullet list

Return ONLY the cleaned text. Nothing else."
```

```
CLEANUP_SYSTEM_PROMPT_HI:
"You are a voice transcription editor specializing in Hindi-English mixed speech (Hinglish).
You receive raw speech-to-text and return only the cleaned version — no commentary, no explanation.

Rules:
• Remove filler words: um, uh, like, you know, haan, acha, matlab, basically, actually, toh, na, yaar
• Fix run-on sentences with proper punctuation
• Correct obvious grammar errors
• CRITICAL — preserve the language each word was spoken in:
  - If the speaker said a word in English (e.g. "practice", "meeting", "laptop"), write it
    in English — even if a Hindi equivalent exists
  - If the speaker said a word in Hindi, write it in Devanagari script
  - Never translate or substitute a word into the other language
  - Never transliterate Hindi into Roman letters or English into Devanagari
• If tone=formal: use professional Hindi with English terms where the speaker used them
• If tone=casual: keep it conversational, preserving the original code-switching
• If tone=bullets: convert to a clean bullet list, maintaining each word's original language

Return ONLY the cleaned text. Nothing else."
```

---

## App State

```js
const state = {
  isRecording:        false,
  mediaRecorder:      null,
  audioChunks:        [],
  recognition:        null,
  rawTranscript:      '',
  selectedTone:       'casual',
  stream:             null,
  recordingStartTime: null,
  timerInterval:      null,
};
```

---

## Recording Flow

**Start:**
1. `navigator.mediaDevices.getUserMedia({ audio: true })` — on `NotAllowedError` / `PermissionDeniedError`, call `showMicBlocked()`
2. Set `state.isRecording = true`, clear `audioChunks` + `rawTranscript`, hide result section
3. Add `.recording` class to `#record-btn` and `#bar-viz`
4. Start a 1-second interval that updates `#status-sub` with elapsed time `M:SS` (starts at `0:00`)
5. Set `#status` text to `'Recording'` in accent color
6. If `browser` provider: call `TranscriptionService.startBrowserRecognition(onInterim, onFinal, onError)`
   - Show Hinglish tip toast if `LANGUAGE_MODE === 'hi-en'`
   - If recognition returns null, toast error and `forceStopRecording()`
7. Else: pick best supported MIME type (`audio/webm;codecs=opus` → `audio/webm` → `audio/ogg;codecs=opus` → `audio/mp4`), create `MediaRecorder`, start with 100 ms timeslice, `ondataavailable` pushes chunks, `onstop` calls `handleBlobStop()`

**Stop:**
1. Clear timer interval
2. Set `state.isRecording = false`, remove `.recording`, add `.processing` to record btn
3. Stop recognition or MediaRecorder
4. Release mic stream tracks
5. For `browser`: if `rawTranscript` is empty → toast "No audio detected. Try again." and reset; else call `processCleanup(rawTranscript)`
6. For API providers: `handleBlobStop()` builds the Blob, calls `TranscriptionService.transcribeBlob(blob)`, then `processCleanup()`

**`processCleanup(rawText)`:**
1. `setStatus('Cleaning up…', 'Polishing with AI')`
2. Show `#result-skeleton`, hide `#result-text`
3. If no `CLEANUP_API_KEY`: show raw text immediately, toast "Add a Cleanup API key in Settings to enable AI cleanup.", `setStatus('Ready to copy', 'Tap mic to record again')`
4. Else: call `CleanupService.cleanup(rawText, state.selectedTone)`, put result in `#result-text`, auto-resize textarea
5. On error: show raw text + error toast
6. Always: `setStatus('Ready to copy', 'Tap mic to record again')`, remove `.processing` from record btn

**`setStatus(main, sub)`:**
```js
function setStatus(main, sub = '') {
  $status.textContent = main;
  $status.style.color = state.isRecording ? 'var(--accent)' : '';
  $statusSub.textContent = sub;
}
```

Idle state: `setStatus('Tap to record', 'Hold steady, speak naturally')`

---

## Visual Design

### Color Palette
```css
:root {
  --bg:           #08080f;
  --card-bg:      #0e0e18;
  --border:       #14141e;
  --text:         #e7e7ef;
  --text-muted:   #8a8aae;
  --text-dim:     #3a3a52;
  --accent:       #6366f1;
  --accent-hover: #4f46e5;
  --btn-bg:       #0e0e18;
  --btn-hover:    #141428;
}
```

### Font
Load **Geist** from Google Fonts CDN in `<head>`. Fallback: `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`.

### Layout (mobile-first, centered, max 480px)
```
body
└── .app-container (flex column, max-width 480px, full height)
    ├── .header (brand + settings button)
    ├── .main-stage (flex:1, centered column, scrollable)
    │   ├── .bar-viz (12 .bar divs)
    │   ├── .record-container > #record-btn
    │   ├── .status-block (#status + #status-sub)
    │   └── #result-section (.result-skeleton + #result-text textarea + .result-actions)
    └── .controls-dock (Tone row + Lang row)
```

### Body
```css
body {
  background: var(--bg);
  background-image: radial-gradient(ellipse 360px 280px at 50% -60px, rgba(99,102,241,0.13) 0%, transparent 80%);
  min-height: 100vh;
  min-height: 100dvh;   /* Safari address bar fix */
  display: flex;
  flex-direction: column;
  align-items: center;
}
html { height: 100%; background: var(--bg); }  /* prevents bounce-scroll gap */
```

### Header
```css
.header {
  padding: calc(18px + env(safe-area-inset-top, 0px)) 24px 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-shrink: 0;
}
```
- `.header-brand`: flex row, gap 9px
  - `.brand-icon`: 26×26px, `border-radius: 8px`, `background: var(--accent)`, `box-shadow: 0 4px 14px rgba(99,102,241,0.35)` — contains a 14×14 white mic SVG
  - `.app-title`: 16px, weight 600, `letter-spacing: -0.02em`, color `#f2f2f7`
- `.settings-btn`: 34×34px, `border-radius: 10px`, `border: 1px solid #1b1b2a`, `background: var(--btn-bg)`, color `#4a4a60` — contains a 16×16 gear SVG

### Bar Visualizer (12 bars, CSS-only — no Web Audio API)
```css
.bar-viz { display: flex; align-items: center; justify-content: center; gap: 4px; height: 54px; }
.bar { width: 3.5px; height: 48px; border-radius: 3px; background: var(--accent); transform-origin: center; opacity: 0.12; transition: opacity 0.3s; }
```

Idle scaleY values (nth-child 1–12): `.333, .625, .958, .458, .792, .292, .583, .917, .417, .708, .542, .250`

Recording (`.bar-viz.recording .bar { opacity: 0.95; }`), assign animations per bar:
- bar 1 → `bar-c 1.10s`
- bar 2 → `bar-a 0.90s`
- bar 3 → `bar-d 1.30s`
- bar 4 → `bar-b 0.80s`
- bar 5 → `bar-e 1.20s`
- bar 6 → `bar-a 1.00s`
- bar 7 → `bar-c 0.95s`
- bar 8 → `bar-d 1.15s`
- bar 9 → `bar-b 1.30s`
- bar 10 → `bar-e 0.85s`
- bar 11 → `bar-a 1.05s`
- bar 12 → `bar-c 1.20s`

All `ease-in-out infinite`. Keyframes:
```css
@keyframes bar-a { 0%,100%{ transform:scaleY(.18); } 50%{ transform:scaleY(1.00); } }
@keyframes bar-b { 0%,100%{ transform:scaleY(.45); } 50%{ transform:scaleY(.90);  } }
@keyframes bar-c { 0%,100%{ transform:scaleY(.70); } 50%{ transform:scaleY(.28);  } }
@keyframes bar-d { 0%,100%{ transform:scaleY(.25); } 50%{ transform:scaleY(.95);  } }
@keyframes bar-e { 0%,100%{ transform:scaleY(.60); } 50%{ transform:scaleY(.15);  } }
```

### Record Button
84×84px circle, `background: var(--accent)`, `box-shadow: 0 8px 32px rgba(99,102,241,0.4)`.

Three child elements; only one visible per state:
- `.mic-icon` (32×32 SVG) — default
- `.stop-icon` (26×26, `background:#fff`, `border-radius:6px`) — when `.recording`
- `.spinner-icon > .spinner` (26×26 spinner, `border-top-color: var(--accent)`) — when `.processing`

Recording: `animation: pulse-ring 1.6s ease-in-out infinite`
```css
@keyframes pulse-ring {
  0%   { box-shadow: 0 0 0  0px rgba(99,102,241,0.45); }
  60%  { box-shadow: 0 0 0 22px rgba(99,102,241,0);    }
  100% { box-shadow: 0 0 0  0px rgba(99,102,241,0);    }
}
```
Processing: `background:#13131f; border:1.5px solid #24243c; box-shadow:none; pointer-events:none`

### Status Block
```css
.status-block { display:flex; flex-direction:column; align-items:center; gap:5px; min-height:40px; }
.status     { font-size:15px; font-weight:500; color:var(--text); transition:color 0.2s; }
.status-sub { font-size:12px; color:var(--text-dim); font-variant-numeric:tabular-nums; min-height:16px; }
```

### Result Section
Animate in with:
```css
@keyframes vc-rise { from { opacity:0; transform:translateY(10px); } to { opacity:1; transform:translateY(0); } }
.result-section { animation: vc-rise 0.45s cubic-bezier(.2,.8,.2,1) both; }
```

- `.result-skeleton`: shimmer loading bar, `min-height:78px`, `border-radius:14px`
- `#result-text`: `background:var(--card-bg); border:1px solid #1a1a2a; border-radius:14px; color:#c5c5d8; font-size:14px; padding:15px 16px; resize:none; min-height:80px; overflow-y:hidden` — auto-resize on input via `el.style.height = el.scrollHeight + 'px'`
- `.copy-btn`: flex:1, 44px tall, `border-radius:13px`, accent background, weight 600. On copy: text becomes `'✓ Copied!'`, add `.copied` class (`background:#1d8a52; box-shadow:0 6px 20px rgba(29,138,82,0.30)`) for 2 s
- `.reclean-btn`: fixed width, same height, dark bordered button

### Controls Dock
```css
.controls-dock {
  padding: 16px 24px calc(20px + env(safe-area-inset-bottom, 0px));
  border-top: 1px solid #14141f;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.dock-label { font-size:10px; letter-spacing:0.14em; text-transform:uppercase; color:#33334a; font-weight:600; width:36px; flex-shrink:0; }
```

Two rows:
1. **Tone** label + chip group: `Casual` (active by default), `Formal`, `Bullets`
2. **Lang** label + chip group: `EN` (active by default), `हिं / EN`

Chip styles:
```css
.chip { padding:6px 14px; border-radius:20px; background:transparent; border:1px solid #181826; color:var(--text-dim); font-size:13px; font-weight:500; }
.chip.active { background:rgba(99,102,241,0.13); border-color:rgba(99,102,241,0.40); color:var(--accent); }
```

### Settings Bottom Sheet
- Triggered by gear button in header
- `.sheet-overlay` (fixed, full-screen, flex align-items:flex-end) containing:
  - `.sheet-backdrop` (absolute full-screen, `background:rgba(2,2,8,0.66)`, `backdrop-filter:blur(3px)`) — clicking closes the sheet
  - `.sheet` (slides up with animation, `border-radius:26px 26px 0 0`, max-height:85vh, scrollable, `padding-bottom:env(safe-area-inset-bottom,0)`)

Sheet animation:
```css
@keyframes sheet-up { from{transform:translateY(100%);} to{transform:translateY(0);} }
.sheet { animation: sheet-up 0.34s cubic-bezier(.2,.85,.25,1) both; }
```

Sheet contents:
- `.sheet-handle`: 38×4px pill, `background:#23233a`, centered
- `.sheet-header`: "Settings" title + "Done" button (color: var(--accent))
- `.sheet-body`: two `<details>`-free sections — **Transcription** and **AI Cleanup** — each with a `.sheet-card` (dark card, `border-radius:14px`, `overflow:hidden`) containing `.sheet-row` divs (label left, `<select>` or `<input type="password">` right, right-aligned in accent color)
- Transcription card rows: Provider `<select>`, API Key `<input>`, Language `<select>`
- AI Cleanup card rows: Provider `<select>`, API Key `<input>`
- Advanced `<details>` section (collapsed by default): Transcription Model + Cleanup Model text inputs
- `.sheet-footer`: sticky bottom, full-width "Save Settings" button

Input IDs (must match exactly for JS to work): `cfg-transcription-provider`, `cfg-transcription-key`, `cfg-language-mode`, `cfg-cleanup-provider`, `cfg-cleanup-key`, `cfg-transcription-model`, `cfg-cleanup-model`

On save: write all values to `CONFIG`, persist to `localStorage`, close sheet, show "Settings saved." toast. Also sync `.lang-row` chip active state to match saved `LANGUAGE_MODE`.

Backdrop/Done button both close the sheet. Implement backdrop click by checking `e.target.id === 'sheet-backdrop'` (it's a child element, not the overlay itself).

### Toasts
- Position: `fixed; bottom: max(24px, env(safe-area-inset-bottom, 24px)); left:50%; transform:translateX(-50%)` — stacked column
- `.toast`: dark card with border, 14px text, close button (`✕`)
- `.toast.error`: `background:#1a0a0a; border-color:#7f1d1d`
- Auto-dismiss after `duration` ms (default 6000); `duration=0` means persistent
- Slide up on entry: `@keyframes toast-in { from{opacity:0;transform:translateY(12px);} to{opacity:1;transform:translateY(0);} }`

---

## iOS PWA Requirements

Add to `<head>`:
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="VoiceClip">
<link rel="apple-touch-icon" href="icon-192.svg">
```

**iOS input zoom prevention** — iOS zooms the viewport when focusing an input with `font-size < 16px`. Fix:
```css
@media (max-width: 768px) {
  input, textarea, select { font-size: 16px !important; }
}
button, a, input, textarea, select { touch-action: manipulation; }
```

**HTTP warning banner** — show `#http-warning` (amber banner) when `location.protocol === 'http:'` and hostname is not `localhost` / `127.0.0.1`.

---

## Microphone Blocked

`showMicBlocked()`:
- Set `$status.textContent = 'Microphone blocked'`, `$status.style.color = '#ef4444'`
- Toast with platform-specific instructions:
  - iOS: "Mic blocked. To allow: Settings app → scroll to VoiceClip → enable Microphone."
  - Other: "Mic blocked. Click the lock icon in your browser address bar to allow microphone access."
  - Persistent toast (`duration=0`)

On startup, query `navigator.permissions` for `'microphone'`; if `denied` call `showMicBlocked()`. Listen for `status.onchange` to recover automatically.

---

## Language Selector Logic

- Clicking a lang chip sets `CONFIG.LANGUAGE_MODE` and persists to `localStorage` immediately (no need to press Save)
- On startup, sync active chip to match `CONFIG.LANGUAGE_MODE`
- Saving Settings also syncs chip active state from the `cfg-language-mode` select

---

## Service Worker (`service-worker.js`)

```js
const CACHE_NAME = 'voiceclip-v2';
const APP_SHELL  = ['./', './index.html', './app.js', './style.css', './manifest.json', './icon-192.svg', './icon-512.svg'];
```

- `install`: cache all APP_SHELL, `self.skipWaiting()`
- `activate`: delete old caches, `self.clients.claim()`
- `fetch`: cache-first for same-origin GET requests; pass-through for cross-origin (API calls)

---

## File Structure

```
voiceclip/
├── index.html          ← shell markup only, no inline JS
├── app.js              ← all JavaScript
├── style.css           ← all CSS
├── manifest.json       ← PWA metadata
├── service-worker.js   ← offline cache
├── icon-192.svg        ← mic icon on dark bg, 192×192
└── icon-512.svg        ← mic icon on dark bg, 512×512
```

Icons: simple SVG, dark background (`#0f0f0f`), indigo circle, white mic path.

---

## How to use this prompt

Paste into any AI coding assistant (Claude, GPT-4o, Gemini, etc.) and it will generate all 7 files in one shot. The only thing you need to add manually is your API key — either paste it into the Settings sheet inside the app, or set `CLEANUP_API_KEY` in the `CONFIG` block at the top of `app.js`.

To swap providers: tap the gear → change Provider → paste your key → Save. That's it.
