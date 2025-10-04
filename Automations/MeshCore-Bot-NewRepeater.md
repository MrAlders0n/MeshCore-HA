# 🛰️ MeshCore-HA Automation · New Repeater Detected (with Duplicate-ID Check)

This automation listens for **MeshCore repeater advertisements** and notifies when a **new repeater** is detected.  
It also checks for **duplicate IDs** by comparing the **first two hex characters** (prefix) of the repeater’s **public key** against existing repeaters.  
If a duplicate prefix is found (indicating a potential **ID collision**), it sends a distinct warning message.  
All events are **forwarded to a Discord channel** for city-wide visibility.

---

## 📘 Overview

**Purpose:**  
- Detect new **repeater** adverts (`EventType.ADVERTISEMENT`)  
- Compare the advert’s public key **prefix** against known repeaters to flag **duplicates**  
- Notify your HA companion + **forward to Discord**  
- Append the repeater public key to `/config/www/MeshCore_Known_Repeaters.txt`

> **📌 Important Note**  
> This automation **must** persist the known-repeaters list in a **JSON file on disk**.  
> Home Assistant **entities (sensors/helpers)** cannot store large lists (≈255 char limit), so **JSON on disk** is required.  
> Using `/config/www/MeshCore_Known_Repeaters.txt` avoids this limitation and keeps data across restarts.

