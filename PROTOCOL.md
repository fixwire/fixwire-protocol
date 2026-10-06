# Fixwire protocol (v1)

How Fixwire SDKs, OpenTelemetry SDKs and tools send data to Fixwire, and
what Fixwire answers.

- **Traces, logs and errors** travel as [OTLP/HTTP](https://opentelemetry.io/docs/specs/otlp/),
  the OpenTelemetry protocol, on its standard paths. Any OpenTelemetry SDK can
  send them, in any language.
- **Sessions, check-ins, feedback and files** have small Fixwire endpoints
  that take JSON (files: multipart).
- **Fixwire SDKs** speak both, and add what OpenTelemetry does not define:
  structured stack traces for grouping, breadcrumbs, release health,
  redaction on the device.

## 1. The DSN

One string per project tells an SDK where to send and with which key:

```
{scheme}://{key}@{host}[:{port}][/{path}]
https://fw_pk_live_4c1f2a…@ingest.eu.fixwire.io
```

- `key` is the project's publishable key (`fw_pk_live_…`, or `fw_pk_test_…`
  for test data). It can only send data to its own project, so it is safe in
  browser bundles and mobile apps. Secret keys (`fw_sk_…`) are refused.
- The **base URL** is the DSN without the key: `{scheme}://{host}[:{port}][/{path}]`.
  Every endpoint below is relative to it. A path is for self-hosted servers
  behind a prefix.
- SDKs read the `dsn` option, else the `FIXWIRE_DSN` environment variable.
  Without a DSN an SDK does nothing.

