# Backend — Omni Browser Agent Server

The backend is a local-only FastAPI application that turns natural-language prompts into autonomous, real-time browser actions. It owns LLM selection, browser lifecycle management, and chat session persistence.

## Developer

Shyamala Hari Prasad

---

## Architecture

### Component responsibilities

| File | Responsibility |
|---|---|
| `server.py` | FastAPI app: HTTP surface, browser lifecycle (`create_browser`), agent execution (`_execute_agent`), NDJSON streaming, cancellation |
| `config/env_sync.py` | Injects/normalizes environment variables needed both by the browser agent and by an optional local Claude Code CLI integration (routes it to Ollama instead of Anthropic's cloud API) |
| `config/llm_factory.py` | Constructs the primary (Gemini) + fallback (Ollama) LLM pair per request, based on `model_choice` and available API keys |
| `config/session_store.py` | Flat-file JSON persistence for chat sessions — no database dependency |
| `config/step_format.py` | Converts a `browser-use` agent step's `next_goal` and `action` list into a readable string for the live activity feed |
| `legacy/agent_backend.py` | An earlier, standalone CLI prototype (LangChain + `BrowserConfig`) kept for reference; **not used by `server.py`** and not wired into the FastAPI app |

### Request lifecycle (server-side)

1. `POST /run-agent-stream` receives `{ prompt, model_choice, language, session_id }`.
2. `_execute_agent()` resolves/creates the chat session via `session_store`, appends the user message.
3. `llm_factory.build_llm_pair(model_choice)` returns `(primary_llm, fallback_llm, primary_provider)`.
4. `create_browser()` attempts, in order: attach to an already-running Chrome via CDP → launch a new Chrome instance that copies your profile → raise a descriptive error if Chrome is open with debugging disabled.
5. A `browser-use` `Agent` is constructed with the task, LLM pair, a fixed system prompt, and tuned execution parameters (`max_steps=25`, `step_timeout=90`, `max_actions_per_step=8`, `flash_mode=True`, vision/planning/judge features disabled for speed).
6. Each agent step triggers `step_callback`, which formats the step and pushes it onto an `asyncio.Queue` that the streaming response generator drains and yields as NDJSON.
7. On completion, the final result is persisted to the session and returned with metadata: which LLM actually answered (`llm_used`), the specific model (`model_used`), and whether a fallback occurred (`fallback_triggered`).
8. The `finally` block detaches (if Chrome was attached to) or fully kills (if Chrome was launched) the browser handle.

### API Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | Reports Gemini key presence, Ollama reachability, and whether an agent is currently running |
| `GET` | `/sessions` | Lists all chat sessions (sorted by last update) and the currently active session ID |
| `POST` | `/sessions` | Creates a new chat session |
| `GET` | `/sessions/{id}` | Retrieves full message history for one session |
| `POST` | `/sessions/{id}/activate` | Marks a session as active (used when switching in the sidebar) |
| `POST` | `/run-agent` | Runs the agent to completion, returns a single JSON result (non-streaming) |
| `POST` | `/run-agent-stream` | Runs the agent, streaming NDJSON `action` events followed by one `final` event |
| `POST` | `/stop-agent` | Cancels the in-flight agent task and interrupts any spawned Claude CLI subprocess |

---

## Local Installation & Setup

### Prerequisites
- Python 3.11+ (developed against 3.13)
- Google Chrome installed
- (Optional, for offline/fallback mode) [Ollama](https://ollama.com) installed and running locally

### 1. Create and activate a virtual environment
```
cd backend
python -m venv .venv
.venv\Scripts\activate        # Windows
source .venv/bin/activate     # macOS/Linux
```

### 2. Install dependencies
```
pip install -r requirements.txt
```
This installs `browser-use`, `fastapi`, `python-dotenv`, and `uvicorn[standard]`. `browser-use` will also pull in the Chromium automation stack it depends on.

### 3. Configure environment variables
Copy `.env.example` to `.env` and fill in your key:
```
GEMINI_API_KEY=your_key_here
```
Optional overrides (all have sane defaults if omitted):
```
PREFER_OLLAMA=false
GEMINI_MODEL_FLASH=gemini-2.5-flash
GEMINI_MODEL_PRO=gemini-2.5-pro
GEMINI_ALLOW_PRO=false
OLLAMA_HOST=http://127.0.0.1:11434
OLLAMA_MODEL_FAST=qwen2.5-coder:3b
OLLAMA_MODEL_PRO=qwen2.5-coder:7b
CHROME_MODE=attach
CHROME_PROFILE_DIRECTORY=Default
BROWSER_HEADLESS=false
CHROME_CDP_URL=
```

### 4. (Optional) Pull local Ollama models for fallback
```
ollama pull qwen2.5-coder:3b
ollama pull qwen2.5-coder:7b
```

### 5. Start Chrome with remote debugging enabled
Close all Chrome windows, then run:
```
scripts\start_chrome_for_agent.cmd
```
This launches Chrome on `--remote-debugging-port=9222` using your existing Default profile, so the agent can attach to your logged-in session rather than a blank one.

### 6. Start the server
```
scripts\start_server.cmd
```
or directly:
```
python server.py
```
The server starts on `http://127.0.0.1:8000`. Verify with:
```
curl http://127.0.0.1:8000/health
```

### 7. (Optional) Claude Code + local Ollama bridge
`scripts\start_claude_ollama.cmd` launches the Claude Code CLI configured to route its Anthropic API calls to your local Ollama instance instead of Anthropic's cloud (via `ANTHROPIC_BASE_URL=http://localhost:11434`). This is unrelated to the browser agent's own LLM calls — it's a convenience for developers who also use Claude Code locally and want it running against the same local model, at no API cost.

### Deployment note
This backend is designed for **local, single-user use** — it binds to `127.0.0.1`, has CORS wide open (`allow_origins=["*"]`) for the extension's convenience, and stores sessions as unencrypted local JSON. Do not expose this server directly to the internet without adding authentication, restricting CORS, and reviewing the CDP-attach behavior (which grants control over your live, logged-in browser).
