# Dead Code Audit Findings

Date: 2026-06-18

Scope: trim/removal candidates only. No behavioral changes were made in this pass.

## Findings

### 1. Legacy unused code-runner copy helper

1. Description: `static/js/codeRunner.js` keeps a retired helper, `addCopyBtn_unused()`, that is explicitly described as legacy and no longer participates in the current code path.
2. Why it appears low-value: it is not exported, not referenced elsewhere, and the surrounding comment says the inline copy UI replaced it.
3. Files/lines involved:
   - `static/js/codeRunner.js:90`
   - `static/js/codeRunner.js:93`
4. Removal risk: low
5. Recommended follow-up action: remove now

### 2. Dead email poller task slot

1. Description: `routes/email_pollers.py` still defines `_summarize_task` even though the module comment now says the legacy auto-summary/reply poller no longer starts here.
2. Why it appears low-value: `_summarize_task` is only initialized to `None`, listed in a `global`, and reset to `None` again. It does not drive behavior.
3. Files/lines involved:
   - `routes/email_pollers.py:1090`
   - `routes/email_pollers.py:1119`
   - `routes/email_pollers.py:1124`
4. Removal risk: low
5. Recommended follow-up action: remove now

### 3. Startup memory load runs twice

1. Description: `static/app.js` loads memories immediately on startup, then schedules the same call again one second later.
2. Why it appears low-value: this looks like an old rendering-race workaround rather than a clean dependency. It duplicates work on page load and is not guarded by any state check.
3. Files/lines involved:
   - `static/app.js:3939`
   - `static/app.js:3942`
4. Removal risk: medium
5. Recommended follow-up action: confirm usage first

### 4. Skills tab has both eager and lazy boot paths

1. Description: the Skills UI is loaded and fetched through multiple overlapping paths.
2. Why it appears low-value: `index.html` eagerly loads `skills.js`, `skills.js` auto-fetches on `DOMContentLoaded`, and `memory.js` still lazy-imports `skills.js` again when the Skills tab opens. That is duplicate loader logic and likely duplicate network/UI work.
3. Files/lines involved:
   - `static/index.html:2431`
   - `static/js/skills.js:1963`
   - `static/js/skills.js:1965`
   - `static/js/memory.js:1453`
   - `static/js/memory.js:1455`
4. Removal risk: medium
5. Recommended follow-up action: merge duplicates

### 5. Chat module is loaded under two different ES-module specifiers

1. Description: the page loads `chat.js` directly with a cache-busting query string, while `app.js` imports the plain specifier.
2. Why it appears low-value: browsers treat `/static/js/chat.js?v=...` and `./js/chat.js` as distinct modules, so this can create double evaluation and split module state. This is especially suspicious because `chat.js` mutates `window.chatModule`.
3. Files/lines involved:
   - `static/index.html:2447`
   - `static/app.js:13`
   - `static/js/chat.js:5016`
   - `static/js/chat.js:5017`
4. Removal risk: medium
5. Recommended follow-up action: merge duplicates

### 6. Permanent modal auto-wire polling loop

1. Description: `static/js/modalManager.js` keeps a `setInterval(..., 1000)` loop running to scan the DOM for modal wiring opportunities.
2. Why it appears low-value: it is an always-on polling path in a frontend that otherwise uses event-driven wiring heavily. It may still be needed, but it is a strong cleanup/perf candidate.
3. Files/lines involved:
   - `static/js/modalManager.js:1462`
   - `static/js/modalManager.js:1463`
4. Removal risk: medium
5. Recommended follow-up action: confirm usage first

### 7. Real duplicate research handler implementation

1. Description: `services/research/research_handler.py` is a second real implementation, not a compatibility alias, and it has already drifted from `src/research_handler.py`.
2. Why it appears low-value: the `services` layer instantiates the `services` copy directly, while app startup wires the `src` copy. The copies differ materially in validation, probing, formatting, persistence, and timeout handling.
3. Files/lines involved:
   - `services/research/service.py:8`
   - `services/research/service.py:45`
   - `services/research/research_handler.py:1`
   - `services/research/research_handler.py:235`
   - `services/research/research_handler.py:333`
   - `src/app_initializer.py:22`
   - `src/research_handler.py:24`
   - `src/research_handler.py:35`
   - `src/research_handler.py:738`
