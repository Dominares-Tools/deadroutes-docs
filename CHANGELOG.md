# Changelog

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show deadroutes`, `deadroutes.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (Dominares Tools Commercial License, version 1.3).
- One public contact address, support@dominares.org, in the licence and the README.

## 1.0.0 (2026-09-30)

First release.

- `deadroutes.install(app)` / `DeadRoutesMiddleware`: plain ASGI middleware for FastAPI and
  Starlette that counts every HTTP request, reads included, keyed by method, ROUTE TEMPLATE (never
  the raw path) and caller. Mounted sub-applications and included routers resolve to their full
  template; requests that match no route share one `(unmatched)` key.
- Bounded by design. `max_keys` bounds distinct `(method, route)` pairs and
  `max_callers_per_route` bounds named callers per route; the rest share `(other callers)`, so a
  flood of user agents never fills the table. HTTP methods outside the standard verbs are filed as
  `OTHER`. Only a NEW pair past the cap is dropped: `dropped_keys` goes up, known keys keep
  counting, and reports refuse to judge until the store is reset. User agents are scrubbed of
  credentials, addresses and telephone numbers and truncated; `record_callers = false` keeps none.
- `install(app)` reads `deadroutes.toml` (or `config_path=`) for the counting settings. A request
  that matches only a Starlette `Mount` prefix is counted as unmatched, never as a route.
- Nothing slow on the request path: a dictionary update under a lock, flushed by a timer, at
  application shutdown and at interpreter exit. A failing store is logged once and retried, never
  raised into a request. Fork-safe.
- `SQLiteStore` (standard library) is safe across worker processes, is read without being written
  to (a read-only copy works; another program's SQLite file is refused); `MemoryStore` for tests; any
  object with `add`, `snapshot` and `reset` can stand in.
- Every report carries `observing_since`, `observed_days` and `safe_to_judge` (minimum window
  `min_observed_days`, 30 by default). A counter that stopped (`stale_after_days`) is never safe to
  judge, and a quiet app writes a heartbeat so it is not mistaken for one that stopped.
- `deadroutes scan`: reads the route table from `--app module:app`, falling back to parsing
  decorators with `ast`; reads `--frontend` globs (JS, TS, HTML, templates) and `server` Python
  files; classifies each route as called by the front end (strings, template literals, `fetch` and
  `axios` URLs, `url_for`, path parameters matched), machine-only (configured patterns, each with a
  reason) or unreferenced. Prefixes are configuration. `--max-unreferenced` is a ratchet for CI.
  The method is read where the code states it (`axios.delete`, `fetch(url, {method: 'PUT'})`,
  `<form method>`); commented-out calls are ignored; the source fallback keeps same-named modules
  in different folders apart and adds the `FastAPI()` documentation routes as machine-only.
- `deadroutes report`: joins the scan with the counted traffic; lists candidates to review only
  over a mature window, and lists unreferenced routes that are called, by caller. It never says
  "delete". It also lists routes the code names that had no calls, so a dead `DELETE` beside a
  used `GET` stays visible.
- `deadroutes window-check`: exit 0 / 1 / 2 / 3 for CI, and `examples/github-action.yml`, a weekly
  workflow template.
- `UsageEndpoint`: serves the counts as JSON behind a bearer token, for CI to read.
- `deadroutes usage`, `deadroutes reset --yes`, `deadroutes demo DIR`. A `--store` flag wins over
  `$DEADROUTES_STORE`.
- Settings from `deadroutes.toml`, `deadroutes.json` or `[tool.deadroutes]`; unknown keys are
  errors.