An app with OpenTelemetry and no Fixwire SDK sets the same two things:

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://ingest.eu.fixwire.io
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer fw_pk_live_4c1f2a…
```

## 2. Requests

### Authentication

Every request carries the key, from the first of:

1. `Authorization: Bearer {key}`
2. the `key` query parameter, for clients that cannot set headers (a
   browser sending while a page closes, `curl` in a cron job).

A browser request's `Origin` must be allowed by the project (its allowed
origins setting, empty by default: any origin).

### Who is sending

OTLP requests name their SDK with the standard resource attributes
`telemetry.sdk.name`, `telemetry.sdk.version` and `telemetry.sdk.language`.
Fixwire JSON bodies carry `"sdk": {"name": …, "version": …}`. Fixwire SDK
names are `fixwire.{language}[.{platform}]`: `fixwire.javascript.browser`,
`fixwire.javascript.node`, `fixwire.python`, `fixwire.go`, …

### Bodies

- `Content-Encoding`: `gzip`, `deflate`, `br` or `zstd`, or none.
- Up to 20 MB as sent and 50 MB decompressed per request; per kind of data,
  the limits in each section.
- JSON bodies may be sent as `text/plain;charset=UTF-8`, so browsers skip
  the CORS preflight.

### Answers

| Status | Meaning | The SDK |
|---|---|---|
| `200` | Accepted, perhaps in part (below) | Done |
| `400` | Unreadable body | Drops it |
| `401` | No key, or an unknown, revoked or secret one | Drops it |
| `403` | Not allowed from this origin | Drops it |
| `404` | No such endpoint | Drops it |
| `413` | Too large | Drops it |
| `415` | Unsupported content type or encoding | Drops it |
| `429` | Rate limited | Waits `Retry-After` seconds, then retries |
| `5xx` | Unavailable | Retries with backoff, honouring `Retry-After` |

OTLP answers are the OTLP response messages, in the request's encoding.
When only part of a request is accepted (a quota is used up for one kind of
data, a record is too large), the answer is `200` with OTLP's
`partial_success` counting what was rejected and why. Fixwire endpoints
answer JSON: `{"id": …}` when something was created, `{"error": …}`
otherwise.

### Rate limits

`429` and `503` answers carry `Retry-After` (seconds). Any answer may also
carry `Fixwire-Rate-Limits`, which tells Fixwire SDKs to pause some kinds of
data while the rest keeps flowing:

```
Fixwire-Rate-Limits: 60:log;span, 3600:file
```

A comma-separated list of `{seconds}:{categories}`, categories separated by
`;`: `error`, `log`, `span`, `session`, `check_in`, `feedback`, `file`. An
empty list means all of them. Each kind of data has its own quota, so a full
log quota never stops errors.

## 3. Traces: `POST /v1/traces`

An OTLP `ExportTraceServiceRequest`, as `application/x-protobuf` or
`application/json` (OTLP's JSON encoding: hex trace and span ids).

| From | Fixwire reads |
|---|---|
| Resource | `service.name`; `service.version` (the **release**); `deployment.environment.name` (the **environment**, default `production`); `telemetry.sdk.*` |
| Span | name, kind, start and end, status, parent, attributes (OpenTelemetry semantic conventions: `http.*`, `db.*`, `gen_ai.*`, …), the last `exception` event |
| `fixwire.op` (optional) | The operation Fixwire shows (`http.server`, `db.query`, `task`, …); without it Fixwire derives one from the attributes |

- A span is a **segment** (a request, a job, a page load) when it has no
  parent, or when its parent is remote (OTLP span `flags` bits 8 and 9).
- **AI agent runs** are spans with the OpenTelemetry GenAI conventions
  (`gen_ai.*`). Fixwire also maps OpenInference, OpenLLMetry, the Vercel AI
  SDK, LiteLLM and Claude Code.
- **Errors on spans.** For senders without a Fixwire SDK, an `exception`
  event on a span whose status is error becomes an error (section 4).
  Fixwire SDKs report errors as log records only, so nothing counts twice;
  Fixwire ignores span exception events from `fixwire.*` SDKs for issues.
- Up to 5 MB of spans per request; links are not read yet.

## 4. Logs and errors: `POST /v1/logs`

An OTLP `ExportLogsServiceRequest`, encoded as for traces. A log record is
one of three things:

| Log record | Becomes |
|---|---|
| `event_name` = `exception`, or no event name and an `exception.type` or `exception.message` attribute | An **error**: an event, grouped into an issue |
| `event_name` = `fixwire.message` | A **message**: an event, grouped into an issue (`captureMessage`) |
| Anything else | A **log line**: stored, searchable, linked to its trace |

The level comes from `severity_number`: `TRACE` and `DEBUG` are `debug`,
`INFO` is `info`, `WARN` is `warning`, `ERROR` is `error`, `FATAL` is
`fatal`. Errors without a severity are `error`. `trace_id` and `span_id`
link an error or a log line to its trace. Release and environment come from
the resource, as for traces.

### Error and message attributes

The first three are OpenTelemetry's; the rest are what Fixwire SDKs add.
Everything but the exception's type or message is optional.

| Attribute | Type | Meaning |
|---|---|---|
| `exception.type` | string | `TypeError`, `java.lang.IllegalStateException`, … |
| `exception.message` | string | The exception's message |
| `exception.stacktrace` | string | The stack as the language prints it. Fixwire reads frames from it when `fixwire.exceptions` is missing (Go, Java, .NET, Python, JavaScript, Ruby and PHP formats) |
| `fixwire.event_id` | string | 32 hex characters, made by the SDK: removes duplicates, and links feedback and files to the event |
| `fixwire.exceptions` | array | The chain of exceptions, **outermost first** (the one caught, then its cause, …). Each: `type`, `message`, `module`, `mechanism` (`type`, `handled`), `frames` |
| (a frame) | kvlist | `function`, `module`, `file`, `abs_path`, `line`, `column`, `in_app`, `context_line`, `pre_context`, `post_context`, `vars` (local variables, already redacted). Frames run from the oldest call to the newest (the throwing line last) |
| `fixwire.handled` | bool | `false` for a crash or an unhandled exception (default `true`) |
| `fixwire.fingerprint` | array of strings | A custom grouping; `{{ default }}` stands for Fixwire's own |
| `fixwire.tags` | kvlist of strings | Searchable tags |
| `fixwire.contexts` | kvlist | Named groups of details, such as `order: {id, items}` |
| `fixwire.breadcrumbs` | array | What happened before, oldest first. Each: `timestamp` (Unix seconds), `type`, `category`, `message`, `level`, `data` |
| `fixwire.debug_images` | array | For source maps and symbols. Each: `type` (`sourcemap`), `code_file`, `debug_id` |
| `fixwire.suppressed` | int | Occurrences the SDK folded into this one (its duplicate budget) |
| `fixwire.transaction` | string | The route or task the error happened in |
| `user.id`, `user.email`, `user.name` | string | The user (OpenTelemetry conventions); `client.address` is their IP when the app sends personal data |
| `http.request.method`, `url.full`, `http.route`, `user_agent.original`, … | | The request (OpenTelemetry conventions); the user agent lets filters drop web crawlers |

The log record's `body`, a string, is the message of a `fixwire.message`
record; on an error it is optional. An error or a message is at most 1 MB;
a request of log records at most 5 MB.

### Example

An error from a Fixwire SDK, in OTLP JSON with attributes shown as a map
for reading (on the wire, attributes are OTLP key-value lists):

```json
{
  "timeUnixNano": "1791190800000000000",
  "eventName": "exception",
  "severityNumber": 17,
  "traceId": "5b8efff798038103d269b633813fc60c",
  "spanId": "eee19b7ec3c1b174",
  "attributes": {
    "exception.type": "TypeError",
    "exception.message": "amount 500 exceeds the limit",
    "exception.stacktrace": "TypeError: amount 500 exceeds the limit\n    at chargeCard (/app/cart.js:12:11)\n    at checkout (/app/cart.js:20:3)",
    "fixwire.event_id": "9ec79c33ec9942ab8353589fcb2e04dc",
    "fixwire.exceptions": [{
      "type": "TypeError",
      "message": "amount 500 exceeds the limit",
      "mechanism": { "type": "generic", "handled": true },
      "frames": [
        { "function": "checkout", "file": "/app/cart.js", "line": 20, "column": 3, "in_app": true },
        { "function": "chargeCard", "file": "/app/cart.js", "line": 12, "column": 11, "in_app": true,
          "context_line": "    throw new TypeError(`amount ${amount} exceeds the limit`);" }
      ]
    }],
    "fixwire.tags": { "plan": "team" },
    "fixwire.breadcrumbs": [
      { "timestamp": 1791190799.2, "category": "cart", "message": "checkout started", "level": "info" }
    ],
    "user.id": "user-1"
  }
}
```

## 5. Sessions: `POST /v1/sessions`

Release health: how many sessions and users of a release ended well, with an
error, or in a crash.

```json
{
  "sdk": { "name": "fixwire.javascript.browser", "version": "1.0.0" },
  "release": "web@1.4.0",
  "environment": "production",
  "sessions": [
    { "sid": "b0e2…", "did": "c6c2…", "init": true, "started": "2026-10-05T10:00:00Z",
      "timestamp": "2026-10-05T10:04:12Z", "status": "exited", "errors": 0, "duration": 252.1 }
  ],
  "aggregates": [
    { "started": "2026-10-05T10:00:00Z", "did": "c6c2…", "exited": 41, "errored": 2, "crashed": 0, "abnormal": 0 }
  ]
}
```

- `sessions` are single sessions (a browser page, a mobile app's run): sent
  when one starts (`init: true`) and when it ends. `status` is `ok`,
  `exited`, `crashed` or `abnormal`.
- `aggregates` count a server's requests per minute (`started`, the start of
  the minute) and user: `exited`, `errored`, `crashed`, `abnormal`.
- `did` is the user, hashed on the device (the first 16 bytes of the
  SHA-256 of their id, as 32 hex characters), never the raw id; Fixwire
  hashes it again with a key of its own.
- `user_agent` (optional, browsers) helps tell bots from people.
- At most 1 MB.

## 6. Check-ins: `/v1/check-ins/{monitor}`

`{monitor}` is the monitor's slug. `POST` with JSON:

```json
{
  "check_in_id": "8b3e…",
  "status": "ok",
  "duration": 42.5,
  "environment": "production",
  "monitor_config": {
    "schedule": { "type": "crontab", "value": "0 3 * * *" },
    "checkin_margin": 5,
    "max_runtime": 30,
    "timezone": "Europe/Berlin"
  }
}
```

- `status` is `in_progress`, `ok` or `error`. A job sends `in_progress` when
  it starts and `ok` or `error` with the same `check_in_id` when it ends.
- `duration` is in seconds. `monitor_config` creates or updates the monitor
  (schedule `crontab`, or `interval` with `value` and `unit`).
- Jobs without an SDK use `GET` with the query instead:
  `GET /v1/check-ins/nightly-report?status=ok&environment=production&key=fw_pk_live_…`.
- The answer is `{"id": "{check_in_id}"}` (made by Fixwire when not sent).
  At most 100 kB.

## 7. Feedback: `POST /v1/feedback`

```json
{
  "message": "The answer about my refund was wrong.",
  "score": -1,
  "event_id": "9ec79c33ec9942ab8353589fcb2e04dc",
  "trace_id": "5b8efff798038103d269b633813fc60c",
  "name": "Ada",
  "email": "ada@example.com",
  "url": "https://shop.example.com/help",
  "source": "widget",
  "release": "web@1.4.0",
  "environment": "production"
}
```

- A `message`, a `score` or both: a thumbs-up or -down alone is feedback.
  `score` runs from `-1` (bad) to `1` (good).
- `event_id` ties feedback to an error; `trace_id` to a trace or an AI
  agent run (a negative score there opens a `user_feedback` agent issue).
- At most 1 MB. The answer is `{"id": …}`.

## 8. Files: `POST /v1/files`

`multipart/form-data` with:

- `event_id`: the event the file belongs to (32 hex characters);
- `type`: `attachment` (the default); `minidump` comes with native crashes;
- `file`: the file, with its filename and content type.

At most 20 MB per file. The event may arrive before or after its file. The
answer is `{"id": …}`. Files are billed by size.

## 9. Trace propagation

- Fixwire SDKs continue and start traces with the W3C `traceparent` and
  `tracestate` headers, on incoming and outgoing HTTP requests.
- A trace's sampling decision travels in `traceparent`'s sampled flag;
  spans follow their parent's. A new trace is kept when its trace id's last
  56 bits, as a fraction of 2^56, are at least `1 - sample_rate`, so every
  service decides the same for the same trace.
- `baggage` passes through untouched (within the limits of section 13).
  There are no Fixwire-specific propagation headers.
- Outgoing requests carry trace headers only to the app's
  `trace_propagation_targets` (section 13); without the option, browsers
  send them to the page's own origin and servers send them nowhere.
- A server-rendered page passes its trace to the browser SDK with
  `<meta name="traceparent" content="…">`.

## 10. Source maps and debug files

Uploads do not change in v1: `fixwire-cli` sends source maps and debug
files to the API host. A bundle carries its debug ID as a
`//# debugId={uuid}` comment (the source map debug ID convention), and at
runtime in `globalThis._fixwireDebugIds`, which maps stack locations to
debug IDs. Fixwire SDKs send them in `fixwire.debug_images`.

