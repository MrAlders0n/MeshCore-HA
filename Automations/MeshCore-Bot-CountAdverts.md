# 📈 MeshCore-HA Automation · Advert Counter & Hourly Graph

This automation tracks how many `EventType.ADVERTISEMENT` packets your Home Assistant hears, **counts them per hour**, and **graphs the hourly totals** with ApexCharts.

---

## 📘 Overview

**Purpose**
- Increment a counter every time an Advert is received
- Reset the counter at the **top of each hour**, producing clean per-hour buckets
- Display a rolling graph (last 48h by default) of hourly totals

**How it works**
1. **Increment**: On every `meshcore_raw_event` with `EventType.ADVERTISEMENT`, bump a counter.
2. **Reset**: At minute `00` of every hour, reset the counter back to zero.  
   - The graph then uses the **hourly max** for each bucket as that hour’s count.
3. **Graph**: ApexCharts groups values by `60min` and displays the **max** per bucket (i.e., “how high the counter got before the hourly reset”).

> 🧠 Why `group_by: max`?  
> Because we reset the counter every hour, the **maximum value reached** in that hour equals the **total adverts** heard in that hour.

---

## ✅ Prerequisites

- **MeshCore-HA** events flowing into Home Assistant (so `meshcore_raw_event` fires for adverts).
- **ApexCharts Card** installed in your HA frontend (HACS or manual resource).  
  Once installed, you can add the card via “Manual” YAML.

---

## 🧰 Helper: Counter

Create a Counter Helper named **`counter.meshcore_advert_total`**.

### Option A — UI (recommended)
**Settings → Devices & Services → Helpers → Create Helper → Counter**  
- Name: `MeshCore Advert Total`  
- Step: `1`  
- (Optional) Turn **off** “Restore” if you want a clean slate after HA restarts

This will create an entity like `counter.meshcore_advert_total`.

---

## ⚙️ Automations

### 1) Increment on every Advert
```yaml
alias: MeshCore - Total Advert - Increment
description: ""
triggers:
  - trigger: event
    event_type: meshcore_raw_event
    event_data:
      event_type: EventType.ADVERTISEMENT
conditions: []
actions:
  - action: counter.increment
    target:
      entity_id: counter.meshcore_advert_total
    data: {}
mode: parallel
```

### 2) Reset at the top of each hour
```yaml
alias: Meshcore - Total Advert - Reset
description: ""
triggers:
  - trigger: time_pattern
    minutes: "0"
    hours: "*"
conditions: []
actions:
  - action: counter.reset
    target:
      entity_id: counter.meshcore_advert_total
    data: {}
mode: parallel
```

---

## 📊 Lovelace: ApexCharts (Hourly Totals)

Add this card (Dashboard → Edit → Add Card → **Manual**):

```yaml
type: custom:apexcharts-card
header:
  show: true
  title: Adverts Heard
  colorize_states: true
  show_states: true
all_series_config:
  stroke_width: 3
  curve: smooth
  group_by:
    func: max
    duration: 60min
graph_span: 48h
apex_config:
  legend:
    show: false
  chart:
    zoom:
      enabled: true
    toolbar:
      show: true
      tools:
        zoom: true
        zoomin: true
        zoomout: true
        pan: true
        reset: true
series:
  - entity: counter.meshcore_advert_total
    name: Adverts
    show:
      in_header: false
      legend_value: false
```

**What you’ll see**
- A smooth line where each point is the **hourly advert count**
- Pan/zoom controls to inspect bursts and quiet periods
- 48-hour window (adjust `graph_span` as you like)

---

## 🧪 Verify It Works

1. **Watch the counter**: Developer Tools → States → `counter.meshcore_advert_total`  
   - It should increment on each Advert
2. **Check the reset**: At `HH:00`, the counter should reset to `0`
3. **Graph**: After one full hour, you’ll see the first bucket populated

---

## 🛠️ Troubleshooting

- **No increments?** Confirm you’re seeing `meshcore_raw_event` with `EventType.ADVERTISEMENT` in **Settings → Logs** 
- **Weird hourly numbers?** Ensure only **one** reset automation runs at `:00`. Duplicate resets (or a restart at `:00`) can skew the hour’s max.
- **Gaps on the graph?** If HA restarts during an hour, the counter restarts too. That hour’s max may be lower than expected.

---

## 🧭 Variations

- **Different window**: Change `graph_span: 48h` to `7d` for a weekly trend.
- **Bar chart look**: In `apex_config`, set `chart: { type: bar }` for columns.

---
