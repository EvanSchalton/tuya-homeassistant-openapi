> AS OF RIGHT NOW THIS IS ONLY USEFUL FOR `IBS-M2` POOL SENSOR. It's a hacky approach to limitations with the tuya apigw repo.

# Background

This is all just a workaround to a breaking API change: https://github.com/tuya/tuya-device-sharing-sdk/issues/11

I'd ideally like to merge this PR: https://github.com/home-assistant/core/pull/113214
into the base Tuya integration, but the API change means the non-standard data points aren't available in home assistant.

I forked an old version of the HA repo, using the legacy Tuya SDK.

# Setup:
1. This integration reverts the Tuya core integration to use the [tuya-iot-python-sdk](https://github.com/tuya/tuya-iot-python-sdk) which returns the `DP Instruction` set instead of just the `Standard Instruction` so follow it's standard [setup](https://github.com/tuya/tuya-iot-python-sdk)

>I've logged [an issue](https://github.com/tuya/tuya-device-sharing-sdk/issues/11) with the Tuya team, if they merge that I'll be able to merge my [PR](https://github.com/tuya/tuya-device-sharing-sdk/issues/11) into core

2. Toggle the `DP Instruction`

In the cloud portal, the edit pencil on the device card links to the `Change Control Instruction Mode`

![Change Control Instruction](screenshots/device-card.png)

Select `DP Instruction`, you should see the additional value populate.

![DP Instruction](screenshots/instruction-mode.png)


# Hardware
- Sensor: [IBS-M2](https://amzn.to/3wX0Ir3)
- Switch: [Dwenwils Pool Timer](https://amzn.to/3wX0Ir3)

All in I spent ~$150, the cheapest alternative I could find was ~$650

# Configuration
I have two of the Dwenwils Pool Timers, one controlling my pump and the other controlling my heater.

## Other Automations
> Add a variable to detect when the heater was last turned off, configure your pump to stay on (or turn back on) for 20 minutes after the heater was last one. Some heaters say they don't need the pump to remain on, but this will almost certainly increase the longevitiy of your heater

Use the generic_thermostat to control a pool heater

```yaml
climate:
  - platform: generic_thermostat
    name: Pool Heater
    heater: switch.pool_heater_socket_1
    target_sensor: sensor.ibs_m2_temperature_2
    min_temp: 75
    max_temp: 100
    ac_mode: false
    target_temp: 85
    cold_tolerance: 2.0
    hot_tolerance: 2.0
    min_cycle_duration:
      seconds: 600
    initial_hvac_mode: "off"
    away_temp: 80
    precision: 0.1
```

# Dashboard

## Current Pool Dashboard (for Inspiration)

![Mobile Dashboard](screenshots/mobile-dashboard.png)

### Glance Yaml:
```yaml
show_name: true
show_icon: true
show_state: true
type: glance
entities:
  - entity: sensor.ibs_m2_temperature_3
    name: Temperature
  - entity: sensor.ibs_m2_ch_2_battery
    name: Battery
state_color: true
```

### Pump Yaml

`It's disabled right now because the breaker is off for my pool house`
```yaml
type: custom:button-card
name: Pump
show_state: false
show_icon: true
show_name: true
entity: switch.pool_pump_socket_1
icon: mdi:water-pump
state:
  - operator: template
    value: |
      [[[ return entity.state === 'on'; ]]]
    styles:
      icon:
        - color: rgb(16, 30, 194)
  - operator: template
    value: |
      [[[ return entity.state === 'off'; ]]]
    icon: mdi:water-pump-off
    styles:
      icon:
        - color: rgb(90, 90, 90)
  - operator: template
    value: >
      [[[ return entity.state === 'unknown'; ]]]  # Set "unknown" as the default
      state
    icon: mdi:alert-circle-outline
    styles:
      icon:
        - color: rgb(255, 0, 0)
tap_action:
  action: toggle

```

### Heater

`It's disabled right now because the breaker is off for my pool house`

```yaml
type: custom:button-card
name: Heater
show_state: false
show_icon: true
show_name: true
entity: switch.pool_heater_socket_1
icon: mdi:heat-wave
state:
  - operator: template
    value: |
      [[[ return entity.state === 'on'; ]]]
    styles:
      icon:
        - color: rgb(194, 28, 16)
  - operator: template
    value: |
      [[[ return entity.state === 'off'; ]]]
    icon: mdi:heat-wave
    styles:
      icon:
        - color: rgb(90, 90, 90)
  - operator: template
    value: >
      [[[ return entity.state === 'unknown'; ]]]  # Set "unknown" as the default
      state
    icon: mdi:alert-circle-outline
    styles:
      icon:
        - color: rgb(255, 0, 0)
tap_action:
  action: toggle

```

# Automations
Turn on the pump anytime the heater is on:
```yaml
- id: 'xxxxxx'
  alias: Pump turns on with heater
  description: ''
  trigger:
  - platform: state
    entity_id:
    - switch.pool_heater_socket_1
    attribute: 'On'
  condition: []
  action:
  - type: turn_on
    device_id: xxxxxx
    entity_id: xxxxxx
    domain: switch
  mode: single
```

Run the pump for 20 minutes after the heater turns off
```yaml
alias: Turn off pump first
description: ""
trigger:
  - platform: device
    type: changed_states
    device_id: __pump_device_id__
    entity_id: __pump_entity_id__
    domain: switch
condition:
  - condition: device
    type: is_on
    device_id: __heater_device_id__
    entity_id: __heater_entity_id__
    domain: switch
action:
  - type: turn_on
    device_id: __pump_device_id__
    entity_id: __pump_entity_id__
    domain: switch
  - type: turn_off
    device_id: __heater_device_id__
    entity_id: __heater_entity_id__
    domain: switch
  - delay:
      hours: 0
      minutes: 20
      seconds: 0
      milliseconds: 0
  - if:
      - condition: device
        type: is_off
        device_id: __heater_device_id__
        entity_id: __heater_entity_id__
        domain: switch
    then:
      - type: turn_off
        device_id: __pump_device_id__
        entity_id: __pump_entity_id__
        domain: switch
mode: restart
```
> This should be improved upon: I could store pump/heater on/off timestamps to action if the pump should stay on after the heater turns off -- I haven't tested this w/ the new themostat controller so they may interact oddly, in which case I'll need to update.

# Other Ideas
1. I'll probably add weather integration that controls the pool temp based on the weather (no point in heating when it's raining or on a particularly cool night or overcast day where we wont want to swim)
2. I happen to have multiple sensors, instead of having one sit on the shelf I'm going to play with creating a sensor that averages the and is used as the `target_sensor` in the climate control.