## 11. Not in v1

- Metrics: `POST /v1/metrics` answers `200` with `partial_success`
  rejecting every data point, so OpenTelemetry SDKs with metrics on don't
  log errors.
- Profiles, session replay and the SDKs' own counts of dropped data.

## 12. Versions

Everything lives under `/v1`. New attributes, fields and endpoints are
added within it; Fixwire ignores what it doesn't know, and so do SDKs. A
change that breaks a client gets `/v2`, served next to `/v1`.

## 13. What every Fixwire SDK guarantees

An SDK runs inside someone else's app, on data an attacker may shape
(messages, URLs, headers, request bodies) and answers it doesn't control.
Every Fixwire SDK keeps to the rules below, with the same numbers, and
tests each one. Numbers are upper bounds unless an option is named.

### Never in the app's way

- No SDK call throws, panics or rejects into the app. The app's own
  callbacks (`before_send`, `before_breadcrumb`, samplers) and the app's
  objects the SDK reads (messages, `toString`, getters) are guarded: what
  fails is skipped, and said in the debug log.
- `init` never throws either: a broken DSN or option is reported (as an
  error value where the language returns one, else as a warning on
  stderr) and the SDK stays off, so a typo in configuration can't stop the
  app from starting.
- Work on the app's threads is linear in its input: no pattern that
  backtracks on untrusted text, no scan inside a loop over the same data.
  The duplicate budget reads at most a message's first 1,024 characters.
