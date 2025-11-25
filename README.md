# Home Assistant – Smart Roof Heat Tape Automation

This repository contains a set of Home Assistant automations designed to intelligently control **roof de-icing heat tape** using:

- Melt-band temperature logic  
- Time-of-use (TOU) off-peak scheduling  
- A “recent snowfall” window using a timer  
- A high-power Z-Wave relay (Zooz ZEN78 800LR)  

These automations help prevent ice dams while significantly reducing electricity cost by running the tape **only when conditions justify it**.

## Hardware Used

### **Zooz ZEN78 800LR High-Power Relay (40A)**
Product page:  
https://www.thesmartesthouse.com/products/zooz-z-wave-long-range-high-power-relay-zen78-800lr

This relay is ideal for roof heat tape control because:

- **40 A resistive load rating** (supports large self-regulating heat-tape circuits)  
- Compatible with **120–277V systems**  
- Uses **Z-Wave 800 Long Range** for excellent connectivity  
- Provides **energy monitoring**, helpful for tracking cost  
- Eliminates the need for an external contactor for most installations  

### ⚠️ Safety Notes  
- Roof de-icing circuits commonly require **GFCI or GFPE** protection.  
- Use a **dedicated circuit** sized for your heat tape load.  
- Follow NEC / local code requirements for conductor gauge & torque specs.  
- If unsure, consult a **licensed electrician**.

## Required Home Assistant Entities

Create these in **Settings → Devices & Services → Helpers**:

### 1. `input_boolean.freeze_latch`  
Tracks whether the melt cycle is actively engaged.

### 2. `input_boolean.snow_in_last_12_days`  
Indicates whether snow has fallen recently.  
Set automatically or manually from your dashboard.

### 3. `timer.snow_recent_12d_timer`  
Represents how long snow should be considered “recent.”

- **12 days = 288 hours**  
- **14 days = 336 hours**  
Enter this as hours in the GUI.

## Weather Integration Requirements

Install the **OpenWeatherMap** integration (API key required).  
You must have access to these entities:

- `sensor.openweathermap_temperature` (°F)  
- `sensor.openweathermap_snow` (in/hr)  

A threshold of **0.02 in/hr** is used to avoid false snow triggers.

## Automations Layout

Each automation is stored as its own YAML file inside `automations/`:

```
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
| **snow_start_timer.yaml** | Starts the 12-day snow window when snow intensity exceeds threshold or manually toggled ON |
| **snow_clear_flag.yaml** | Clears the “snow in last 12 days” flag when the timer expires |
| **snow_cancel_timer.yaml** | Cancels the timer if snow flag is manually turned OFF |
| **freeze_daily_reset.yaml** | Ensures the tape is OFF before TOU peak pricing begins (3:55 PM) |
| **freeze_latch_on.yaml** | Turns the system ON during daylight only when temp is in melt band (25–39°F) & recent snow exists |
| **freeze_maintain_latched.yaml** | Keeps tape ON while latched and conditions remain valid; shuts off when out of melt zone |

## Configurable Values

| Setting | Default | Notes |
|--------|---------|-------|
| Melt band (°F) | **25–39°F** | Below 25°F tape is ineffective; above 39°F roof is warm enough to melt naturally |
| Daylight window | **08:00 → 15:55** | Optimizes melting during sun exposure and avoids peak pricing |
| Snow window duration | **12 days** (288 hr) | Suitable for north-facing roofs that hold snow for weeks |
| Snow detection threshold | **0.02 in/hr** | Prevents false positives from sensor noise |

## Installation

### Option A — Using Home Assistant UI  
For each file in `automations/`:

1. Open the file and copy its contents.  
2. In Home Assistant, go to **Settings → Automations & Scenes → Create Automation**.  
3. Click the 3-dot menu → **Edit in YAML**.  
4. Paste the contents.  
5. Save & enable.

### Option B — YAML Includes  
If you manage automations as files, add this to `configuration.yaml`:

```yaml
automation: !include_dir_merge_list automations
```

Then drop the `*.yaml` files from this repo's `automations/` directory into your Home Assistant `automations/` folder.

## License

This project is licensed under the **Apache License 2.0**.  
See the included `LICENSE` file.

## Disclaimer

Roof heating systems contain **high-amperage electrical loads** and may pose hazards if installed incorrectly.  
Always consult applicable electrical codes and a licensed electrician where needed.
