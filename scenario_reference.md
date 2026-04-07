# LogCraft Scenario Reference

Complete reference for every configuration key available in a LogCraft scenario file.
All scenarios are defined in YAML under a top-level `scenario:` key.

---

## Table of Contents

- [Scenario (root)](#scenario-root)
- [Environment](#environment)
- [Duration & Seed](#duration--seed)
- [Outputs](#outputs)
- [Agents](#agents)
- [Fields & Generators](#fields--generators)
- [Phases](#phases)
- [Latency](#latency)
- [Templates](#templates)
- [Interactions](#interactions)
- [Rules](#rules)
- [Incidents](#incidents)
- [Health State Machine](#health-state-machine)
- [State & Effects](#state--effects)
- [Rate Modulation](#rate-modulation)
- [Auto Cascade](#auto-cascade)
- [Noise](#noise)
- [Users & Personas](#users--personas)
- [Entity Pool & Field Variations](#entity-pool--field-variations)
- [Log Format](#log-format)
- [Request Flow](#request-flow)
- [Contention & Slow Queries](#contention--slow-queries)
- [Failures & Availability](#failures--availability)
- [Registry](#registry)
- [Includes](#includes)
- [Replay](#replay)

---

## Scenario (root)

Every file starts with a `scenario:` block:

```yaml
scenario:
  name: "my-scenario"         # Human-readable name (required)
  seed: 42                     # Fixed seed for deterministic output (optional)
  duration: 5m                 # How long the scenario runs (required)
```

---

## Duration & Seed

### Duration

Controls how long the scenario runs. Accepts human-readable strings or raw seconds.

| Format   | Example   | Meaning       |
|----------|-----------|---------------|
| Seconds  | `30s`     | 30 seconds    |
| Minutes  | `5m`      | 5 minutes     |
| Hours    | `1h`      | 1 hour        |
| Combined | `24h`     | 24 hours      |
| Raw      | `120`     | 120 seconds   |
| Infinite | `0`       | Runs forever  |

### Seed

When `seed` is set, every run produces the exact same logs in the same order. Remove the `seed` line for randomized output each time.

```yaml
scenario:
  seed: 42        # Same seed → same logs every time
  duration: 5m
```

---

## Environment

Optional metadata that gets embedded in every log record. Use it to tag your logs with deployment context.

```yaml
environment:
  region: us-east-1
  cluster: prod-ecommerce
  version: v3.2.1
```

All keys under `environment` are free-form — add any key/value pair you need.

### Example: Embedded in Logs

Every log record will include these fields:

```json
{
  "timestamp": "2026-04-07T14:23:45Z",
  "level": "info",
  "message": "GET /api/products -> 200",
  "region": "us-east-1",
  "cluster": "prod-ecommerce",
  "version": "v3.2.1"
}
```

---

## Outputs

Define where and how logs are written. You can configure multiple outputs — the same log events are sent to all of them simultaneously.

```yaml
outputs:
  - type: console
    format: json

  - type: file
    format: clf
    path: logs/access.log
    max_size_bytes: 10485760    # 10 MB rotation threshold
    max_files: 5                # Keep up to 5 rotated files

  - type: http
    url: "http://localhost:9200/_bulk"
    format: ecs
    batch_size: 100             # Records per HTTP request
    flush_interval_ms: 1000     # Max time before flushing a batch
```

### Output Types

| Type         | Description                                    |
|--------------|------------------------------------------------|
| `console`    | Print to stdout                                |
| `file`       | Write to disk with optional rotation           |
| `http`       | Send batched HTTP POST requests                |
| `prometheus` | Expose a scrape endpoint for metrics           |
| `statsd`     | Push metrics over UDP                          |
| `recording`  | Write a JSONL replay file for later playback   |

### File Output Options

| Key              | Type    | Description                              |
|------------------|---------|------------------------------------------|
| `path`           | string  | File path to write to                    |
| `max_size_bytes` | integer | Rotate when file exceeds this size       |
| `max_files`      | integer | Maximum number of rotated files to keep  |

### HTTP Output Options

| Key                 | Type    | Description                              |
|---------------------|---------|------------------------------------------|
| `url`               | string  | Destination URL                          |
| `batch_size`        | integer | Records per request (default: 100)       |
| `flush_interval_ms` | integer | Max ms before flushing (default: 1000)   |

### Prometheus Output Options

| Key               | Type    | Description                               |
|-------------------|---------|-------------------------------------------|
| `port`            | integer | Port for the scrape endpoint              |
| `metrics_interval`| string  | How often metrics are updated (e.g. `15s`)|

### StatsD Output Options

| Key    | Type    | Description                         |
|--------|---------|-------------------------------------|
| `host` | string  | StatsD server host                  |
| `port` | integer | StatsD server port (default: 8125)  |

### Output Formats

Every output (except `prometheus` and `statsd`) requires a `format` field:

| Format                         | Description                                |
|--------------------------------|--------------------------------------------|
| `json`                         | Structured JSON                            |
| `text`                         | Plain text                                 |
| `clf`                          | Common Log Format (Apache/Nginx access)    |
| `apache_error`                 | Apache error log format                    |
| `log4j`                        | Log4j pattern                              |
| `syslog`                       | BSD syslog (RFC 3164)                      |
| `rfc5424`                      | IETF syslog (RFC 5424)                     |
| `nginx_error`                  | Nginx error log format                     |
| `kv` / `logfmt`                | Key=value pairs                            |
| `android_logcat`               | Android logcat format                      |
| `windows_cbs`                  | Windows CBS log format                     |
| `spark_hdfs`                   | Apache Spark / HDFS format                 |
| `health_app`                   | Health application format                  |
| `proxifier`                    | Proxifier log format                       |
| `cloudwatch`                   | AWS CloudWatch Logs format                 |
| `systemd_journal`              | Systemd journal format                     |
| `hpc` / `bgl`                  | HPC / Blue Gene/L format                   |
| `iis_w3c`                      | IIS W3C extended log format                |
| `ecs`                          | Elastic Common Schema                      |
| `otel` / `opentelemetry` / `otlp` | OpenTelemetry log format               |

### Per-Agent Output Routing

Agents can send logs to specific named outputs instead of all of them:

```yaml
outputs:
  - name: main-logs
    type: console
    format: json
  - name: error-file
    type: file
    format: text
    path: logs/errors.log

agents:
  - name: my-agent
    outputs: [main-logs]          # Only writes to "main-logs"
```

---

## Agents

Agents are the log-producing services in your simulation. Each agent represents one component of your system.

```yaml
agents:
  - name: api-gateway             # Unique name (required)
    type: web_server              # Service type (descriptive label)
    rate: 100/s                   # Log records per second
    error_rate: 0.03              # Probability of ERROR level (0.0 – 1.0)
    log_level: info               # Default log level: info, debug, warn, error
    instances: 3                  # Run N copies of this agent
    start_after: 5s               # Delay before this agent starts emitting
    use: my_template              # Inherit defaults from a template
    outputs: [main-logs]          # Route to specific named outputs
    message_template: "{method} {path} -> {status_code}"
    fields: [...]
```

### Agent Keys

| Key                | Type          | Description                                                        |
|--------------------|---------------|--------------------------------------------------------------------|
| `name`             | string        | Unique agent identifier (required)                                 |
| `type`             | string        | Descriptive label (e.g. `web_server`, `database`)                  |
| `rate`             | string/number | Records per second (`"200/s"` or `rate_per_second: 200`)           |
| `error_rate`       | number        | Probability of emitting an ERROR-level log (0.0–1.0)               |
| `log_level`        | string        | Default level: `info`, `debug`, `warn`, `error`                    |
| `instances`        | integer       | Number of parallel copies of this agent                            |
| `start_after`      | string        | Delay before first log emission (e.g. `"5s"`)                      |
| `use`              | string        | Name of a template to inherit from                                 |
| `outputs`          | list          | Named outputs this agent writes to                                 |
| `message_template` | string        | Log message with `{field_name}` placeholders                       |
| `fields`           | list          | Field definitions (see [Fields & Generators](#fields--generators)) |
| `phases`           | list          | Time-based behavior changes (see [Phases](#phases))                |
| `latency_ms`       | object/array  | Latency distribution (see [Latency](#latency))                     |
| `dependencies`     | list          | Names of agents this agent depends on                              |
| `health_state`     | object        | Health state machine config                                         (see [Health State Machine](#health-state-machine))                                                       |
| `state`            | object        | Internal state variables (see [State & Effects](#state--effects))  |
| `effects`          | list          | Conditional behavior changes based on state                        |
| `noise`            | object        | Per-agent noise overrides                                          |
| `request_flow`     | list          | Distributed call chain (see [Request Flow](#request-flow))         |
| `contention`       | object        | Connection pool limits (see [Contention & Slow Queries](#contention--slow-queries))                                                                              |
| `slow_queries`     | object        | Probabilistic slow operations                                      |
| `availability`     | object        | Uptime and failure mode config                                     |
| `failures`         | object        | Burst failure patterns                                             |
| `rate_modulation`  | object        | Time-varying rate changes (see [Rate Modulation](#rate-modulation))                                                                                       |

### Rate Format

Rates can be written two ways:

```yaml
rate: 200/s           # Shorthand
rate_per_second: 200  # Explicit numeric
```

---

## Fields & Generators

Fields define the dynamic data inside each log record. Each field has a `name` and a `generator` that determines how values are produced.

```yaml
fields:
  - name: method
    generator: weighted_choice
    values: [GET, POST, PUT, DELETE]
    weights: [60, 25, 10, 5]
```

### Generator Types

#### `weighted_choice`

Pick from a list of values with specified probabilities.

```yaml
- name: status_code
  generator: weighted_choice
  values: ["200", "404", "500"]
  weights: [90, 7, 3]
```

| Key       | Type | Description                                    |
|-----------|------|------------------------------------------------|
| `values`  | list | Possible values                                |
| `weights` | list | Relative probabilities (same length as values) |

#### `choice`

Pick randomly from a list with equal probability.

```yaml
- name: path
  generator: choice
  values: [/api/users, /api/orders, /health]
```

#### `range`

Generate a random number within a range.

```yaml
- name: bytes_sent
  generator: range
  min: 200
  max: 50000
  integer: true      # true for integers, false/omit for decimals
```

| Key       | Type    | Description                       |
|-----------|---------|-----------------------------------|
| `min`     | number  | Minimum value (inclusive)         |
| `max`     | number  | Maximum value (inclusive)         |
| `integer` | boolean | Round to integer (default: false) |

#### `sequence`

Auto-incrementing counter, optionally with a string prefix.

```yaml
- name: request_id
  generator: sequence
  prefix: "req-"
  start: 1000
```

| Key      | Type    | Description                         |
|----------|---------|-------------------------------------|
| `prefix` | string  | Prepend this string to the number   |
| `start`  | integer | Starting value (default: 0)         |

#### `static`

Always produces the same fixed value.

```yaml
- name: host
  generator: static
  value: "web-prod-01"
```

#### `timestamp`

Produces a formatted date/time string.

```yaml
- name: ts
  generator: timestamp
  format: "%Y-%m-%dT%H:%M:%S"    # strftime format (default)
```

#### `normal`

Gaussian (bell curve) distribution.

```yaml
- name: response_time
  generator: normal
  mean: 50
  stddev: 12
```

#### `percentile`

Distribution defined by percentile targets (p50, p95, p99).

```yaml
- name: latency
  generator: percentile
  p50: 10
  p95: 80
  p99: 300
```

#### `conditional`

Produce different values based on another field's value.

```yaml
- name: message
  generator: conditional
  depends_on: status_code
  branches:
    "200": "OK"
    "404": "Not Found"
    "500": "Internal Server Error"
```

| Key          | Type   | Description                            |
|--------------|--------|----------------------------------------|
| `depends_on` | string | Name of the field to branch on         |
| `branches`   | object | Map of field value → output value      |

---

## Phases

Phases let an agent change its behavior over time. Each phase overrides the agent's rate, error rate, or latency for a set duration.

```yaml
agents:
  - name: api-server
    rate: 50/s                    # Base rate — used when no phase is active
    phases:
      - name: warmup
        duration: 30s
        rate: 10/s                # Override: ramp down to 10/s
        latency_ms: [5, 20]
      - name: steady
        duration: 90s
        rate: 50/s                # Back to base rate
        latency_ms:
          distribution: normal
          mean: 25
          stddev: 8
      - name: peak
        duration: 60s
        rate: 150/s               # Override: ramp up to 150/s
        error_rate: 0.08
        latency_ms:
          distribution: percentile
          p50: 40
          p95: 200
          p99: 600
```

### Phase Keys

| Key          | Type          | Description                              |
|--------------|---------------|------------------------------------------|
| `name`       | string        | Label for this phase                     |
| `duration`   | string        | How long this phase lasts                |
| `rate`       | string/number | Override the agent's rate                |
| `error_rate` | number        | Override the agent's error rate          |
| `latency_ms` | object/array | Override the agent's latency distribution |

Phases run in order. When the last phase ends, the scenario ends (or the agent continues at its base rate if the scenario duration is longer).

---

## Latency

Latency controls the simulated response time for each log record. It can be defined in three ways:

### Uniform (array)

Random value between min and max.

```yaml
latency_ms: [5, 100]       # 5ms to 100ms, uniformly distributed
```

### Normal (Gaussian)

Bell-curve distribution centered on the mean.

```yaml
latency_ms:
  distribution: normal
  mean: 50
  stddev: 10
```

### Percentile

Define by target percentiles — most realistic for production systems.

```yaml
latency_ms:
  distribution: percentile
  p50: 10         # Median latency
  p95: 80         # 95th percentile
  p99: 300        # 99th percentile
```

---

## Templates

Templates let you define reusable agent configurations. Any agent can inherit from a template using `use:` and then override specific fields.

```yaml
templates:
  go_microservice:
    rate: 100/s
    error_rate: 0.008
    latency_ms:
      distribution: normal
      mean: 12
      stddev: 5

agents:
  - name: user-service
    use: go_microservice          # Inherits rate, error_rate, latency_ms
    error_rate: 0.05              # Override: increase error rate to 5%
    message_template: "..."       # Add specific fields
    fields: [...]
```

Templates can include any agent-level key (rate, error_rate, latency_ms, etc.).

---

## Interactions

Interactions describe the communication topology between agents. They declare which agents call which other agents.

```yaml
interactions:
  - from: api-gateway
    to: order-service
    type: request
  - from: order-service
    to: postgres-primary
    type: dependency
```

### Interaction Keys

| Key    | Type   | Description                                        |
|--------|--------|----------------------------------------------------|
| `from` | string | Name of the calling agent                          |
| `to`   | string | Name of the called agent                           |
| `type` | string | Relationship label (e.g. `request`, `dependency`, `query`, `cache_lookup`) |

Interactions are used by the cascading engine to determine how errors and latency propagate between agents.

---

## Rules

Rules define conditional error and latency propagation based on agent metrics.

```yaml
rules:
  - when: "postgres-primary.error_rate > 0.05"
    propagate:
      to: order-service
      latency_multiplier: 3.0
  - when: "order-service.error_rate > 0.1"
    propagate:
      to: api-gateway
      error_rate: 0.08
```

### Rule Keys

| Key                            | Type   | Description                                     |
|--------------------------------|--------|-------------------------------------------------|
| `when`                         | string | Condition expression (e.g. `"agent.metric > N"`)|
| `propagate.to`                 | string | Target agent to affect                          |
| `propagate.latency_multiplier` | number | Multiply the target's latency by this factor    |
| `propagate.error_rate`         | number | Set the target's additional error rate          |

---

## Incidents

Incidents simulate unexpected events — outages, slowdowns, cascading failures — that change agent behavior at specific times.

```yaml
incidents:
  - name: database_overload
    trigger: "time > 5m"
    duration: 2m
    effects:
      - target: postgres-primary
        latency_multiplier: 8
        error_rate: 0.20
      - target: order-service
        latency_multiplier: 3
        error_rate: 0.10
```

### Incident Keys

| Key                   | Type   | Description                                                   |
|-----------------------|--------|---------------------------------------------------------------|
| `name`                | string | Label for this incident                                       |
| `trigger`             | string | When it fires: `"time > 5m"` (time-based)                     |
| `trigger_probability` | number | Alternative: probability of firing each second (e.g. `0.005`) |
| `duration`            | string | How long effects last (omit or `0` for permanent)             |
| `effects`             | list   | Agent modifications while incident is active                  |

### Effect Keys

| Key                  | Type   | Description                                  |
|----------------------|--------|----------------------------------------------|
| `target`             | string | Agent name to affect                         |
| `latency_multiplier` | number | Multiply the agent's latency by this factor  |
| `error_rate`         | number | Set the agent's error rate during incident   |

### Trigger Types

- **Time-based:** `"time > 5m"` — fires once after 5 minutes
- **Probabilistic:** `trigger_probability: 0.005` — 0.5% chance per second of firing (simulates sporadic failures)

---

## Health State Machine

Agents can have a health state that transitions automatically between four states: **Healthy**, **Degraded**, **Failing**, and **Recovering**. Each state has its own latency multiplier and error rate.

```yaml
agents:
  - name: order-service
    health_state:
      latency_multipliers: [1.0, 2.0, 5.0, 1.5]    # [Healthy, Degraded, Failing, Recovering]
      error_rates: [0.0, 0.05, 0.30, 0.02]          # [Healthy, Degraded, Failing, Recovering]
      transitions:                                    # Probability of transitioning per tick
        healthy_to_degraded: 0.01
        degraded_to_failing: 0.05
        failing_to_recovering: 0.10
        recovering_to_healthy: 0.15
```

### Health States

| Index | State       | Description                                      |
|-------|-------------|--------------------------------------------------|
| 0     | Healthy     | Normal operation                                 |
| 1     | Degraded    | Elevated latency and error rate                  |
| 2     | Failing     | High latency and error rate                      |
| 3     | Recovering  | Slightly elevated while returning to normal      |

The state machine transitions probabilistically — each tick there is a chance of moving to the next state. This produces realistic, unpredictable health fluctuations.

**Default multipliers** (when not specified):
- Latency: `[1.0, 2.0, 5.0, 1.5]`
- Error rates: `[0.0, 0.05, 0.30, 0.02]`

---

## State & Effects

State variables track internal metrics of an agent (e.g. connection pool usage, queue depth). Effects change agent behavior when a state variable crosses a threshold.

```yaml
agents:
  - name: order-service
    state:
      queue_depth:
        initial: 0
        max: 10000
        growth_per_request: 0.1
    effects:
      - when: "queue_depth > 5000"
        latency_multiplier: 3
      - when: "queue_depth > 8000"
        error_rate: 0.15
```

### State Variable Keys

| Key                  | Type   | Description                                |
|----------------------|--------|--------------------------------------------|
| `initial`            | number | Starting value                             |
| `max`                | number | Upper bound                                |
| `growth_per_request` | number | How much the value increases per log event |

### Effect Keys

| Key                  | Type   | Description                                  |
|----------------------|--------|----------------------------------------------|
| `when`               | string | Condition expression (e.g. `"variable > N"`) |
| `latency_multiplier` | number | Multiply latency when condition is true      |
| `error_rate`         | number | Override error rate when condition is true   |

---

## Rate Modulation

Rate modulation makes an agent's traffic vary over time, simulating realistic load patterns.

### Sinusoidal

Smooth periodic wave — useful for simulating day/night cycles or repeating traffic patterns.

```yaml
rate_modulation:
  type: sinusoidal
  amplitude_factor: 0.5          # ±50% variation around the base rate
  period_seconds: 86400          # Full cycle = 24 hours
  phase_offset_seconds: 0        # Shift the wave start
```

### Business Hours

Higher traffic during work hours, lower outside.

```yaml
rate_modulation:
  type: business_hours
  peak_start_hour: 9             # Peak begins at 09:00
  peak_end_hour: 18              # Peak ends at 18:00
  peak_multiplier: 1.0           # Rate multiplier during peak
  off_peak_multiplier: 0.1       # Rate multiplier outside peak
```

---

## Auto Cascade

Automatically propagates errors and latency between agents that have declared interactions, without needing explicit rules.

```yaml
auto_cascade:
  error_threshold: 0.10          # Start cascading when error rate > 10%
  latency_threshold_ms: 0.0      # 0 = disabled (only error-based cascading)
  dampening_factor: 0.5          # Effect is halved at each hop
  blast_radius: 2                # Maximum number of hops from the source
```

| Key                    | Type   | Description                                                    |
|------------------------|--------|----------------------------------------------------------------|
| `error_threshold`      | number | Minimum error rate to trigger cascade (default: 0.10)          |
| `latency_threshold_ms` | number | Minimum latency to trigger cascade; 0 = disabled               |
| `dampening_factor`     | number | Fraction of the effect passed to each downstream hop (0.0–1.0) |
| `blast_radius`         | integer| Maximum hops from the failing agent (default: 3)               |

When an agent's error rate exceeds `error_threshold`, its direct dependents (defined via `interactions`) receive a dampened portion of the failure. The effect continues to propagate up to `blast_radius` hops, getting weaker with each hop.

---

## Noise

Noise simulates imperfections in your logging pipeline — duplicated entries, missing fields, and timing jitter.

```yaml
noise:
  log_duplication_rate: 0.005    # 0.5% of logs are duplicated
  missing_fields_rate: 0.002     # 0.2% of logs have fields stripped
  random_delay_ms: [0, 10]       # Random timing jitter up to 10ms
```

| Key                    | Type       | Description                             |
|------------------------|------------|-----------------------------------------|
| `log_duplication_rate` | number     | Probability of duplicating a log entry  |
| `missing_fields_rate`  | number     | Probability of removing random fields   |
| `random_delay_ms`      | [min, max] | Uniform delay added to each log event   |

Noise can be set at the scenario level (applies to all agents) or per-agent (overrides the scenario level for that agent).

---

## Users & Personas

Simulate realistic user traffic patterns with session-based behavior.

### Users

```yaml
users:
  count: 50000
  sessions:
    duration: [30s, 10m]              # Each session lasts 30s to 10m
    requests_per_session: [3, 80]     # Each session makes 3 to 80 requests
```

### Personas

Personas define different user behavior profiles. Each persona has a weight (relative probability) and a behavior description.

```yaml
personas:
  - name: power_user
    weight: 2.0
    behavior:
      request_rate: high
      endpoints: ["/api/checkout", "/api/search"]
  - name: casual_browser
    weight: 5.0
    behavior:
      request_rate: low
      endpoints: ["/api/products", "/api/categories"]
```

| Key                    | Type   | Description                                        |
|------------------------|--------|----------------------------------------------------|
| `name`                 | string | Persona label                                      |
| `weight`               | number | Relative frequency (higher = more common)          |
| `behavior.request_rate`| string | `high`, `medium`, or `low`                         |
| `behavior.endpoints`   | list   | Endpoints this persona tends to access             |

---

## Entity Pool & Field Variations

### Entity Pool

A fixed list of entity identifiers that get reused across logs, creating realistic correlations (e.g. the same order ID appearing across multiple services).

```yaml
entity_pool:
  - "order-10001"
  - "order-10002"
  - "customer-201"
  - "product-5001"
```

### Field Variations

Add random jitter to numeric fields for additional realism.

```yaml
field_variations:
  - field: latency_ms
    jitter_percent: 8.0           # ±8% random variation
  - field: response_bytes
    jitter_percent: 15.0
```

---

## Log Format

Define the shape and field ordering of log records.

```yaml
log_format:
  type: json
  fields:
    - timestamp
    - level
    - service
    - trace_id
    - span_id
    - message
    - latency_ms
```

| Key      | Type   | Description                                |
|----------|--------|--------------------------------------------|
| `type`   | string | Format type (`json`)                       |
| `fields` | list   | Ordered list of fields in each log record  |

---

## Request Flow

Simulates distributed call chains between services, modeling network latency and timeouts.

```yaml
agents:
  - name: api-gateway
    request_flow:
      - call: auth-service
        timeout_ms: 30
      - call: product-service
        timeout_ms: 100
      - call: order-service
        timeout_ms: 200
```

| Key          | Type    | Description                                     |
|--------------|---------|-------------------------------------------------|
| `call`       | string  | Name of the agent being called                  |
| `timeout_ms` | integer | Maximum wait time before treating as a timeout  |

The calls are executed in order, simulating a sequential request chain. If a downstream agent is slow, the timeout triggers an error in the caller.

---

## Contention & Slow Queries

### Contention

Models connection pool exhaustion — when connections are saturated, new requests queue up with additional latency.

```yaml
contention:
  max_connections: 200
  queue_latency_ms: [1, 50]     # Additional latency when queued
```

### Slow Queries

Probabilistic outlier queries that are much slower than normal.

```yaml
slow_queries:
  probability: 0.015             # 1.5% of queries are slow
  latency_ms: [500, 5000]       # Slow queries take 500ms–5s
```

---

## Failures & Availability

### Availability

Controls the overall uptime of an agent.

```yaml
availability:
  uptime: 0.985                  # 98.5% uptime
  failure_mode: timeout          # What happens during downtime
```

### Failures

Configures burst failure patterns that activate under specific conditions.

```yaml
failures:
  mode: burst
  trigger: latency_threshold
  threshold_ms: 500
  duration: 30s
  error_rate: 0.20               # Error rate during the burst
```

| Key            | Type   | Description                                   |
|----------------|--------|-----------------------------------------------|
| `mode`         | string | Failure pattern (`burst`)                     |
| `trigger`      | string | What triggers the failure (`latency_threshold`) |
| `threshold_ms` | number | Latency threshold to activate burst           |
| `duration`     | string | How long the burst lasts                      |
| `error_rate`   | number | Error rate during the failure burst           |

---

## Registry

The registry system lets you maintain a library of reusable agent definitions in external YAML files. Agents in the registry can be referenced by name and version, with optional overrides.

> **Note:** Registry requires access to the local filesystem and is not available from the online Lab.

```yaml
registry:
  sources: [agents/]              # Directories to scan for agent files
  named_agents:
    nginx:
      v1: agents/nginx_v1.yaml
      v2: agents/nginx_v2.yaml
    auth: agents/auth.yaml
    db: agents/db.yaml

  agents:
    - ref: "nginx:v2"            # Load nginx version 2
      name: nginx-primary        # Give it a name in this scenario
      overrides:
        rate_per_second: 500
        error_rate: 0.01
        instances: 3
    - ref: auth
      name: auth-service
```

### Registry Keys

| Key                  | Type   | Description                                         |
|----------------------|--------|-----------------------------------------------------|
| `sources`            | list   | Directories to scan for `.yaml` agent files         |
| `named_agents`       | object | Map of agent name → file path (or name → version map) |
| `agents`             | list   | Agent instances to create from the registry         |
| `agents[].ref`       | string | Registry reference (`"name"` or `"name:version"`)  |
| `agents[].name`      | string | Instance name in this scenario                      |
| `agents[].overrides` | object | Override any agent-level key (rate, error_rate, etc.) |

---

## Includes

Include external YAML files to compose a scenario from multiple files. Included files can contain agent definitions, templates, or any scenario section.

> **Note:** Includes requires access to the local filesystem and is not available from the online Lab.

```yaml
includes:
  - includes/web_server_agents.yaml
  - includes/monitoring_agents.yaml
```

Paths are relative to the scenario file's location.

---

## Replay

Record logs to a JSONL file for later deterministic playback.

```yaml
replay:
  path: recordings/my_scenario.jsonl
```

The recording output type must also be configured in `outputs` for recording to work:

```yaml
outputs:
  - type: recording
    path: recordings/my_scenario.jsonl
```
