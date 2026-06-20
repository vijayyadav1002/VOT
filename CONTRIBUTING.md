# Contributing to VoiceClip

Thanks for your interest! VoiceClip is intentionally simple — no build step, no bundler, no dependencies. Please keep it that way.

## Ground rules

- All JavaScript stays in `voiceclip/app.js`
- All CSS stays in `voiceclip/style.css`
- All markup stays in `voiceclip/index.html`
- No npm packages, no transpilation, no bundler
- Test on both desktop Chrome and iOS Safari before opening a PR

## Getting started

```bash
cd voiceclip
npx serve .
# Open http://localhost:3000
```

For iOS device testing (mic requires HTTPS):

```bash
ngrok http 3000
# Open the https://*.ngrok-free.app URL in Safari on your device
```

## How to add a transcription provider

1. Add a new `if (p === 'your-provider')` branch in `TranscriptionService.transcribeBlob()` in `app.js`
2. Add the option to the `#cfg-transcription-provider` select in `index.html`
3. Update the provider table in `voiceclip/README.md`

## How to add a cleanup provider

1. Add a new `if (p === 'your-provider')` branch in `CleanupService.cleanup()` in `app.js`
2. Add the option to the `#cfg-cleanup-provider` select in `index.html`
3. Update the provider table in `voiceclip/README.md`

## Submitting a PR

- Keep PRs focused — one feature or fix per PR
- Describe what you changed and why
- Include any new provider API docs or references in the PR description
- If you're adding a provider, include a note on whether it has a free tier

## Reporting bugs

Open an issue with:
- Browser and OS version
- Steps to reproduce
- What you expected vs. what happened
- Any console errors (open DevTools → Console)

## Questions

Open a discussion or issue — happy to help.
