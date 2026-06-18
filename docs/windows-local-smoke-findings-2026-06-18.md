# Windows Local Smoke Findings (2026-06-18)

This note captures the confirmed findings from a native Windows PowerShell boot
and authenticated browser smoke test of Odysseus on June 18, 2026.

## Test setup

- Launch path: native Windows `venv` + `python -m uvicorn app:app --host 127.0.0.1 --port 7000`
- Browser check: Playwright CLI against `http://127.0.0.1:7000`
- Authenticated user: `admin`

## Confirmed findings

### 1. Login page triggers authenticated-only requests before login

The public `/login` page performs requests that require authentication on first
paint, which produces guaranteed `401` errors in the browser console before the
user signs in.

Observed requests:

- `GET /api/auth/policy`
- `GET /api/prefs/theme`
- `GET /api/prefs/custom-themes`

Relevant code:

- [static/login.html](../static/login.html) fetches auth policy at line 366.
- [static/js/theme.js](../static/js/theme.js) fetches theme prefs at lines 480
  and 2067.
- [app.py](../app.py) auth exemptions at lines 203-216 do not include those
  endpoints.

Impact:

- Noisy console on the public login page
- Misleading auth failures during normal first-load behavior

### 2. Saved chat sessions produce noisy `404` polling errors after login

After login, the main chat workspace polls research and stream status for a
normal saved session and receives `404` responses, which surface as console
errors even though the UI continues working.

Observed requests:

- `GET /api/research/status/<session_id>` -> `404`
- `GET /api/chat/stream_status/<session_id>` -> `404`

Relevant code:

- [static/js/sessions.js](../static/js/sessions.js) polls research status at
  line 2044.
- [static/js/sessions.js](../static/js/sessions.js) polls chat stream status at
  line 2156.
- [routes/research_routes.py](../routes/research_routes.py) returns `404` for
  missing research status at lines 170-173.
- [routes/chat_routes.py](../routes/chat_routes.py) returns `404` for no active
  stream at line 1458.

Impact:

- Browser console shows errors during normal chat-page load
- Network panel looks unhealthy even when the workspace is usable

### 3. `/api/ready` is not usable as an unauthenticated readiness probe

With auth enabled, `GET /api/ready` returns `401` before login, while
`GET /api/health` succeeds.

Relevant code:

- [app.py](../app.py) defines `/api/ready` at lines 868-877.
- [app.py](../app.py) auth exemptions at lines 203-216 include `/api/health`
  but not `/api/ready`.

Observed behavior:

- `/api/health` -> `200`
- `/api/ready` -> `401` until authenticated

Impact:

- Readiness checks cannot be used by unauthenticated local/service probes
- The route behaves more like an authenticated diagnostics endpoint than a
  public readiness endpoint

## Windows-specific startup caveats

The app booted and the main authenticated UI was usable, but vector-backed
features started in a degraded state because ChromaDB was not running locally.

Observed startup degradation:

- `VectorRAG` unavailable
- `MemoryVectorStore` unavailable
- Tool index warmup failed

Impact:

- Chat, Calendar, Notes, and Settings were still usable
- Vector-backed RAG/memory behavior was not a full-fidelity native test in this
  run

## What worked

The following paths were verified as working during this smoke test:

- Native Windows boot completed successfully
- Login with the provided admin credentials succeeded
- Main chat workspace loaded
- Calendar modal opened and rendered
- Notes modal opened and rendered
- Settings modal opened and rendered

## Suggested next fixes

1. Make the login page avoid authenticated-only fetches before sign-in, or make
   those specific reads fail quietly without console noise.
2. Treat "no active stream" and "no research for this session" as expected
   states in the client polling flow instead of noisy console errors.
3. Decide whether `/api/ready` should be public like `/api/health` or be
   renamed/reframed as an authenticated diagnostics route.
