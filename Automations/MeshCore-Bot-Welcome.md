# 🛰️ MeshCore-HA Automation · New Client Node Welcome

This automation detects when a **new client node** sends its **first message on the Public channel (ch0)** and sends an automated welcome message.  
It also updates a persistent JSON list of known companions to prevent duplicate welcomes.

---

## 📘 Overview

**Purpose:**  
- Detect new nodes that speak in default public channel
- Welcome them in-channel with onboarding links  
- Log the event to your own notifier  
- Append companion name to `/config/www/MeshCore_Known_Companions.txt`

> **📌 Important Note**  
> This automation **must** persist the known-companions list in a **JSON file on disk**.  
> Home Assistant **entities (sensors/helpers)** cannot store long text (limit ≈255 chars),  
> so JSON on disk is required.  
>  
> Using `/config/www/MeshCore_Known_Companions.txt` avoids this limitation and ensures persistence across restarts.

---

## ⚙️ Trigger Conditions

- **Event Type:** `meshcore_message`  
- **Message Type:** `channel`  
- **Channel Index:** `0` (Public)

Only runs when the sender is **not already in the known list** stored in `/config/www/MeshCore_Known_Companions.txt`.

---

## 🧩 Files Used

| File | Purpose |
|------|----------|
| *(Created via GUI)* | Main automation logic |
| `/config/scripts/meshcore_add_companion.sh` | Bash helper to append + dedupe names |
| `/config/www/MeshCore_Known_Companions.txt` | Persistent JSON list of known nodes |
| `/config/configuration.yaml` | Shell command + REST sensor definitions |

---

## 🪄 Automation: “MeshCore - Messages from New (Unknown) Node”

Create the following automation in the Home Assistant GUI

```yaml
alias: MeshCore - Messages from New (Unknown) Node
description: >-
  Notify once when a node on ch0 speaks and sender is not in known list
  (reads/writes /config/www/MeshCore_Known_Companions.txt)
triggers:
  - event_type: meshcore_message
    event_data:
      message_type: channel
      channel_idx: 0
    trigger: event
conditions:
  - condition: template
    value_template: >-
      {% set sender = trigger.event.data.sender_name | trim %} {% set known =
      state_attr('sensor.meshcore_known_companions','companions') | default([])
      %} {{ sender not in known }}
actions:
  - action: meshcore.send_channel_message
    enabled: true
    data:
      channel_idx: 0
      message: >-
        Welcome @[{{ trigger.event.data.sender_name }}] to Ottawa MeshCore!
        Check out https://ottawamesh.ca and join our Discord (link on the
        wiki).👋
  - delay:
      seconds: 2
    enabled: true
  - action: meshcore.send_channel_message
    enabled: true
    data:
      channel_idx: 0
      message: "@[{{ trigger.event.data.sender_name }}] Join #ottawa + #testing."
  - action: notify.mralders0n
    data:
      title: MeshCore
      message: >-
        New node spoke on ch{{ trigger.event.data.channel_idx }} ({{
        trigger.event.data.channel }}): {{ trigger.event.data.sender_name }} →
        "{{ trigger.event.data.message }}" @ {{ trigger.event.data.timestamp }}
  - data:
      name: "{{ trigger.event.data.sender_name | trim }}"
    action: shell_command.meshcore_add_companion
  - action: homeassistant.update_entity
    metadata: {}
    data:
      entity_id:
        - sensor.meshcore_known_companions
mode: queued
max: 10

```

---

## 🧰 Helper Script: `/config/scripts/meshcore_add_companion.sh`

Create the following script on the HomeAssistant CLI

```bash
#!/bin/bash
set -euo pipefail

FILE="/config/www/MeshCore_Known_Companions.txt"
TMP="${FILE}.tmp"
NAME="$1"

# Initialize file if missing/empty
if [ ! -s "$FILE" ]; then
  echo '{"companions":[]}' > "$FILE"
fi

# Append and de-duplicate
jq --arg name "$NAME" '
  .companions += [$name]
  | .companions |= unique
' "$FILE" > "$TMP" && mv "$TMP" "$FILE"
```

---

## 📄 JSON List Example: `/config/www/MeshCore_Known_Companions.txt`

Create the the json file using the CLI. 

```json
{
  "companions": [
    "Alice",
    "Bob-Mobile",
    "Gadgets",
    "LoMo",
    "Mewtwo",
    "Sydney",
    "Svil_RAK",
    "willsonlilly"
  ]
}
```

Automatically managed by the automation and REST sensor.

---

## 🧩 Configuration Additions (`configuration.yaml`)

Edit your Home Assistant configuration.yaml file and add these. Restart Home Assistant after this is completed to update its configuration

Note: Replace <<HA-IPADDRESS>> with your HomeAssistant IP

```yaml
shell_command:
  meshcore_add_companion: /bin/bash /config/scripts/meshcore_add_companion.sh "{{ name }}"

sensor:
  - platform: rest
    name: MeshCore Known Companions
    resource: http://<<HA-IPADDRESS>>:8123/local/MeshCore_Known_Companions.txt
    method: GET
    value_template: "{{ (value_json.companions | count) | default(0) }}"
    json_attributes:
      - companions
    scan_interval: 300
```

---

## 🧠 Notes & Tips

- Uses *Public / Channel 0*  
- File path `/config/www/` allows HA to expose the JSON via `/local/`  
- To reset: delete the JSON file or clear its contents  

---

## 💬 Example Output

```
Welcome @LoMo to Ottawa MeshCore! 👋
Check out https://ottawamesh.ca and join our Discord (link on the wiki).
@LoMo Join #ottawa and #testing to meet everyone
```

---

## 🧾 Credits

- MeshCore-HA Integration by the MeshCore-HA project
  https://github.com/meshcore-dev/meshcore-ha
- Ottawa MeshCore community for field testing  
- Example automations by @MrAlders0n