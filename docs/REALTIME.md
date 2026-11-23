# Realtime Connectivity, 2-Hour Broker Expiry & Resilience

[← Back to README](../README.md) · [← Architecture](./ARCHITECTURE.md) · [Storage & State →](./STORAGE.md)

Estimateys provides live multiplayer synchronization across desktop and mobile devices **without any dedicated backend server or database**. It combines a custom **OASIS MQTT 5.0 over WebSocket** client with the browser's native **`BroadcastChannel` API**.

---

## 1. Dual-Channel Transport & Broker Failover

Every published update fans out simultaneously over:
1. **MQTT 5.0 over WSS** (`src/shared/realtime/mqttClient.ts`) for cross-device internet sync across teams, with automatic **2-hour `Message Expiry Interval`** (`7,200 s`).
2. **`BroadcastChannel`** for instant zero-latency sync between tabs in the same browser.

```mermaid
flowchart LR
    subgraph DeviceA["Host / Player Browser A"]
        HookA["Room Hook\n(usePokerRoom / useRaceRoom /\nuseTrainRoom / useSwitcherRoom)"]
        MqttA["MqttWsClient\n(OASIS MQTT 5.0 Binary Encoder)"]
        BCA["BroadcastChannel"]
    end

    subgraph Brokers["Public WSS MQTT 5.0 Failover Pool"]
        direction TB
        Hive["1. wss://broker.hivemq.com:8884/mqtt\n(Primary — Retained + 7200s TTL verified)"]
        Mosq["2. wss://test.mosquitto.org:8081/mqtt\n(Failover 1 — Retained + 7200s TTL verified)"]
        EMQX["3. wss://broker.emqx.io:8084/mqtt\n(Failover 2 — Live pub/sub fallback)"]
        Hive -.->|"Timeout / Close"| Mosq
        Mosq -.->|"Timeout / Close"| EMQX
    end

    subgraph DeviceB["Teammate Browser B"]
        MqttB["MqttWsClient"]
        HookB["Room Hook + Merge Engine"]
    end

    subgraph TabA2["Second Tab on Browser A"]
        BCA2["BroadcastChannel"]
    end

    HookA --> MqttA
    HookA --> BCA
    BCA <-->|"Instant local tab sync"| BCA2
    MqttA <-->|"WSS TLS + LWT + 7200s TTL"| Hive
    Hive <-->|"Retained & Transient Pub/Sub"| MqttB
    MqttB --> HookB
```

### Built-in Transport Safeguards (`mqttClient.ts`)
- **Zero Dependencies**: Implements raw OASIS MQTT 5.0 `CONNECT` (protocol level `0x05`), `CONNACK`, `SUBSCRIBE`, `PUBLISH` (with Property `0x02` `Message Expiry Interval`), `PINGREQ`, `PINGRESP`, and `DISCONNECT` binary framing directly over standard `WebSocket`.
- **Zombie-Socket Watchdog**: Sends `PINGREQ` keep-alives every `17 s` (`keepAliveSec = 25 s`) and forces a reconnect/failover if `PINGRESP` is missed (critical for mobile Safari backgrounding).
- **Foreground Wake Re-Sync**: Listens to `visibilitychange`, `focus`, `pageshow`, and `online` events to immediately call `refreshSync()` (re-subscribing to pull fresh retained broker state when returning from another app on iOS/Android).

---

## 2. MQTT Topic Architecture Across All 4 Games

Every room is isolated under a namespaced topic root keyed by its 6-character room code:

| Game | Topic Root | Retained Topics (Auto-Expire After 2h) | Transient Topics (Never Stored) |
| :--- | :--- | :--- | :--- |
| **Planning Poker** | `estimateys-planning/v1/<CODE>` | `/doc` (room & rounds)<br>`/presence/<playerId>`<br>`/sealed/<roundId>/<playerId>` | `/events` (`START_REVEAL_COUNTDOWN`, `DOC_EVENT`, `LEAVE`) |
| **Fruit Race 3D** | `estimateys-race/v1/<CODE>` | `/doc` (race config, seed, placements)<br>`/presence/<playerId>` | `/events` (`CHEER`, `LEAVE`) |
| **Sydney Express 3D** | `estimateys-train/v1/<CODE>` | `/doc` (journey, stops, host)<br>`/presence/<playerId>`<br>`/ans/<journeyId>/<stopIdx>/<playerId>` | `/events` (`REACT`, `LEAVE`) |
| **Track Switcher 3D** | `estimateys-switch/v1/<CODE>` | `/doc` (options, seed, winner)<br>`/presence/<playerId>` | `/events` (`CHEER`, `LEAVE`) |

