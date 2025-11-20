---
layout: base
title: Setup meshtastic for use in Eye-Set ecosystem
permalink: /meshtastic
---
# Setup meshtastic for use in eye-set ecosystem

## About Meshtastic

Meshtastic is a mesh network for the lora devices. It allows you to communicated over the radio with free bandwidth and without internet connection. It works over long distances, too. Catch is that it is extremely low bandwidth. Even sending small file will take significant amount of time.

## Setting up "gateway" node

This is the node that has both Lora and Wifi connectivity so this will be used to send data from your lora device to mqtt so it can be accessed over internet.  

First you need to upload meshstastic firmware. You can do that from https://flasher.meshtastic.org/ and follow the instructions. Once done, you need set radio. 

We will be using meshtastic cli:

```
meshtastic --nodes
meshtastic --set lora.region EU_868
meshtastic --set-owner 'Photogateway' --set-owner-short  'GAT'
meshtastic --set lora.modemPreset MEDIUM_FAST
meshtastic --set network.wifi_enabled true --set network.wifi_ssid "my network" --set network.wifi_psk mypassword

#get admin public key to be able to configure devices remotely
meshtastic --get security.public_key
#configure private channel generate other random psk
meshtastic --ch-set psk base64:puavdd7vtYJh8NUVWgxbsoG2u9Sdqc54YvMLs+KNcMA= --ch-index 0

#configure mqtt settings
meshtastic --ch-set uplink_enabled true --ch-index 0
meshtastic --ch-set downlink_enabled true --ch-index 0
meshtastic --set mqtt.enabled true --set mqtt.json_enabled true --set mqtt.encryption_enabled false --set mqtt.address 192.168.1.35:1883 --set mqtt.username lora  --set mqtt.password password


#Set fixed position of gateway node
meshtastic --set position.fixed_position true --setlat 37.8651 --setlon -119.5383
meshtastic --ch-set module_settings.position_precision 32 --ch-index 0
```


## Configuring mqtt and homeassistant

Configure mosquitton on hassio: https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md

Checking that everything is working and you see json messages from the nodes:
```
mosquitto_sub -h 192.168.1.10 -p 1883 -t "#" -v -u mqttuser -P mqttpass
```

What you should see is json messages from the nodes (please note that both binary and json is send in the correct configuration above). 
```
msh/EU_868/2/e/MediumFast/!ea8f8630 
??Geo??:m??%xÁ=(??54??
=??GhHX
x?0
MediumFast	!ea8f8630
msh/EU_868/2/json/MediumFast/!ea8f8630 {"channel":0,"from":3935274544,"hop_start":3,"hops_away":0,"id":178295092,"payload":{"air_util_tx":0.0633611083030701,"battery_level":101,"channel_utilization":1.50333333015442,"uptime_seconds":43266,"voltage":-0.00100000004749745},"sender":"!ea8f8630","timestamp":1749534388,"to":4294967295,"type":"telemetry"}
msh/EU_868/2/e/MediumFast/!ea8f8630 
??b?????"
     %??Gh(x?? 56?&?=??GhHX
x?0
MediumFast	!ea8f8630
msh/EU_868/2/json/MediumFast/!ea8f8630 {"channel":0,"from":3935274544,"hop_start":3,"hops_away":0,"id":2334574902,"payload":{"latitude_i":518495880,"longitude_i":195866210,"precision_bits":32,"time":1749534914},"sender":"!ea8f8630","timestamp":1749534914,"to":4294967295,"type":"position"}
```

More details: https://meshtastic.org/docs/software/integrations/mqtt/

Create automation to update location that we can use to display on map:
```
alias: Update Node 1 location
description: Update Meshtastic node when corresponding MQTT messages are seen.
trigger:
  - platform: mqtt
    topic: msh/US/2/json/LongFast/!67ea9400
    payload: "on"
    value_template: |-
      {% if value_json.from == 4038675309 and
            value_json.payload.latitude_i is defined and 
            value_json.payload.longitude_i is defined %}on{% endif %}
condition: []
action:
  - service: device_tracker.see
    metadata: {}
    data:
      dev_id: node_1
      gps:
        - "{{ (trigger.payload | from_json).payload.latitude_i | int * 1e-7 }}"
        - "{{ (trigger.payload | from_json).payload.longitude_i | int * 1e-7 }}"
      battery: "{{ states('sensor.node_1_battery_percent')|float(0) }}"
mode: single
```

Configure node basic data:
```
sensor:

# Node #1 

  - name: "Node 1 Battery Voltage"
    unique_id: "node_1_battery_voltage"
    state_topic: "msh/US/2/json/LongFast/!67ea9400"
    state_class: measurement
    value_template: >-
      {% if value_json.from == 4038675309 and
            value_json.payload.voltage is defined and
            value_json.payload.temperature is not defined %}
        {{ (value_json.payload.voltage | float) | round(2) }}
      {% else %}
        {{ this.state }}
      {% endif %}
    unit_of_measurement: "Volts"
    # Telemetry packets come in two flavors: The default node telemetry, and the I2C sensor data.
    # Both packets contain "voltage" so we check for temperature to ignore the sensor packet here.

  - name: "Node 1 Battery Percent"
    unique_id: "node_1_battery_percent"
    state_topic: "msh/US/2/json/LongFast/!67ea9400"
    state_class: measurement
    value_template: >-
      {% if value_json.from == 4038675309 and value_json.payload.battery_level is defined %}
        {{ (value_json.payload.battery_level | float) | round(2) }}
      {% else %}
        {{ this.state }}
      {% endif %}
    device_class: "battery"
    unit_of_measurement: "%"

  - name: "Node 1 ChUtil"
    unique_id: "node_1_chutil"
    state_topic: "msh/US/2/json/LongFast/!67ea9400"
    state_class: measurement
    value_template: >-
      {% if value_json.from == 4038675309 and value_json.payload.channel_utilization is defined %}
        {{ (value_json.payload.channel_utilization | float) | round(2) }}
      {% else %}
        {{ this.state }}
      {% endif %}
    unit_of_measurement: "%"

  - name: "Node 1 AirUtilTX"
    unique_id: "node_1_airutiltx"
    state_topic: "msh/US/2/json/LongFast/!67ea9400"
    state_class: measurement
    value_template: >-
      {% if value_json.from == 4038675309 and value_json.payload.air_util_tx is defined %}
        {{ (value_json.payload.air_util_tx | float) | round(2) }}
      {% else %}
        {{ this.state }}
      {% endif %}
    unit_of_measurement: "%"
```

This is how it can look like:
![Hassio dashboard view](/img/hassio_dashboards.png)

## Configuring individual nodes before deployment

Before deploying individual nodes, it's important to configure them according to your mesh settings so each device can join the network.

```
#Base
meshtastic --set lora.region EU_868
meshtastic --set-owner 'Photonode1' --set-owner-short  'PHO1'
meshtastic --set lora.modemPreset MEDIUM_FAST

#Set key for remote administration
meshtastic --set security.admin_key base64:PooQyXA2TYzkV0oMKgs+mNgM9jBKsPg/hpti0H1d51Q=

#Channel
meshtastic --ch-set psk base64:puavdd7vtYJh8NUVWgxbsoG2u9Sdqc54YvMLs+KNcMA= --ch-index 0
meshtastic --ch-set module_settings.position_precision 32 --ch-index 0
```
