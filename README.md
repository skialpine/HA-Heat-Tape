# Home Assistant – Smart Roof Heat Tape Automation

This repository contains a set of Home Assistant automations designed to intelligently control **roof de‑icing heat tape** in cold, snowy climates (especially mountain and north‑facing roofs).

The goal is to **prevent ice dams while minimizing electricity cost**, by running heat tape **only when it is effective** and **only during off‑peak hours**.

This project evolved over real‑world winter use and incorporates lessons learned from actual snow, temperature, and power‑usage data.

---

## Design Goals

- Prevent ice dams without running heat tape continuously
- Avoid running heat tape when it is ineffective (too cold or already melting)
- Respect Time‑of‑Use (TOU) electric pricing
- Handle real‑world weather data imperfections
- Allow safe manual override without fighting automation

---

## Hardware Used

### Zooz ZEN78 800LR High‑Power Relay (40A)

Product page:  
https://www.thesmartesthouse.com/products/zooz-z-wave-long-range-high-power-relay-zen78-800lr

Why this relay works well for heat tape:

- **40A resistive load rating** (suitable for large self‑regulating heat tape circuits)
- Supports **120–277V**
- **Z‑Wave 800 Long Range** (excellent reach to garages, attics, exterior panels)
- Built‑in **energy monitoring** (useful for cost analysis)
- Eliminates the need for an external contactor in most residential installs

> ⚠️ **Electrical safety**  
> Heat tape is a high‑amperage load. Use a properly sized breaker, follow local code (often GFCI/GFPE), and consult a licensed electrician if unsure.

---

## Helpers Required

Create the following helpers in  
**Settings → Devices & Services → Helpers**

### `input_boolean.freeze_latch`
Tracks whether the system is allowed to run during the current melt cycle.

### `input_boolean.snow_in_last_12_days`
Indicates whether snow has fallen recently.
- Set automatically by snow detection
- Can be manually toggled from the dashboard

### `timer.snow_recent_12d_timer`
Represents how long snow should be considered “recent.”

Recommended duration:
- **288 hours (12 days)**

---

## Weather Integration (Pirate Weather)

All weather logic uses **Pirate Weather**.

OpenWeatherMap was removed because it did not reliably expose snow intensity and accumulation signals needed for automation‑grade decisions in mountain conditions.

### Required entities

Enable these Pirate Weather entities:

- `weather.pirateweather`
- `sensor.pirateweather_temperature`
- `sensor.pirateweather_precip_intensity`
- `sensor.pirateweather_current_day_snow_accumulation`

### Recommended update interval

Snow detection uses a debounce window, so Pirate Weather must update frequently:

```yaml
update_interval: 600   # seconds (10 minutes)
```

This effectively requires **two consecutive updates** confirming snow before acting.

---

## Snow Detection Logic (Final)

A snow event is detected when **any** of the following are true **for at least 10 continuous minutes**:

```jinja2
'snow' in weather state
or precip_intensity > 0.02 in/hr
or current_day_snow_accumulation >= 0.2 in
```

### Why accumulation is trusted

Real‑world data showed that:
- Snow accumulation often increases while `precip_intensity` reports `0.00`
- Overnight and light mountain snow is frequently under‑reported by intensity alone

Using accumulation ≥ **0.2 inches** reliably captures real snowfall while filtering noise.

---

## 12‑Day Snow Window Logic

When snow is detected:

- `input_boolean.snow_in_last_12_days` → **ON**
- `timer.snow_recent_12d_timer` → restarted to **12 days**

When the timer expires:
- `input_boolean.snow_in_last_12_days` → **OFF**

### Why the timer does NOT restart mid‑storm

The timer only restarts when snow conditions transition:

```
false → true (and stay true for 10 minutes)
```

During a continuous storm, the condition never becomes false, so the timer **counts down normally** instead of constantly restarting.

This prevents thrashing and preserves a meaningful “time since last snow” window.

---

## Manual Override Behavior (Important)

If you manually turn **OFF** `input_boolean.snow_in_last_12_days` during an active storm:

- The system treats this as a **manual override for the remainder of that storm**
- The flag will **not** re‑enable automatically until:
  1. Snow conditions go false, and
  2. A **new** snow event begins and lasts 10 minutes

This prevents the automation from fighting intentional user actions.

---

## Melt‑Band Logic (25–39°F)

Heat tape is allowed to run only when **all** of the following are true:

- Snow occurred recently (`snow_in_last_12_days` = ON)
- Temperature is between **25°F and 39°F**
- Time is within the off‑peak daylight window
- `freeze_latch` is ON

### Why 25–39°F?

- Below ~25°F, heat tape is largely ineffective at moving meltwater
- Above ~39°F, natural melting typically reduces ice‑dam risk
- This band focuses energy where heat tape is most effective

---

## Time‑of‑Use / Daylight Window

The system runs only during:

- **06:00 → 15:55**

At **15:55**, a daily reset automation:
- Turns OFF the heat tape
- Clears the freeze latch

This guarantees the system never runs into TOU peak pricing at 16:00.

---

## Automation Files

```
automations/
├── snow_start_timer.yaml
├── snow_clear_flag.yaml
├── snow_cancel_timer.yaml
├── freeze_daily_reset.yaml
├── freeze_latch_on.yaml
└── freeze_maintain_latched.yaml
```

### Summary

| File | Purpose |
|-----|--------|
| snow_start_timer | Detect snow and start 12‑day timer |
| snow_clear_flag | Clear snow flag when timer expires |
| snow_cancel_timer | Cancel timer on manual OFF |
| freeze_daily_reset | Shut system down before TOU peak |
| freeze_latch_on | Enable system during melt band |
| freeze_maintain_latched | Maintain safe on/off behavior |

---

## Verification Checklist

After a storm:

- `snow_in_last_12_days` should be **ON**
- `timer.snow_recent_12d_timer` should show ~288 hours remaining
- Heat tape should turn on during mornings if temp is 25–39°F
- Timer should **count down**, not restart repeatedly

---

## License

Apache License 2.0  
See `LICENSE` for details.

---

## Disclaimer

This project controls high‑amperage electrical equipment.  
Always follow local electrical codes and consult a qualified electrician when needed.
