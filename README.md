# Home Assistant – Smart Roof Heat Tape Automation

This repository contains a **package-based Home Assistant automation** designed to intelligently control **roof de-icing heat tape** in cold, snowy climates (especially mountain and north-facing roofs).

The goal is to **prevent ice dams while minimizing electricity cost**, by running heat tape **only when it is effective** and **only during off-peak hours**.

This project evolved over real-world winter use and incorporates lessons learned from actual snow, temperature, and power-usage data.

---

## Design Goals

- Prevent ice dams without running heat tape continuously  
- Avoid running heat tape when it is ineffective (too cold or already melting)  
- Respect Time-of-Use (TOU) electric pricing  
- Handle real-world weather data imperfections  
- Allow safe manual override without fighting automation  
- Allow normal UI automations to coexist safely  

---

## Architecture (Important)

This project is implemented as a **Home Assistant package**.

That means:
- Heat-tape automations are **file-managed and GitHub-tracked**
- All other automations may continue to be created/edited in the **Home Assistant UI**
- No duplicate IDs or automation conflicts

### Package file
```
packages/
└── heat_tape.yaml
```

All heat-tape automations live inside this single file.

---

## Installation

### 1. Enable packages
Add (or merge) this into `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 2. Install the package file
Copy `heat_tape.yaml` into:

```
/config/packages/heat_tape.yaml
```

### 3. Restart Home Assistant
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

## Helpers Required

Create these helpers in  
**Settings → Devices & Services → Helpers**

### `input_boolean.freeze_latch`
Internal latch used by automations.  
**Do not manually control this in normal use.**

### `input_boolean.snow_in_last_12_days`
Indicates whether snow has fallen recently.
- Set automatically by snow detection
- Safe for **manual override** from the dashboard

### `timer.snow_recent_12d_timer`
Represents how long snow should be considered “recent.”

**Recommended duration:**  
- **288 hours (12 days)**

---

## Weather Integration (Pirate Weather)

All weather logic uses **Pirate Weather**.

OpenWeatherMap was removed due to unreliable snow signals in mountain conditions.

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
- Safe for **shared API keys**
- Prevents rate-limit errors (429)
- Snow logic uses debounce + long windows, so faster updates provide no benefit

---

## Snow Detection Logic

A snow event is detected when **any** of the following are true **for at least 10 continuous minutes**:

```jinja2
'snow' in weather state
or precip_intensity > 0.02 in/hr
or current_day_snow_accumulation >= 0.2 in
```

Accumulation is critical because intensity often under-reports mountain snowfall.

---

## 12-Day Snow Window Logic

When snow is detected:
- `snow_in_last_12_days` → ON
- `timer.snow_recent_12d_timer` → restarted to 12 days

The timer **does not restart continuously during a storm**.  
It only restarts on a new snow transition.

---

## Manual Override Behavior

If you manually turn **OFF** `snow_in_last_12_days` during an active storm:
- The system respects this override
- Snow must clear and restart for automation to resume

This prevents the system from fighting user intent.

---

## Melt-Band Logic (25–39°F)

Heat tape runs only when:
- Snow occurred recently
- Temperature is 25–39°F
- Time is within the off-peak daylight window
- Freeze latch is active

Below ~25°F heat tape is ineffective.  
Above ~39°F natural melting dominates.

---

## Time-of-Use / Daylight Window

System runs only during:
- **06:00 → 15:55**

At 15:55 daily:
- Heat tape turns OFF
- Freeze latch clears

This guarantees no TOU peak usage at 16:00.

---

## Automations Included

| Automation | Purpose |
|-----------|--------|
| Heat Tape – Freeze – Latch ON | Enables system during melt band |
| Heat Tape – Freeze – Maintain ON | Enforces correct on/off behavior |
| Heat Tape – Freeze – Daily reset | Stops system before TOU peak |
| Heat Tape – Snow – Start timer | Detects snow and starts 12-day window |
| Heat Tape – Snow – Clear flag | Clears snow flag when timer ends |
| Heat Tape – Snow – Cancel timer | Respects manual override |

---

## Verification Checklist

After a storm:
- `snow_in_last_12_days` = ON
- Timer ≈ 288 hours
- Heat tape runs only in 25–39°F window
- Timer counts down normally

---

## License

Apache License 2.0

---

## Disclaimer

This project controls high-amperage electrical equipment.  
Always follow local electrical codes and consult a qualified electrician.
