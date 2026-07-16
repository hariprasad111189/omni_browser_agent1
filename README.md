# omni_browser_agent1

# Omni Browser Agent

### An autonomous, dual-LLM browser automation copilot with live voice interaction and graceful offline fallback

**Developed by:** Shyamala Hari Prasad

---

## Overview

Omni Browser Agent is a Chrome extension + local Python backend that lets you control a real Chrome browser using natural language. You type (or speak) a task — *"search for the cheapest flight to Tokyo next month and open the booking page"* — and an autonomous agent plans and executes the necessary clicks, typing, and navigation steps inside your actual browser session, streaming its reasoning back to you in real time.

The system is built around a **cost-resilient, dual-LLM architecture**: it defaults to Google Gemini for fast, high-quality reasoning, but automatically and silently falls back to a fully local Ollama model the moment Gemini's free-tier quota is exhausted or unreachable — so the agent never simply stops working.

---

## Core System Architecture

The system has four cooperating layers: the **Extension UI** (what you see), the **FastAPI Server** (orchestration), the **LLM + Agent Core** (reasoning and execution), and the **Chrome Browser** itself (the actual environment being controlled).

```
┌────────────────────────────────────────┐
│           CHROME EXTENSION UI            │
│         (popup.html + popup.js)          │
│                                           │
│  • Chat input (text or 🎤 voice)         │
│  • Session sidebar (persisted history)   │
│  • Model selector: Fast / Smart          │
│  • Language selector: EN / ES / HI       │
│  • Live "Gemini / Ollama" status pill    │
└─────────────────┬─────────────────────────┘
                  │ fetch() → NDJSON stream over HTTP
                  ▼
┌────────────────────────────────────────┐
│           FASTAPI SERVER (server.py)     │
│                                           │
│  GET  /health                            │
│  GET  /sessions        POST /sessions    │
│  POST /run-agent-stream (NDJSON)         │
│  POST /stop-agent                        │
└───────┬─────────────────────┬────────────┘
        ▼                     ▼
┌───────────────┐     ┌─────────────────────┐
│  LLM FACTORY   │     │   SESSION STORE      │
│ (llm_factory)  │     │ (session_store.py)   │
│                │     │                       │
│ Gemini Flash/  │     │ JSON files on disk:   │
│ Pro (primary)  │     │ data/sessions/*.json  │
│      ⇅ auto    │     │ + index.json          │
│ Ollama qwen2.5 │     │                       │
│ (fallback +    │     └─────────────────────┘
│ page-extract)  │
└───────┬────────┘
        ▼
┌────────────────────────────────────────┐
│      BROWSER-USE AGENT CORE              │
│                                           │
│  • Receives task + LLM pair              │
│  • Plans next action each step           │
│  • register_new_step_callback → NDJSON   │
│    "action" events streamed to extension │
│  • Retries/falls back on step failure    │
│  • Emits "done" when task is complete     │
└─────────────────┬─────────────────────────┘
                  │ Chrome DevTools Protocol (CDP)
                  ▼
┌────────────────────────────────────────┐
│           REAL CHROME BROWSER            │
│                                           │
│  Mode A: ATTACH to your already-open      │
│  Chrome (via --remote-debugging-port      │
│  9222) — keeps your logins/cookies        │
│                                           │
│  Mode B: LAUNCH a fresh instance that      │
│  copies your Default profile directory    │
└────────────────────────────────────────┘
```

### Data flow, step by step

