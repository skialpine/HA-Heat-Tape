# Home Assistant – Smart Roof Heat Tape Automation

This repository contains a set of Home Assistant automations designed to intelligently control **roof de-icing heat tape** using:

- Melt-band temperature logic  
- Time-of-use (TOU) off-peak scheduling  
- A “recent snowfall” window using a timer  
- A high-power Z-Wave relay (Zooz ZEN78 800LR)  
- **Pirate Weather** for accurate, real-time snow detection and temperature  

These automations help prevent ice dams while significantly reducing electricity cost by running the tape **only when conditions justify it**.

---

## Hardware Used

### Zooz ZEN78 800LR High-Power Relay (40A)

Product page:  
https://www.thesmartesthouse.com/products/zooz-z-wave-long-range-high-power-relay-zen78-800lr

This relay is ideal for roof heat tape control because:

- **40 A resistive load rating** (supports large self-regulating heat-tape circuits)  
- Compatible with **120–277V systems**  
- Uses **Z-Wave 800 Long Range** for excellent connectivity  
- Provides **energy monitoring**, helpful for tracking cost  
- Eliminates the need for an external contactor for most installations  

> ⚠️ **Safety notes**  
> - Use a dedicated circuit sized for your heat tape.  
> - Roof de-icing circuits often require **GFCI/GFPE protection**.  
> - Follow conductor-size and torque specifications.  
> - When in doubt, consult a licensed electrician.

---

## Required Home Assistant Entities

Create these in **Settings → Devices & Services → Helpers**:

### 1. `input_boolean.freeze_latch`  

Tracks whether the melt cycle is actively engaged.

### 2. `input_boolean.snow_in_last_12_days`  

Indicates whether snow has fallen recently.  
Set automatically by the snow timer automation, or manually from your dashboard.

### 3. `timer.snow_recent_12d_timer`  

Represents how long snow should be considered “recent.”

- **12 days = 288 hours**  
- **14 days = 336 hours**  

Because the timer helper only supports hours, enter one of those values for the duration (288 or 336).

---

## Weather Integration Requirements

These automations now use **Pirate Weather for both snow detection and temperature**. OpenWeatherMap is no longer required for the core logic.

### Pirate Weather

Install the **Pirate Weather** integration via HACS, then add it as an integration with your API key.

You should expose the following entities:

- `weather.pirateweather`  
- `sensor.pirateweather_temperature`  
- `sensor.pirateweather_precip_intensity`  
- `sensor.pirateweather_current_day_snow_accumulation`  

#### Snow detection signals

The automations consider it “actively snowing” when **any** of the following are true:

- `weather.pirateweather` state contains `"snow"` (e.g. `snowy`, `light_snow`, `heavy_snow`)  
- `sensor.pirateweather_precip_intensity` is above **0.02 in/hr**  
- `sensor.pirateweather_current_day_snow_accumulation` is **> 0** and `sensor.pirateweather_precip_intensity` is **> 0**

To avoid reacting to brief glitches or very short flurries, this condition must remain true for **at least 10 minutes** before the “recent snow” timer starts.

#### Temperature signal

The melt-band temperature checks use:

- `sensor.pirateweather_temperature` (°F)

The automations treat the **effective melt band** as:

- Between **25°F and 39°F**  

Below 25°F, heat tape is much less effective at creating meltwater; above 39°F, the roof is typically warm enough that natural melting reduces the need for tape.

### Recommended Pirate Weather update interval

Because snow detection is debounced with a `for:` condition, Pirate Weather must update more frequently than the debounce window.

Recommended configuration (in `configuration.yaml` or via integration options):

```yaml
pirateweather:
  api_key: YOUR_API_KEY
  update_interval: 600  # seconds (10 minutes)
```

With a 600-second update interval and a 10-minute debounce, the system effectively requires **two consecutive “snowy” updates** before treating it as real snowfall and starting the 12-day timer.

---

## 12-Day Snow Window Logic

A snow event is considered “active” when **any** of the following are true for **10 continuous minutes**:

- `weather.pirateweather` state contains `"snow"`  
- `sensor.pirateweather_precip_intensity` > **0.02 in/hr**  
- `sensor.pirateweather_current_day_snow_accumulation` > 0 **and** `sensor.pirateweather_precip_intensity` > 0  

When this condition is met:

- `input_boolean.snow_in_last_12_days` is turned **ON**
- `timer.snow_recent_12d_timer` is (re)started to the full 12-day duration

When the timer finishes, the flag is turned **OFF**.

### Manual override behavior (important)

If you manually toggle **`input_boolean.snow_in_last_12_days`** to **OFF** while it is currently snowing, the system treats this as a **manual override for the remainder of that storm**.

This is because the snow detection automation only triggers when the snow condition changes:

