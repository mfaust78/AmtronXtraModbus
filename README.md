# Use Home assistant to get data from your Amtron Xtra / Premium
Original challenge was to get energy data from Amtron Xtra to Home Assistant.
First I tried to get the registers directly with Modbus from Home Assistant, but that somewhat overloaded the Amtron. Therefore I changed it to struct and using templates.
This works fine with my Amtron Xtra and the polling setting in the code.

## Requirements
- Mennekes Amtron Xtra / Premium Wall Box
- IP Address of Amtron Xtra / Premium
- Software Version 1.13 on Amtron Xtra / Premium
- Home Assistant 2022 or newer

## configuration.yaml changes
```YAML
# I don't want to record all registers every time they are read, let's exclude them
recorder:
  exclude:
    entities:
    - sensor.amtron_registers

# I don't want a logbook entry either, let's exclude them
logbook:
  exclude:
    entities:
      - sensor.amtron_registers

# Modbus config and sensor to read  the modbus registers from the Amtron Xtra/Premium in one go
modbus:
  - name: Amtron
    type: tcp
    host: <PutYourAmtronIPhere>
    port: 502
    retry_on_empty: true
    retries: 1
    timeout: 10
    delay: 1
    sensors:
      - name: "Amtron Registers"
        address: 0x0300
        slave: 255
        scan_interval: 120
        lazy_error_count: 1
        input_type: input
        count: 38
        data_type: custom
        structure: ">2h15H22B10H"

# Convert the structure to useful, separated values using templates
template:
  - sensor:
      - name: "Amtron HMI Temp Internal"
        unique_id: amtron_hmi_temp_int
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[0] }}"
        device_class: temperature
        unit_of_measurement: "°C"
        state_class: measurement
  - sensor:
      - name: "Amtron HMI Temp External"
        unique_id: amtron_hmi_temp_ext
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[1] }}"
        device_class: temperature
        unit_of_measurement: "°C"
        state_class: measurement
  - sensor:
      - name: "Amtron CP State"
        unique_id: amtron_cp_state
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: >-
          {% set mapper =  {
              '0' : 'illegal/bad',
              '1' : 'A1 - Not connected',
              '2' : 'A2 - Not connected',
              '3' : 'B1 - Connected',
              '4' : 'B2 - Connected',
              '5' : 'C1 - Charging',
              '6' : 'C2 - Charging',
              '7' : 'D1 - Charging with Ventilation',
              '8' : 'D2 - Charging with Ventilation' } %}
          {% set state =  states('sensor.amtron_registers').split(',')[2] %}
          {{ mapper[state] if state in mapper else 'Unknown' }}
  - sensor:
      - name: "Amtron PP State"
        unique_id: amtron_pp_state
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: >-
          {% set mapper =  {
              '0' : 'illegal/bad',
              '1' : 'Open',
              '2' : '13A',
              '3' : '20A',
              '4' : '32A' } %}
          {% set state =  states('sensor.amtron_registers').split(',')[3] %}
          {{ mapper[state] if state in mapper else 'Unknown' }}
  - sensor:
      - name: "Amtron State"
        unique_id: amtron_state
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: >-
          {% set mapper =  {
              '0' : 'Idle',
              '1' : 'Standby Authorize',
              '2' : 'Standby Connect',
              '3' : 'Charging',
              '4' : 'Paused',
              '5' : 'Terminated',
              '6' : 'Error' } %}
          {% set state =  states('sensor.amtron_registers').split(',')[5] %}
          {{ mapper[state] if state in mapper else 'Unknown' }}
  - sensor:
      - name: "Amtron Phases"
        unique_id: amtron_phases
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: >-
          {% set mapper =  {
              '0' : 'Unknown',
              '1' : '1 Phase',
              '3' : '3 Phases' } %}
          {% set state =  states('sensor.amtron_registers').split(',')[8] %}
          {{ mapper[state] if state in mapper else 'Unknown' }}
  - sensor:
      - name: "Amtron Rated Current"
        unique_id: amtron_rated_current
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[9] }}"
        device_class: current
        unit_of_measurement: "A"
        state_class: measurement
  - sensor:
      - name: "Amtron Installation Current"
        unique_id: amtron_installation_current
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[10] }}"
        device_class: current
        unit_of_measurement: "A"
        state_class: measurement
  - sensor:
      - name: "Amtron Serial Number"
        unique_id: amtron_serial_number
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: >-
          {% set sn = (states('sensor.amtron_registers').split(',')[11]|int + states('sensor.amtron_registers').split(',')[12]|int * 65536) | string %}
          135{{ sn[:4] }}.{{ sn[4:] }}
  - sensor:
      - name: "Amtron Energy"
        unique_id: amtron_energy
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[13]|int + states('sensor.amtron_registers').split(',')[14]|int * 65536 }}"
        device_class: energy
        unit_of_measurement: "Wh"
        state_class: total_increasing
  - sensor:
      - name: "Amtron Power"
        unique_id: amtron_power
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[15]|int + states('sensor.amtron_registers').split(',')[16]|int * 65536 }}"
        device_class: power
        unit_of_measurement: "W"
        state_class: measurement
  - sensor:
      - name: "Amtron Wallbox Name"
        unique_id: amtron_wallbox_name
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: >-
          {% set ns = namespace(name = '') -%}
          {% set input = states('sensor.amtron_registers').split(',')[17:40] -%}
          {% for i in range(0,11) -%}
            {% set ns.name = ns.name ~ "%c"%input[i*2+1]|int ~ "%c"%input[i*2]|int -%}
          {% endfor %}
          {{ ns.name.replace('\x00','') }}
  - sensor:
      - name: "Amtron Max Current T1"
        unique_id: amtron_max_current_t1
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[40] }}"
        device_class: current
        unit_of_measurement: "A"
        state_class: measurement
  - sensor:
      - name: "Amtron Max Current T2"
        unique_id: amtron_max_current_t2
        availability: "{{ has_value('sensor.amtron_registers') }}"
        state: "{{ states('sensor.amtron_registers').split(',')[44] }}"
        device_class: current
        unit_of_measurement: "A"
        state_class: measurement
```

## References
- [Mennekes Software Download](https://www.mennekes.de/emobility/services/software-updates/)
- [Mennekes Modbus Description](Description Modbus_AMTRON HCC3_v01_2021-06-25_en.pdf)
