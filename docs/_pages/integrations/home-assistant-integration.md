---
title: Home Assistant
category: Integrations
order: 21
---
## Home Assistant Integration

### MQTT Broker
At first you need a MQTT Broker (if not already present).
Install and configure the "Mosquitto Broker" Addon from the Official Addon Repository withing HomeAssistant. Follow the documentation of the addon and don't forget to create a dedicated user for this addon.

### Valetudo Settings
When Addon and MQTT Broker Integration is present, you can do the Valetudo MQTT configuration (Settings -> MQTT).
Enable MQTT, Add the Server IP of your Homeassistant instance as "Server" option. For Username/Password you should now use the dedicated user which was previous created for the Homeassistant MQTT Broker. Ensure the Autodiscovery Settings (For Homeassistant AND Homie) are enabled. Then Save the Settings to let the magic happen.

### Homeassistant
Homeassistant will now discover lots of entities you can now read and use.
Some basic functions like starting, stopping or returning to base can now be called with the appropriate homeassistant vacuum integration.
Since Valetudo 2021.04.0 "vacuum.send_command" is no longer supported (which was used for things like segment cleaning or goto location).
Now the MQTT publish Homeassistant Component must be used for advanced commands.

### Examples:

#### Basic Services

Assuming Robot entity = vacuum.robot

Starting and stopping the robot
```
service: vacuum.stop
target:
  entity_id: vacuum.robot
```

```
service: vacuum.start
target:
  entity_id: vacuum.robot
```

#### Advanced Services

For using the Homeassistant MQTT Publish component, you need to know the topic prefix and the identifier. These Settings can be found in the Valetudo MQTT settings.

For these examples we are assuming topic prefix=valetudo and identifier=robot

For the segment cleaning capability, you should first go ahead to valetudo and rename your segments (rooms). Then you can go and check out the entity "sensor.map_segments" which provides a list of your rooms like this:

```
'16': livingroom
'17': kitchen
'18': floor
'19': office
'20': bathroom
```

The resulting Homeassistant Service to clean the bathroom, floor and livingroom in this order 2 times would then look like this:

```
service: mqtt.publish
data:
  topic: valetudo/robot/MapSegmentationCapability/clean/set
  payload: '{"segment_ids": ["20", "18", "16"], "iterations": 2, "customOrder": true}'
```

For more features check out the [MQTT documentation](/pages/integrations/mqtt.html).


#### Segment cleaning lovelace

