# Observability with OpenTelemetry

Skill Scanner ships with optional, fully opt-in OpenTelemetry (OTel)
instrumentation for **traces**, **metrics**, and **logs**. When disabled (the
default), there is zero overhead and no SDK dependencies are loaded.

---

## Installation

Install the `otel` extra together with the base package:

```bash
# Using uv (recommended)
uv pip install cisco-ai-skill-scanner[otel]

# Using pip
pip install cisco-ai-skill-scanner[otel]
```

This pulls in the OpenTelemetry SDK plus OTLP exporters (gRPC and HTTP).  If
you only need HTTP export:

```bash
pip install cisco-ai-skill-scanner opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

---

## Activation

Telemetry is **off by default**. Enable it in one of two ways:

### CLI flag

```bash
skill-scanner scan /path/to/skill --use-behavioral --enable-otel
skill-scanner scan-all /path/to/skills --enable-otel --format json
```

### Environment variable

```bash
export SKILL_SCANNER_OTEL_ENABLED=true
skill-scanner scan /path/to/skill
```

The environment variable is also respected by the API server
(`skill-scanner-api`) without requiring a restart.

---

## Configuration

Skill Scanner follows the standard OpenTelemetry environment variables so any
compatible backend works without code changes.

| Variable | Description | Default |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP collector endpoint (gRPC or HTTP) | Console exporter |
| `OTEL_EXPORTER_OTLP_HEADERS` | Comma-separated `key=value` auth headers | — |
| `OTEL_SERVICE_NAME` | Service name reported to the backend | `skill-scanner` |
| `OTEL_RESOURCE_ATTRIBUTES` | Additional resource attributes | — |
| `OTEL_SDK_DISABLED` | Set to `true` to hard-disable the SDK | `false` |

### Console export (no backend, development)

When `OTEL_EXPORTER_OTLP_ENDPOINT` is not set, spans and metrics are printed
to stdout — useful for debugging locally without a collector.

```bash
export SKILL_SCANNER_OTEL_ENABLED=true
skill-scanner scan /path/to/skill --enable-otel
```

---

## Backend examples

### Jaeger (local)

```bash
# Start Jaeger all-in-one
docker run -d --name jaeger \
  -p 4317:4317 -p 16686:16686 \
  jaegertracing/all-in-one:latest

export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_SERVICE_NAME=skill-scanner
skill-scanner scan /path/to/skill --enable-otel
```

Open Jaeger UI: http://localhost:16686

### Grafana Tempo + OpenTelemetry Collector

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export SKILL_SCANNER_OTEL_ENABLED=true
skill-scanner scan-all ./skills --recursive
```

### Dash0

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=https://ingress.eu-west-1.aws.dash0.com
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer YOUR_DASH0_TOKEN"
export OTEL_SERVICE_NAME=skill-scanner
skill-scanner scan /path/to/skill --enable-otel
```

### Honeycomb

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io
export OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=YOUR_API_KEY"
export OTEL_SERVICE_NAME=skill-scanner
skill-scanner scan /path/to/skill --enable-otel
```

---

## Traces

A **root span** is created for every scan operation with the following
attributes:

| Span name | Attribute | Description |
|---|---|---|
| `skill_scanner.scan_skill` | `skill.name` | Name from `SKILL.md` manifest |
| | `skill.directory` | Absolute path to the skill directory |
| | `skill.file_count` | Number of files in the skill package |
| | `findings.total` | Total findings after post-processing |
| | `findings.is_safe` | Whether the result is safe (no HIGH/CRITICAL) |
| | `scan.duration_seconds` | Wall-clock scan time |
| | `analyzers.used` | Comma-separated list of analyzer names |
| `skill_scanner.scan_directory` | `skills.directory` | Root directory scanned |
| | `skills.count` | Skills discovered |
| | `scan.recursive` | Whether recursive search was used |
| | `scan.check_overlap` | Whether cross-skill analysis ran |
| | `skills.scanned` | Skills successfully scanned |
| | `skills.skipped` | Skills skipped due to load/scan errors |
| `skill_scanner.load_skill` | `skill.directory` | Directory being loaded |
| | `skill.lenient` | Whether lenient loading was used |
| | `skill.name` | Parsed skill name |
| | `skill.file_count` | Files discovered in the package |
| `skill_scanner.extract_archives` | `skill.archive_input_count` | Files considered for extraction |
| | `skill.extracted_file_count` | Files produced by extraction |
| | `skill.extraction_findings_count` | Findings raised during extraction |

Separating `load_skill` and `extract_archives` from the root span makes
"load time" and "extraction I/O" distinguishable from "analyze time".

Each **analyzer** gets its own child span:

```
skill_scanner.scan_skill
  ├── skill_scanner.load_skill          (sibling: runs before the root span)
  ├── skill_scanner.extract_archives
  ├── skill_scanner.analyzer.static
  ├── skill_scanner.analyzer.pipeline
  ├── skill_scanner.analyzer.behavioral
  └── skill_scanner.analyzer.llm_analyzer   (phase=llm)
        └── skill_scanner.external.llm      (provider call)
```

