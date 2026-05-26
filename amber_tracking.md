## Input Helpers

| Name | Entity ID | Settings |
|------|-----------|----------|
| Amber Last 5min Cost | `input_number.amber_last_5min_cost` | Min: -999, Max: 999, Step: 0.0001, Unit: $ |
| Amber Daily Cost Cumulative | `input_number.amber_daily_cost_cumulative` | Min: -999, Max: 9999, Step: 0.0001, Unit: $ |
| Grid Energy Previous | `input_number.grid_energy_previous` | Min: 0, Max: 999999, Step: 0.0001, Unit: kWh |
| Amber Last 5min Feed-in | `input_number.amber_last_5min_feed_in` | Min: -999, Max: 999, Step: 0.0001, Unit: $ |
| Amber Daily Feed-in Cumulative | `input_number.amber_daily_feed_in_cumulative` | Min: -999, Max: 9999, Step: 0.0001, Unit: $ |
| Solar Return Previous | `input_number.solar_return_previous` | Min: 0, Max: 999999, Step: 0.0001, Unit: kWh |

## Automations

### Main Cost Tracker (Import + Export)

```yaml
alias: Amber 5-Minute Cost Tracker
description: ""
triggers:
  - trigger: state
    entity_id:
      - sensor.home_general_price
conditions: []
actions:
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - sensor.home_general_price
        attribute: estimate
        to:
          - "false"
        from:
          - "true"
    timeout:
      hours: 0
      minutes: 0
      seconds: 30
      milliseconds: 0
    continue_on_timeout: true

  # Calculate import cost
  - target:
      entity_id: input_number.amber_last_5min_cost
    data:
      value: >
        {% set current = states('sensor.house_total_energy') | float(0) %}
        {% set previous = states('input_number.grid_energy_previous') | float(0) %}
        {% set energy_kwh = [current - previous, 0] | max %}
        {% set price = states('sensor.home_general_price') | float(0) %}
        {{ (energy_kwh * price) | round(4) }}
    action: input_number.set_value

  # Calculate export feed-in
  - target:
      entity_id: input_number.amber_last_5min_feed_in
    data:
      value: >
        {% set current = states('sensor.solar_return_kwh') | float(0) %}
        {% set previous = states('input_number.solar_return_previous') | float(0) %}
        {% set energy_kwh = [current - previous, 0] | max %}
        {% set price = states('sensor.home_feed_in_price') | float(0) %}
        {{ (energy_kwh * price) | round(4) }}
    action: input_number.set_value

  # Accumulate import cost
  - target:
      entity_id: input_number.amber_daily_cost_cumulative
    data:
      value: >
        {% set last = states('input_number.amber_last_5min_cost') | float(0) %}
        {% set total = states('input_number.amber_daily_cost_cumulative') | float(0) %}
        {{ (total + last) | round(4) }}
    action: input_number.set_value

  # Accumulate export feed-in
  - target:
      entity_id: input_number.amber_daily_feed_in_cumulative
    data:
      value: >
        {% set last = states('input_number.amber_last_5min_feed_in') | float(0) %}
        {% set total = states('input_number.amber_daily_feed_in_cumulative') | float(0) %}
        {{ (total + last) | round(4) }}
    action: input_number.set_value

  # Store previous energy values for next cycle
  - target:
      entity_id: input_number.grid_energy_previous
    data:
      value: "{{ states('sensor.house_total_energy') | float(0) }}"
    action: input_number.set_value
  - target:
      entity_id: input_number.solar_return_previous
    data:
      value: "{{ states('sensor.solar_return_kwh') | float(0) }}"
    action: input_number.set_value
mode: single
```

### Daily Reset

```yaml
alias: Amber Daily Cost Reset
description: "Reset cumulative values at midnight and seed previous energy values"
triggers:
  - trigger: time
    at: "00:00:00"
conditions: []
actions:
  - target:
      entity_id: input_number.amber_daily_cost_cumulative
    data:
      value: 0
    action: input_number.set_value
  - target:
      entity_id: input_number.amber_daily_feed_in_cumulative
    data:
      value: 0
    action: input_number.set_value
  - target:
      entity_id: input_number.grid_energy_previous
    data:
      value: "{{ states('sensor.house_total_energy') | float(0) }}"
    action: input_number.set_value
  - target:
      entity_id: input_number.solar_return_previous
    data:
      value: "{{ states('sensor.solar_return_kwh') | float(0) }}"
    action: input_number.set_value
mode: single
```

## Optional: Net Daily Cost Template Sensor

```yaml
template:
  - sensor:
      - name: "Amber Net Daily Cost"
        unique_id: amber_net_daily_cost
        state: >
          {% set import = states('input_number.amber_daily_cost_cumulative') | float(0) %}
          {% set export = states('input_number.amber_daily_feed_in_cumulative') | float(0) %}
          {{ (import - export) | round(4) }}
        unit_of_measurement: "$"
        device_class: monetary
        state_class: total
```

## Dashboard Entities Summary

| Purpose | Entity |
|---------|--------|
| Import cost this interval | `input_number.amber_last_5min_cost` |
| Import cost today | `input_number.amber_daily_cost_cumulative` |
| Export credit this interval | `input_number.amber_last_5min_feed_in` |
| Export credit today | `input_number.amber_daily_feed_in_cumulative` |
| Net amount owed | `sensor.amber_net_daily_cost` (template) |
