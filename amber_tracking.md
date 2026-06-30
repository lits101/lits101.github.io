# Amber Electric Demand Charge Tracking

Home Assistant configuration to track monthly peak demand during Amber Electric's demand window (16:00–21:00) and estimate the resulting monthly demand charge.

---

## Table of Contents

- [Overview](#overview)
- [Entities](#entities)
- [Sensor Configuration](#sensor-configuration)
- [Automation Configuration](#automation-configuration)
- [Calculation Logic](#calculation-logic)
- [Notes & Assumptions](#notes--assumptions)

---

## Overview

This setup:

1. Monitors house energy consumption via a 30-minute rolling statistics sensor.
2. Restricts demand tracking to the Amber Electric demand window: **16:00–21:00**.
3. Captures the highest 30-minute kWh value seen during the month.
4. Converts that peak kWh value to kW.
5. Calculates an estimated monthly demand charge.

---

## Entities

| Entity | Type | Purpose |
|---|---|---|
| `sensor.house_total_energy` | Pre-existing sensor | All-time house energy in kWh. |
| `sensor.house_energy_30_minutes_rolling` | Statistics sensor | 30-minute rolling 50th-percentile energy in kWh. |
| `sensor.house_energy_demand_window` | Template sensor | House energy restricted to the 16:00–21:00 demand window. |
| `sensor.current_demand` | Template sensor | Current rolling demand, non-zero only during the demand window. |
| `input_number.monthly_peak_demand_kwh` | Input number | Highest 30-minute kWh value observed this month. |
| `sensor.monthly_peak_demand_kw` | Template sensor | Converted monthly peak demand in kW. |
| `sensor.monthly_demand_charge` | Template sensor | Estimated monthly demand charge in dollars. |

> **Note:** `sensor.house_energy_demand_window` may also be named `sensor.garage_solaredge_i1_house_energy_demand_window` in your instance.

---

## Sensor Configuration

### `sensor.house_energy_30_minutes_rolling`

Statistics sensor configuration:

```yaml
- platform: statistics
  name: "House Energy 30 Minutes Rolling"
  entity_id: sensor.house_total_energy
  state_characteristic: percentile
  percentile: 50
  sampling_size: 1000
  max_age:
    minutes: 30
  precision: 2
```

### `sensor.house_energy_demand_window`

```yaml
- platform: template
  sensors:
    house_energy_demand_window:
      friendly_name: "House Energy Demand Window"
      unit_of_measurement: "kWh"
      value_template: >
        {{ states('sensor.house_total_energy') | float(0)
           if now().hour >= 16 and now().hour < 21
           else 0 }}
```

### `sensor.current_demand`

```yaml
- platform: template
  sensors:
    current_demand:
      friendly_name: "Current Demand"
      unit_of_measurement: "kWh"
      value_template: >
        {% if now().hour >= 16 and now().hour < 21 %}
          {{ states('sensor.house_energy_30_minutes_rolling') | float(0) }}
        {% else %}
          0
        {% endif %}
```

### `input_number.monthly_peak_demand_kwh`

```yaml
input_number:
  monthly_peak_demand_kwh:
    name: Monthly Peak Demand kWh
    min: 0
    max: 100
    step: 0.001
    mode: box
    unit_of_measurement: kWh
```

### `sensor.monthly_peak_demand_kw`

```yaml
- platform: template
  sensors:
    monthly_peak_demand_kw:
      friendly_name: "Monthly Peak Demand kW"
      unit_of_measurement: "kW"
      value_template: >
        {{ states('input_number.monthly_peak_demand_kwh') | float(0) * 2 }}
```

### `sensor.monthly_demand_charge`

```yaml
- platform: template
  sensors:
    monthly_demand_charge:
      friendly_name: "Monthly Demand Charge"
      unit_of_measurement: "$"
      value_template: >
        {{ (states('sensor.monthly_peak_demand_kw') | float(0) * 7) | round(2) }}
```

---

## Automation Configuration

### Update Demand Peak

Captures the highest 30-minute rolling energy value seen during the demand window.

```yaml
- alias: "Update Demand Peak"
  description: "Capture the highest 30-minute rolling demand during the Amber demand window."
  trigger:
    - platform: state
      entity_id: sensor.house_energy_30_minutes_rolling
  condition:
    - condition: time
      after: "16:00:00"
      before: "21:00:00"
  action:
    - if:
        - condition: template
          value_template: |
            {{ states('sensor.house_energy_30_minutes_rolling') | float(0) >
               states('input_number.monthly_peak_demand_kwh') | float(0) }}
      then:
        - service: input_number.set_value
          target:
            entity_id: input_number.monthly_peak_demand_kwh
          data:
            value: "{{ states('sensor.house_energy_30_minutes_rolling') | float(0) }}"
  mode: single
```

### Reset Usage & Peak Demand Counter

Resets the stored monthly peak demand at midnight on the 1st of each month.

```yaml
- alias: "Reset Usage & Peak Demand Counter"
  description: "Reset the monthly peak demand counter on the first day of each month."
  trigger:
    - platform: time
      at: "00:00:00"
  condition:
    - condition: template
      value_template: "{{ now().day == 1 }}"
  action:
    - service: input_number.set_value
      target:
        entity_id: input_number.monthly_peak_demand_kwh
      data:
        value: 0
  mode: single
```

---

## Calculation Logic

| Step | Formula | Example |
|---|---|---|
| 30-minute peak energy | Stored in `input_number.monthly_peak_demand_kwh` | `3.500 kWh` |
| Convert to kW | `kWh × 2` | `3.500 × 2 = 7.00 kW` |
| Demand charge | `kW × $7` | `7.00 × 7 = $49.00` |

---

## Notes & Assumptions

- The demand window is hardcoded as **16:00–21:00**. Update the time conditions in both the template sensors and the automation if your tariff window differs.
- The `$7` multiplier in `sensor.monthly_demand_charge` should match your Amber Electric plan's demand charge rate.
- `sensor.house_energy_demand_window` is defined but is **not used** by the peak-capture automation. It can be used for dashboards or debugging, or safely removed.
- `sensor.current_demand` is a convenience display sensor that zeroes outside the demand window.
- `sensor.house_total_energy` is assumed to be a pre-existing all-time house energy kWh sensor.