### External-service spans

Outbound calls to third-party services are wrapped in
`skill_scanner.external.<service>` spans, so provider latency and retries are
visible instead of being hidden inside an opaque analyzer span:

| Span name | Operations | Notes |
|---|---|---|
| `skill_scanner.external.virustotal` | `file_report`, `file_upload`, `analysis_poll` | Carries `http.status_code`, `vt.attempt` |
| `skill_scanner.external.aidefense` | `inspect` | Carries `http.status_code`, `external.attempt` |
| `skill_scanner.external.llm` | `completion`, `meta_analysis`, `alignment` | Carries `llm.model`, `external.attempt` |

Installing the `[otel]` extra also enables
`opentelemetry-instrumentation-httpx`, which adds automatic client spans for
**all** outbound HTTP requests with no analyzer code changes.

### Span events

Soft failures — paths that catch an exception and continue — are recorded as
span events rather than disappearing silently:

| Event | Emitted when |
|---|---|
| `skill_load_failed` | A skill in a directory scan fails to load |
| `skill_scan_failed` | A skill raises an unexpected error mid-scan |
| `cross_skill_overlap_check_failed` | Description-overlap analysis fails |
| `cross_skill_pattern_detection_failed` | `CrossSkillScanner` fails |
| `meta_analysis_failed` | API meta-analysis fails for a single scan |
| `batch_meta_analysis_failed` | Batch meta-analysis run fails |

Exceptions are automatically recorded on the span with `ERROR` status.

---

## Metrics

| Metric name | Type | Unit | Description |
|---|---|---|---|
| `skill_scanner.scans.total` | Counter | `{scan}` | Total scans completed |
| `skill_scanner.scan.duration` | Histogram | `s` | Wall-clock scan time |
| `skill_scanner.findings.total` | Counter | `{finding}` | Findings, labelled by `finding.severity` |
| `skill_scanner.analyzer.duration` | Histogram | `s` | Per-analyzer timing, labelled by `analyzer.name` |
| `skill_scanner.scan.errors` | Counter | `{error}` | Failed or skipped skills, labelled by `error.type` |
| `skill_scanner.external.duration` | Histogram | `s` | External-call latency, labelled by `external.service` |
| `skill_scanner.external.retries` | Counter | `{retry}` | Retries against external services, labelled by `retry.reason` |
| `skill_scanner.external.errors` | Counter | `{error}` | External failures, labelled by `error.type` |

Scan instruments carry `skill.name`, `scan.is_safe`, and `analyzers` labels.
External instruments carry `external.service` and `external.operation`.

`error.type` values include `rate_limit`, `timeout`, `auth_failure`, and
`http_<status>`, which makes it possible to alert on VirusTotal rate-limit
pressure or LLM provider degradation directly.

The duration histograms use explicit bucket boundaries tuned for scan
workloads rather than SDK defaults, so percentiles are meaningful.

---

## Logs

When OTel log export is enabled, the `skill_scanner.*` logger hierarchy is
bridged into the OTLP log pipeline. Each log record is correlated with the
active trace context so you can jump from a log entry directly to its span in
your backend.

---

## Python SDK usage

```python
from skill_scanner.telemetry import setup_telemetry, TelemetryConfig, shutdown_telemetry
from skill_scanner import SkillScanner

# Initialise once at process start
setup_telemetry(TelemetryConfig(
    service_name="my-scanner-service",
    otlp_endpoint="http://localhost:4317",
))

scanner = SkillScanner()
result = scanner.scan_skill("/path/to/skill")
print(result.is_safe)

# Flush all pending telemetry before exit
shutdown_telemetry()
```

---

## CI/CD integration

OpenTelemetry traces work seamlessly in CI pipelines. Add the following
environment variables to your CI configuration:

```yaml
# GitHub Actions example
env:
  SKILL_SCANNER_OTEL_ENABLED: "true"
  OTEL_EXPORTER_OTLP_ENDPOINT: ${{ secrets.OTEL_ENDPOINT }}
  OTEL_EXPORTER_OTLP_HEADERS: ${{ secrets.OTEL_HEADERS }}
  OTEL_SERVICE_NAME: skill-scanner-ci
  OTEL_RESOURCE_ATTRIBUTES: ci.pipeline=${{ github.workflow }},ci.run_id=${{ github.run_id }}
```

This produces a trace per CI run that you can search by `ci.run_id` to
correlate with test results and understand scan performance trends over time.

---

## Troubleshooting

**"opentelemetry-sdk not installed"** – Run `pip install cisco-ai-skill-scanner[otel]`.

**No spans visible in backend** – Check `OTEL_EXPORTER_OTLP_ENDPOINT` is reachable from the
host. Omit the variable to fall back to the console exporter and confirm spans are produced.

**High overhead on LLM scans** – LLM spans are expected to be long (several seconds).
The `skill_scanner.analyzer.llm_analyzer` span duration reflects the actual LLM API latency.

**Duplicate spans after re-import** – `setup_telemetry()` is idempotent; calling it
multiple times is safe and only the first call initialises the SDK.
