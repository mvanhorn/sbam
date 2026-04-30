# Feature: MQTT feed (telemetry + control) with Home Assistant Discovery

> Slug: `64-issue-mqtt-feed` · Created: 2026-04-30
> Source issue: [#64](https://github.com/atbore-phx/sbam/issues/64)
> Target release: **v2.0.0** (major — first user-visible integration surface)

## Summary

Today the only way to observe sbam is to scrape its logs. This feature adds a
first-class **MQTT integration** that lets sbam:

1. **Publish telemetry** about every scheduling tick (forecast, battery state,
   computed net power, decision taken, charge percentage written to the
   inverter, errors).
2. **React to commands** received over MQTT so external orchestrators
   (Node-RED, Home Assistant automations, n8n, …) can trigger / inhibit /
   override charging without modifying sbam itself.
3. **Auto-register entities in Home Assistant** via the
   [MQTT Discovery](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery)
   protocol, so users get sensors and controls in HA with zero YAML.

The feature is opt-in: when no broker is configured, sbam keeps behaving
exactly as it does today.

## Motivation / User Story

> "If sbam could send & receive MQTT messages then we would be able to (1)
> monitor & record what it's doing, (2) enable the control of sbam for
> complex tasks via Node-RED or similar, eliminating the requirement for
> complex logic within sbam itself."
> — issue [#64](https://github.com/atbore-phx/sbam/issues/64), @travellingkiwi

> "SBAM exists either as Home Assistant add-on or as standalone binary. No
> sensors are available at the moment […] I think the task deserves a major
> v2.0.0."
> — @atbore-phx, same issue.

The HA add-on user base in particular wants entities (sensors, switches,
buttons) to wire into dashboards and automations. Doing this via MQTT
Discovery is strictly better than building and maintaining a bespoke REST API
because:

- The HA add-on already ships with the official Mosquitto add-on as a common
  dependency; users already have a broker.
- Discovery means **zero HA-side configuration**: sbam describes its own
  entities.
- The same payloads work for non-HA users (Node-RED, Grafana via
  telegraf-mqtt, etc.) without any HA coupling in sbam.

## Scope

In scope:

- A new `pkg/mqtt` package implementing connect / publish / subscribe with
  reconnection, last-will, and TLS support.
- Telemetry publishing from the `schedule` workflow (both single-shot and
  cron modes) on every tick.
- Home Assistant MQTT Discovery payloads for the published entities.
- Command topics for the runtime overrides listed in **Functional
  Requirements** below.
- Configuration via the standard sbam precedence chain
  (flag > env > yaml > default), wired into the HA add-on schema.
- Unit tests using an in-process MQTT broker
  ([`mochi-mqtt/server`](https://github.com/mochi-mqtt/server) v2) — same
  pattern as `tbrandon/mbserver` for Modbus.

Out of scope (explicitly deferred to follow-up issues):

- A REST/HTTP API on sbam itself (MQTT Discovery removes the need for v2.0).
- Persistence / history of published values (broker / HA handle this).
- Authentication beyond username/password and TLS (no client certs in v2.0).
- A web UI.
- Re-publishing arbitrary inverter Modbus registers — only the values sbam
  already computes are published.

## Functional Requirements

### Telemetry (publish)

- **FR1.** When `mqtt_enabled=true`, on every successful schedule tick sbam
  MUST publish to `<base_topic>/state` a single retained JSON document with
  at least the following fields:
  - `ts` (RFC3339 timestamp)
  - `forecast_wh` (daily solar production estimate, Wh)
  - `consumption_wh` (configured `pw_consumption`)
  - `battery_soc_pct` (state of charge, %)
  - `battery_capacity_wh`, `battery_remaining_wh`, `battery_to_charge_wh`
  - `pw_net_wh` (net power = battery + (forecast − consumption))
  - `decision` (one of: `idle`, `forecast_charge`, `reserve_charge`,
    `battery_full`, `out_of_window`)
  - `charge_pct` (the int16 charge percentage written to the inverter; `0`
    when no charge command was issued)
  - `forecast_charge_window` (`true` while inside `[start_hr, end_hr]`)
  - `reserve_charge_window` (`true` while inside
    `[batt_reserve_start_hr, batt_reserve_end_hr]`)
- **FR2.** Errors raised during a tick MUST be published to
  `<base_topic>/error` (retained, JSON: `{ts, source, message}`) so that HA
  / Node-RED can react to outages without parsing logs.
- **FR3.** Connection liveness MUST be exposed via:
  - `<base_topic>/availability` = `online` on connect, `offline` as MQTT
    Last-Will (retained).
- **FR4.** All telemetry payloads MUST be valid JSON (one object per
  message); no binary, no CSV.

### Commands (subscribe)

- **FR5.** sbam MUST subscribe to `<base_topic>/cmd/#` and act on the
  following sub-topics (payloads are simple strings or JSON):
  - `cmd/charge`           → payload `{"pct": <0-100>, "duration_s": <int>}`
                              forces a charge for `duration_s` (or until next
                              tick if omitted) at `pct` percent. Equivalent
                              to a manual `sbam configure --force_charge`.
  - `cmd/stop`             → payload empty / `{}`. Restores defaults
                              (equivalent to `sbam configure -d`).
  - `cmd/pause`            → payload `{"until": "<RFC3339>"}` or duration
                              string. While paused, scheduled ticks still
                              publish telemetry but DO NOT write Modbus.
  - `cmd/resume`           → cancels an active pause.
  - `cmd/refresh`          → forces an immediate schedule tick out-of-band
                              (useful from HA automations / scripts).
- **FR6.** Every command MUST be acknowledged on `<base_topic>/cmd/ack`
  (JSON: `{ts, command, accepted, error}`) so callers can implement
  request/response.
- **FR7.** Unknown sub-topics under `cmd/#` MUST be logged at debug and
  acknowledged with `accepted=false, error="unknown command"`.

### Home Assistant MQTT Discovery

- **FR8.** When `mqtt_ha_discovery=true` (default `true` when MQTT is
  enabled), sbam MUST publish, once per connect, retained discovery configs
  under `<discovery_prefix>/{sensor,binary_sensor,button}/sbam_<unique_id>/…/config`
  for at least:
  - sensors: `forecast_wh`, `consumption_wh`, `battery_soc_pct`,
    `battery_remaining_wh`, `pw_net_wh`, `charge_pct`, `decision`
  - binary sensors: `forecast_charge_window`, `reserve_charge_window`,
    `paused`
  - buttons: `refresh`, `stop`, `resume`
  - The `device` block MUST group all entities under one HA device named
    "sbam — Smart Battery Advanced Manager", carrying `sw_version`
    (sbam version string injected at build time), `manufacturer="atbore-phx"`,
    `model="sbam"`, `identifiers=[<unique_id>]`.
- **FR9.** `unique_id` MUST be derived deterministically from
  `<fronius_ip>` (sanitised) so that re-deployments reuse the same HA
  entities and do not pollute the registry.
- **FR10.** On graceful shutdown (SIGINT/SIGTERM) sbam MUST publish
  `offline` to `<base_topic>/availability` before disconnecting (in
  addition to the LWT safety net).

### Reliability & safety

- **FR11.** MQTT MUST never block or break the scheduling loop. A broker
  outage is logged at warn and the scheduling tick proceeds normally; the
  client reconnects in the background with exponential backoff.
- **FR12.** Modbus writes triggered by `cmd/charge` MUST go through the
  exact same `fronius.ForceCharge` path used by the `configure` command —
  no parallel/duplicated write code.
- **FR13.** When MQTT is disabled (`mqtt_enabled=false`, the default),
  sbam MUST behave **byte-identically** to v1.x: no broker connection, no
  extra log lines, no new dependencies loaded into the hot path.

## Non-functional Requirements

- Backward compatibility: 100% — flags / env / yaml keys added are all
  optional and default to disabled. Existing deployments are unaffected.
- Performance: telemetry publish is fire-and-forget at QoS 0 by default
  (configurable up to QoS 1); no measurable impact on scheduling latency.
- Idiomatic Go: a `pkg/mqtt` package exposes a `Client` interface with a
  `New()` constructor; the schedule loop depends on the interface, not on
  the concrete paho client. No global state besides the package-level
  `utils.Log`.
- Security:
  - TLS supported (`mqtt_tls=true`, optional `mqtt_ca_file` for self-signed
    brokers; system roots used otherwise).
  - Credentials never logged — `mqtt_password` MUST be added to
    `utils.SecretKeys` so it shows as `***` in the startup parameter dump
    introduced by issue #70.
  - Command handler MUST validate inputs (range-check `pct ∈ [0, 100]`,
    `duration_s ≥ 0`, RFC3339 parsing) and reject malformed payloads with a
    structured ack — never panic on attacker-controlled input.

## Configuration Impact

New CLI flags / env vars / yaml keys (defaults shown):

| Key                    | Default              | Notes                                                |
| ---------------------- | -------------------- | ---------------------------------------------------- |
| `mqtt_enabled`         | `false`              | Master switch.                                       |
| `mqtt_broker`          | `""`                 | e.g. `tcp://core-mosquitto:1883`.                    |
| `mqtt_username`        | `""`                 | Optional.                                            |
| `mqtt_password`        | `""`                 | **Secret** (added to `utils.SecretKeys`).            |
| `mqtt_client_id`       | `sbam-<host-hash>`   | Auto-derived if empty.                               |
| `mqtt_base_topic`      | `sbam`               | Root for `state`, `error`, `availability`, `cmd/#`.  |
| `mqtt_qos`             | `0`                  | Allowed: 0, 1.                                       |
| `mqtt_tls`             | `false`              |                                                      |
| `mqtt_ca_file`         | `""`                 | PEM bundle for self-signed brokers.                  |
| `mqtt_ha_discovery`    | `true`               | Effective only when `mqtt_enabled=true`.             |
| `mqtt_discovery_prefix`| `homeassistant`      | HA standard prefix.                                  |

Home Assistant add-on (`home-assistant/addons/sbam/config.json`):

- All new keys above are exposed under `options` and validated under
  `schema` (passwords use the `password` schema type).
- `run.sh` exports each as the corresponding upper-cased env var.

## External Integrations Touched

- Solcast: none.
- Fronius Solar API: none (telemetry reads from existing in-memory values).
- Fronius Modbus: only via the existing `fronius.ForceCharge` /
  `fronius.Setdefaults` paths invoked by `cmd/charge` and `cmd/stop`.
- New: an MQTT broker (any v3.1.1 / v5 broker; tested against Mosquitto and
  the in-process `mochi-mqtt/server` mock).

## Acceptance Criteria

- [ ] With `mqtt_enabled=false` (default), `make test` passes and there is
      no functional or log diff vs. v1.x.
- [ ] With `mqtt_enabled=true` and a reachable broker, every successful
      `schedule` tick publishes one JSON message on `<base>/state`
      containing all FR1 fields, verified by an integration test against
      the in-process broker.
- [ ] Errors thrown inside a tick produce one message on `<base>/error`,
      verified by injecting a Modbus failure via `mbserver`.
- [ ] `<base>/availability` flips to `offline` (via LWT) when the sbam
      process is killed mid-run — verified by killing the test client and
      asserting on the broker's retained value.
- [ ] Publishing `1` to `<base>/cmd/refresh` triggers an extra schedule
      tick out-of-band; an ack appears on `<base>/cmd/ack`.
- [ ] Publishing a malformed `cmd/charge` payload yields
      `{accepted:false, error:"..."}` and does NOT touch the inverter.
- [ ] On connect, retained discovery configs appear under
      `homeassistant/sensor/sbam_<id>/…/config` for every entity listed in
      FR8; a real HA instance subscribed to the same broker shows the
      device with all entities populated.
- [ ] Broker outage: stopping the broker mid-run logs a warn but does NOT
      stop the scheduling loop; on broker restart the client reconnects and
      republishes discovery + availability.
- [ ] `mqtt_password` shows as `***` in the startup parameter dump
      (`DEBUG=true sbam schedule …`).
- [ ] `make build` passes with `CGO_ENABLED=0`.
- [ ] HA add-on `config.json` schema validates the new keys; the add-on
      starts cleanly with the new options left at defaults.

## Test Strategy

Following the conventions in [.github/copilot-instructions.md](../../../.github/copilot-instructions.md):

- `pkg/mqtt/mqtt_test.go` (new): expected, edge, failure cases against an
  in-process `mochi-mqtt/server` broker spun up in `TestMain` and torn down
  with `defer broker.Close()`. Cases include:
  - publish round-trip on `state` with full payload (expected),
  - reconnect after a forced server stop/start (edge),
  - publish attempt with a bad QoS and credential failure (failure).
- `pkg/cmd/schedule_mqtt_test.go` (new): wires a fake `mqtt.Client`
  (interface) into the schedule loop and asserts that one tick produces
  exactly one `state` publish with the expected decision string for the
  classic three branches (`forecast_charge`, `reserve_charge`,
  `battery_full`). Combined with existing `tbrandon/mbserver` and
  `httptest.NewServer` mocks for Modbus and Solcast.
- `pkg/mqtt/discovery_test.go` (new): asserts the JSON structure of the
  discovery configs (presence of `device.identifiers`, `unique_id`,
  `availability_topic`, `state_topic`, `value_template`).
- `pkg/mqtt/commands_test.go` (new): table-driven test covering every
  `cmd/*` topic: payload validation, ack content, and that `cmd/charge`
  ultimately calls the same Modbus path as `configure --force_charge`
  (asserted via the mock Modbus server).

## Risks / Open Questions

- **Library choice.** Default proposal:
  [`github.com/eclipse/paho.mqtt.golang`](https://github.com/eclipse/paho.mqtt.golang)
  — battle-tested, MIT-licensed, supports auto-reconnect, TLS, LWT, QoS 0/1/2.
  Alternative: `github.com/mochi-mqtt/server` only ships a server; the
  natural client pairing remains paho. **Decision deferred to PLAN.**
- **HA Discovery payload churn.** Discovery is published once per connect;
  if the entity set changes between sbam versions, stale entries can
  accumulate. Mitigation: ship a `cmd/cleanup_discovery` admin command that
  publishes empty retained payloads to old config topics. v2.0 ships the
  initial set; cleanup is deferred.
- **Cron + MQTT trigger interleaving.** `cmd/refresh` and `cmd/charge` may
  race with the cron tick. Mitigation: serialise all schedule executions
  through a single goroutine fed by a buffered channel; cron and MQTT
  commands enqueue intents.
- **HA add-on dependency on Mosquitto.** Recommend (don't require) the
  Mosquitto add-on in `home-assistant/addons/sbam/DOCS.md`; sbam itself
  remains broker-agnostic.

## References

- Issue: [#64](https://github.com/atbore-phx/sbam/issues/64)
- HA MQTT Discovery: <https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery>
- Eclipse Paho Go: <https://github.com/eclipse/paho.mqtt.golang>
- In-process broker for tests: <https://github.com/mochi-mqtt/server>
- Related: issue #70 (startup parameter dump — must redact `mqtt_password`).
- Related: issue #68 (flag > env > yaml precedence — new keys must follow it).

## Clarifications

> 2026-04-30 — initial draft assumptions (open for the maintainer to
> confirm/revise before PLAN execution):
>
> - Slug: `64-issue-mqtt-feed`.
> - Target release: v2.0.0 (major; MQTT is the first user-visible
>   integration surface beyond Modbus/HTTP).
> - HA integration path: **MQTT Discovery only** in v2.0 (no parallel REST
>   API). Rationale: zero HA-side configuration, broker-agnostic, no
>   maintenance of a second transport.
> - Library: `github.com/eclipse/paho.mqtt.golang` for the client; tests
>   use `github.com/mochi-mqtt/server` v2 in-process.
> - Default behavior: MQTT off — strict zero-impact for v1.x users.