- The SDK's own requests are never instrumented (no spans, breadcrumbs or
  trace headers), its debug lines are never captured, and a logging
  integration skips what is logged while the SDK is capturing (from a
  callback, say).
- `flush(timeout)` and `close(timeout)` return within their timeout; at
  exit the SDK waits at most its shutdown timeout (default 2 s).
- Where processes fork (pre-forking servers), a child starts its own
  sender at once and leaves the parent's queued data to the parent; locks
  held at the fork are not waited on.

### Sending

- **No redirects.** A `3xx` is a refusal: the key in `Authorization` goes to
  the DSN's host and nowhere else.
- **Answers** are read for their status and headers, and at most 64 kB of
  body.
- **Pauses.** `Retry-After` (seconds or an HTTP date) and the seconds in
  `Fixwire-Rate-Limits` count from 0 to 86,400 (a day): larger values are a
  day, broken ones are ignored. A `429` without `Fixwire-Rate-Limits`
  pauses all data for `Retry-After`, at least 60 s; a `5xx` with
  `Retry-After` pauses all data for that long. Only the categories of
  section 2 are kept.
- **Retries.** A request is sent at most 4 times in all: again after no
  answer or a `5xx`, waiting about 1 s, then twice as long each time, and
  again after a `429`'s pause (an SDK that sends at exit, like PHP's, may
  try fewer). A request whose next try would be more than 5 minutes away
  is dropped.