1. **Input capture** — The user types a task in the extension popup, or speaks it (captured via the browser's Web Speech API `SpeechRecognition`, converted to text client-side — no audio ever leaves the browser).
2. **Request dispatch** — `popup.js` POSTs `{ prompt, model_choice, language, session_id }` to `POST /run-agent-stream` and opens a streaming `ReadableStream` reader on the response body.
3. **Session resolution** — `server.py` resolves or creates a chat session via `session_store.py`, appends the user's message, and persists it to `data/sessions/<uuid>.json`.
4. **LLM pairing** — `llm_factory.build_llm_pair()` inspects `GEMINI_API_KEY` and `PREFER_OLLAMA`: if a Gemini key is present, Gemini becomes primary and a local Ollama model becomes both the fallback LLM *and* the dedicated page-extraction LLM (since Ollama is cheaper for repetitive DOM-reading calls). If no key is present, Ollama alone drives everything.
5. **Browser acquisition** — `create_browser()` first tries to attach to a Chrome instance already running with a debugger port open (discovered via `DevToolsActivePort` files or a port scan on 9222–9232). If none is found, it launches a new Chrome instance that copies your existing profile directory, so bookmarks/extensions/logins carry over.
6. **Agent execution loop** — A `browser-use` `Agent` is instantiated with the task, the LLM pair, and a system prompt instructing it to prefer direct DOM interaction (click/input/navigate) over full-page text extraction, and to finish with a `done` action. Every step invokes `step_callback`, which formats the model's `next_goal` and chosen actions via `step_format.py` and pushes an `{"type": "action", ...}` NDJSON line onto an `asyncio.Queue`.
7. **Live streaming** — The FastAPI `StreamingResponse` generator drains that queue and yields each NDJSON line immediately, so the extension renders each planning/action step as it happens rather than waiting for the whole task to finish.
8. **Fallback transparency** — If Gemini returns a quota/429 error mid-task, `browser-use`'s built-in `fallback_llm` mechanism swaps to Ollama for subsequent steps. The final response reports `fallback_triggered: true` and which model actually answered, and the extension surfaces this with a visible "↩️ Ollama fallback" note.
9. **Completion** — On `done`, the agent's final result is appended to the session, a `{"type": "final", ...}` event is queued, and the extension speaks the result aloud (`SpeechSynthesisUtterance`) if voice output is enabled, then refreshes the session sidebar.
10. **Cleanup** — In the `finally` block, if the browser was *attached* to your existing Chrome, only the debugging session is detached (`browser.stop()`) so your browser stays open; if it was a *launched* copy, the whole instance is killed (`browser.kill()`).

---

## Detailed Feature Breakdown

### Dual-LLM cost & reliability engine
`config/llm_factory.py` doesn't just pick one model — it builds a **primary/fallback pair** every request. Gemini Flash is the default primary (Gemini Pro is gated behind `GEMINI_ALLOW_PRO=true` since the free tier commonly has a `0` request limit for Pro). Ollama (`qwen2.5-coder:3b` for "Fast" mode, `qwen2.5-coder:7b` for "Smart" mode) is *always* wired in as the fallback and as the dedicated `page_extraction_llm`, since DOM-reading calls are frequent and cheaper to run locally. Setting `PREFER_OLLAMA=true` skips Gemini entirely for a fully offline, zero-API-key mode.

### Live NDJSON action streaming
Rather than a single blocking request/response, `/run-agent-stream` uses newline-delimited JSON over a chunked HTTP response. The frontend reads the stream incrementally with `response.body.getReader()`, splitting on `\n` and parsing each line independently — meaning the chat UI shows the agent's reasoning ("Step 3 → clicking search button") *while it's still working*, not after the fact.

### Dual browser-attachment strategy
`discover_cdp_url()` actively probes for a Chrome instance already running with remote debugging enabled — checking an explicit `CHROME_CDP_URL` override, then `browser-use`'s own discovery helper, then each Chrome user-data directory's `DevToolsActivePort` file, then a raw port scan of 9222–9232. This lets the agent reuse your *existing, logged-in* browser tab instead of spinning up a sterile new profile — critical for tasks that need existing session cookies (e.g., "check my Gmail inbox").

### File-based, dependency-free session persistence
`session_store.py` implements a minimal but functional chat-history system using plain JSON files (`data/sessions/*.json` + an `index.json` manifest) instead of a database — zero setup, human-readable, and trivially portable. Each session auto-titles itself from the first 48 characters of the user's first message.

### Voice input/output with graceful text fallback
Voice input uses the browser-native Web Speech API (`SpeechRecognition`), so no audio is ever sent to the backend — transcription happens entirely client-side. If the browser doesn't support it, the microphone button is hidden (`micBtn.style.display = "none"`) and the text input remains the sole, fully-functional entry point. Voice *output* uses `SpeechSynthesisUtterance`, is toggleable, and is automatically cancelled when a new response starts or the user stops the agent.

### Multi-language task execution
The `language` field is passed straight into the agent's system prompt (`"Language: '{data.language}'."`), letting the same task phrasing work across English, Spanish, and Hindi — the underlying LLM interprets and responds according to the selected language rather than the extension performing any local translation.

### Cooperative cancellation
`/stop-agent` doesn't just drop the connection — it calls `agent.stop()` on the active `browser-use` agent, cancels the underlying `asyncio.Task`, and (for the Claude Code integration path) sends `CTRL_BREAK_EVENT`/`SIGINT` to any spawned Claude CLI subprocess, ensuring in-flight browser actions and LLM calls are interrupted cleanly rather than orphaned.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Browser Extension | Manifest V3, vanilla JavaScript, HTML/CSS (glassmorphism UI, no build tooling) |
| Voice I/O | Web Speech API (`SpeechRecognition`, `SpeechSynthesis`) |
| Backend Framework | FastAPI + Uvicorn (ASGI) |
| Agent Runtime | `browser-use` (Agent + Browser abstraction over Chrome DevTools Protocol) |
| Primary LLM | Google Gemini (`gemini-2.5-flash` / `gemini-2.5-pro`) via `ChatGoogle` |
| Fallback / Local LLM | Ollama (`qwen2.5-coder:3b` / `qwen2.5-coder:7b`) via `ChatOllama` |
| Browser Control Protocol | Chrome DevTools Protocol (CDP) |
| Persistence | Flat-file JSON session store (no external database) |
| Streaming Transport | NDJSON over chunked HTTP (`StreamingResponse`) |
| Config Management | `python-dotenv` + a Claude Code/Ollama environment bridge |
| Language | Python 3.13 (backend), JavaScript ES2020+ (frontend) |

---

## Troubleshooting & Verification

Use this section to systematically verify each layer of the stack when something isn't working. Start from the top (extension) and work down (browser) — most issues are visible in the `/health` endpoint or the extension's status pill before you need to touch code.

### 1. Quick system verification
Run this first for any issue:
```
curl http://127.0.0.1:8000/health
```
A healthy response looks like:
```json
{
  "status": "ok",
  "primary": "google",
  "gemini_configured": true,
  "ollama_reachable": true,
  "fallback_enabled": true,
  "agent_running": false
}
```
If `status` is `"degraded"`, neither Gemini nor Ollama is usable — see sections 2 and 3 below.

### 2. Gemini / API key issues
- **Symptom:** `"gemini_configured": false`, or error mentioning `GEMINI_API_KEY`.
- **Fix:** Confirm `GEMINI_API_KEY=your_key_here` exists in `.env` at the project root with no quotes or trailing spaces, then restart the server (env vars are loaded once at boot via `load_dotenv()`).
- **Symptom:** Error containing `429`, `resource_exhausted`, or `quota`.
- **This is expected behavior, not a bug** — the agent should auto-switch to Ollama on the next step. Confirm this occurred by checking the response for `"fallback_triggered": true`.

### 3. Ollama / local model issues
- **Symptom:** `"ollama_reachable": false`, or error `Cannot reach Ollama`.
- **Fix:**
```
  ollama serve
  ollama pull qwen2.5-coder:7b
  ollama pull qwen2.5-coder:3b
```
  Confirm `OLLAMA_HOST` in `.env` matches where Ollama is actually listening (default `http://127.0.0.1:11434`).
- **Symptom:** Error `Ollama model not found` / `404`.
- **Fix:** The model tag you selected in the extension ("Fast"/"Smart") hasn't been pulled locally. Run `ollama pull <model-name>` for the tag reported in the error.

### 4. Chrome / CDP attachment failures
- **Symptom:** Server error mentioning `no_cdp`, or `RuntimeError("Chrome is open but remote debugging is off...")`.
- **Fix:** Close all Chrome windows completely, then launch `scripts\start_chrome_for_agent.cmd` instead of the normal Chrome shortcut — this starts Chrome with `--remote-debugging-port=9222` on your Default profile.
- **Fix:** If you'd rather not modify how you launch Chrome, set `CHROME_MODE=launch` in `.env` to force the agent to spin up its own Chrome copy using your profile directory instead of attaching to a running one.
- **Symptom:** Agent launches Chrome but pages look wrong / extensions missing.
- **Fix:** Confirm `CHROME_PROFILE_DIRECTORY` in `.env` matches the profile folder name you actually use (check `chrome://version` → "Profile Path").

### 5. Extension shows "Server offline"
- Confirm `server.py` is actually running (`scripts\start_server.cmd`) and bound to `127.0.0.1:8000` — the extension is hardcoded to call `http://127.0.0.1:8000`.
- Confirm no firewall/antivirus rule is blocking local loopback traffic on port 8000.
- Open the extension's popup DevTools (right-click popup → Inspect) and check the Network tab for the actual failing request/status code.

### 6. Microphone / voice input not working (hardware input failure path)
This is the most common "it just doesn't respond to my voice" report, and it should be diagnosed in this order:

1. **Browser-level permission.** Voice capture uses the Web Speech API inside the extension popup, which requires microphone permission at the *browser* level. Go to `chrome://settings/content/microphone` and confirm the extension/site is not blocked, and that a default microphone is selected.
2. **OS-level privacy permissions.** The browser cannot access a microphone the operating system is blocking:
   - **Windows:** Settings → Privacy & security → Microphone → ensure "Microphone access" and "Let apps access your microphone" are both On, and that Google Chrome specifically is allowed.
   - **macOS:** System Settings → Privacy & Security → Microphone → ensure Google Chrome is checked.
3. **Correct input device selected.** Unlike a native desktop audio pipeline, this project does **not** manage its own device index — microphone selection is delegated entirely to the OS/browser default input device. Check your OS sound settings (Windows Sound Settings → Input, or macOS Sound → Input) and confirm the intended physical microphone is set as the *default* device, since the browser will only ever request the system default.
4. **Known MV3 popup limitation.** Chrome extension popups close on loss of focus, which can interrupt `SpeechRecognition` sessions mid-capture in some Chrome versions. If recognition starts (mic button turns red) but never returns a transcript, this is the likely cause — click the microphone button again to retry, or use text input instead.
5. **Graceful fallback is automatic and by design.** If `window.SpeechRecognition`/`window.webkitSpeechRecognition` is unavailable in the browser at all, `popup.js` detects this at load time and hides the microphone button entirely (`micBtn.style.display = "none"`), leaving the text input field as the fully-functional primary input path. **No task functionality is lost** — every capability reachable by voice is equally reachable by typing the same instruction into the input field. If voice capture is unreliable in your environment, simply use text input as the supported fallback rather than troubleshooting further.

### 7. Voice output (text-to-speech) not working
- Confirm the 🔊 "Voice ON" toggle in the header is active (not toggled to "🔇 Voice OFF").
- `SpeechSynthesisUtterance` relies on the OS having at least one installed TTS voice for the selected language (`en-US`/`es-ES`/`hi-IN`) — check your OS's speech/accessibility settings if audio never plays despite the toggle being on.

### 8. Session history not persisting
- Confirm the process running `server.py` has write permission to `data/sessions/` in the project directory.
- If `data/sessions/index.json` becomes corrupted (e.g., from a forced process kill mid-write), delete `index.json` only — individual session files are independent and will be re-indexed as sessions are re-created; existing per-session `.json` files can be manually re-added to a fresh index if needed.

---

## Future Roadmap

- **Multi-tab orchestration** — allow a single task to coordinate actions across multiple browser tabs concurrently.
- **Vision-fallback mode** — opt-in `use_vision=True` path for pages where DOM-only reasoning fails (canvas-heavy or heavily obfuscated sites).
- **Encrypted session storage** — replace flat JSON with an encrypted-at-rest store for sensitive chat history.
- **Additional language packs** — expand beyond EN/ES/HI to cover more Web Speech API-supported locales.
- **Firefox/Edge parity** — port the MV3 extension to Firefox's WebExtensions API.
- **Packaged installer** — bundle backend + Ollama setup into a single installer instead of manual `.cmd` scripts.
- **Usage/cost dashboard** — visualize Gemini-vs-Ollama call ratios and estimated API spend over time.
- **Streaming token-level output** — move from step-level NDJSON events to token-level streaming for smoother perceived latency.

---

## License

This project is provided as-is for educational and personal automation use. See `LICENSE` for details.