```
false → true and stays true for 10 minutes
```

During an ongoing storm, the combined snow expression remains **true** continuously, so there is no new **false → true** transition and the automation does **not** re-latch the flag.

What this means:

- Turning the helper **OFF** during an active storm **keeps it OFF** until the storm ends.  
- Once snow conditions drop so the expression becomes **false**, the **next** snow event that satisfies the 10‑minute condition will automatically re‑arm the flag and restart the 12‑day timer.

This prevents the system from “fighting” you if you intentionally override it during a storm, while still allowing full automation on future storms.

---

## Melt-Band Heat Tape Logic

The tape should run when:

- Snow has fallen recently (`snow_in_last_12_days` is ON)  
- Temperature is within the melt band (**25–39°F**)  
- Time is within the **daylight / off‑peak window (06:00–15:55)**  
- The **freeze latch** is ON  

### Why 25–39°F?

- Below ~25°F, heat tape is relatively inefficient at creating meltwater.  
- Above ~39°F, the roof and gutters tend to melt naturally without assistance.  
- Centering operation in this band focuses electricity on the temperatures where heat tape is most useful for preventing ice dams.

---

## Time-of-Use / Daylight Window

The automations use a **daylight + off‑peak** window of:

- **Start:** 06:00  
- **End:** 15:55 (5 minutes before TOU peak starts at 16:00)

At **15:55**, a daily reset automation:

- Turns **OFF** the heat tape switch  
- Turns **OFF** the `freeze_latch`  

This guarantees the tape is not accidentally left on into the expensive TOU peak period.

---

## Automations Layout

Each automation is stored as its own YAML file inside `automations/`:

```text
automations/
├── snow_start_timer.yaml
├── snow_clear_flag.yaml
├── snow_cancel_timer.yaml
├── freeze_daily_reset.yaml
├── freeze_latch_on.yaml
└── freeze_maintain_latched.yaml
```

### What each file does

| File | Purpose |
|------|---------|
| `snow_start_timer.yaml` | Starts the 12‑day snow window when Pirate Weather shows meaningful snow (state or intensity/accumulation) **for at least 10 minutes**, or when manually toggled ON |
| `snow_clear_flag.yaml` | Clears the `snow_in_last_12_days` flag when the timer expires |
| `snow_cancel_timer.yaml` | Cancels the timer if the snow flag is manually turned OFF |
| `freeze_daily_reset.yaml` | Ensures the tape is OFF before TOU peak pricing begins (3:55 PM) |
| `freeze_latch_on.yaml` | Turns the system ON during off‑peak daylight (06:00–15:55) when Pirate Weather temp is in melt band (25–39°F) **and** recent snow exists |
| `freeze_maintain_latched.yaml` | Keeps tape ON while latched and conditions remain valid; shuts off when temp leaves the melt zone or outside the 06:00–15:55 window |

---

## Installation

### Option A – Using Home Assistant UI  

For each file in `automations/`:

1. Open the file and copy its contents.  
2. In Home Assistant, go to **Settings → Automations & Scenes → Create Automation**.  
3. Click the 3‑dot menu → **Edit in YAML**.  
4. Paste the contents.  
5. Save & enable.

### Option B – YAML Includes  

If you manage automations as files, add this to `configuration.yaml`:

```yaml
automation: !include_dir_merge_list automations
```

Then drop the `*.yaml` files from this repo's `automations/` directory into your Home Assistant `automations/` folder.

---

## Verification Tips

To verify everything is working as expected:

1. **Confirm Pirate Weather update interval**  
   - In Developer Tools → States, watch `weather.pirateweather` “Last updated” timestamps.  
   - They should advance roughly every 10 minutes.

2. **Confirm snow debounce**  
   - When Pirate Weather shows snowy conditions and/or `precip_intensity > 0.02`, the `input_boolean.snow_in_last_12_days` should remain OFF for the first 10 minutes and then flip ON, and `timer.snow_recent_12d_timer` should start.

3. **Confirm manual override**  
   - Toggling `input_boolean.snow_in_last_12_days` ON should immediately start or restart the timer.  
   - Toggling it OFF should cancel the timer and, if done during a storm, prevent auto‑re‑arming until the next storm.

4. **Confirm melt-band behavior**  
   - When `sensor.pirateweather_temperature` is between 25°F and 39°F, within 06:00–15:55, and `snow_in_last_12_days` is ON, the heat tape relay should be ON.  
   - When temp stays above 39°F or below 25°F for an hour, the system should shut off and unlatch.

---

## License

This project is licensed under the **Apache License 2.0**.  
See the included `LICENSE` file.

---

## Disclaimer

Roof heating systems contain **high-amperage electrical loads** and may pose hazards if installed incorrectly.  
Always consult applicable electrical codes and a licensed electrician where needed.
