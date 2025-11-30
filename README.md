# Home Assistant – Smart Roof Heat Tape Automation

This repository contains a set of Home Assistant automations designed to intelligently control **roof de-icing heat tape** using:

- Melt-band temperature logic  
- Time-of-use (TOU) off-peak scheduling  
- A “recent snowfall” window using a timer  
- A high-power Z-Wave relay (Zooz ZEN78 800LR)  
- **Pirate Weather** for accurate, real-time snow detection  

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
- Use a dedicated circuit sized for your heat tape.  
- Roof de-icing circuits often require **GFCI/GFPE protection**.  
- Follow conductor-size and torque specifications.  
- When in doubt, consult a licensed electrician.

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
Enter this as hours in the GUI.

## Weather Integration Requirements

You’ll use two weather sources:

### OpenWeatherMap (temperature only)

Install the **OpenWeatherMap** integration and ensure you have:

- `sensor.openweathermap_temperature` (°F)  

This is used for melt-band temperature checks (25–39°F).

### Pirate Weather (real-time snow detection)

Install the **Pirate Weather** integration via HACS and add it as an integration with your API key.

You must have:

- `weather.pirateweather`

Pirate Weather’s **state** is used for snow detection.  
Examples:

- `snowy`
- `light_snow`
- `heavy_snow`
- `rainy`
- `cloudy`
- `clear`

The automation considers it “snowing now” when:

```jinja2
states('weather.pirateweather') | lower | contains('snow')
```

### ⚙ Recommended Pirate Weather update interval

The snow detection automation uses a time-based debounce (`for:`).  
For this to work correctly, Pirate Weather must update **more frequently** than the debounce window.

Recommended configuration (in `configuration.yaml` or the integration options):

```yaml
pirateweather:
  api_key: YOUR_API_KEY
  update_interval: 600  # seconds (10 minutes)
```

With a 600‑second update interval and a 10‑minute debounce (see below), the system effectively requires **two consecutive "snowy" updates** before treating it as real snowfall.

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
| **snow_start_timer.yaml** | Starts the 12-day snow window when Pirate Weather reports snow (state contains “snow”) **for at least 10 minutes**, or when manually toggled ON |
| **snow_clear_flag.yaml** | Clears the “snow in last 12 days” flag when the timer expires |
| **snow_cancel_timer.yaml** | Cancels the timer if the snow flag is manually turned OFF |
| **freeze_daily_reset.yaml** | Ensures the tape is OFF before TOU peak pricing begins (3:55 PM) |
| **freeze_latch_on.yaml** | Turns the system ON during off-peak daylight (06:00–15:55) when temp is in melt band (25–39°F) & recent snow exists |
| **freeze_maintain_latched.yaml** | Keeps tape ON while latched and conditions remain valid; shuts off when out of melt zone or outside 06:00–15:55 window |

## Configurable Values

You may customize these depending on climate and snow behavior:

| Setting | Default | Notes |
|--------|---------|-------|
| Melt band (°F) | **25–39°F** | Below 25°F tape is ineffective; above 39°F roof is warm enough to melt naturally |
| Daylight / off-peak window | **06:00 → 15:55** | Starts earlier to pre-warm north-facing roofs while staying in off-peak |
| Snow window duration | **12 days** (288 hr) | Suitable for north-facing roofs that hold snow for weeks |
| Snow detection source | **Pirate Weather state contains “snow” for ≥ 10 minutes** | Debounced; requires at least two snowy updates at 10‑min interval |

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
