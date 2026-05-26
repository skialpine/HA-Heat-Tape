# Home Assistant – Smart Roof Heat Tape Automation

This repository contains a **package-based Home Assistant automation** designed to intelligently control **roof de-icing heat tape** in cold, snowy climates (especially mountain and north-facing roofs).


The goal is to **prevent ice dams while minimizing electricity cost**, by running heat tape **only when it is effective** and **only during off-peak hours**.

---

## Design Goals

- Prevent ice dams without running heat tape continuously
- Avoid running heat tape when it is ineffective (too cold or already melting)
- Respect Time-of-Use (TOU) electric pricing
- Handle real-world weather data imperfections
- Allow safe manual override without fighting automation
- Allow normal UI automations to coexist safely

---

## Architecture

This project is implemented as a **Home Assistant package**.

- Heat-tape automations are **file-managed and GitHub-tracked**
- All other automations may continue to be created/edited in the **Home Assistant UI**
- No duplicate IDs or automation conflicts

### Package file
```
packages/
└── heat_tape_package.yaml
```

All heat-tape automations live inside this single file.

---

## Installation

### 1) Enable packages

Add (or merge) this into `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 2) Install the package file

Copy `heat_tape_package.yaml` into:

```
/config/packages/heat_tape_package.yaml
```

(If you prefer a different filename you may rename the file during install; the automations expect the entities and helpers described below.)

### 3) Create the required Helpers (2 toggles + timer + last-snow datetime)

Go to: **Settings → Devices & Services → Helpers → Create Helper**

#### Helper 1: Freeze latch (toggle)
- Type: **Toggle**
- Name: `Freeze latch`
- Entity ID (recommended): `input_boolean.freeze_latch`

**Purpose:** internal latch used by automations.  
**Normal use:** do **not** manually control this.

#### Helper 2: Snow in last 12 days (toggle)
- Type: **Toggle**
- Name: `Snow in last 12 days`
- Entity ID (required): `input_boolean.snow_in_last_12_days`

**Purpose:** “permit” switch that gates heat tape operation.
- Set automatically by snow detection
- Safe for **manual override** from a dashboard

#### Helper 3: Snow recent timer (12 days)
- Type: **Timer**
- Name: `Snow recent 12d timer`
- Entity ID (required): `timer.snow_recent_12d_timer`
- Duration: **288:00:00** (12 days)

Notes:
- Some UIs show only hours/min/sec. Enter **288 hours**.
- Timers can show **idle** after Home Assistant restarts; this project restores the timer correctly (see “Reboot-safe” section below).

#### Helper 4: Last snow detected (date+time)
- Type: **Date and/or Time**
- Name: `Last snow detected`
- Entity ID (required): `input_datetime.last_snow_detected`
- Enable: **Date and Time**

**Purpose:** reboot-safe tracking of the last detected snow (or manual enable).
This allows Home Assistant to correctly restore the “recent snow” window even after a restart.

Important: configure this input_datetime with both date and time (including seconds) to ensure correct parsing of timestamps written by the automations.

### 4) Restart Home Assistant

A full restart is required.

---

## Hardware Used

### Zooz ZEN78 800LR High-Power Relay (40A)

Product page:  
https://www.thesmartesthouse.com/products/zooz-z-wave-long-range-high-power-relay-zen78-800lr

Why this relay works well for heat tape:
- 40A resistive load rating
- Supports 120–277V
- Z-Wave 800 Long Range
- Built-in energy monitoring
- Eliminates need for external contactor in most installs

⚠️ **Electrical safety**  
Heat tape is a high-amperage load. Use a properly sized breaker, follow local code (often GFCI/GFPE), and consult a licensed electrician if unsure.

---

## Weather Integration (Pirate Weather)

All weather logic uses **Pirate Weather**.

### Required entities

Enable:
- `weather.pirateweather`
- `sensor.pirateweather_temperature`
- `sensor.pirateweather_precip_intensity`
- `sensor.pirateweather_current_day_snow_accumulation`

### Recommended update interval

```yaml
update_interval: 1200   # seconds (20 minutes)
```

Why 1200 seconds:
- Safe for **shared API keys** (especially if you have two HA instances)
- Prevents rate-limit errors (429)
- Snow logic uses debounce + long windows, so faster updates provide little benefit

---

## Snow Detection Logic

A snow event is detected when **any** of the following are true **for at least 10 continuous minutes**:

```jinja2
'snow' in weather state
or precip_intensity > 0.02 in/hr
or current_day_snow_accumulation >= 0.2 in
```

---

## 12-Day Snow Window Logic (Reboot-safe)

This project uses **three layers**:

1) `input_datetime.last_snow_detected` (truth / durable)
2) `input_boolean.snow_in_last_12_days` (permit flag)
3) `timer.snow_recent_12d_timer` (UI countdown)

### When snow is detected (or you manually enable the permit)
- `last_snow_detected` is stamped with the current time
- `snow_in_last_12_days` turns ON
- the 12-day timer starts/resets

### After a Home Assistant restart
Timers often restore as **idle**, even if the snow flag is still ON.

To fix that, the package includes an automation that:
- recomputes how long it has been since `last_snow_detected`
- turns the permit ON/OFF correctly
- restarts the timer with the **remaining** duration

Note: the automation formats the remaining duration as an explicit HH:MM:SS string when restarting the timer for robust behaviour across HA versions.

### Manual OFF behavior
If you manually turn **OFF** `snow_in_last_12_days`, the package:
- cancels the timer
- clears `last_snow_detected` (the automation sets it to the sentinel value `1970-01-01 00:00:00`)
- prevents a restart from immediately re-enabling the snow window

---

## Melt-Band Logic (25–39°F)

Heat tape runs only when:
- Snow occurred recently (permit is ON)
- Temperature is 25–39°F
- Time is within the off-peak daylight window (06:00 → 15:55)

Below ~25°F heat tape is often ineffective.  
Above ~39°F natural melting dominates.

### Forecast-aware behavior

**Startup skip:** Before latching ON, the package fetches the next two hourly forecast periods. If either period exceeds 39°F, startup is skipped entirely — natural melting will dominate shortly and running the tape would be wasteful. The automation re-evaluates if the temperature remains in the melt band.

**Immediate shutoff:** When temperature rises above 39°F, the package checks the next-hour forecast. If the forecast also exceeds 39°F, the tape is turned off immediately rather than waiting the standard one-hour delay. This saves approximately 30% runtime on warm days compared to relying on delay-based triggers alone. If the forecast is unavailable or uncertain, the one-hour delay trigger acts as a backstop.

---

## Seasonal-Off Gate (rolling 7-day window)

To automatically suppress heat-tape operation when the season has turned, the package tracks a rolling 7-day window of the past week's temperatures via two `statistics` platform sensors:

- `sensor.pirateweather_temp_min_7d` — rolling 7-day MIN of `sensor.pirateweather_temperature`
- `sensor.pirateweather_temp_max_7d` — rolling 7-day MAX of `sensor.pirateweather_temperature`

These feed a template binary sensor:

- `binary_sensor.heat_tape_seasonal_off` — ON when **both** of these are true:
  - 7-day MIN > 32°F (no freezing all week)
  - 7-day MAX > 50°F (warm enough to call it spring)

While `binary_sensor.heat_tape_seasonal_off` is ON, the **Latch ON** automation will not start the tape, and the **Maintain ON** automation will shut it off if currently running.

**Fail-safe:** the binary sensor reports OFF (allow runs) whenever either statistics sensor is unavailable — for example, during the first 7 days after a fresh deploy, or if PirateWeather stops reporting. The default is to keep the system armed.

**Why rolling min/max (not "average daily high/low")**
Using a strict 7-day minimum keeps the system armed if even one night dropped to freezing in the past week. Averaging daily lows could drown out a single cold snap and prematurely declare "summer," leaving the tape disabled on a morning when ice dams are still possible. For heat tape, failing to run when needed is worse than running unnecessarily — so the rule errs toward "stay armed."

These three entities are auto-created by the package — no manual helper setup is required.

---

## Time-of-Use / Daylight Window

System runs only during:
- **06:00 → 15:55**

At 15:55 daily:
- heat tape turns OFF
- freeze latch clears

Note: triggers at exact boundary times combined with `after`/`before` conditions can be sensitive to inclusivity semantics; if you observe missed events at 06:00, consider adjusting the condition bounds slightly or use a template-based time condition.

---

## Automations Included

| Automation | Purpose |
|-----------|--------|
| Heat Tape – Freeze – Latch ON | Enables system during melt band; skips if forecast shows warming within 2h or seasonal-off gate is ON |
| Heat Tape – Freeze – Maintain ON | Enforces correct on/off behavior while latched; respects seasonal-off gate |
| Heat Tape – Freeze – Daily reset | Stops system before TOU peak (15:55) |
| Heat Tape – Forecast – Immediate shutoff | Turns off immediately when current + next-hour forecast both > 39°F |
| Heat Tape – Snow – Start timer | Detects snow and stamps last snow time |
| Heat Tape – Snow – Recompute after restart | Restores recent-snow window after HA restart |
| Heat Tape – Snow – Clear flag | Clears snow permit when timer ends |
| Heat Tape – Snow – Cancel timer | Respects manual OFF and clears last-snow time |

---

## License

Apache License 2.0

---

## Disclaimer

This project controls high-amperage electrical equipment.  
Always follow local electrical codes and consult a qualified electrician.