### Why Per-Player Subtopics Matter
- **Conflict-Free Concurrent Writes**: In Planning Poker (`/sealed/<roundId>/<playerId>`) and Sydney Express (`/ans/<journeyId>/<stopIdx>/<playerId>`), every player publishes to **their own retained subtopic**. Ten teammates submitting at the exact same millisecond never overwrite each other.
- **Hidden Until Reveal**:
  - Planning Poker obfuscates sealed votes (`sealVote`) while `round.status === 'voting'` and only unfolds them into `doc.rounds[i].revealedVotes` when the synchronized 3-2-1 countdown hits zero.
  - Sydney Express keeps per-stop answers hidden in UI until the Conductor transitions `status` from `answering` → `revealed`, folding all live answers into the room document for history and latecomers.

---

## 3. Deterministic Synchronized 3D Playback

Instead of streaming 60 FPS positions over the network (which causes jitter and high bandwidth), **Fruit Race 3D** and **Track Switcher 3D** broadcast a single **random seed + start timestamp**:

```mermaid
sequenceDiagram
    participant Host as Host Browser
    participant Broker as WSS MQTT 5.0 Broker
    participant Peer as Teammate Browser

    Host->>Host: Generate random seed + startedAt
    Host->>Broker: PUBLISH retained /doc (status="racing", seed, expiry=7200s)
    Broker->>Peer: Deliver /doc
    par Deterministic Local Simulation
        Host->>Host: Mulberry32(seed) -> identical race/switch plan
        Peer->>Peer: Mulberry32(seed) -> identical race/switch plan
    end
    Peer->>Broker: PUBLISH /events (CHEER emoji, retain=false)
    Broker->>Host: Deliver CHEER bubble
    Note over Host,Peer: Both devices animate 60 FPS locally and finish on the exact same winner!
```

---

## 4. Disconnect Handling, LWT & Automatic Host Failover

Every live room implements **four layers of disconnect protection**:

```mermaid
sequenceDiagram
    participant Host as Host (Alice)
    participant Broker as MQTT 5.0 Broker
    participant Peer as Earliest Peer (Bob)

    Note over Host,Broker: On CONNECT, Host registers LWT on presence/alice (empty retained payload)

    alt Clean Exit (Leave button / tab close)
        Host->>Host: Confirm leave modal / beforeunload warning
        Host->>Broker: PUBLISH retained /doc (hostId = Bob, expiry = 7200s)
        Host->>Broker: PUBLISH retained /presence/alice = "" (delete)
        Host->>Broker: PUBLISH /events (LEAVE, retain = false)
        Broker->>Peer: Bob immediately becomes Host
    else Sudden Crash / Wi-Fi Drop / Phone Killed
        Host--xBroker: TCP/WebSocket drops unexpectedly
        Broker->>Peer: Broker fires LWT -> /presence/alice = "" (deleted)
        Peer->>Peer: Detects host offline -> shows "Take Over as Host" banner
        Peer->>Broker: One-click claim -> PUBLISH /doc (hostId = Bob, version + 1)
    else Silent Ghost (Missed LWT)
        Peer->>Peer: Ghost sweeper prunes peers silent > 75s
    end
```

### Safeguard Summary
1. **Last Will & Testament (LWT)**: Registered at MQTT 5.0 `CONNECT` time (`willFlag = 1`, `willRetain = 1`) so if a browser crashes or loses power, the broker automatically deletes that player's retained `/presence/<playerId>` topic.
2. **bfcache-Safe `pagehide`**: Cleanly clears presence and triggers automatic host handover (`pickNextHost` / `pickNextTrainHost`) when closing a tab, while ignoring `evt.persisted === true` so iOS Safari back/forward cache doesn't falsely drop active players.
3. **Host Failover + Manual Takeover**: When a host exits cleanly, host controls transfer deterministically to the earliest-joined human peer (bots are excluded). If the host drops abruptly, teammates immediately see a **"Take Over as Host / Conductor / Signal Master"** banner (or after `32 s` of silence).
4. **Leave Warnings**: Native `beforeunload` prompt + in-app confirmation modal guard against accidental closes during active voting, racing, train travel, or track switching.

