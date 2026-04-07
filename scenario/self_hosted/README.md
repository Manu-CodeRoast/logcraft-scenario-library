# Self-Hosted Scenarios

These scenarios use **includes** and/or **registry** directives, which require access to the local filesystem.

They are **not available from the online Lab** — you need a self-hosted LogCraft installation (CLI or Server) with the full `logcraft-scenario-library` cloned locally.

## What are includes?

Includes let you compose a scenario from multiple YAML files:

```yaml
includes:
  - includes/web_server_agents.yaml
```

## What is the registry?

The registry loads reusable agent definitions from external files, with versioning and overrides:

```yaml
registry:
  named_agents:
    nginx:
      v1: agents/nginx_v1.yaml
      v2: agents/nginx_v2.yaml
  agents:
    - ref: "nginx:v2"
      name: nginx-primary
      overrides:
        rate_per_second: 500
```

## Scenarios in this folder

| File | Features |
|------|----------|
| `registry_demo.yaml` | External agent loading with versioning |
| `registry_templates.yaml` | Registry + templates combined |
| `feature_showcase.yaml` | Every major feature (registry, incidents, rules, auto_cascade, health states) |
| `web_server.yaml` | Includes-based agent composition |
| `includes/web_server_agents.yaml` | Included agent definitions used by web_server.yaml |
