# Wiegand Keypad

Based on the code [https://github.com/algorytmix02/wiegand-2-mqtt/blob/main/RFID_Wiegand_MQTT.ino](https://github.com/algorytmix02/wiegand-2-mqtt/blob/main/RFID_Wiegand_MQTT.ino)
with several adaptations to make the code compatible with an ESP32.
Additionally, the MQTT labels have been adjusted to be more descriptive.

# Overall Operation

An entry on the keypad sends information to the controller (ESP32 in this case).
The controller sends the information via MQTT.
Home Assistant (HA) retrieves the sent data, which is then verified via a Python script. If the user is authorized, the validated data is sent via MQTT to a dedicated topic. If not, it is sent to a different topic. These two topics are monitored by HA; every time one of them is modified, an automation is triggered to process the request.

# rfidwiegand Package

```yaml=
rfidwiegand:
  mqtt:
    sensor:
      - name: "RFID status"
        unique_id: clavier_rfid_status
        state_topic: "rfid/status"
        icon: mdi:lan-connect

      - name: "Last event"
        unique_id: clavier_rfid_last_event
        state_topic: "rfid/event"
        icon: mdi:card-account-details

      - name: "RFID Code"
        unique_id: clavier_rfid_code
        state_topic: "rfid/event"
        value_template: "{{ value_json.code }}"
        icon: mdi:card-account-details

      - name: "RFID Type"
        unique_id: clavier_rfid_type
        state_topic: "rfid/event"
        value_template: "{{ value_json.type }}"
        icon: mdi:card-account-details

      - name: "RFID Input"
        unique_id: clavier_rfid_input
        state_topic: "rfid/input"
        icon: mdi:login-variant
        
      - name: "RFID entry OK"
        unique_id: rfid_access_ok
        state_topic: "rfid/ok"
        icon: mdi:login-variant
    
      - name: "RFID entry Nok"
        unique_id: rfid_access_nok
        state_topic: "rfid_nok"
        icon: mdi:login-variant

    switch:
      - name: "RFID Relay 1"
        unique_id: clavier_rfid_relay1
        state_topic: "rfid/portail"
        command_topic: "rfid/portail"
        payload_on: 1
        payload_off: 0
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:electric-switch

      - name: "RFID Relay 2 - LED"
        unique_id: clavier_rfid_relay2_led
        state_topic: "rfid/ledclavier"
        command_topic: "rfid/ledclavier"
        payload_on: 1 # Green
        payload_off: 0 # Red
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:electric-switch

      - name: "RFID Relay 3 - Buzzer"
        unique_id: clavier_rfid_relay3_buzzer
        state_topic: "rfid/buzzer"
        command_topic: "rfid/buzzer"
        payload_on: 1
        payload_off: 0
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:bell

      - name: "RFID Relay 4"
        unique_id: clavier_rfid_relay4
        state_topic: "rfid/relay4"
        command_topic: "rfid/relay4"
        payload_on: 1
        payload_off: 0
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:electric-switch

      - name: "RFID Door Pulse"
        unique_id: clavier_rfid_pulse1
        state_topic: "rfid/pulseporte"
        command_topic: "rfid/pulseporte"
        payload_on: 1
        payload_off: 0
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:flash

      - name: "RFID LED Pulse"
        unique_id: clavier_rfid_pulse2
        state_topic: "rfid/pulseled"
        command_topic: "rfid/pulseled"
        payload_on: 1
        payload_off: 0
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:flash

      - name: "RFID Light"
        unique_id: clavier_rfid_light
        state_topic: "rfid/light"
        command_topic: "rfid/light"
        payload_on: 1
        payload_off: 0
        state_on: 1
        state_off: 0
        optimistic: true
        icon: mdi:lightbulb

```

This package allows the retrieval of keypad data via MQTT.

# User Configuration File

```yaml=
user1:
  label: "User 1"
  code: "1234"
  type: "pin"
  active: true

user2:
  label: "User 2"
  code: "2345"
  type: "pin"
  active: true

userTemp:
  label: "Temporary User"
  code: "9876"
  type: "pin"
  active: false

u1_rfid:
  label: "User 1"
  code: "6414522"
  type: "rfid"
  active: true
  

```

Users are managed through this file. The `active` parameter allows for the deactivation of non-regular users.

# Python Script

```python=
#!/usr/bin/env python3
import yaml
from datetime import datetime
import sys
import os
import paho.mqtt.client as paho
import json

# Retrieved parameters
code = sys.argv[1]
type_ = sys.argv[2]

# Log file
log_file = "/config/digicode_logs.txt"

#MQTT
broker = "mymqtt.local" # IP or alias of the broker
port = 1883 # mqtt port
userMqtt = 'myUser' # mqtt user
passwdMqtt = 'fuck1ngDifficultPassword!' # mqtt password
topicOk = 'rfid/ok'
topicNok = 'rfid/nok'

def on_publish(client, userdata, result):  # create function for callback
    pass

def sendMqtt(topic, data):
    client = paho.Client("digicodeCheck")  # create client object
    client.username_pw_set(userMqtt, passwdMqtt)
    client.on_publish = on_publish  # assign function to callback
    client.connect(broker, port)  # establish connection
    ret = client.publish(topic, data)  # publish

# Load users
with open("/config/digicode_users.yaml", "r", encoding="utf-8") as f:
    users = yaml.safe_load(f)

# Search for the user
matched_label = None
for key, user in users.items():
    if user.get("active")== True and str(user.get("code")) == code and user.get("type") == type_:
        matched_label = user.get("label")
        break

# Current time
now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# Log
if matched_label:
    log_line = f"[{now}] ✅ Access authorized: {matched_label} ({type_})\n"
    dataOk = {
        "dateHeure": now,
        "type": type_,
        "user": matched_label
    }
    sendMqtt(topicOk, json.dumps(dataOk, indent=4))
else:
    log_line = f"[{now}] ❌ Access denied: code={code}, type={type_}\n"
    dataNok = {
        "dateHeure": now,
        "type": type_,
        "code": code
    }
    sendMqtt(topicNok, json.dumps(dataNok, indent=4))
    
print(log_line)
with open(log_file, "a", encoding="utf-8") as log:
    log.write(log_line)

```

# Digicode Package

```yaml=
digicode:
  homeassistant:
    customize:
      input_text.digicode_log:
        friendly_name: Last user
      input_text.digicode_lastcode:
        friendly_name: Last scanned code

  input_text:
    digicode_log:
      max: 30
    digicode_lastcode:
      max: 20
      
  shell_command:
    digicode_check: "python3 /config/scripts/digicode_check.py {{ code }} {{ type }}"

  automation:
    - alias: Digicode - Verify code via Python script
      trigger:
        - platform: mqtt
          topic: "rfid/event"
      variables:
        payload: "{{ trigger.payload_json }}"
        code: "{{ payload.code }}"
        type: "{{ payload.type }}"
      action:
        - service: shell_command.digicode_check
          data:
            code: "{{ code }}"
            type: "{{ type }}"
            

```

Once all information is gathered, it can be displayed in HA.
![last access](./img/digicode-lastAccess.jpg)