**HACS requirements: [auto-entities](https://github.com/thomasloven/lovelace-auto-entities), [button-card](https://github.com/custom-cards/button-card).** 

![image](./img/ha-lovelace-segments.png)

Add the following card to your lovelace dashboard (Replace `vacuum.dreamez10pro` with your vacuum entry)
```yaml
{% raw %}
type: vertical-stack
cards:
  - type: custom:auto-entities
    card:
      type: entities
      state_color: true
      title: Rooms to Vacuum
    filter:
      include:
        - group: group.vacuum_rooms
      exclude: []
    show_empty: true
    sort:
      method: friendly_name
      reverse: false
      numeric: false
  - type: custom:button-card
    tap_action:
      action: call-service
      service: script.vacuum_clean_segments
      confirmation: true
      service_data: {}
      target: {}
    lock:
      enabled: >-
        [[[return states['group.vacuum_rooms'].state !== 'on' ||
        states['vacuum.dreamez10pro'].state !== 'docked']]]
      exemptions: []
    entity: script.vacuum_clean_segments
    name: Vacuum selected segments
    show_state: false
    show_icon: false
{% endraw %}
```

Now change the following config files:

`/config/configuration.yaml`
```yaml
input_boolean:
  vacuum_hallway:
    name: Hallway
    icon: mdi:foot-print
  vacuum_livingroom:
    name: Livingroom
    icon: mdi:sofa
  vacuum_bedroom:
    name: Bedroom
    icon: mdi:bed-empty
  vacuum_kitchen:
    name: Kitchen
    icon: mdi:silverware-fork-knife
  vacuum_study:
    name: Studyroom
    icon: mdi:laptop
```

Make sure your `room_id` matches the segments from the `sensor.map_segments` attributes, example:

```yaml
'16': livingroom
'17': kitchen
'18': floor
'19': study
'20': bedroom
```

`/config/customize.yaml`
```yaml
input_boolean.vacuum_hallway:
  room_id: "18"
input_boolean.vacuum_livingroom:
  room_id: "16"
input_boolean.vacuum_bedroom:
  room_id: "20"
input_boolean.vacuum_kitchen:
  room_id: "17"
input_boolean.vacuum_study:
  room_id: "19"
```

`/config/groups.yaml`
```yaml
vacuum_rooms:
  name: Vacuum Rooms
  entities:
    - input_boolean.vacuum_bedroom
    - input_boolean.vacuum_hallway
    - input_boolean.vacuum_kitchen
    - input_boolean.vacuum_livingroom
    - input_boolean.vacuum_study
```

`/config/scripts.yaml`
```yaml
{% raw %}
vacuum_clean_segments:
  sequence:
  - service: script.turn_on
    target:
      entity_id: script.vacuum_clean_segments_message
    data:
      variables:
        segments: '{{expand("group.vacuum_rooms") | selectattr("state","eq","on")
          | map(attribute="attributes.room_id") | list | to_json}}'
  mode: single
  alias: vacuum_clean_segments
  icon: mdi:arrow-right
vacuum_clean_segments_message:
  alias: vacuum_clean_segments_message
  sequence:
  - service: mqtt.publish
    data:
      topic: valetudo/robot/MapSegmentationCapability/clean/set
      payload_template: '{"segment_ids": {{segments}}}'
  mode: single
{% endraw %}
```

Restart HA and everything should work!

##### Re-using the script for single segment cleaning

The `vacuum_clean_segments_message` script accepts the variable `segments` also as manual input, please check [passing variables to script](https://www.home-assistant.io/integrations/script/#passing-variables-to-scripts) how to integrate it into a button or automation.

If you use the [vacuum-card](https://github.com/denysdovhan/vacuum-card) you can also add single room clean buttons as followed:

```yaml
{% raw %}
type: custom:vacuum-card
entity: vacuum.dreamez10pro
stats:
  default:
    - entity_id: sensor.main_filter_hour
      unit: hours
      subtitle: Filter
    - entity_id: sensor.main_brush_hour
      unit: hours
      subtitle: Main brush
    - entity_id: sensor.right_brush_hour
      unit: hours
      subtitle: Right brush
    - entity_id: sensor.sensor_cleaning_hour
      unit: hours
      subtitle: Sensors
actions:
  - name: Clean living room
    service: script.vacuum_clean_segments_message
    service_data:
      segments: '["{{state_attr("input_boolean.vacuum_livingroom", "room_id")}}"]'
    icon: mdi:sofa
  - name: Clean Hallway
    service: script.vacuum_clean_segments_message
    service_data:
      segments: '["{{state_attr("input_boolean.vacuum_hallway", "room_id")}}"]'
    icon: mdi:foot-print
  - name: Clean bedroom
    service: script.vacuum_clean_segments_message
    service_data:
      segments: '["{{state_attr("input_boolean.vacuum_bedroom", "room_id")}}"]'
    icon: mdi:bed-empty
  - name: Clean Study
    service: script.vacuum_clean_segments_message
    service_data:
      segments: '["{{state_attr("input_boolean.vacuum_study", "room_id")}}"]'
    icon: mdi:laptop
  - name: Clean kitchen
    service: script.vacuum_clean_segments_message
    service_data:
      segments: '["{{state_attr("input_boolean.vacuum_kitchen", "room_id")}}"]'
    icon: mdi:silverware-fork-knife
compact_view: false
show_status: true
show_name: true
show_toolbar: true
{% endraw %}
```


### Map display

If you are on Hass.io and want the map also on your dashboards of Home Assistant, you can use the [Lovelace Valetudo Map Card
](https://github.com/TheLastProject/lovelace-valetudo-map-card).