- **Memory.** At most `max_queue` requests (default 100) wait to be sent,
  and as many wait for a retry; past that, new data is dropped.
- **Sizes.** An error or a message is at most 1 MB of JSON: over it, the
  SDK leaves out `fixwire.breadcrumbs`, then frames' `vars`, then
  `fixwire.contexts`, and drops the record if it is still over. Log records
  and spans go in requests of at most 100 items and 5 MB; one item that
  can't fit is dropped alone. A sessions request holds at most 5,000
  aggregates.

### What is sent

- **Strings** are at most `max_value_length` (default 1,024) bytes of
  UTF-8, cut on a character boundary and ending in `...` (the `...` within
  the limit). Two kinds of text get more room: `exception.stacktrace` is
  at most 64 kB, and the content of AI calls (`gen_ai.*` prompts,
  completions, tool arguments and results) at most 16 kB. Redaction runs
  before the cut, over the part kept and the next 16 kB, so a secret the
  cut goes through is still found.
- **Values** (contexts, extras, attributes, local variables) are at most 10
  levels deep and 100 items wide, and at most 10,000 objects are walked per
  value. A container inside itself is `"[Circular ~]"`, one deeper than
  the limit `"[Object]"` or `"[Array]"`, one that can't be read
  `"[Unreadable]"`. `NaN` and infinities are the strings `"NaN"`,
  `"Infinity"` and `"-Infinity"`.
- **Breadcrumbs**: the last `max_breadcrumbs` (default 100); adding one
  takes constant time.
- **Exceptions**: a chain of at most 10, cut where it comes back to an
  exception already in it; at most `max_stack_frames` (default 100) frames
  each, the newest kept.
- **Source lines**: 5 above and below the frame's line, read only from
  regular files of at most 10 MB, through a cache of at most 64 files and
  32 MB.
- **Spans**: a segment keeps at most 1,000 child spans, a span at most 128
  attributes.
- **Sessions**: at most 5,000 users are counted apart per send; past that,
  requests are counted without their user.

### Trace context

- An incoming `traceparent` is used only when it is well formed, as W3C
  defines it: version `00`, exactly four fields, lower-case hex, a
  non-zero 32-hex trace id, a non-zero 16-hex span id, 2-hex flags. An
  incoming `tracestate` over 512 bytes, or `baggage` over 8,192 bytes, or
  either with a control character other than tab (W3C's list whitespace),
  is not passed on.
- `trace_propagation_targets` decide which outgoing requests carry trace
  headers. A URL is compared without its user info, query and fragment:
  - a string with `://` matches URLs that start with it
    (`https://api.example.com/v2`);
  - a string starting with `/` matches requests to the page's own origin
    whose path starts with it (browsers);
  - any other string is a host, with a port if it has one: it matches that
    host and its subdomains (`example.com` matches `api.example.com`, not
    `badexample.com` or `example.com.evil.net`);
  - a regular expression is searched for in the URL as compared.

### Redaction

- Every string the SDK sends from the app's data goes through redaction
  (unless the app turns it off): messages, attributes, span names and
  status messages, breadcrumbs, feedback, URLs and their queries, local
  variables, and the keys of maps. The app's own configuration (release,
  environment, service name, monitor slugs) is cut but sent as given:
  masking `api@1.2.3.example` as an email would break release health. The
  rules are the server's; each SDK tests its copy against the server's
  test vectors.
- Detectors run in linear time, and numbering keys that mask alike is
  linear too.
- A value redaction fails on (a timeout, an error) is sent as
  `[Filtered]`, never unmasked.