> ⚠️ *Discord bot setup and authentication are **not** included here.*  
> Configuration of Discord bots, webhooks, and notify services is well-documented on  
> the official [Discord Developer Portal](https://discord.com/developers/docs/intro)  
> and the [Home Assistant Discord integration documentation](https://www.home-assistant.io/integrations/discord/).

---

## ⚙️ Trigger Conditions

- **Event Type:** `meshcore_raw_event`  
- **Payload Event:** `EventType.ADVERTISEMENT`  

---

## 🧩 Files Used

| File | Purpose |
|------|---------|
| *(Created via GUI)* | Main automation logic |
| `/config/scripts/meshcore_add_repeater.sh` | Bash helper to append + dedupe repeater public keys |
| `/config/www/MeshCore_Known_Repeaters.txt` | Persistent JSON list of known repeater public keys |
| `/config/configuration.yaml` | Shell command + REST sensor definitions (`sensor.meshcore_known_repeaters`) |

---

## 🪄 Automation: “Meshcore – Bot – New Repeater Detected”

Create the following automation in the **Home Assistant GUI**.

```yaml
alias: Meshcore - Bot - New Repeater Detected
description: Detects new repeater and sends message to discord
triggers:
  - trigger: event
    event_type: meshcore_raw_event
    event_data:
      event_type: EventType.ADVERTISEMENT
conditions:
  - condition: template
    value_template: >-
      {% set repeater = trigger.event.data.payload.public_key | trim %} {% set
      known = state_attr('sensor.meshcore_known_repeaters','repeaters') |
      default([]) %} {{ repeater not in known }}
actions:
  - variables:
      pub: >-
        {{ (trigger.event.data.payload.public_key | default('')) | string |
        lower }}
      contacts_ids: |-
        {{ states.binary_sensor
           | map(attribute='entity_id')
           | select('search','^binary_sensor\\.meshcore_.*_contact$')
           | list }}
      eid: |-
        {% set ns = namespace(eid='') %}
        {% for id in contacts_ids %}
          {% set pk = (state_attr(id,'public_key') or state_attr(id,'Public key') or '') | string | lower %}
          {% if pk == pub %}
            {% set ns.eid = id %}
          {% endif %}
        {% endfor %}
        {{ ns.eid }}
      adv_name: "{{ state_attr(eid,'adv_name') or state_attr(eid,'Adv name') }}"
      node_type_str: "{{ state_attr(eid,'node_type_str') or state_attr(eid,'Node type str') }}"
      adv_lat: "{{ state_attr(eid,'adv_lat') or state_attr(eid,'Adv lat') }}"
      adv_lon: "{{ state_attr(eid,'adv_lon') or state_attr(eid,'Adv lon') }}"
      ts: "{{ trigger.event.data.timestamp | default(now().timestamp()) }}"
  - if:
      - condition: template
        value_template: "{{ node_type_str == 'Repeater' }}"
    then:
      - delay:
          hours: 0
          minutes: 1
          seconds: 0
          milliseconds: 0
        enabled: false
      - action: notify.benderha
        metadata: {}
        data:
          target: "000000000000000001"
          message: >+
            {% set prefix = pub[0:2] %}

            {% set known =
            state_attr('sensor.meshcore_known_repeaters','repeaters') |
            default([]) | map('string') | map('lower') | list %}

            {% set candidates = known | select('search','^' ~ prefix) |
            reject('equalto', pub) | list %}

            {% set known_pk = (candidates[0] if candidates|count > 0 else '') %}

            {# find a contact entity for the known_pk to pull its name #}

            {% set ns = namespace(keid='', kname='') %}

            {% for id in contacts_ids %}
              {% set pk = (state_attr(id,'public_key') or state_attr(id,'Public key') or '') | string | lower %}
              {% if known_pk and pk == known_pk %}
                {% set ns.keid = id %}
              {% endif %}
            {% endfor %}

            {% set ns.kname = (state_attr(ns.keid,'adv_name') or
            state_attr(ns.keid,'Adv name')) %}

            {% set duplicate = (known_pk != '') %}

            {% if duplicate %}

            ⚠️ **Duplicate Repeater ID Detected**

            **Prefix:** {{ prefix }}

            **Advert name:** {{ adv_name if adv_name else "Unknown" }}  **Advert
            pubkey:**   ```{{ pub }}```

            **Known name:** {{ ns.kname if ns.kname else "Unknown" }} **Known
            pubkey:**   ```{{ known_pk }}```

            {% else %}

            📡 **New repeater detected**

            **Name:** {{ adv_name if adv_name else "Unknown" }} 

            **Last adv coords:** {{ adv_lat ~ ", " ~ adv_lon if adv_lat is not
            none and adv_lon is not none else "Unknown" }} 

            **Public key:**   ```{{ pub }}``` 

            **MeshCore URL**

            {% endif %}

        enabled: true
      - data:
          name: "{{ trigger.event.data.payload.public_key | trim }}"
        action: shell_command.meshcore_add_repeater
      - action: homeassistant.update_entity
        metadata: {}
        data:
          entity_id:
            - sensor.meshcore_known_repeaters
mode: parallel
max: 10

```

---

## 🧰 Helper Script: `/config/scripts/meshcore_add_repeater.sh`

Create the following script on your Home Assistant host.

```bash
#!/bin/bash
set -euo pipefail

FILE="/config/www/MeshCore_Known_Repeaters.txt"
TMP="${FILE}.tmp"
NAME="$1"

# Initialize file if missing/empty
if [ ! -s "$FILE" ]; then
  echo '{"repeaters":[]}' > "$FILE"
fi

# Append and de-duplicate
jq --arg name "$NAME" '
  .repeaters += [$name]
  | .repeaters |= unique
' "$FILE" > "$TMP" && mv "$TMP" "$FILE"
```

---

## 📄 JSON List Example: `/config/www/MeshCore_Known_Repeaters.txt`

Create/edit the JSON file (one public key per entry).

```json
{
  "repeaters": [
    "03c7e4c6abd49dac36043ed3276e7cb9cfac57bc428b704de3787530ce3f814e",
    "fcb7f2db3d45bcf4994d72354405751819b3b0498cc6864f975198d4f20cd334",
    "fe653f47e232adab960c273221c693d9709439aa0d811424ed9622febca93c4d"
  ]
}
```

Automatically managed by the automation and REST sensor.

---

## 🧩 Configuration Additions (`configuration.yaml`)

Add the shell command and REST sensor. **Restart Home Assistant** after editing.  
Replace `<<HA-IPADDRESS>>` with your HA IP/hostname.

```yaml
shell_command:
  meshcore_add_repeater: /bin/bash /config/scripts/meshcore_add_repeater.sh "{{ name }}"

sensor:
  - platform: rest
    name: MeshCore Known Repeaters
    resource: http://<<HA-IPADDRESS>>:8123/local/MeshCore_Known_Repeaters.txt
    method: GET
    value_template: "{{ (value_json.repeaters | count) | default(0) }}"
    json_attributes:
      - repeaters
    scan_interval: 300
```

---

## 🧠 Notes & Tips

- **Duplicate detection** uses the **first two hex characters** of the public key (`prefix`).  
  This quickly surfaces likely ID collisions, especially when many repeaters are deployed.
- File path `/config/www/` lets HA expose the JSON via `/local/…` and simplifies REST reads.  
- To reset: delete or clear the JSON file (the automation will re-create it as needed).  

---

## 💬 Example Outputs

**New repeater:**
```
📡 New repeater detected (ADVERTISEMENT)
Name: Ashton_R1
Last adv coords: 45.1234, -75.9876
Public key: fe653f47e232adab960c273221c693d9709439aa0d811424ed9622febca93c4d
```

**Duplicate ID:**
```
⚠️ Duplicate Repeater ID Detected
Prefix: fe
Advert name: Ashton_R1
Advert pubkey: fe653f47e232adab960c273221c693d9709439aa0d811424ed9622febca93c4d
Known name: Orleans_R1
Known pubkey: fe653f47e232adab960c273221c693d9709439aa0d811424ed9622febca93c4a
```

---

## 🧾 Credits

- MeshCore-HA Integration by the MeshCore-HA project
  https://github.com/meshcore-dev/meshcore-ha
- Ottawa MeshCore community for field testing  
- Example automations by @MrAlders0n  