# 🧠 MeshCore-HA Automations Collection

> **⚠️ Experimental & Work-In-Progress**  
> Everything in this repository is experimental and actively evolving.  
> These automations are shared to **help guide other Home Assistant users** integrating with **MeshCore-HA** or building similar event-driven bots and tools.  
> Expect frequent updates, refactors, and the occasional rough edge.

---

## 📘 Overview

This repository is a collection of **Home Assistant automations, scripts, and templates** that extend the **MeshCore-HA** integration.  
They are used within the **Ottawa MeshCore** deployment and related community projects.

---

## 🧰 Requirements

- Home Assistant 2025.9+  
- [MeshCore-HA Integration](https://github.com/MeshCoreCanada/meshcore-ha)  
- Working MeshCore node connected to your HA instance  
- Basic familiarity with YAML automations, shell commands, and REST sensors  


## 🧩 Automations Index

A quick overview of what’s included. Each page has full YAML, screenshots, and setup notes.

- **New Client Node Welcome**  
  Detects the first message from a *new client* on the Public channel (ch0), posts a friendly welcome with onboarding links, and appends the node to a persistent known-companions list to avoid duplicate welcomes.  
  → [`Automations/MeshCore-Bot-Welcome.md`](Automations/MeshCore-Bot-Welcome.md)

- **New Repeater Detected (with Duplicate-ID Check)**  
  Watches for `EventType.ADVERTISEMENT` and announces newly heard repeaters. Compares the first bytes of the public key against your known set to flag potential duplicate/overlapping IDs, and forwards details to Discord.  
  → [`Automations/MeshCore-Bot-NewRepeater.md`](Automations/MeshCore-Bot-NewRepeater.md)

- **Forward Channel #testing → Discord**  
  Listens to Channel 1 (`#testing`) traffic and relays messages to a Discord channel/thread via your HA notify service — handy for city-wide RX visibility during deployments.  
  → [`Automations/MeshCore-Bot-FWD-Testing2Discord.md`](Automations/MeshCore-Bot-FWD-Testing2Discord.md)

- **Ping / Pong (with Path + Hops)**  
  A lightweight bot command: users send `!ping` and the bot replies with `pong`, including hop count and the RF path. Uses a companion RX-log caching automation to extract and store path bytes between events.  
  → [`Automations/MeshCore-Bot-Ping.md`](Automations/MeshCore-Bot-Ping.md)

- **Advert Counter & Hourly Graph**  
  Increments a counter on every `EventType.ADVERTISEMENT` received, resets it hourly, and graphs the totals using ApexCharts for 48-hour rolling visibility of network activity.  
  → [`Automations/MeshCore-Bot-CountAdverts.md`](Automations/MeshCore-Bot-CountAdverts.md)