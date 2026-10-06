<div align="center">

_Bugs reach production. Fixwire finds them first: errors, traces, logs and
AI agent runs in one place, an AI debugger on every plan, and your data
kept in Europe._

[![Discord](https://img.shields.io/badge/Discord-join%20us-5865F2?logo=discord&logoColor=white)](https://fixwire.io/discord)
[![Slack](https://img.shields.io/badge/Slack-community-4A154B?logo=slack&logoColor=white)](https://fixwire.io/slack)
[![X](https://img.shields.io/badge/X-follow%20us-000000?logo=x&logoColor=white)](https://fixwire.io/x)
[![Protocol](https://img.shields.io/badge/protocol-v1-blue)](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-OTLP%2FHTTP-425CC7?logo=opentelemetry&logoColor=white)](https://opentelemetry.io/docs/specs/otlp/)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/fixwire/fixwire-protocol/blob/main/LICENSE)

<br/>

</div>

# The Fixwire protocol

How Fixwire SDKs, OpenTelemetry SDKs and tools send data to
**[Fixwire](https://fixwire.io)**, what Fixwire answers, and what every
Fixwire SDK guarantees the app it runs in. The specification is
[PROTOCOL.md](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md).

## 📦 In short

- **Traces, logs and errors** travel as OpenTelemetry's OTLP/HTTP on its
  standard paths (`/v1/traces`, `/v1/logs`), so any OpenTelemetry SDK can
  send them, in any language.
- **Sessions, check-ins, feedback and files** have small JSON endpoints
  (`/v1/sessions`, `/v1/check-ins/{monitor}`, `/v1/feedback`, `/v1/files`).
- **One DSN** per project, `https://<publishable key>@<host>`, says where to
  send and with which key; the key goes in `Authorization: Bearer`.
- **Fixwire SDKs** add what OpenTelemetry doesn't define: structured stack
  traces for grouping, breadcrumbs, release health and redaction on the
  device, within the limits of section 13.

## 🔭 Sending from OpenTelemetry

An app that already uses OpenTelemetry needs no Fixwire SDK to send traces,
logs and errors:

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=https://ingest.eu.fixwire.io
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer fw_pk_live_…"
```

## 📚 Contents

| Section | What it covers |
|---|---|
| [1. The DSN](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#1-the-dsn) | Keys, the base URL, `FIXWIRE_DSN` |
| [2. Requests](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#2-requests) | Authentication, bodies, answers, rate limits |
| [3. Traces](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#3-traces-post-v1traces) | Spans, segments, AI agent runs |
| [4. Logs and errors](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#4-logs-and-errors-post-v1logs) | Errors and messages as log records, their attributes |
| [5. Sessions](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#5-sessions-post-v1sessions) | Release health |
| [6. Check-ins](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#6-check-ins-v1check-insmonitor) | Cron monitors |
| [7. Feedback](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#7-feedback-post-v1feedback) | User feedback and AI answer ratings |
| [8. Files](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#8-files-post-v1files) | Attachments |
| [9. Trace propagation](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#9-trace-propagation) | W3C trace context across services |
| [10. Source maps and debug files](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#10-source-maps-and-debug-files) | Debug IDs |
| [12. Versions](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#12-versions) | How the protocol changes |
| [13. What every Fixwire SDK guarantees](https://github.com/fixwire/fixwire-protocol/blob/main/PROTOCOL.md#13-what-every-fixwire-sdk-guarantees) | Limits, safety and redaction every SDK keeps |

## 🧩 SDKs that speak it

| Language | Repository |
|---|---|
| JavaScript (Node.js, browsers, edge, React) | [fixwire-js](https://github.com/fixwire/fixwire-js) |
| Python | [fixwire-python](https://github.com/fixwire/fixwire-python) |
| Go | [fixwire-go](https://github.com/fixwire/fixwire-go) |
| Java and Kotlin | [fixwire-java](https://github.com/fixwire/fixwire-java) |
| .NET | [fixwire-dotnet](https://github.com/fixwire/fixwire-dotnet) |
| PHP, Laravel, Symfony | [fixwire-php](https://github.com/fixwire/fixwire-php), [fixwire-laravel](https://github.com/fixwire/fixwire-laravel), [fixwire-symfony](https://github.com/fixwire/fixwire-symfony) |
| Ruby and Rails | [fixwire-ruby](https://github.com/fixwire/fixwire-ruby) |
| Rust | [fixwire-rust](https://github.com/fixwire/fixwire-rust) |

## 🙌 Want to contribute?

Found something unclear, or a case the protocol doesn't cover? Open an
[issue](https://github.com/fixwire/fixwire-protocol/issues): the protocol
changes through the same review as the server and every SDK that speaks it.

## 🛟 Need help?

Ask on [Discord](https://fixwire.io/discord) or [Slack](https://fixwire.io/slack).
Found a security issue? Please don't open an issue; follow the
[security policy](https://github.com/fixwire/fixwire-protocol/blob/main/SECURITY.md).

## 📃 License

The protocol is published under the MIT license; see
[LICENSE](https://github.com/fixwire/fixwire-protocol/blob/main/LICENSE).
