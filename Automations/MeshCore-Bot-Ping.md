# 🛰️ MeshCore Bot – Ping Automation (Home Assistant)

This automation lets **Bender** (The name of our MeshCore Bot) respond to `!ping` messages in the **Public channel**, showing both the hop count and the actual RF path the message took through the mesh.

---

## 📦 Overview

When a packet is received by Home Assistant via the MeshCore integration, it emits two key events:

- `EventType.RX_LOG_DATA` → raw LoRa payload, includes header, path, and channel-hash.  
- `EventType.CHANNEL_MSG_RECV` → decrypted channel message text.

We use these together:

1. The **RX_LOG_DATA** event is decoded to extract:
   - Hop count (`path_len`)
   - Route path bytes (`path_bytes`)
   - Channel-hash (`0x11` → Public channel)
   - Timestamp (for temporal matching)
2. These values are cached in **input helpers**.
3. When someone sends `@[Bender] !ping` on the Public channel,
   the **CHANNEL_MSG_RECV** automation compares timestamps and responds:
   > Pong 🏓 — your ping was received in 3 hops • Path: CF→92→4E

---

## ⚙️ Prerequisites

- [Home Assistant](https://www.home-assistant.io/)
- [MeshCore-HA Integration](https://github.com/MeshCore-HA/meshcore-ha)
- Your MeshCore node configured to forward RX log events to HA.

---

## 🧰 Step 1 — Create Helpers

Add these helpers (in UI: *Settings → Devices & Services → Helpers → + Add Helper → Text/Number*):

```yaml
input_text:
  meshcore_last_path_bytes:
    name: MeshCore Last Path Bytes
    max: 255

input_number:
  meshcore_last_path_len:
    name: MeshCore Last Path Len
    min: 0
    max: 32
    step: 1

input_number:
  meshcore_last_rx_time:
    name: MeshCore Last RX Time (epoch)
    min: 0
    max: 32503680000
    step: 0.001
```

These act as temporary storage between the RX and Ping automations.

---

## 🛰️ Step 2 — Cache RX Path (Public Channel Only)

This automation listens for raw RX logs, extracts the hop path, and caches it.

```yaml
alias: MeshCore - Cache RX Path (ch0)
description: Extract path from RX_LOG_DATA and cache it for quick lookups
triggers:
  - trigger: event
    event_type: meshcore_raw_event
    event_data:
      event_type: EventType.RX_LOG_DATA
conditions: []
actions:
  - variables:
      _p: "{{ (trigger.event.data.payload.payload | default('') | string) }}"
      _r: "{{ (trigger.event.data.payload.raw_hex | default('') | string) }}"
      hx: >-
        {% set base = _p if _p|length > 0 else (_r[4:] if _r|length > 4 else '')
        %} {{ base | replace(' ', '') | replace('\n', '') | lower }}
      ok: "{{ hx is string and (hx|length) >= 4 and (hx|length) % 2 == 0 }}"
      path_len: "{{ (hx[2:4] if ok else '00') | int(16) }}"
      have_full_path: "{{ ok and (hx|length) >= (4 + path_len * 2) }}"
      path_bytes_str: |-
        {% if have_full_path and path_len > 0 %}
          {% set ns = namespace(s='') %}
          {% for i in range(path_len) %}
            {% set b = hx[4 + i*2 : 4 + i*2 + 2] %}
            {% set ns.s = ns.s + ((' ' if ns.s else '') + b) %}
          {% endfor %}
          {{ ns.s }}
        {% else %}{% endif %}
      chan_hash_hex: >-
        {% set i = 4 + path_len * 2 %} {{ hx[i:(i+2)] if have_full_path and
        (hx|length) >= (i+2) else '' }}
      rx_ts: "{{ (trigger.event.data.timestamp | float(0)) | round(0) }}"
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ have_full_path and path_len > 0 }}"
          - condition: template
            value_template: |-
              {{ (chan_hash_hex | default('') | string
                  | regex_replace('[^0-9a-fA-F]','')) | int(base=16) == 0x11 }}
        sequence:
          - action: input_text.set_value
            target:
              entity_id: input_text.meshcore_last_path_bytes
            data:
              value: "{{ path_bytes_str | default('') }}"
          - action: input_number.set_value
            target:
              entity_id: input_number.meshcore_last_path_len
            data:
              value: "{{ path_len }}"
          - action: input_number.set_value
            target:
              entity_id: input_number.meshcore_last_rx_time
            data:
              value: "{{ rx_ts | float(0) }}"
          - action: persistent_notification.create
            data:
              title: RX Path gate
              message: >-
                chan_hash_hex={{ chan_hash_hex | tojson }} | norm={{
                (chan_hash_hex | string | lower | trim) | tojson }}
            enabled: false
mode: parallel
max: 10
```

---

## 🤖 Step 3 — Ping Bot (Respond to !ping)

```yaml
alias: MeshCore - Bot - Pingv3
description: When ch0 message contains '@[bender] !ping', reply with Pong and path length
triggers:
  - trigger: event
    event_type: meshcore_raw_event
    event_data:
      event_type: EventType.CHANNEL_MSG_RECV
conditions:
  - condition: template
    value_template: >-
      {% set p = trigger.event.data.payload %} {% set msg = (p.text | string |
      lower) %} {{ p.channel_idx == 0 and '@[bender]' in msg and '!ping' in msg
      }}
actions:
  - variables:
      p: "{{ trigger.event.data.payload }}"
      hops: "{{ p.path_len | int(0) }}"
      now_ts: "{{ (trigger.event.data.timestamp | float(0)) | round(0) }}"
      rx_ts: "{{ states('input_number.meshcore_last_rx_time') | float(0) | round(0) }}"
      rx_len: "{{ states('input_number.meshcore_last_path_len') | int(0) }}"
      rx_bytes_str: "{{ states('input_text.meshcore_last_path_bytes') | string }}"
      rx_age: >-
        {{ ((trigger.event.data.timestamp | float(0)) | round(0)) -
        (states('input_number.meshcore_last_rx_time') | float(0) | round(0)) }}
      show_path: >-
        {{ 0 <= rx_age <= 5 and (states('input_number.meshcore_last_path_len') |
        int(0)) == (p.path_len | int(0)) and
        (states('input_text.meshcore_last_path_bytes') | length > 0) }}
      path_arrow: |-
        {% if show_path %}
          {% set parts = rx_bytes_str.split() %}
          {{ parts | map('upper') | join('→') }}
        {% else %}{{ '' }}{% endif %}
  - action: meshcore.send_channel_message
    data:
      channel_idx: "{{ p.channel_idx }}"
      message: >-
        {% set hop_word = 'hop' if hops == 1 else 'hops' %} {% if show_path and
        path_arrow %}
          Pong 🏓 — your ping was received in {{ hops }} {{ hop_word }} • Path: {{ path_arrow }}
        {% else %}
          Pong 🏓 — your ping was received in {{ hops }} {{ hop_word }}
        {% endif %}
  - delay:
      seconds: 5
mode: single
```

---

## ✅ Example Output

```
EssentialTrash_T114: @[Bender] !ping
```

**Bender replies:**
> Pong 🏓 — your ping was received in 3 hops • Path: CF→92→4E

---

## 🧠 How It Works

1. Every RX packet is decoded; if the channel hash is `0x11` (Public), it saves path info.
2. When `!ping` arrives, HA matches it to the last RX log (by timestamp and hop length).
3. The bot sends back a dynamic response that shows *real RF hop data*.

This means your ping response isn’t simulated — it’s showing the actual route that message traveled through the mesh. ⚡

---

## 🛠️ Troubleshooting

- **Ping shows no path:**  
  Increase the join window in the Ping automation from `<= 5` to `<= 10`.

- **No helpers updating:**  
  Check that the Cache automation trigger uses `EventType.RX_LOG_DATA`.  
  Confirm channel hash `0x11` in traces.

---

## 🧾 Credits

- MeshCore-HA Integration by the MeshCore-HA project
  https://github.com/meshcore-dev/meshcore-ha
- Ottawa MeshCore community for field testing  
- Example automations by @MrAlders0n  