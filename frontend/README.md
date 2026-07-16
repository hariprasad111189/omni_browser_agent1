# Frontend — Omni Browser Agent Chrome Extension

A Manifest V3 Chrome extension that provides the chat interface for controlling the backend agent — no build step, no framework, just HTML/CSS/vanilla JS.

## Developer

Shyamala Hari Prasad

---

## UI Layout

The popup (`popup.html`) is a fixed 440×560px panel structured as:

```
┌─────────────────────────────────────────┐
│ HEADER                                    │
│  ⚡ Omni Copilot        [status pill]     │
│  [Fast/Smart ▾] [EN/ES/HI ▾] [🔊 Voice]   │
├───────────┬─────────────────────────────┤
│ SIDEBAR    │ CHAT AREA                    │
│ History    │  ┌ message bubbles ┐        │
│ + New chat │  │  (user/agent/   │        │
│ [session]  │  │   action/system)│        │
│ [session]  │  └─────────────────┘        │
│            │  [typing indicator]         │
│            ├─────────────────────────────┤
│            │ [🎤] [ text input ] [➔/■]   │
└───────────┴─────────────────────────────┘
```

- **Header** — brand, a live connection status pill (Gemini+Ollama / Gemini only / Ollama only / No LLM / Offline), a model-speed selector, a language selector, and a voice-output toggle.
- **Sidebar** — lists all persisted chat sessions (most recently updated first), a "+" button to start a new chat, and highlights the currently active session.
- **Chat area** — renders four distinct message types with distinct styling: `user` (right-aligned), `agent` (left-aligned, with a "via gemini (model)" or "↩️ Ollama fallback" meta line), `action` (monospace, green-tinted live step feed), and `system` (centered, muted status text).
- **Input row** — a microphone button (hidden automatically if the browser lacks Web Speech API support), the text input, and a send button that morphs into a stop button (`■`) while the agent is running.

## State Rendering & Management

The extension keeps everything in plain module-level JS variables — there is no framework or state library:

- `isRunning` — gates whether the send button submits or stops, disables the input/model selector, and shows/hides the typing indicator.
- `activeSessionId` — tracks which backend session the current chat maps to; updated whenever a session is created, switched, or returned in a `final` streaming event.
- `speakEnabled` — toggled by the 🔊/🔇 button; gates whether `speakText()` fires after a response.
- `activeRequestController` — an `AbortController` tied to the in-flight `fetch`, allowing `stopAgent()` to cancel the network request client-side in addition to telling the backend to stop.
- `recognition` — the `SpeechRecognition` instance (or `null` if unsupported), lazily driven by the microphone button's click handler.

**Rendering pattern:** the DOM is never diffed — `renderMessages()` clears `chatBox.innerHTML` and rebuilds it from a `messages` array whenever a session is loaded or switched; new messages during an active run are appended directly via `appendMessage()` without a full re-render, for responsiveness.

**Streaming consumption:** `runAgent()` opens `response.body.getReader()` against `/run-agent-stream`, maintains a rolling text `buffer`, splits on `\n`, and JSON-parses each complete line as it arrives — rendering `action` events immediately and holding the `final` event until the stream ends, at which point it renders the agent's answer, speaks it aloud if enabled, and refreshes the session sidebar.

## Running Locally

### 1. Ensure the backend is running first
The extension talks to `http://127.0.0.1:8000` (hardcoded in `popup.js`) — start the backend server before loading the extension. See `backend/README.md`.

### 2. Load the unpacked extension in Chrome
1. Open `chrome://extensions`.
2. Enable **Developer mode** (top-right toggle).
3. Click **Load unpacked**.
4. Select the `frontend/extension/` folder (the one containing `manifest.json`).
5. Pin the "Omni Browser Assistant" icon to your toolbar for quick access.

### 3. Use it
- Click the extension icon to open the popup.
- Confirm the status pill shows something other than "Offline" — this reflects the backend's `/health` response.
- Type (or click 🎤 and speak) a task, choose Fast/Smart and your language, and press send.
- Watch live step-by-step actions stream into the chat as green monospace lines, followed by the agent's final answer.
- Use the sidebar to revisit previous sessions; use "+" to start a new one.

### 4. Reloading after changes
Any edit to `popup.html`, `popup.js`, or `manifest.json` requires clicking the refresh icon on the extension's card in `chrome://extensions`, then reopening the popup.

### Known limitation
Chrome extension popups can lose focus and close abruptly, which occasionally interrupts an in-progress `SpeechRecognition` capture. If this happens, simply retry the microphone button or type the request instead — the text path is fully equivalent in capability and is the supported fallback if voice input is unreliable in your environment.
