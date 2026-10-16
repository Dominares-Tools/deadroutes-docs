# deadroutes — Find the API endpoints nobody calls

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/deadroutes**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

deadroutes counts every request your FastAPI or Starlette app serves, by route template, and reads your route table and front-end files to see which endpoints your own code names. It joins the two and lists the routes nothing references and nobody called over a long enough window as candidates to review, never as something to delete.

## Get it

Buy deadroutes at https://dominares.org/tools/deadroutes: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the deadroutes command, with an example project, a weekly CI workflow template and the full test suite.

Requirements: A FastAPI or Starlette app to count requests in. The [fastapi] and [starlette] extras install the framework.

This page documents version 1.0.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

**Find the API endpoints nobody calls, with evidence before you remove anything.**

A FastAPI or Starlette backend that has grown for a year or two, especially one extended by AI
coding agents, collects endpoints. Nobody remembers who calls half of them. Deleting one on a
hunch is how a partner's nightly script, a mobile app release from last spring or the monthly
finance export stops working, and you hear about it from them.

deadroutes gives you the two pieces of evidence that make a removal reviewable:

1. **What your own code says.** It reads your route table and your front-end files (JavaScript,
   TypeScript, HTML, templates) and sorts every route into one of three buckets: *called by the
   front end* (a string, a template literal, a `fetch` or `axios` URL, path parameters matched),
   *machine-only* (webhooks, health checks: patterns you configure) or *unreferenced*.
2. **What production actually did.** A small ASGI middleware counts every request, reads
   included, by route template, into a store on disk. It adds nothing slow to the request path.

`deadroutes report` joins the two and lists the routes that are unreferenced **and** had zero
calls over a window long enough to mean something. They are *candidates to review*. The tool never
says "delete", and every report states how long it has been watching, because "zero calls" on day
one means nothing:

```
24 routes; 311 requests counted since 2026-08-22T01:12:41Z (40 days observed, 30 required). Safe to judge: yes

Candidates to review (no reference in the scanned code, zero calls in 40 days): 4
  DELETE  /api/orders/{order_id}/notes/{note_id}    declared at shopapp.orders.delete_note
  POST    /api/queue/{item_id}/escalate    declared at shopapp.queue.escalate
  GET     /api/reports/monthly    declared at shopapp.reports.monthly
  GET     /v2/ping    declared at shopapp.main.ping
  Before changing any of these, ask what each one is for. A yearly export, a disaster-recovery route or a caller you do not know about can look exactly like this.

Unreferenced, but called (a caller outside the scanned code): 1
  GET     /api/reports/legacy-export  9 calls, last 2026-10-01T01:12:41Z  by partner-sync/2.3 (9)
```

That last line is the point: no file in the repository names `legacy-export`, yet something calls
it every few days, by name. A scan alone would have listed it as dead.

**Who it is for:** teams with a sprawling FastAPI or Starlette backend who want proof before they
remove endpoints, and who cannot afford to break a caller they do not know about.

The runtime is the Python standard library only (3.10 or newer; on 3.10 the one dependency,
`tomli`, installs itself). FastAPI and Starlette are needed only by your own app, and by `--app`
when the scan imports it.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

### 60-second quickstart

`deadroutes demo` writes a small shop API, its front end and a settings file into a new folder.
Run these from the folder where you installed it:

```bash
deadroutes demo shop-demo
cd shop-demo
deadroutes scan
```

