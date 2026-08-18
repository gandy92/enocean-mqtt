# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`enocean-mqtt` is a small daemon that bridges an EnOcean radio interface (USB/serial transceiver)
and an MQTT broker. It receives EnOcean radio packets, decodes them via the `enocean` Python
library's EEP (EnOcean Equipment Profile) definitions, and publishes the decoded values to MQTT.
It can also go the other direction: MQTT messages trigger outgoing EnOcean packets (commands or
learn-acknowledgements) to configured devices.

## Running it

```bash
python -m venv venv && source venv/bin/activate
pip install -e .
enoceanmqtt [--debug] [--logfile PATH] [--envfile PATH] [config_file ...]
```

- Broker connection details should go in a `.env` file (`MQTT_HOST`/`MQTT_PORT`/`MQTT_USER`/
  `MQTT_PASSWORD`, see `enoceanmqtt.env.sample`), never in a tracked `enoceanmqtt.conf` — `.env`
  is gitignored specifically so this never needs to touch a config file that could be committed.
  `.env` is auto-discovered from the working directory upward, or pointed at via `--envfile`.
  Env values take precedence; `mqtt_host`/`mqtt_port`/`mqtt_user`/`mqtt_pwd` in the `[CONFIG]`
  section remain a fallback for backward compatibility only, not meant for new setups.

- No test suite, linter config, or CI checks exist in this repo — the only GitHub Action
  (`.github/workflows/pythonpublish.yml`) publishes to PyPI on GitHub release, it does not run
  tests. There is nothing to "run the tests" for; verify changes by exercising the daemon against
  a real or mocked EnOcean interface/MQTT broker, or by careful code reading.
- Config files are merged in order: `/etc/enoceanmqtt.conf`, then (in the Docker image)
  `/enoceanmqtt-default.conf`, then any files given as CLI args — later files override earlier
  ones. `enoceanmqtt.conf.sample` documents all recognized keys.
- `setup.py` still declares the version (`0.1.4`) and the two runtime dependencies (`enocean`,
  `paho-mqtt>=2`) — bump the version there for releases.

## Architecture

The whole application is two modules under `enoceanmqtt/`:

- **`enoceanmqtt.py`** — entry point. Parses CLI args, loads and merges `ConfigParser` INI files
  into a flat `global_config` dict plus a list of per-sensor dicts (one per non-`[CONFIG]`
  section), sets up logging, then hands off to `Communicator.run()`.
  - Sensor config values are coerced with `int(value, 0)` (so hex `0x...` and decimal both work)
    except for the string-typed keys `command`, `channel`, `publish_json`.
  - Each sensor's MQTT topic name is `mqtt_prefix + section_name` (default prefix `enocean/`).

- **`communicator.py`** — the entire runtime, in one `Communicator` class. Owns both the paho-mqtt
  client and the `enocean.communicators.SerialCommunicator`, and glues them together:
  - **EnOcean → MQTT**: `run()` blocks on `self.enocean.receive` (a queue fed by the EnOcean
    library's own thread), routes `PACKET.RADIO` packets to `_process_radio_packet()`, which
    matches the packet sender address against configured sensors, applies RORG-specific learn-flag
    quirks (VLD vs RPS vs 1BS/4BS), then calls `_read_packet()` → `_handle_data_packet()` (EEP
    decode via `packet.parse_eep()`) → `_publish_mqtt()`. Auxiliary fields `_RSSI_` and `_DATE_`
    are injected before decoding and stripped out / republished under a sensor's `STATE` subtopic
    or as separate topics depending on `publish_rssi`/`publish_date`/`publish_json` flags.
  - **MQTT → EnOcean**: `_on_mqtt_message()` dispatches to `_mqtt_message_normal()` (plain scalar
    payloads on `<sensor>/req/<field>` and `<sensor>/req/send`) or `_mqtt_message_json()` (JSON
    object payloads on `<sensor>/req`, with a `"send"` key triggering transmission). Both paths
    accumulate values into `sensor["data"]` until a send is triggered, then `_send_packet()`
    builds a `RadioPacket` via `packet.set_eep()`, applying `default_data` as a base and the
    sensor's configured `sender`/`direction`/`command` as needed.
  - Per-sensor and per-message boolean flags (`publish_json`, `publish_rssi`, `publish_date`,
    `persistent`, `log_learn`, `answer`, `ignore`, `mqtt_ssl*`, `mqtt_debug`, `log_packets`) are
    all read as loosely-typed config strings and compared against `("True", "true", "1")` — there
    is no central bool-parsing helper except `_sensor_enabled_for()`, which additionally supports
    falling back from a per-sensor key to a global `mqtt_<key>` config default.
  - `channel` on a sensor (`IO/CMD` syntax) groups published values into subtopics named after the
    values of those EEP fields, e.g. `<sensor>/IO1/CMD2`.

There are no other modules, no persistence layer, and no web/API surface — everything is this one
request/response loop between two blocking I/O sources (an MQTT client thread and an EnOcean
serial-reader thread), coordinated through the single-threaded `run()` loop.

## Working conventions

- Keep runtime logic confined to `communicator.py` unless it's genuinely CLI/config-loading
  concerned (which belongs in `enoceanmqtt.py`) — the project intentionally stays a two-file daemon.
- Config values arriving from either the INI file or MQTT are always strings/JSON-decoded scalars
  first; follow the existing pattern of explicit int-then-float coercion with logged warnings on
  failure rather than raising.


# Karpathy Principles

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

