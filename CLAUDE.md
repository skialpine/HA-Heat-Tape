# CLAUDE.md — HA-Heat-Tape Project Guide

## Project Purpose

Smart roof heat tape automation for a **mountain home in Edwards, CO** (high-altitude, snowy climate).
Controls a Zooz ZEN78 relay via Home Assistant to run heat tape **only when it's effective and economical**:
- Prevents ice dams by melting snow in the 25–39°F melt band
- Avoids running heat tape when temps are too cold (< 25°F, ineffective) or too warm (> 39°F, natural melting works)
- Respects Time-of-Use (TOU) electricity pricing: runs **only 06:00–15:55 Mountain Time** (off-peak daylight)
- Uses a 12-day snow window to prevent tape from running long after snow is gone

## Repository Structure

```
heat_tape_package.yaml   # All automations — file-managed, GitHub-tracked
tools.yaml               # Official Portainer MCP tool definitions (not HA-related)
README.md                # End-user install instructions
CLAUDE.md                # This file
.mcp.json                # Claude Code project-level MCP server config
```

## MCP Servers (`.mcp.json`)

Three MCP servers are pre-configured for this project:

| Key | Description | Connection |
|-----|-------------|------------|
| `portainer` | Docker/Portainer on the home server | `docker-vm.local:9443` (official Go binary at `/Users/jeffreyh/bin/portainer-mcp`) |
| `home-assistant` | **Chivington (CHV)** — primary home HA | `http://homeassistant-chv.local:8123` |
| `home-assistant-ed` | **Edwards (ED)** — mountain home HA | `http://homeassistant-ed.tail7752d4.ts.net:8123` (Tailscale) |

**Heat tape runs on the Edwards (ED) instance.** Use `mcp__home-assistant-ed__*` tools for heat tape work.

