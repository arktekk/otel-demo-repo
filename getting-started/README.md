# Getting started with OpenTelemetry

This folder is a companion source to [Part 2](https://arktekk.no/blogs/2025_otel_part_2_agent)

## Contents:

- `camelo.jar` is the unknown application the team is taking over. Run it at your own peril, we don't have
  the source code yet. Arktekk is not liable in the event that it orders 1000 litres of milk.
- `otel-collector-debug-config.yaml` contains the configuration for the otel-collector used to debug telemetry collection
- `docker-compose.yml` runs an otel collector with the debug config and makes it available on ports 4317 and 4318
- `opentelemetry-javaagent.jar` is a checked in copy of the otel java agent
- `.mise.toml` can be used to install dependencies with [mise](https://mise.jdx.dev/)

## Commands

### Collector

```shell
docker compose up
```

To change the config, you must take it down first:

```shell
docker compose down
```

The data is logged to the console where it runs.

### `mise`

Install dependencies (java): 

```shell
mise install
```

Start app: 
```shell
mise run server-start
```

Start app with telemetry: 

```shell
mise run server-start-with-otel
```

Start app with telemetry and otel debug log (noisy):

```shell
mise run server-start-with-otel-debug
```