4. Removal risk: medium
5. Recommended follow-up action: merge duplicates

### 8. Likely orphaned config subsystem with import-time side effects

1. Description: `src/config.py` appears to be a largely unused configuration layer that still runs initialization side effects at import time.
2. Why it appears low-value: `app.py` imports `config` but does not read from it. `src/config.py` creates directories and validates config during import, while `src/app_initializer.py` duplicates directory creation separately. I did not find real runtime consumers of `config.data`, `config.llm`, `config.search`, or `config.security` outside `src/config.py`.
3. Files/lines involved:
   - `app.py:513`
   - `src/config.py:178`
   - `src/config.py:181`
   - `src/config.py:194`
   - `src/config.py:210`
   - `src/app_initializer.py:28`
   - `src/app_initializer.py:43`
4. Removal risk: medium
5. Recommended follow-up action: confirm usage first

### 9. Compatibility shims that are now mostly migration ballast

1. Description: several modules are now pure compatibility shims rather than real implementations.
2. Why it appears low-value: these files only exist to preserve older import paths. They are not harmful right now, but they are structural cleanup candidates once callers are normalized.
3. Files/lines involved:
   - `core/constants.py:2`
   - `core/constants.py:11`
   - `src/search/core.py:1`
   - `src/search/providers.py:1`
   - `src/search/content.py:1`
   - `src/search/query.py:1`
   - `src/search/cache.py:1`
   - `src/search/analytics.py:1`
   - `src/search/ranking.py:1`
   - `src/youtube_handler.py:1`
   - `tests/test_search_module_consolidation.py:10`
   - `tests/test_search_module_consolidation.py:38`
4. Removal risk: medium
5. Recommended follow-up action: confirm usage first

### 10. Stale JS module inventory doc

1. Description: `static/js/MODULE_SUMMARY.md` is explicitly marked as partial and historical, and its dependency-order section no longer reflects the current ES-module bootstrap.
2. Why it appears low-value: the document admits it is not authoritative, and the listed script ordering is from an older non-current loading model. That makes it maintenance clutter rather than reliable documentation.
3. Files/lines involved:
   - `static/js/MODULE_SUMMARY.md:6`
   - `static/js/MODULE_SUMMARY.md:10`
   - `static/js/MODULE_SUMMARY.md:111`
   - `static/js/MODULE_SUMMARY.md:113`
4. Removal risk: low
5. Recommended follow-up action: remove now

### 11. Assistant module still self-boots a polling watcher after its sidebar wiring was removed

1. Description: `assistant.js` says its former sidebar wiring is gone and that Tasks now owns the UI surface, but the module still self-starts a periodic watcher that polls for the active assistant session and injects a gear button into the chat header.
2. Why it appears low-value: this may still be useful, but it is another legacy-style polling path living beside newer Tasks-owned flows. It is a good cleanup candidate if the assistant affordance can be attached from session/task lifecycle events instead.
3. Files/lines involved:
   - `static/js/assistant.js:416`
   - `static/js/assistant.js:442`
   - `static/js/assistant.js:444`
   - `static/js/assistant.js:460`
4. Removal risk: medium
5. Recommended follow-up action: confirm usage first

## Brief Summary

Highest-confidence removals:

- `static/js/codeRunner.js:addCopyBtn_unused()`
- `routes/email_pollers.py:_summarize_task`
- `static/js/MODULE_SUMMARY.md`

Highest-value consolidation targets for a scoped cleanup PR:

- unify `chat.js` bootstrap so the app does not load the same module under two specifiers
- collapse eager and lazy Skills loading into one path
- merge `services/research/research_handler.py` back to a single research handler
- decide whether `src/config.py` is still a real runtime dependency or just legacy import-time scaffolding

Lower-priority structural cleanup after import normalization:

- retire shim layers such as `core/constants.py`, `src/search/*`, and `src/youtube_handler.py`
- replace or remove polling-style frontend watchers where lifecycle/event hooks now exist
