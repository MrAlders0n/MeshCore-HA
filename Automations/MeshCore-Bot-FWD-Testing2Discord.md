# 🛰️ MeshCore-HA Automation · Forward Channel 1 Messages to Discord

This automation listens for **messages sent on Channel 1** (our `#testing` channel) and automatically forwards them to a **dedicated Discord channel** using a Home Assistant **notify service** connected to a Discord bot.  

It’s used across the **Ottawa MeshCore** deployment to visualize where packets are being received city-wide — each repeater or listener posts to its own Discord thread so we can see which nodes picked up the message.

---

## 📘 Overview

**Created in:**  
Home Assistant Automations GUI (`Settings → Automations & Scenes → Automations`)  

**Purpose:**  
- Listen for all Channel 1 (`#testing`) messages  
- Forward the message and sender to a Discord channel  
- Allow public visibility into message propagation across repeaters  

> ⚠️ *Discord bot setup and authentication are **not** included here.*  
> Configuration of Discord bots, webhooks, and notify services is well-documented on  
> the official [Discord Developer Portal](https://discord.com/developers/docs/intro)  
> and the [Home Assistant Discord integration documentation](https://www.home-assistant.io/integrations/discord/).


---

## ⚙️ Trigger Conditions

- **Event Type:** `meshcore_message`  
- **Message Type:** `channel`  
- **Channel Index:** `1` (Testing)

This will fire on **every** message sent in Channel 1 regardless of sender.

---

## 🪄 Automation: “MeshCore – Bot – Forward test to discord-test”

```yaml
alias: MeshCore - Bot - Forward test to discord-test
description: Forward test to discord-test
triggers:
  - trigger: event
    event_type: meshcore_message
    event_data:
      message_type: channel
      channel_idx: 1
    enabled: true
conditions: []
actions:
  - action: notify.benderha
    data:
      target: "0000000000000002"
      message: "{{ trigger.event.data.sender_name }}: {{ trigger.event.data.message }}"
mode: parallel
max: 20
```

---

## 🧰 Explanation

| Step | Description |
|------|--------------|
| **Trigger** | Listens for any `meshcore_message` event where `channel_idx` = 1 |
| **Condition** | None – all Channel 1 traffic is included |
| **Action** | Sends a formatted string (`sender_name: message`) via the Discord notify service |
| **Notify Target** | Discord channel ID `0000000000000002` (replace with your own) |
| **Mode** | `parallel` so multiple mesh messages can forward concurrently |

---

## 🧠 Notes & Tips

- Works best when paired with a **dedicated Discord test channel** (e.g., `#meshcore-testing`).  
- You can create separate automations per channel index if you want to log other channels differently.  
- The `notify.benderha` service name depends on your own Discord bot integration name in Home Assistant.

---

## ✅ Example Output

```
Sydney: Test message across the mesh!
```

---

## 🧾 Credits

- MeshCore-HA Integration by the MeshCore-HA project
  https://github.com/meshcore-dev/meshcore-ha
- Ottawa MeshCore community for field testing  
- Example automations by @MrAlders0n  