The scan lists 24 routes: 8 referenced by the front end, 11 machine-only (two webhooks, a health
probe and the framework's own documentation routes), and 5 unreferenced:

```
Unreferenced: nothing in the scanned code names these: 5
  DELETE  /api/orders/{order_id}/notes/{note_id}    declared at shopapp.orders.delete_note
  POST    /api/queue/{item_id}/escalate    declared at shopapp.queue.escalate
  GET     /api/reports/legacy-export    declared at shopapp.reports.legacy_export
  GET     /api/reports/monthly    declared at shopapp.reports.monthly
  GET     /v2/ping    declared at shopapp.main.ping
```

Nothing has been counted yet, so the CI check refuses to call that a pass:

```bash
deadroutes window-check; echo "exit $?"
```
```
No usage has been recorded. Either the middleware is not installed, or it has not flushed yet. An empty table looks exactly like a new one, so this is not a pass.
exit 2
```

In a real project the observation window fills in as time passes. For the demo, `simulate.py`
sends traffic through the real app and middleware with the clock set back 40 days:

```bash
python3 simulate.py
deadroutes report
deadroutes window-check; echo "exit $?"
```

The report now lists four candidates, and `legacy-export` under *Unreferenced, but called*, with
the caller's name. The check exits `1`: the window has matured and there is something to review.

Now do it for your own app. Add two lines where the app is created:

```python
import deadroutes

app = FastAPI()
recorder = deadroutes.install(app)   # counts every request into deadroutes.sqlite3
```

Then, from your project root, put your settings in `deadroutes.toml` (copy
`examples/deadroutes.toml`; it lists every setting with its default):

```toml
app = "myapp.main:app"
frontend = ["static/**/*.js", "templates/**/*.html"]

[machine_only]
"GET /health" = "the load balancer's health probe"
"POST /webhooks/*" = "event callbacks from outside providers"
```

and run `deadroutes scan`. Leave the app running for as long as your slowest real caller takes
to come round (see the window under *Reading a report*), then run `deadroutes report`.

### Counting requests

```python
app = FastAPI()
recorder = deadroutes.install(app)
```

`install` adds `DeadRoutesMiddleware` to a FastAPI or Starlette app and returns the recorder (also
kept on `app.state.deadroutes`). It reads your settings file the way the command line does
(`deadroutes.toml` in the working directory, `$DEADROUTES_CONFIG`, or
`deadroutes.install(app, config_path="...")`), so the counting settings (`include`, `exclude`,
`store`, `flush_every_seconds`, `max_keys`, `max_callers_per_route`, `max_caller_length` and
`record_callers`) apply to counting as well as to the commands. Pass your own
`deadroutes.Config(...)` to skip the file. The same middleware is plain ASGI, so with Starlette's
own list it is:

```python
from starlette.middleware import Middleware

app = Starlette(routes=[...], middleware=[
    Middleware(deadroutes.DeadRoutesMiddleware, config=deadroutes.Config(exclude=("/health",))),
])
```

What it records, per `(method, route template, caller)`: a count and the time of the last request.

- **Keyed by route template, never the raw path.** `/users/{user_id}` is one key, however many users
  call it. The table stays small and no email address or order number in a URL is ever written
  down. Mounted sub-applications and included routers are resolved to their full template.
- **Reads count.** A `GET` is a call like any other, so a route only a dashboard reads is not
  mistaken for dead.
- **Unmatched requests share one route key**, `(unmatched)`, because their raw path is whatever
  the caller typed. A rising count there means something is trying doors you do not have. A request
  that matched a path but not its method (a 405) lands there too. The method is bounded the same
  way: anything outside the standard HTTP verbs is filed as `OTHER`, so a scanner inventing verbs
  cannot make keys either. A starlette `Mount` that matches a prefix but nothing inside it (a 404
  under `/api/...`) is unmatched too, never a route called `/api`.
- **The caller is the user agent, scrubbed and truncated.** Bearer tokens, API-key shapes (AWS,
  Slack, GitHub and similar), JWTs, email addresses, IP addresses, telephone numbers, `id=`/`user=`
  style values and long opaque strings are replaced with `[redacted]`; version numbers such as
  `Chrome/120.0.0.0` are left alone; control characters are
  dropped; the label is cut to `max_caller_length`. That is how `partner-sync/2.3` shows up by name.
  Set `record_callers = false` to keep only the route and the count.
- **Bounded, and loud at the bound.** A key is a `(method, route)` pair, and at most `max_keys` of
  them are kept. Callers are bounded separately: each route keeps at most `max_callers_per_route`
  named callers and the rest share one `(other callers)` row, so a flood of different user agents
  (on real routes or on junk paths) never fills the table and never costs a route its row. Only a
  NEW pair that arrives after `max_keys` are held is dropped, and then `dropped_keys` goes up;
  pairs already known keep counting. While anything has been dropped the report refuses to
  judge, because a zero might be a route that was never stored.
- **Nothing slow on the request path.** A request does one dictionary update under a lock, after
  the response has been produced. A background thread writes the pending counts every
  `flush_every_seconds`, and the tail is written when the app shuts down (and at interpreter exit as
  a backstop). A store that fails to write is logged once on the `deadroutes` logger and retried;
  it never raises into a request. A quiet app writes a small heartbeat every few hours, so an
  internal tool nobody opens over a holiday does not look like a counter that stopped.
- **Safe across worker processes.** The default store is one SQLite file (standard library),
  written in short transactions that add to what is there. Several `uvicorn` or `gunicorn`
  workers flush into the same file without losing counts. Put the file on a local disk that
  survives deploys: set `DEADROUTES_STORE=/var/lib/yourapp/deadroutes.sqlite3`.

To keep counts somewhere else (a table in your own database), pass any object with
`add(batch, *, now, max_keys, max_callers_per_route, dropped=0)`, `snapshot()` and `reset()`
as `store=`. `add` must add to what is stored, never replace it, and an empty batch is a
heartbeat that only refreshes the last-written time. See `deadroutes/store.py`.

#### Letting CI read the counts

If the counts live on a server, serve them to the CI job from the app itself:

```python
app.add_route("/internal/deadroutes/usage",
              deadroutes.UsageEndpoint(recorder, token=os.environ["DEADROUTES_TOKEN"]))
```

The endpoint refuses every request without `Authorization: Bearer <token>` (compared in constant
time) and refuses to serve at all while the token is empty. `deadroutes ... --usage URL` sends the
token from `$DEADROUTES_TOKEN`. The scan files this route as machine-only for you. Prefer to keep
it off the public internet as well; the token is the second lock, not the only one.

### Scanning

```bash
deadroutes scan                       # uses deadroutes.toml
deadroutes scan --app myapp.main:app --frontend 'static/**/*.js' --frontend 'templates/**/*.html'
deadroutes scan --list unreferenced   # one "METHOD /path" per line
deadroutes scan --json
```

**Reading the routes.** With `--app module:app` (or `app = ...`), deadroutes imports your app and
reads its route table. That is exact: it sees routers included with a prefix, mounted
sub-applications and routes added at run time. If the app cannot be imported (a missing
dependency, an environment variable it insists on), it falls back to parsing the Python files
with `ast`, never executing them, and says so in a warning. The parse understands `@app.get(...)`
style decorators, `APIRouter(prefix=...)`, `include_router`, `Route` and `Mount` lists and
`add_api_route`. Modules are told apart by their dotted path (`src.auth.router` and
`src.posts.router` are two modules), absolute and relative imports are followed, and an import
that could mean more than one module is left out with a warning instead of guessed. It can miss a
prefix added at run time. Give it the files with `source = [...]`
or let it use the files of the app's own package.

**Matching the front end.** Every quoted string in your front-end files is a candidate, including
template literals (`` `/api/orders/${id}` ``), strings joined with `+`, `fetch` and `axios` calls,
HTML attributes and template syntax. A value in a path (`${id}`, `{{ order.id }}`, `:id`, `%s`)
fits any one segment, so `/api/orders/${id}/cancel` matches `/api/orders/{order_id}/cancel`.
`url_for('name')` in a template matches the route with that name. A literal segment must match
exactly: `/api/orders/${id}/cancel` does not match `/api/orders/{order_id}/refund`.

**Methods.** Where the code states the method in plain sight, the reference counts for that
method only: `axios.delete(url)`, `$.get(url)`, `fetch(url, {method: 'PUT'})`,
`{method: isNew ? 'POST' : 'PUT'}`, `<form action="..." method="post">`. So a front end that
only ever GETs `/api/items/{item_id}` leaves the `DELETE` and `PUT` beside it unreferenced. When
the code does not say (`fetch(url)`, `fetch(url, options)`, a wrapper such as `api(url, 'POST')`),
the reference counts for every method on that path, which errs towards keeping a route out of the
candidate list. `deadroutes report` also lists routes that code names but that had no calls over a
mature window, so a dead sibling is still visible when the method could not be read.

**Comments are not callers.** `//`, `/* */`, `<!-- -->` and `{# #}` comments are blanked out
before the scan looks for URLs, because a call that was commented out and forgotten does not keep
a route alive.

**Server files.** Some URLs reach the browser from Python: a work queue item that carries
`{"approve": f"/api/requests/{rid}/approve"}` is how a button calls a route no JavaScript file
names. List those files under `server`. They are parsed, and route declarations and docstrings are
left out so that a route never counts as referencing itself.

**Prefixes are configuration.** Nothing is assumed about your URL layout. `include` and `exclude`
choose which routes are in scope; `public_prefixes` names a prefix your front end adds that the
app never sees, such as a reverse proxy's `/backend`.

**Machine-only** routes are the ones a person's browser never calls. You list them, each with who
calls it, so the list means something later:

```toml
[machine_only]
"POST /webhooks/*"  = "event callbacks from outside providers"
"GET /health"       = "the load balancer's health probe"
```

A pattern is an optional method followed by a path; `*` matches any run of characters. An entry that
no longer matches any route is called out, so stale entries do not pile up. FastAPI's own
`/docs`, `/redoc` and `/openapi.json` are filed as machine-only automatically, both when the app
is imported and when routes are read from source (where `FastAPI(docs_url=None)` and the other
documentation settings are honoured, a mounted `FastAPI()` sub-application gets its own under the
mount, and a setting that is not a literal is skipped rather than guessed).

`deadroutes scan --max-unreferenced N` exits `1` when more than `N` routes are unreferenced. Set `N`
to today's number and it becomes a ratchet: a new endpoint needs a caller in the scanned code, or an
entry that says who calls it.

### Reading a report

```bash
deadroutes report                          # the configured store
deadroutes report --usage prod.sqlite3     # a store copied from a server
deadroutes report --usage https://app.example.com/internal/deadroutes/usage
deadroutes report --usage host-a.json --usage host-b.json   # several hosts, merged
deadroutes report --json
```

Every report opens with the window: **`observing_since`**, **`observed_days`** and
**`safe_to_judge`**. `safe_to_judge` is true only when the counter has run for at least
`min_observed_days` (30 by default), it is still running (it wrote within `stale_after_days`;
a counter that ran for a day two months ago and then stopped has an old start and says nothing),
nothing was dropped at the cap, and something has been recorded. Until then, the routes with no reference and no calls are listed as *waiting*, not as
candidates, and the report says why.

Choose the window to match your slowest legitimate caller. A job that runs once a month needs more
than 30 days; a year-end export needs a year. A route that looks dead in a quiet month is not
evidence of anything.

The sections, in order:

- **Candidates to review**: unreferenced in the scanned code, zero calls, and a mature window.
  Each shows where it is declared. These are questions to ask, not answers.
- **Unreferenced, but called**: nothing in your code names it, but something calls it, with the
  callers' names and counts. This is the section that stops a bad removal.
- **Called, but no longer declared**: counts for routes the app does not have (a route you already
  removed). If one was called in the last `recent_days`, something outside the code still depends
  on it. Counts for paths outside your `include`/`exclude` settings are left out of this list.
- **Named in the scanned code, but not called**: routes some file names that had no calls over a
  mature window. Not candidates (the code does name them), but where an unused `DELETE` or `PUT`
  beside a used `GET` shows up when the method could not be read from the code.
- **Referenced by the front end** and **Machine-only**, with their call counts.
- **Requests that matched no route.**
- **Warnings**: the counts were last written a long time ago (the counter has stopped), keys were
  dropped, a pattern matched no file, the app could not be imported.

`--usage` accepts a SQLite store, a JSON snapshot (`deadroutes usage > snapshot.json`) or a URL.
Reading a store never writes to it: it is opened read-only, so a copy you cannot write to works
(copy the `.sqlite3` file while the app is idle, or use `deadroutes usage > snapshot.json`), and an
SQLite file that is not a deadroutes store is refused with an error rather than read as empty or
altered. Merging hosts is deliberately conservative: the window starts at the *latest* start (the host that
began counting last has seen the least) and `updated_at` is the *earliest* last write (one host
whose counter stopped makes the whole picture stale).

### Checking in CI

```bash
deadroutes window-check
```

| Exit | Meaning |
|------|---------|
| `0` | The window is still open, or it has matured and nothing is both unreferenced and uncalled. |
| `1` | The window has matured and some routes are unreferenced with zero calls: review them. |
| `2` | The counts cannot be trusted: nothing recorded, the counter stopped, keys were dropped, a setting is wrong. |
| `3` | A route the app no longer declares is still being called. Something outside the code depends on it. |

The check stays green while the window is open, because a zero is not evidence yet. It goes red once
it can say something, and a failing scheduled run is the alert; GitHub emails the people watching
the repository.

`examples/github-action.yml` is a workflow that runs the check weekly, reads the counts from your
`UsageEndpoint`, and keeps the JSON report as a build artifact. It installs your app's own
requirements first (`requirements.txt`; change the step to match how you install them), so the
routes are read from the real app; without them it falls back to your source files, which cannot
always see everything. Requests to the framework's own `/docs` and `/openapi.json` pages never
trigger exit `3`. Copy it to
`.github/workflows/`, and set two repository secrets: `DEADROUTES_USAGE_URL` and
`DEADROUTES_TOKEN`. Weekly is deliberate: a job that fires daily for a task measured in weeks
trains people to ignore it.

### Configuration reference

Settings come from, in order: the file named by `--config`, `$DEADROUTES_CONFIG`, then
`deadroutes.toml`, `deadroutes.json` or a `[tool.deadroutes]` table in `pyproject.toml` in the
current directory. Flags override the file. Relative paths and globs in a file are relative to
that file. **Unknown keys are errors**, so a typo is reported rather than quietly ignored. From
Python: `deadroutes.Config(...)` or `deadroutes.Config.load("deadroutes.toml", min_observed_days=14)`.
`deadroutes.install(app)` reads the same file for its counting settings.

| Setting | Default | What it does |
|---------|---------|--------------|
| `include` | `[]` | Only count and scan routes under these prefixes. Empty means every route. A prefix matches itself and below on a path boundary: `/api` matches `/api/users`, never `/apiary`. |
| `exclude` | `[]` | Leave out routes under these prefixes. |
| `store` | `"deadroutes.sqlite3"` | The SQLite file for counts. `$DEADROUTES_STORE` wins over this, and a `--store` flag on the command line wins over both. |
| `flush_every_seconds` | `60` | Seconds between writes. `0` turns the timer off (counts are still written at shutdown and by `recorder.flush()`). |
| `max_keys` | `5000` | Most distinct `(method, route)` pairs kept. Past it, a new pair is dropped and counted in `dropped_keys`. Callers never count against it. |
| `max_callers_per_route` | `50` | Named callers kept per route; later ones share `(other callers)`, and nothing is dropped. |
| `max_caller_length` | `80` | Characters kept of a user agent. |
| `record_callers` | `true` | `false` keeps only the route and the count. |
| `min_observed_days` | `30` | Days of counting before a zero may be read. |
| `recent_days` | `3` | A removed route called within this many days counts as still in use (exit `3`). |
| `stale_after_days` | `7` | Counts last written this long ago mean the counter stopped: the report is not safe to judge and the check exits `2`. `0` turns it off. |
| `app` | `""` | `module:attribute` of your FastAPI or Starlette app. |
| `source` | `[]` | Python files to read routes from without importing. Used when `app` cannot be imported. |
| `frontend` | `[]` | Globs of files that name URLs: JS, TS, HTML, templates. `**` works; `node_modules` is skipped. |
| `server` | `[]` | Python files whose strings count as references (URLs your server hands the browser). |
| `public_prefixes` | `[]` | Prefixes the front end adds that the app does not see. |
| `machine_only` | `{}` | Table of `"[METHOD] /path/*" = "who calls it"`. A reason is required. |

Environment variables: `DEADROUTES_CONFIG` (settings file), `DEADROUTES_STORE` (store path),
`DEADROUTES_TOKEN` (bearer token, for a `--usage` URL and for your `UsageEndpoint`). Logging goes
to the `deadroutes` logger.

#### Commands

| Command | What it does |
|---------|--------------|
| `deadroutes scan` | Classify every route by what your code names. `--json`, `--list BUCKET`, `--verbose`, `--max-unreferenced N`. |
| `deadroutes report` | The scan joined with the counted traffic. `--usage FILE\|URL` (repeatable), `--min-days N`, `--json`. |
| `deadroutes window-check` | The CI gate; exit codes above. Takes the same flags as `report`. |
| `deadroutes usage` | Print the counted traffic as a JSON snapshot. `--store FILE` reads that store. |
| `deadroutes reset --yes` | Empty the store and start a new observation window. `--store FILE` names the store; it wins over `$DEADROUTES_STORE`. |
| `deadroutes demo DIR` | Write the example project used in the quickstart. |

`scan`, `report` and `window-check` also take `--app`, `--source`, `--frontend`, `--server`,
`--include`, `--exclude` and `--public-prefix`, each overriding the file. Globs on the command
line are relative to where you run the command.

### From Python

```python
import deadroutes

config = deadroutes.Config.load("deadroutes.toml")
result = deadroutes.scan(config)                          # the static half
snapshot = deadroutes.SQLiteStore(config.store_path()).snapshot()
report = deadroutes.build_report(result, snapshot, config)
code, lines = deadroutes.window_check(report)             # the same gate the command uses
```

For your own tests of the middleware there is a client with no HTTP and no network:

```python
from deadroutes.testing import call
status, body = call(app, "GET", "/api/orders", headers={"user-agent": "my-script/1.0"})
```

### Honest limits

- **A scan cannot see a caller outside the files you give it.** A mobile app, a partner's script,
  another service's cron job: none of them show up in your front-end files. That is what the
  counter is for, and also why neither half decides alone.
- **A count is only as long as the window.** A quarterly report that did not run during your window
  looks exactly like a dead route. Pick `min_observed_days` from your real cycles, and ask the
  people who own a candidate before touching it.
- **Evidence is not proof.** A route that is unreferenced and uncalled over a mature window is a
  good candidate to ask about. A disaster-recovery route, a rarely used admin action or a caller
  you have not met can look the same. deadroutes never edits or removes anything.
- **A path is not always a method.** The scan reads the method when the code states it literally,
  and otherwise counts a reference for every method on the path. A `fetch(url, options)` whose
  options come from a variable cannot tell a read from a delete. The report's *named in the
  scanned code, but not called* section exists for exactly that case.
- **Comments are skipped, strings that look like comments may not be.** A forgotten call in a
  comment is ignored, but unusual quoting can hide a comment from the scan; that errs towards
  "referenced".
- **URLs built entirely at run time are not matched.** `fetch(base + path)` with both parts
  variable names nothing a scan can read. Calls like `` `/api/${resource}/export` `` are listed under
  *URLs built at run time* in `scan --verbose`, and a route that fits is counted as referenced, but
  deadroutes cannot know every value.
- **Reading source is a fallback.** When the app cannot be imported, prefixes added at run time can
  be missed, as can routes added by a library. Prefer `--app`, and install your app's requirements
  in the CI job so it can import.
- **HTTP only.** WebSocket connections are not counted and WebSocket routes are not listed.
- **FastAPI and Starlette routing.** The middleware relies on the router leaving the matched route
  in the request scope. Another ASGI framework's requests are counted under `(unmatched)`.
- **Counts can be lost if the process is killed hard.** A normal shutdown writes the tail. A
  `kill -9` or a crash loses at most `flush_every_seconds` of counts; a lower value narrows that.
- **SQLite means one machine.** It is safe across the worker processes on one host, on a local
  disk. It is not reliable on a network file system. With several hosts, keep one store per host and
  pass each to `--usage` (or serve each with a `UsageEndpoint`); they are merged conservatively.
- **Health checks count as calls** unless you `exclude` them. A load balancer probing `/health`
  every ten seconds is a caller, so list it under `machine_only` or `exclude` it.
- **The counter must be running.** If the middleware is removed in a refactor, the counts stop and
  every route starts to look idle. The `stale_after_days` check turns that into exit `2` instead of
  a clean-looking zero.

### Files

- `deadroutes/`: the package. `middleware.py` and `recorder.py` count, `store.py` keeps, `routes.py`
  and `references.py` and `scan.py` read, `report.py` joins, `cli.py` is the command.
- `deadroutes/demo/shop/`: the example project `deadroutes demo` writes out.
- `examples/deadroutes.toml`: every setting with its default and a comment.
- `examples/github-action.yml`: the weekly CI check.
- `examples/usage_endpoint.py`: serving the counts from your app.
- `tests/`: the suite (`python3 -m pip install -e ".[dev]" && python3 -m pytest`). No network.

---

Docs mirror of the private repository Dominares-Tools/deadroutes; updated with each release.
