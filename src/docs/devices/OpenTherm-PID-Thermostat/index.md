---
title: OpenThem PID Thermostat using DIYless shield
date-published: 2025-12-06
type: misc
standard: uk eu us
board: esp32
---

## Introduction
This project uses an esp32-s2 mini with a [DIYless OpenTherm master shield] (https://diyless.com/product/master-opentherm-shield) to communicate and control Opentherm equipped HVAC systems. I used a master shield but I believe one could use a gateway shield instead which can operate in the presence of a master, like an off the shelf thermostat to provide some resliance in case your HA is down.

## Assembly
The OpenTherm master shield is designed in such a way as to fit directly on an esp8266 D1 mini or an esp32-s2 mini. You just have to solder the headers on and push them together

## Configuration
Each manufacturer supports usually a subset of the OpenTherm protocol. Check to see what is available in the integration, what is documented by your vendor, and finally what works and what sensors you are interested in. The following is for an Intergas system boiler. Non working/unsupported sensors may return zero rather than nan/not available.

PID tuning was the hardest part. Autotune may get you somewhere close depending on the time of year and outside temperature. The goal here is to have the PID thermostat controlling the boiler with small changes to the desired flow temperature such that the boiler is not turning off and on all the time (cycling) but instead modulating gradually - This is best for efficiency, boiler longevity and comfort.

PID control works best with accurate and frequent temperature readings. In the config below we pull two temperature sensors from Home Assistant and take the mean of them. Those sensors are in two rooms and are dallas ds18b20 with two decimal places and updates every 10s of course running ESPHome. A zigbee temperature sensor is less than ideal for this due to lack of resolution and infrequent updates.

```yaml
esphome:
  name: opentherm
  platform: ESP32
  board: lolin_s2_mini
  on_boot:
    priority: -100
    then:
      - number.set:
          id: t_dhw_set
          value: 55
      - number.set:
          id: max_t_set
          value: 80

logger:
# "NONE", "ERROR", "WARN", "INFO", "CONFIG", "DEBUG", "VERBOSE", "VERY_VERBOSE"
  level: INFO
  logs:
    api: WARN
    binary_sensor: WARN
    climate: INFO
    homeassistant.sensor: WARN
    homeassistant: WARN
    logger: INFO
    mdns: WARN
    number: INFO
    opentherm: ERROR
    component: ERROR
    pid.autotune: INFO
    pid: INFO
    template: INFO
    text_sensor: INFO


api:
 
ota:
  platform: esphome
wifi:
  ap:
    ssid: "Thermostat"
captive_portal:

web_server:
  version: 3
  log: false

opentherm:
  in_pin: 33
  out_pin: 35
  sync_mode: true
  ch_enable: true
  dhw_enable: true
  cooling_enable: false
  otc_active: false
  ch2_active: false

output:
  - platform: opentherm
    t_set:
      id: boiler_setpoint    # setpoint range for boiler water temp in C
      min_value: 45
      max_value: 80
      zero_means_zero: true

number:
  - platform: template
    name: "Boiler Setpoint Setting"
    id: tset_setting         # Number input for boiler_setpoint for testing or other thermostat control in HA, boiler_setpoint is controlled by esphome PID thermostat normally
    min_value: 45
    max_value: 80
    mode: box
    step: 1
    lambda: "return id(boiler_setpoint).state;"
    set_action:
      - lambda: "id(boiler_setpoint).write_state(x);"

  - platform: opentherm
    t_dhw_set:
      id: t_dhw_set         # Domestic Hot Water Setpoint
      name: "Boiler DHW Setpoint"
    max_t_set:
      id: max_t_set
      name: "Boiler Max Setpoint"
#    otc_hc_ratio:
#      id: ot_hc_ratio       # Heat Curve when using outside temperature sensor and weather compensation
#      name: "OTC heat curve ratio:

  - platform: template
    name: "CH max t_set"
    id: ch_max_t_set         # Maximum boiler setpoint, this is limited by a setting in the boiler but can be set lower
    mode: box
    entity_category: config
    min_value: 0
    max_value: 100
    step: 0.1
    lambda: "return id(boiler_setpoint).get_max_value();"
    set_action:
      - lambda: "id(boiler_setpoint).set_max_value(x);"

  - platform: template
    name: "CH min t_set"     # Minimum boiler setpoint flow temperature, set this to a setting that the boiler can modulate to without cycling off and on execessively 45C works for average radiator setups. Underfloor heating could be lower.
    id: ch_min_t_set
    mode: box
    entity_category: config
    min_value: 0
    max_value: 100
    step: 0.1
    lambda: "return id(boiler_setpoint).get_min_value();"
    set_action:
      - lambda: "id(boiler_setpoint).set_min_value(x);"


  - platform: template
    name: "PID Proportional"
    id: thermostat_kp
    mode: box
    entity_category: config
    min_value: 0
    max_value: 100
    step: 0.00001
    lambda: "return id(central_heating_ot).get_kp();"
    set_action:
      - lambda: "id(central_heating_ot).set_kp(x);"

  - platform: template
    name: "PID Integral"
    id: thermostat_ki
    mode: box
    entity_category: config
    min_value: 0
    max_value: 100
    step: 0.00001
    lambda: "return id(central_heating_ot).get_ki();"
    set_action:
      - lambda: "id(central_heating_ot).set_ki(x);"

  - platform: template
    name: "PID Derivative"
    id: thermostat_kd
    mode: box
    entity_category: config
    min_value: 0
    max_value: 9999
    step: 0.001
    lambda: "return id(central_heating_ot).get_kd();"
    set_action:
      - lambda: "id(central_heating_ot).set_kd(x);"


sensor:
# Not all opentherm sensors provided by the opentherm component are implemented by each boiler manufacturer
# Non-working sensors seem to report zero rather than not available in both sensors and binary_sensors
  - platform: opentherm
    rel_mod_level:
      name: "Boiler Relative modulation level"
      filters:
        - throttle_average: 15s
    ch_pressure:
      name: "Boiler Water pressure in CH circuit"
      filters:
        - throttle_average: 60s
    t_boiler:
      name: "Boiler water temperature"
      filters:
        - throttle_average: 15s
    t_ret:
      name: "Boiler water return temperature"
    t_dhw:
      name: "Boiler DHW temperature"
      filters:
        - throttle_average: 15s
    t_dhw_set:
      name: "Boiler Domestic hot water temperature setpoint"
    dhw_flow_rate:
      name: "DHW Flow Rate"

  # The following gets temperature from two other esphome devices and applies kalman filer
  # These sensors preferably should have two decimals be smoothed with filters to get rid of noise and update frequently
  # This helps the PID algorithm to adjust itself appropriately
  - platform: homeassistant
    name: lounge_dallas
    id: lounge_dallas
    entity_id: sensor.lounge_dallas
    filters:
      heartbeat: 15s
  - platform: homeassistant
    name: kitchen_dallas
    id: kitchen_dallas
    entity_id: sensor.kitchen_dallas
    filters:
      heartbeat: 15s
  - platform: combination
    type: kalman
    id: "t_room"     
    name: "Ground Floor Temperature"
    unit_of_measurement: "°C"
    accuracy_decimals: 2
    process_std_dev: 0.001
    sources:
      - source: lounge_dallas
        error: 0.1
      - source: kitchen_dallas
        error: 0.1

  - platform: homeassistant
    name: Heating Setpoint
    id: t_room_set
    entity_id: climate.central_heating_ot
    attribute: temperature
    internal: false
    state_class: measurement
    unit_of_measurement: °C

  - platform: pid
    name: "PID Climate Result"
    type: RESULT
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Error"
    type: ERROR
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Proportional"
    type: PROPORTIONAL
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Integral"
    type: INTEGRAL
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Derivative"
    type: DERIVATIVE
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Heat"
    id: pid_climate_heat
    type: HEAT
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Kp"
    type: KP
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Ki"
    type: KI
    accuracy_decimals: 6
  - platform: pid
    name: "PID Climate Kd"
    type: KD
    accuracy_decimals: 6
  - platform: uptime
    name: Opentherm Uptime
    id: ot_uptime
    update_interval: 60s
    entity_category: "diagnostic"
  - platform: template
    name: "Boiler Setpoint"
    id: pid_setpoint
    unit_of_measurement: "°C"
    icon: "mdi:fire"
    device_class: "temperature"
    state_class: "measurement"
    accuracy_decimals: 2
    update_interval: 10s
    lambda: return id(boiler_setpoint).state;

binary_sensor:
# Not all opentherm sensors provided by the opentherm component are implemented by each boiler manufacturer
# Non-working sensors seem to report zero rather than not available in both sensors and binary_sensors
  - platform: opentherm
    fault_indication:
      name: "Boiler Fault indication"
    ch_active:
      name: "Boiler Central Heating active"
    dhw_active:
      name: "Boiler Domestic Hot Water active"
    flame_on:
      name: "Boiler Flame on"
    diagnostic_indication:
      name: "Boiler Diagnostic event"
#    dhw_present:
#      name: "Boiler DHW present"
#    control_type_on_off:
#      name: "Boiler Control type is on/off"
#    dhw_storage_tank:
#      name: "Boiler DHW storage tank"
#    master_pump_control_allowed:
#      name: "Boiler Master pump control allowed"
#    dhw_setpoint_transfer_enabled:
#      name: "Boiler DHW setpoint transfer enabled"
#    max_ch_setpoint_transfer_enabled:
#      name: "Boiler CH maximum setpoint transfer enabled"
#    dhw_setpoint_rw:
#      name: "Boiler DHW setpoint read/write"
#    max_ch_setpoint_rw:
#     name: "Boiler CH maximum setpoint read/write"
    service_request: 
      name: "Service Request"
    lockout_reset:
      name: "Lockout Reset"
    low_water_pressure:
      name: "Low Water Pressure"
    flame_fault:
      name: "Gas/Flame Fault"
    air_pressure_fault:
      name: "Air Pressure Fault"
    water_over_temp:
      name: "Water Over Temp"


switch:
  - platform: opentherm
    ch_enable:
      name: "Boiler Central Heating enabled"
      restore_mode: RESTORE_DEFAULT_ON
    dhw_enable:
      name: "Boiler Domestic Hot Water enabled"
      restore_mode: RESTORE_DEFAULT_ON
    cooling_enable:
      name: "Boiler Cooling enabled"
      restore_mode: RESTORE_DEFAULT_OFF
    otc_active:
      name: "Boiler Outside temperature compensation active"
      restore_mode: RESTORE_DEFAULT_OFF
    ch2_active:
      name: "Boiler Central Heating 2 active"
      restore_mode: RESTORE_DEFAULT_OFF
  - platform: restart
    name: "Opentherm Restart"

climate:
  - platform: pid
    id: central_heating_ot
    name: "Central heating OT"
    heat_output: boiler_setpoint
    default_target_temperature: 20.4
    sensor: t_room
    control_parameters: 
# Suggested default: kp: 0.7 ki: 0.003 kd: 0.0
# previous 23/1/24  kp: 0.6790622 ki: 0.0003784035 kd: 304.652
# autotune 23/1 kp: 0.949 ki: 0.00021 kd: 1052
# autotune 25/3 kp: 0.90407 ki: 0.00006 kd: 3196.35376
      kp: 2
      ki: 0.00003
      kd: 0.9
#      min_integral: 0
#      max_integral: 0.25
#      starting_integral_term: 0
      output_averaging_samples: 2
      derivative_averaging_samples: 8
    deadband_parameters:
      threshold_high: 0.5
      threshold_low: -0.3
      kp_multiplier: 0.05
      ki_multiplier: 0.05
      kd_multiplier: 0.0
      deadband_output_averaging_samples: 20
    visual:
      min_temperature: 10
      max_temperature: 23
      temperature_step: 0.1

button:
  - platform: template
    name: "PID Climate Autotune"
    on_press:
      - climate.pid.set_control_parameters:
          id: central_heating_ot
          kp: 0.0
          ki: 0.0
          kd: 0.0
      - climate.pid.autotune: central_heating_ot
  - platform: template
    name: "Reset Integral term"
    on_press:
      - climate.pid.reset_integral_term: central_heating_ot
```
