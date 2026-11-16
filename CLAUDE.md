# CLAUDE.md

Context for working on this repository. This is a fork of
[MarcusWichelmann/EnergyExporter](https://github.com/MarcusWichelmann/EnergyExporter), maintained at
[jamesvarty/EnergyExporter](https://github.com/jamesvarty/EnergyExporter).

## What it does

ASP.NET Core service (.NET 10) that polls energy devices over Modbus TCP and exposes the readings as:

- a management UI at `/` (static page in `src/EnergyExporter/wwwroot/index.html`, backed by `/api/status`)
- Prometheus metrics at `/metrics`
- a REST API at `/devices` (JSON, or XML via `Accept: application/xml`), documented at `/swagger` (`/openapi/v1.json`)
- optional pushes to InfluxDB 2 and MQTT (`Export` section of `appsettings.json`)

Supported devices: SolarEdge inverters, meters and batteries (SunSpec registers) and Janitza power analyzers.

## Layout

| Path | Purpose |
| --- | --- |
| `Devices/` | One class per device type. Properties carry `[ModbusRegister]` / `[ScaledModbusRegister]` / `[StringModbusRegister]` (where to read), `[PrometheusMetric]` and `[InfluxDbMetric]` (how to export). Adding a value means adding an attributed property, nothing else. |
| `Modbus/ModbusReader.cs` | One reader per `host:port`, shared by all devices on that endpoint and serialized with a lock. Groups registers into chunks of at most 125, enforces connect/read timeouts, retries once only if a *reused* connection failed. |
| `Modbus/ScaledModbusRegisterAttribute.cs` | SunSpec `value * 10^sf`. "Not implemented" markers (0xFFFF uint16, 0x8000 int16/scale factor, 0x80000000 int32) become `double.NaN`. uint32 values are acc32 counters, where 0 is valid, so they are never treated as markers. |
| `Services/DeviceService.cs` | Polls all configured devices, tracks a `DeviceStatus` per device (online, failures, last error, last reading) and logs once per outage. |
| `Services/DevicePollingService.cs` | `BackgroundService` loop: no overlapping polls, `PollNow()` trigger, runs each exporter independently and tracks its health. |
| `Controllers/StatusController.cs` | `/api/status`, `POST /api/poll`, `POST /api/reconnect`, `POST /api/devices/{id}/reconnect`. No authentication: LAN use only. |
| `Prometheus/MetricsWriter.cs` | Device metrics (NaN skipped) plus `energy_exporter_device_{up,polls_total,poll_failures_total,poll_duration_seconds}`. |

NaN handling must stay consistent: Prometheus, InfluxDB and MQTT skip NaN values, and JSON writes them as `null`
(`Utils/NaNAsNullDoubleConverter.cs`; System.Text.Json throws on NaN otherwise).

## Building and testing

The .NET SDK is **not installed** on the dev laptop; build and run everything through Docker.

```bash
docker build -t energy-exporter:dev .
# Package checks
docker run --rm -v $PWD/src/EnergyExporter:/w -w /w mcr.microsoft.com/dotnet/sdk:10.0 \
  bash -c 'dotnet list package --vulnerable --include-transitive; dotnet list package --outdated'
# Run against the real inverter (the env var avoids the laptop's inotify limit)
docker run --rm -p 14560:14552 -e DOTNET_hostBuilder__reloadConfigOnChange=false \
  -v /path/to/appsettings.json:/app/appsettings.json:ro energy-exporter:dev
```

There are no unit tests. Changes have been verified end to end by running the container against the real
inverter, and the InfluxDB and MQTT exporters against throwaway `influxdb:2.7` and `eclipse-mosquitto:2` containers.
For offline-device behaviour, configure an extra device at `127.0.0.1:1` (connection refused) or
`10.255.255.1:1502` (connect timeout).

The Dockerfile cross-compiles on the build platform and defaults `BUILDPLATFORM`/`TARGETARCH`/`TARGETOS`, so it also
works with the legacy (non-BuildKit) builder. CI (`.github/workflows/docker.yml`) publishes multi-arch images to
`ghcr.io/<repository owner>/energy-exporter`. GitHub Actions must be enabled on the fork, and new GHCR packages are
private by default.

## Deployment (home lab)

- Host `docker0.localdomain` (10.0.0.183, Ubuntu 22.04, Docker 29, only the legacy `docker-compose` v1). SSH alias
  `docker0` (user `james` in the `docker` group, key `~/.ssh/id_ed25519_docker0`).
- Plain `docker run`, no compose file. The image is built on the host from the committed tree:
  ```bash
  git archive --format=tar HEAD | ssh docker0 'rm -rf ~/src/EnergyExporter && mkdir -p ~/src/EnergyExporter && tar -x -C ~/src/EnergyExporter'
  ssh docker0 'cd ~/src/EnergyExporter && docker build -t energy-exporter:local --label org.opencontainers.image.revision=<sha> . \
    && docker rm -f energy-exporter \
    && docker run -d --name energy-exporter --restart=always -p 14552:14552 \
         -v /usr/energyexporter/appsettings.json:/app/appsettings.json:ro energy-exporter:local'
  ```
  Tag the running image first (e.g. `docker tag energy-exporter:local energy-exporter:<previous>`) to keep a rollback.
- Config: `/usr/energyexporter/appsettings.json` (backup `appsettings.json.bak-20261108`). `Urls` is
  `http://*:14552`, so the port mapping is `14552:14552`.
- Devices: SolarEdge SE10000H single-phase inverter at `10.0.0.95:1502`, unit 1, plus meter index 0
  (SE-MTR-3Y-400V-B). The inverter accepts multiple Modbus TCP connections at once.
- Rollback images: `energy-exporter:net8` (fork on .NET 8, commit `e276c76`) and the original upstream
  `ghcr.io/marcuswichelmann/energy-exporter:3` (container `energy-exporter-old`, stopped, `restart=no`).

## History of this fork (November 2026)

1. **Diagnosis.** The upstream container had logged 82,530 failed polls, each with a stack trace: mostly
   `No route to host` (inverter offline, e.g. at night), plus `Connection refused` and connection resets. The code
   had no connect timeout (hangs of about 2 minutes), let polls overlap via an `async void` timer, blocked startup
   on the first poll and never retried.
2. **`e276c76` Reliability and management UI.** Timeouts, non-overlapping polls, stale-connection retry, one log
   line per outage, independent exporters, device health metrics and the management UI. Data fixes: SunSpec
   "not implemented" values (previously exported as 655.35 A / 6553.5 V on unused phases), floating-point
   artifacts from negative scale factors, register chunking that could skip an isolated trailing register, and
   startup validation of `Index` and `Port`.
3. **`13efdb5` .NET 10.** .NET 8 support ended on 2026-11-10. All packages were updated to their latest versions,
   which removed vulnerable transitive `System.Text.Json` 7.0.2 and `RestSharp` 110.1.0. Swashbuckle was replaced
   by the built-in `Microsoft.AspNetCore.OpenApi` plus `Swashbuckle.AspNetCore.SwaggerUI`, and the MQTT exporter
   was adapted to MQTTnet 5 (`MqttClientFactory`, no `MQTTnet.Client` namespace).
4. **Result.** In the first week on `13efdb5`, 34 of 45,805 polls failed (0.07%) across 16 short outages, each
   logged once.
5. **docker0 housekeeping.** Removed the unused `portainer_agent` (crash-looping: its Docker API 1.42 is below
   Docker 29's minimum of 1.44), 17 stopped containers (stray `cloudflared` runs, the `mcp-grafana` test stack,
   Node-RED, a Bitbucket runner) and their images, freeing about 4.3 GB. The `node_red_data` volume was kept.

Language decision: staying on C#/.NET. Go was considered (smaller image, single binary) but rejected for now;
a rewrite would not improve behaviour.

## Open items

- `SerialNumberOverride` for the meter is still the sample value `123456789`. It should be the meter's real serial,
  which labels its metrics; changing it starts new series.
- Upstream PR: the branch `improve-reliability-and-ui` could be offered upstream (drop the CI image-name change).
- docker0: 54+ pending apt updates and a reboot to load newer kernels (running 5.15.0-176), `grafana/pdc-agent`
  runs a 2-year-old `:latest` image, and `docker-compose` v1 is end-of-life (Ubuntu package `docker-compose-v2`).
  Unused images still on the host include `grafana/otel-lgtm`, `mongo`, `unifi-network-application`,
  `otel/opentelemetry-collector-contrib` and the locally built `unraid_apps_restarter` (not re-pullable).
- `purestorage-health-monitor` on docker0 stopped (exit 0) around 2026-11-10. This was not part of the work above;
  check whether it was stopped on purpose.