The `portainer` MCP uses the official Portainer MCP binary (Go, not ginkida's Python version).
The `-server` arg takes `host:port` **without** `https://` prefix — the binary adds it automatically.

## Target Home Assistant Instance

- **Instance**: Edwards (ED) at `http://homeassistant-ed.tail7752d4.ts.net:8123`
- **Install type**: Home Assistant OS 17.3 (core-2026.5.4) — Apps store available. CHV also HA OS 17.3.
- **Package path**: `/config/packages/heat_tape_package.yaml`
- **`configuration.yaml`** must include:
  ```yaml
  homeassistant:
    packages: !include_dir_named packages
  ```

## Key Entities

### Switches / Hardware
| Entity | Description |
|--------|-------------|
| `switch.heat_tape` | Zooz ZEN78 800LR relay — controls heat tape power |

### Weather (Pirate Weather integration)
| Entity | Description |
|--------|-------------|
| `weather.pirateweather` | Weather state (used for snow detection) |
| `sensor.pirateweather_temperature` | Current temperature (°F) |
| `sensor.pirateweather_precip_intensity` | Precip rate (in/hr) |
| `sensor.pirateweather_current_day_snow_accumulation` | Daily snow accumulation (in) |

Forecast data is fetched on demand via `weather.get_forecasts` (type: hourly).
Recommended update interval: `1200` seconds (20 min) — avoids rate limits on shared API keys.

### Helpers (must be created manually in HA)
| Entity | Type | Description |
|--------|------|-------------|
| `input_boolean.freeze_latch` | Toggle | Internal automation latch — do not manually toggle |
| `input_boolean.snow_in_last_12_days` | Toggle | "Permit" gate — safe to manually override from dashboard |
| `timer.snow_recent_12d_timer` | Timer | 12-day countdown (288h) — UI display of remaining window |
| `input_datetime.last_snow_detected` | Date+Time | Durable timestamp for reboot-safe recovery |

## Automation Logic Overview

### Melt Band: 25–39°F
- **Below 25°F**: heat tape is ineffective (ice won't melt at that temp)
- **25–39°F**: melt band — heat tape is active
- **Above 39°F**: natural melting dominates — tape is wasteful

### TOU Window: 06:00–15:55
- Heat tape only runs during off-peak daylight hours
- `heat_tape_freeze_daily_reset` shuts everything off at 15:55 daily

### 12-Day Snow Window (reboot-safe, 3-layer design)
1. `input_datetime.last_snow_detected` — durable timestamp (truth source)
2. `input_boolean.snow_in_last_12_days` — permit flag checked by freeze automations
3. `timer.snow_recent_12d_timer` — UI countdown, also used for auto-clear

Snow is detected when **any** of these are true for 10+ continuous minutes:
- Weather state contains "snow"
- `sensor.pirateweather_precip_intensity` > 0.02 in/hr
- `sensor.pirateweather_current_day_snow_accumulation` >= 0.2 in

On restart, `heat_tape_snow_recompute_on_restart` recomputes remaining time from `last_snow_detected` and restores the timer with the correct remaining duration.

Manual OFF of `snow_in_last_12_days` also clears `last_snow_detected` (sets to `1970-01-01`) to prevent restart re-enabling.

### Forecast-Aware Logic
Both startup and shutoff use `weather.get_forecasts` (hourly) to avoid wasteful runs:

**Startup skip** (`heat_tape_freeze_latch_on`): Before latching ON, checks next 2 forecast hours.
If either exceeds 39°F, skips turn-on — natural melting will dominate shortly.

**Immediate shutoff** (`heat_tape_forecast_shutoff`): When temp rises above 39°F, checks next-hour
forecast. If also > 39°F, turns off immediately rather than waiting the 1-hour delay in "Maintain ON".
Estimated savings: ~30% runtime reduction on warm days.

### Seasonal-Off Gate (rolling 7-day window)
A `template:` binary sensor `binary_sensor.heat_tape_seasonal_off` suppresses the tape when
the past 7 days indicate the season has turned. Fed by two `statistics` platform sensors:
- `sensor.pirateweather_temp_min_7d` — rolling 7-day MIN of `sensor.pirateweather_temperature`
- `sensor.pirateweather_temp_max_7d` — rolling 7-day MAX

Gate is ON (suppress) when **both**: min > 32°F AND max > 50°F. Fails OFF (allow runs) when
either statistics sensor is unavailable. Rolling min/max chosen over avg-daily because one
freezing night keeps the system armed — the safer direction for heat tape.

The gate condition is wired into both `heat_tape_freeze_latch_on` and the stay-on branch of
`heat_tape_freeze_maintain_latched`.

## Automations Summary

| ID | Alias | Purpose |
|----|-------|---------|
| `heat_tape_freeze_latch_on` | Freeze – Latch ON | Turns on when in melt band; skips if forecast shows warming or seasonal-off gate active |
| `heat_tape_freeze_maintain_latched` | Freeze – Maintain ON | Enforces correct state; turns off when conditions no longer met or seasonal-off gate active |
| `heat_tape_freeze_daily_reset` | Freeze – Daily reset at 3:55 PM | Hard shutoff before TOU peak |
| `heat_tape_forecast_shutoff` | Forecast – Immediate shutoff | Fast shutoff when current + forecast both > 39°F |
| `heat_tape_snow_start_timer` | Snow – Start 12-day timer | Detects snow, stamps timestamp, starts 12-day window |
| `heat_tape_snow_recompute_on_restart` | Snow – Recompute after restart | Restores snow window correctly after HA restart |
| `heat_tape_snow_clear_flag` | Snow – Clear flag | Clears snow permit when 12-day timer finishes |
| `heat_tape_snow_cancel_timer` | Snow – Cancel timer on manual OFF | Handles manual disable, clears timestamp |

## Deployment Process

**Always run a code review before deploying.** Use the `pr-review-toolkit:code-reviewer` agent to check `heat_tape_package.yaml` for bugs, logic errors, and HA best-practice issues. Do not deploy until the review is clean or all flagged issues are resolved.

After modifying `heat_tape_package.yaml`, choose a deploy path based on what changed.

### Decision matrix

| What changed | Use | Restart needed? |
|---|---|---|
| Only `sensor:`, `template:`, `binary_sensor:`, `mqtt:`, `group:`, etc. (anything in `ALLOWED_YAML_KEYS`) AND those keys already existed in the file at last HA startup | `edit_yaml_config` action=`replace`/`add`/`remove` | No for `template`/`mqtt`/`group` (reload service available). YES for `sensor:` and most others — statistics platform needs restart. |
| First time adding a top-level integration key (e.g. introducing `template:` to a file that didn't have one) | Same `edit_yaml_config` call **+ restart** | Yes — modern `template:` integration only bootstraps at HA startup; `reload_all` won't initialize a never-loaded integration |
| Anything touching `automation:` (added/removed/edited automations or their conditions/triggers/actions) | `write_file` (full-file rewrite) → `automation.reload` | No restart (just `automation.reload`) |
| Mixed: both `automation` and new sensor/template blocks | `write_file` full file + restart (because of the new top-level integration) | Yes |

### Preferred path: `edit_yaml_config` (no patch required)

`edit_yaml_config` natively supports `packages/*.yaml`. **No patching needed.** It supports `add` / `replace` / `remove` on top-level keys in the allowed-keys whitelist (`template`, `sensor`, `binary_sensor`, `mqtt`, `command_line`, `group`, `utility_meter`, `shell_command`, `switch`, `light`, `fan`, `cover`, `climate`, `notify`, `rest`, `knx`). **`automation` is intentionally excluded** because upstream's design is "use `ha_config_set_automation` for automation edits" — but that targets `automations.yaml`, not packages, so for package automations we still need `write_file` (see fallback below).

```
mcp__home-assistant-ed__ha_call_service:
  domain: ha_mcp_tools
  service: edit_yaml_config
  return_response: true
  data:
    file: packages/heat_tape_package.yaml
    action: replace        # or "add" / "remove"
    yaml_path: template    # any key in ALLOWED_YAML_KEYS
    content: <YAML list/dict to set as the value under yaml_path>
```

After: if `post_action` in the response is `reload_available`, call its reload service. If `restart_required`, call `homeassistant.restart`.

### Fallback path: `write_file` (full-file rewrite, requires patch)

Needed when:
- Changing the `automation:` block in a package
- Doing a full-file rewrite (faster than N `edit_yaml_config` calls)

`write_file` is hardcoded to `www/`, `themes/`, `custom_templates/`, `dashboards/` — to add `packages/`, the integration must be patched. Upstream issue requesting an opt-in config option: [homeassistant-ai/ha-mcp#1451](https://github.com/homeassistant-ai/ha-mcp/issues/1451).

```
mcp__home-assistant-ed__ha_call_service:
  domain: ha_mcp_tools
  service: write_file
  return_response: true
  data:
    path: packages/heat_tape_package.yaml
    overwrite: true
    content: <full file contents>
```

After `write_file` deployment: call `automation.reload`. Restart only if new top-level integrations were introduced.

### Re-patching `ha_mcp_tools` after a HACS update

If HACS updates `ha_mcp_tools`, the in-place `const.py` is overwritten and the patch is lost. **The staging file at `www/ha_mcp_tools_const.py` is almost certainly stale** (missing constants added in newer upstream versions, e.g. `DASHBOARD_URL_PATH_PATTERN`). If you blindly run the `patch_ha_mcp_tools` shell_command with the stale staging file, the integration will fail to import on next restart with an `ImportError`.

**Correct re-patch runbook:**

1. Refresh the staging file from current upstream + add `packages` to `ALLOWED_WRITE_DIRS`:
   ```bash
   curl -sL "https://raw.githubusercontent.com/homeassistant-ai/ha-mcp/master/custom_components/ha_mcp_tools/const.py" \
     | sed 's|ALLOWED_WRITE_DIRS = \["www", "themes", "custom_templates", "dashboards"\]|ALLOWED_WRITE_DIRS = ["www", "themes", "custom_templates", "dashboards", "packages"]|' \
     > /tmp/patched_const.py
   ```
2. Upload to staging via `ha_mcp_tools.write_file` → `path: www/ha_mcp_tools_const.py` (www/ is in the default allowlist, so no patch needed for this step).
3. Run `shell_command.patch_ha_mcp_tools` to copy staging → in-place.
4. `homeassistant.restart`.

If the integration is already bricked (ImportError on startup), recover by `ha_hacs_download` `homeassistant-ai/ha-mcp` to force a fresh install before re-patching.

### Important: HA readiness polling

When waiting for HA after a restart, **do not** poll unauthenticated HTTP endpoints (`/manifest.json` 404s, `/api/` 401s without a real token — both look like "not ready" to `curl -f` and the loop runs forever). Use `ha_get_state` on a known entity as the readiness probe — when it returns success, HA is up.

## Hardware Reference

**Zooz ZEN78 800LR** — Z-Wave Long Range High-Power Relay
- 40A resistive load rating (handles heat tape circuits)
- 120–277V input
- Built-in energy monitoring (tracks kWh)
- Z-Wave 800 Long Range (better range/reliability than Z-Wave Plus)

Typical heat tape power: ~400–600W for a residential install.
With TOU window (10h/day max) and melt-band gating, realistic peak-week runtime is ~20–35h.

## Keeping README.md in Sync

**Always keep `README.md` consistent with `heat_tape_package.yaml`.**

After modifying any automation logic, check the following README sections and update them if needed:

| README section | What to verify |
|----------------|----------------|
| **Automations Included** table | Matches all automation IDs and aliases in the package |
| **Melt-Band Logic** | Thresholds (25°F, 39°F) match the conditions in the YAML |
| **Time-of-Use / Daylight Window** | Start/end times match triggers and conditions (06:00, 15:55) |
| **Snow Detection Logic** | Thresholds and logic match the template trigger |
| **12-Day Snow Window Logic** | Three-layer description still accurate |
| **Install helpers** section | Helper entity IDs and types still match what the YAML expects |

If new automations are added, add a row to the Automations Included table.
If thresholds or helper entity IDs change, update the relevant README section.

## Common Tasks for Claude

### Check current state
```
mcp__home-assistant-ed__ha_get_state → switch.heat_tape
mcp__home-assistant-ed__ha_get_state → sensor.pirateweather_temperature
mcp__home-assistant-ed__ha_get_state → input_boolean.snow_in_last_12_days
```

### Check recent runtime
```
mcp__home-assistant-ed__ha_get_history → switch.heat_tape (last 7 days)
```

### Check automation traces
```
mcp__home-assistant-ed__ha_get_automation_traces → heat_tape_freeze_latch_on
```

### Manually control
To force heat tape off for summer / maintenance, turn off `input_boolean.snow_in_last_12_days`.
This triggers `heat_tape_snow_cancel_timer` → clears everything → heat tape won't run until snow is re-detected or manually re-enabled.

### Validate config before deploying
```
mcp__home-assistant-ed__ha_check_config
```
