# Home Assistant – Smart Roof Heat Tape Automation

This repository contains Home Assistant automations for controlling a 120 V self-regulating roof
heat-tape circuit using intelligent melt-band temperature logic, TOU-aware daytime operation, and a
“recent snowfall” timer window.

## Hardware Used

### **Zooz ZEN78 800LR High-Power Relay (40A)**
Product page:  
https://www.thesmartesthouse.com/products/zooz-z-wave-long-range-high-power-relay-zen78-800lr

This automation is built around the **40 A** ZEN78 high-power relay. It is appropriate for roof
de-icing loads because:

- 40 A resistive rating (supports heavy heat-tape circuits)
- Z-Wave 800LR Long-Range radio for excellent coverage at a roof/attic location
- Built-in power monitoring (useful for tracking cost)
- Handles 120–277 V resistive loads without an external contactor

**Electrical Safety Notes**
- Use a dedicated circuit sized for your heat tape.
- Roof de-icing circuits often require **GFCI/GFPE protection**.
- Follow conductor-size and torque specifications.
- When in doubt, consult a licensed electrician.

## Required Home Assistant Entities

**Helpers (Settings → Devices & Services → Helpers)**  
You must create:

1. `input_boolean.freeze_latch`
2. `input_boolean.snow_in_last_12_days`
3. `timer.snow_recent_12d_timer`  
   - Duration: 12 days = 288 hr (or 14 days = 336 hr)

**Weather Integration**  
Install **OpenWeatherMap**:
- `sensor.openweathermap_temperature`
- `sensor.openweathermap_snow` (in/hr)

A threshold of **0.02 in/hr** is used to detect real snowfall and avoid noise.

## Automations Layout

This repo stores each automation as its own YAML file under `automations/`:

- `snow_start_timer.yaml`
- `snow_clear_flag.yaml`
- `snow_cancel_timer.yaml`
- `freeze_daily_reset.yaml`
- `freeze_latch_on.yaml`
- `freeze_maintain_latched.yaml`

You can import them one by one via the UI, or include the whole directory via YAML.

## Installation

### Option A – UI Automation Editor
For each file in `automations/`:
1. Open the file and copy its contents.
2. In Home Assistant, go to **Settings → Automations & Scenes → Create Automation**.
3. Switch to **Edit in YAML**, paste, save, and enable.

### Option B – YAML Includes
If you manage automations as files, in `configuration.yaml` use:
```yaml
automation: !include_dir_merge_list automations
```
and drop all `*.yaml` files from this repo's `automations/` directory into your HA `automations/` directory.

## License
Apache 2.0 — see `LICENSE`.

