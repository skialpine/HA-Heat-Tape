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
- **Install type**: Container (not HA OS) — no Add-ons section; uses "Apps" instead
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

## Automations Summary

| ID | Alias | Purpose |
|----|-------|---------|
| `heat_tape_freeze_latch_on` | Freeze – Latch ON | Turns on when in melt band; skips if forecast shows warming |
| `heat_tape_freeze_maintain_latched` | Freeze – Maintain ON | Enforces correct state; turns off when conditions no longer met |
| `heat_tape_freeze_daily_reset` | Freeze – Daily reset at 3:55 PM | Hard shutoff before TOU peak |
| `heat_tape_forecast_shutoff` | Forecast – Immediate shutoff | Fast shutoff when current + forecast both > 39°F |
| `heat_tape_snow_start_timer` | Snow – Start 12-day timer | Detects snow, stamps timestamp, starts 12-day window |
| `heat_tape_snow_recompute_on_restart` | Snow – Recompute after restart | Restores snow window correctly after HA restart |
| `heat_tape_snow_clear_flag` | Snow – Clear flag | Clears snow permit when 12-day timer finishes |
| `heat_tape_snow_cancel_timer` | Snow – Cancel timer on manual OFF | Handles manual disable, clears timestamp |

## Deployment Process

**Always run a code review before deploying.** Use the `pr-review-toolkit:code-reviewer` agent to check `heat_tape_package.yaml` for bugs, logic errors, and HA best-practice issues. Do not deploy until the review is clean or all flagged issues are resolved.

After modifying `heat_tape_package.yaml`, deploy to Edwards HA:

### Option 1: HA Configurator / File Editor (UI)
Use the File Editor add-on (if available) to copy the file to `/config/packages/heat_tape_package.yaml`.

### Option 2: MCP (ha-mcp tools) — preferred
```
mcp__home-assistant-ed__ha_call_service:
  domain: ha_mcp_tools
  service: write_file
  return_response: true
  data:
    path: packages/heat_tape_package.yaml
    content: <file contents>
    overwrite: true
```

`ha_mcp_tools` has been patched to allow writes to `packages/` and edits of the
`automation` YAML key. `write_file` is response-required — always pass `return_response: true`.

Alternatively, use `edit_yaml_config` to replace just the `automation` block:
```
mcp__home-assistant-ed__ha_call_service:
  domain: ha_mcp_tools
  service: edit_yaml_config
  return_response: true
  data:
    file: packages/heat_tape_package.yaml
    action: replace
    yaml_path: automation
    content: <automation list YAML>
```
After `edit_yaml_config`, call `automation.reload` — no restart needed.

### After write_file deployment: reload automations
```
mcp__home-assistant-ed__ha_call_service → domain: automation, service: reload
```

### Re-patching ha_mcp_tools after a HACS update
If ha_mcp_tools is updated via HACS, const.py will be overwritten. Re-apply the patch:
```
mcp__home-assistant-ed__ha_call_service → domain: shell_command, service: patch_ha_mcp_tools
homeassistant.restart
```
The staging file (`www/ha_mcp_tools_const.py`) and shell_command remain in place permanently.

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
