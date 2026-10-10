# Car-Heater 🚗❄️
 
A Home Assistant package that controls engine block heaters (*motorvärmare*) for **two cars**. Set a departure time with sliders, and the heater starts automatically at the right moment, with the lead time adjusted to the outdoor temperature.
 
The cars in this example are named **Kia** and **Subaru**, but you can rename them to anything.
 
## Features
 
- Two independent heaters, one per car
- Departure time set with hour and minute sliders on the dashboard
- Temperature-based lead time: the colder it is, the earlier the heater starts
- Per-car **Activate** switch to enable or disable the schedule
- Heater is switched off automatically at departure time
- Manual control through the heater's own switch
- Works with any outdoor temperature sensor (Z-Wave, Yr/Met.no, SMHI and so on)
- Everything lives in a single package file
## How it works
 
Every 5 minutes, for each car whose **Activate** switch is on, the automation:
 
1. Reads the outdoor temperature.
2. Picks a lead time from the table below.
3. Calculates `now + lead time` and compares it with the departure time.
4. If they match, the heater switch is turned **on**.
5. When the current time equals the departure time, the heater switch is turned **off**.

| Outdoor temperature | Heater starts before departure |
| ------------------- | ------------------------------ |
| below -20 °C        | 180 min (3 h)                  |
| -20 °C to -10 °C    | 120 min (2 h)                  |
| -10 °C to 1 °C      | 60 min (1 h)                   |
| 1 °C or warmer      | 30 min                         |
 
If the temperature sensor is unavailable, the automation falls back to 99 °C, which gives the shortest lead time (30 min).
 
> The minute sliders use steps of 5, which matches the 5-minute automation trigger. If you change the step, change the trigger as well, otherwise the start time may never match exactly.
 
## Requirements
 
- [Home Assistant](https://www.home-assistant.io/) with [packages](https://www.home-assistant.io/docs/configuration/packages/) enabled
- A smart plug or switch per heater, with power measurement if you want to see consumption (for example Z-Wave)
- An outdoor temperature sensor
## Entities
 
The package expects, and creates, the following entities. Entities you must provide yourself are marked in bold.
 
| Entity | Type | Purpose |
| ------ | ---- | ------- |
| **`sensor.garage_temperatur_utomhus`** | sensor (you provide) | Outdoor temperature |
| **`switch.kia_motorvarmare`** | switch (you provide) | Kia heater plug |
| **`switch.subaru_motorvarmare`** | switch (you provide) | Subaru heater plug |
| **`sensor.kia_motorvarmare_electric_consumption_w`** | sensor (optional) | Kia power draw, shown in the group |
| **`sensor.subaru_motorvarmare_electric_consumption_w`** | sensor (optional) | Subaru power draw, shown in the group |
| `input_number.heater_kia_hour` / `_minutes` | input_number | Kia departure time |
| `input_number.heater_subaru_hour` / `_minutes` | input_number | Subaru departure time |
| `input_boolean.heaterstatus_kia` / `_subaru` | input_boolean | Enable or disable the schedule |
| `sensor.departuretime_kia` / `_subaru` | template sensor | Departure time as `HH:MM` |
| `group.heater` | group | Groups everything for a dashboard |
 
## Installation
 
1. Make sure packages are enabled in `configuration.yaml`:
```yaml
   homeassistant:
     packages: !include_dir_named packages
```
 
2. Copy [`carheater.yaml`](carheater.yaml) to `config/packages/carheater.yaml`.
3. Replace the entity IDs of your own devices (see the table above):
   - `sensor.garage_temperatur_utomhus`
   - `switch.kia_motorvarmare`
   - `switch.subaru_motorvarmare`
   - the two `..._electric_consumption_w` sensors
4. Check the configuration under **Developer Tools → YAML → Check configuration**.
5. Restart Home Assistant.
6. Add the entities to a dashboard (see below).
## Dashboard
 
The simplest option is to show the group `group.heater`, which contains everything for both cars. For more control, add an entities card:
 
```yaml
type: entities
title: Motorvärmare
entities:
  - entity: input_boolean.heaterstatus_kia
  - entity: sensor.departuretime_kia
  - entity: input_number.heater_kia_hour
  - entity: input_number.heater_kia_minutes
  - entity: switch.kia_motorvarmare
  - type: divider
  - entity: input_boolean.heaterstatus_subaru
  - entity: sensor.departuretime_subaru
  - entity: input_number.heater_subaru_hour
  - entity: input_number.heater_subaru_minutes
  - entity: switch.subaru_motorvarmare
```
 
## Customization
 
**Change the lead times.** Edit the `lead_minutes` template in both automations:
 
```yaml
lead_minutes: >
  {% if outdoor_temp < -20 %} 180
  {% elif outdoor_temp < -10 %} 120
  {% elif outdoor_temp < 1 %} 60
  {% else %} 30
  {% endif %}
```
 
**Use another temperature source.** Point `outdoor_temp` at any numeric temperature sensor, for example one from the Met.no, SMHI or Yr integration:
 
```yaml
outdoor_temp: "{{ states('sensor.your_outdoor_temperature') | float(99) }}"
```
 
**Change the default departure time.** Edit `initial` on the `input_number` helpers (Kia 06:30, Subaru 06:20 by default).
 
**Add a third car.** Copy one of the car sections (automation, input numbers, input boolean, template sensor and group) and rename the entities.
 
## Troubleshooting
 
- **The heater never starts.** Check that the car's *Activate* switch is on and that the minutes on the slider are a multiple of 5.
- **Always 30 minutes lead time.** The temperature sensor is probably unavailable or has a different entity ID, so the fallback value is used.
- **Configuration check fails.** Check that every entity ID you replaced exists, and that indentation in the YAML is intact.
- **Wrong time.** The automation uses Home Assistant's local time zone. Check it under **Settings → System → General**.
## Known limitations
 
- The heater is started only if `now + lead time` matches the departure time exactly at one of the 5-minute runs. If the temperature crosses a threshold between two runs (for example from -9.9 °C to -10.1 °C, which changes the lead time from 60 to 120 min), the new start point may already have passed and the heater is not started. The same happens if Home Assistant is down or restarting at that moment.
- The heater is switched off only if the automation runs at exactly the departure minute. If Home Assistant is restarting at that moment, the heater stays on until you turn it off manually.
- The schedule runs every day while *Activate* is on. There is no weekday filter.
- The `initial: on` setting on the input booleans resets them to on after every restart.

## Credits
 
Thoughts and ideas from [Gnaget2/Car-Heater-Package](https://github.com/Gnaget2/Car-Heater-Package).