---

## 5. Data Retention, 2-Hour Broker Auto-Expiry & Scaling Path

Estimateys does **not** depend on any paid SaaS subscription, API key, or self-hosted backend. Instead, [`PUBLIC_MQTT_BROKERS`](../src/shared/realtime/mqttClient.ts) routes live traffic through free public MQTT 5.0 WebSocket brokers (`broker.hivemq.com`, `test.mosquitto.org`, `broker.emqx.io`).

### Exact Broker Data Lifecycle (2-Hour Auto-Expiry)

Every non-empty retained `PUBLISH` packet (`retain = true`) automatically stamps **MQTT 5.0 Property `0x02` (`Message Expiry Interval = 7200` seconds / 2 hours)**:

```mermaid
flowchart LR
    A["Active Room\n(Actions publish /doc, /sealed, /ans\nwith Message Expiry = 7,200 s)"] --> B{"Any room activity\nwithin 2 hours?"}
    B -->|"Yes (vote, edit, reveal, spin)"| A
    B -->|"No (2 hours idle)"| C["Broker Automatically Deletes\n/doc, /sealed/*, /ans/*"]
    C --> D["New visitor opening old link\nsees an empty room"]
    C --> E["Original participants keep\nprivate local history in\ntheir own browser localStorage"]
```

| Topic Type | `retain` | Broker Retention | Notes |
| :--- | :---: | :--- | :--- |
| **`/events`** *(cheers, countdowns, reactions, `LEAVE`)* | `false` | **0 seconds** (never stored) | Forwarded to live subscribers and discarded immediately. |
| **`/presence/<playerId>`** *(online status)* | `true` | **Until disconnect** (max 2h safety TTL) | Deleted immediately on Leave / `pagehide` or by LWT on crash; clients also ignore presence older than 2 hours. |
| **`/doc`**, **`/sealed/...`**, **`/ans/...`** *(room state, votes, replies)* | `true` | **Auto-deleted after 2 hours (`7,200 s`) of inactivity** | Every room update resets the 2-hour window. Empty `""` delete packets omit Property `0x02` so deletions apply immediately. |

### What This Means for Privacy & History
- **Sync window = 2 hours of inactivity**: While a team is actively playing (or up to 2 hours after the last update), anyone joining via URL or QR code receives the full current state from the broker.
- **After 2 hours, only participants have a private copy**: Once the broker expires the retained room messages, nothing remains on any public relay. Every participant who played still has their private game history saved locally in their browser (`localStorage`), including Planning Poker recent tables, Fruit Race 3D podiums, Sydney Express 3D Markdown exports, and Track Switcher 3D winners.
- **Resuming a Planning Poker table later**: Because original participants also keep a bounded local recovery snapshot (`estimateys_planning_room_<CODE>`), if an original participant re-opens a Planning Poker room link the next day, their browser restores it from their own `localStorage` and opens a fresh 2-hour live sync window for the team — while anyone who wasn't originally in the room cannot pull past data from the broker.

### Production Tradeoffs & 1-File Upgrade Path
| Aspect | Public MQTT 5.0 Relays | Mitigation in Estimateys |
| :--- | :--- | :--- |
| **Uptime / SLA** | Free community brokers have no formal SLA | 3-broker automatic failover (`HiveMQ → Mosquitto → EMQX`) + `PINGRESP` watchdog + local room snapshots |
| **Topic Namespace** | Shared public namespace keyed by 6-character room code | No accounts/passwords; votes obfuscated until reveal; strict **2-hour automatic broker deletion** |

Because all 4 games share the single [`MqttWsClient`](../src/shared/realtime/mqttClient.ts) abstraction, scaling to a private SLA-backed broker requires **zero changes to any game engine**:
- **Managed Serverless MQTT (Drop-In)**: Point `PUBLIC_MQTT_BROKERS` to a private **HiveMQ Cloud** or **EMQX Cloud Serverless** WSS endpoint (both support MQTT 5.0 `Message Expiry Interval` natively).
- **Edge WebSocket Alternative**: Swap the transport inside `mqttClient.ts` for **Cloudflare Workers + Durable Objects** if you later want server-side room auth or custom quotas.
