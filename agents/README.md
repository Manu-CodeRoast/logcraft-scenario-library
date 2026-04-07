# Agent Reference Library

This folder contains pre-built agent definitions that you can use as templates and inspiration for your own scenarios.

## How to Use

### Option 1: Copy & Customize
Copy an agent definition from this folder and customize it for your scenario:

```yaml
# From agents/nginx_v1.yaml
- name: my-nginx
  type: web_server
  rate: 100/s
  error_rate: 0.02
  latency_ms:
    distribution: normal
    mean: 25
    stddev: 8
  fields: [...]
```

### Option 2: Use as Template
Inline an agent as a template and inherit from it:

```yaml
templates:
  nginx_template:
    rate: 100/s
    error_rate: 0.02
    latency_ms:
      distribution: normal
      mean: 25
      stddev: 8

agents:
  - name: my-nginx
    type: web_server
    use: nginx_template
    fields: [...]
```

### Option 3: Registry (Self-Hosted Only)
If you're using self-hosted LogCraft (CLI or Server), you can reference agents from the registry:

```yaml
registry:
  sources: [agents/]
  named_agents:
    nginx:
      v1: agents/nginx_v1.yaml
      v2: agents/nginx_v2.yaml

  agents:
    - ref: "nginx:v2"
      name: nginx-primary
      overrides:
        rate_per_second: 200
```

> **Note:** Registry is not available from the online Lab.

## Folder Structure

| Folder | Purpose |
|--------|---------|
| `simple/` | Basic agent templates |
| `services/` | Microservice templates (API, worker, notification) |
| `infrastructure/` | Infrastructure agents (load balancer, CDN, Kubernetes) |
| `database/` | Database agents (PostgreSQL, MongoDB, Redis) |
| `cache/` | Cache layer agents |
| `messaging/` | Message queue agents (Kafka, RabbitMQ) |
| `search/` | Search engine agents (Elasticsearch, Solr) |
| `external/` | External API agents (payment gateway, third-party services) |

## Tips

- **Start with `simple/`** — Good for learning agent structure
- **Mix and match** — Combine fields and behaviors from different agents
- **Override as needed** — Templates provide defaults; customize what you need
- **Rate realism** — Check realistic rates for your service type before using
- **Latency distributions** — Percentile distributions are most realistic for production
