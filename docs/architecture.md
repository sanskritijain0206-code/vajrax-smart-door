# VAJRAX Smart Door — System & Technical Architecture Specification

**Document Version:** 1.0.0
**Project:** VAJRAX Smart Door — Robotics Laboratory Access Control System
**Status:** Active Development / Architecture Baseline
**Target Repository:** `vajrax-smart-door`

---

## 1. Architecture Overview

The **VAJRAX Smart Door** system is an edge-centric, multi-modal access-control architecture designed specifically for a robotics research laboratory. In such an environment, researchers and engineers working at distributed assembly benches require fast, low-latency physical door-release control without abandoning active experiments, handling delicate hardware, or depending on active internet service. Simultaneously, laboratory managers require remote, audited, authenticated cloud ingress triggers and status monitoring.

To satisfy these contrasting operational demands without compromise, the system implements a bifurcated dual-path architecture:
1. **The Local Access Path (Validated & Working):** A direct, peer-to-peer radio frequency link utilizing the connectionless **ESP-NOW** protocol operating on the 2.4 GHz ISM band. Workstation Table Nodes (Table 1, Table 2, Table 3) communicate directly with the Door Controller Node via ESP-NOW unicast, actuating a relay interface momentarily upon a physical button press completely independent of local routers, gateways, or the internet.
2. **The Remote Access Path (Planned / Development Stage):** A cloud-mediated control loop utilizing **Google Firebase** (Authentication and Realtime Database) bridging cross-platform client applications (**React Native / Expo** on mobile and **React / Vite** on web) to the Door Controller Node over standard 802.11 b/g/n Wi-Fi.

The system acts strictly as an **authorized auxiliary door-release trigger**. It interfaces with the laboratory's door release circuit (such as an existing authorized door-release input/circuit or access-control power supply trigger) via a relay contact. The system generates a controlled momentary release signal; it does not directly drive or power lock motor coils, and it does not bypass or disable existing laboratory fingerprint scanners, card readers, or physical deadbolts.

---

## 2. Design Goals

The VAJRAX Smart Door architecture is governed by five foundational engineering principles:

- **DG-1: Local Autonomy & Zero WAN Dependency:** Local door actuation from workstation tables must function unconditionally, even during total local router failure, Wi-Fi disconnect, or external internet outages.
- **DG-2: Low-Latency Local Actuation:** Local actuation should provide a fast and consistent response suitable for laboratory door-release operation. Actual end-to-end latency will be measured and documented during system validation.
- **DG-3: Defense-in-Depth Cloud Security:** Remote door actuation must enforce account authentication plus an independent numeric Door PIN, and strict Role-Based Access Control (RBAC).
- **DG-4: Safe Electrical Default:** Power interruption, firmware crash, brownout, or reboot should leave the relay de-energized and should not intentionally trigger the door-release circuit. The physical lock behavior depends on the existing laboratory access-control hardware and must be validated separately.
- **DG-5: Radio Coexistence & Shared-Phy Alignment:** The central edge controller must maintain concurrent operation between connectionless ESP-NOW frame reception and connection-oriented 802.11 b/g/n station infrastructure mode by strictly matching operational RF channels.

---

## 3. High-Level System Architecture

The system decouples local physical bench triggers from remote application clients while converging both control vectors onto the Door NodeMCU as the single authoritative edge gateway.

### 3.1 Architectural Diagram

```mermaid
flowchart TD
    %% Styling Definitions
    classDef client fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef cloud fill:#0f172a,stroke:#f59e0b,stroke-width:2px,color:#f8fafc;
    classDef edge fill:#1e1e2e,stroke:#a855f7,stroke-width:2px,color:#f8fafc;
    classDef local fill:#182234,stroke:#10b981,stroke-width:2px,color:#f8fafc;
    classDef hardware fill:#311515,stroke:#ef4444,stroke-width:2px,color:#f8fafc;

    subgraph REMOTE_PATH["REMOTE COMMUNICATION PATH (Planned Cloud Gateway)"]
        subgraph CLIENTS["Authorized Client Applications"]
            MOB["Mobile Application<br/>(React Native + Expo)"]:::client
            WEB["Web Dashboard<br/>(React + Vite)"]:::client
        end

        subgraph CLOUD["Cloud Infrastructure (Google Firebase)"]
            F_AUTH["Firebase Authentication<br/>(Identity & JWT)"]:::cloud
            F_RTDB["Firebase Realtime Database<br/>(State, Commands, Temporary Access & Activity)"]:::cloud
        end
    end

    subgraph LOCAL_PATH["LOCAL COMMUNICATION PATH (Validated Peer-to-Peer)"]
        subgraph TABLE_NODES["Workstation Table Controllers"]
            T1["Table 1 NodeMCU<br/>(ESP8266 + Physical Button)"]:::local
            T2["Table 2 NodeMCU<br/>(ESP8266 + Physical Button)"]:::local
            T3["Table 3 NodeMCU<br/>(ESP8266 + Physical Button)"]:::local
        end
    end

    subgraph EDGE_LAYER["CENTRAL EDGE CONTROLLER"]
        DOOR_MCU["Door NodeMCU Controller<br/>(ESP8266 Edge Core)<br/>[ESP-NOW Receiver + Wi-Fi Station]"]:::edge
    end

    subgraph ACTUATION_LAYER["ISOLATED ACTUATION INTERFACE"]
        RELAY["5V Relay Module<br/>(Momentary Dry Contact N.O.)"]:::hardware
        DOOR_CIRCUIT["Existing Laboratory<br/>Door-Release Circuit"]:::hardware
    end

    %% Remote Path Connections
    MOB -->|"HTTPS / Auth"| F_AUTH
    WEB -->|"HTTPS / Auth"| F_AUTH
    MOB -->|"HTTPS / Command + PIN"| F_RTDB
    WEB -->|"HTTPS / Command + PIN"| F_RTDB
    F_RTDB <===>|"Wi-Fi (Channel 1)<br/>HTTPS / Firebase RTDB"| DOOR_MCU

    %% Local Path Connections
    T1 ==>|"ESP-NOW Unicast (Channel 1)<br/>Payload: TABLE_1_OPEN"| DOOR_MCU
    T2 ==>|"ESP-NOW Unicast (Channel 1)<br/>Payload: TABLE_2_OPEN"| DOOR_MCU
    T3 ==>|"ESP-NOW Unicast (Channel 1)<br/>Payload: TABLE_3_OPEN"| DOOR_MCU

    %% Edge to Actuation Connections
    DOOR_MCU -->|"GPIO Pulse Signal<br/>(500 ms Momentary Pulse)"| RELAY
    RELAY -->|"Dry Contact Closure<br/>(N.O. to COM)"| DOOR_CIRCUIT
```

### 3.2 Path Differentiation

| Architectural Property | Local Path (`LOCAL`) | Remote Path (`REMOTE`) |
|---|---|---|
| **Origin / Ingress** | Table 1, Table 2, Table 3 push buttons | Mobile App (Expo) / Web Dashboard (Vite) |
| **Protocol** | ESP-NOW (2.4 GHz raw MAC layer frames) | HTTPS / Firebase Realtime Database communication |
| **Intermediary Broker** | None (direct peer-to-peer RF) | Firebase Authentication & Realtime Database |
| **Network Reliance** | Zero. Independent of router, AP, and WAN. | Requires active Wi-Fi AP and internet uplink. |
| **Authentication & Authorization** | MAC address recognition & hardcoded payload validation | Firebase user session token + secondary Door PIN + Time-bounded temporary access grant (when applicable) |
| **Activity Logging Boundary** | **Zero logging.** Physical button events are purely local and strictly excluded from Firebase. | **Targeted logging.** Remote release requests initiated under temporary access grants are recorded in Firebase. |
| **Primary Operational Role** | Immediate lab egress/ingress convenience | Audited administrative, standard, and time-bounded temporary access |
| **Deployment Status** | **Implemented & Bench Validated** | **Planned / Under Development** |

---

## 4. Local Communication Architecture

The local communication architecture provides reliable, low-latency door actuation directly from physical workbenches to the Door NodeMCU.

```
+------------------+         +-------------------+         +------------------+
|  Table 1 Button  |         |  Table 2 Button   |         |  Table 3 Button  |
+--------+---------+         +---------+---------+         +--------+---------+
         | Momentary                   | Momentary                  | Momentary
         | Contact                     | Contact                    | Contact
         v                             v                            v
+--------+---------+         +---------+---------+         +--------+---------+
|  Table 1 NodeMCU |         |  Table 2 NodeMCU  |         |  Table 3 NodeMCU |
+--------+---------+         +---------+---------+         +--------+---------+
         |                             |                            |
         | ESP-NOW (Ch 1)              | ESP-NOW (Ch 1)             | ESP-NOW (Ch 1)
         | "TABLE_1_OPEN"              | "TABLE_2_OPEN"             | "TABLE_3_OPEN"
         +-----------------------------+----------------------------+
                                       |
                                       v
                        +--------------+--------------+
                        |      Door NodeMCU Core      |
                        | (ESP-NOW Receive Callback)  |
                        +--------------+--------------+
                                       |
                                       v
                        +--------------+--------------+
                        |   Command Identification &  |
                        |      Payload Validation     |
                        +--------------+--------------+
                                       | Valid command matched
                                       v
                        +--------------+--------------+
                        |     Relay GPIO Energized    |
                        |    (500 ms Momentary Pulse) |
                        +--------------+--------------+
                                       |
                                       v
                        +--------------+--------------+
                        |     Relay Dry Contact       |
                        |   Momentary Door Release    |
                        +-----------------------------+
```

### 4.1 Physical Button Input & Debouncing
Each table controller (Table 1, Table 2, Table 3) monitors a normally open (NO) mechanical tactile push button wired between a designated GPIO pin and Ground.
- The GPIO is configured with an internal pull-up resistor (`INPUT_PULLUP`).
- Active button depression pulls the logic level to `LOW`.
- Firmware implements software debouncing requiring the pin to remain in a stable `LOW` state for a minimum of 50 ms before registering a valid actuation event. This eliminates false double-triggers caused by mechanical contact chatter.

### 4.2 ESP-NOW Frame Construction & Transmission
ESP-NOW is a connectionless, vendor-specific communication protocol developed by Espressif operating on the 2.4 GHz physical layer (OSI Layers 1 and 2):
- Upon a validated button depression, the Table NodeMCU packages a fixed ASCII command string into an ESP-NOW packet payload.
- The transmission destination is configured to the Door NodeMCU's specific registered MAC address (unicast).
- The transmission executes synchronously without requiring handshakes, network association, or DHCP IP assignment. Total RF on-air transmission time is typically under 5 ms.

### 4.3 Confirmed Local Command Set
The firmware across the three table nodes transmits the following immutable, confirmed command strings:

| Origin Node | Physical Hardware | Confirmed Command String | Packet Size |
|---|---|---|---|
| **Table 1** | ESP8266 NodeMCU + Button 1 | `TABLE_1_OPEN` | 12 Bytes |
| **Table 2** | ESP8266 NodeMCU + Button 2 | `TABLE_2_OPEN` | 12 Bytes |
| **Table 3** | ESP8266 NodeMCU + Button 3 | `TABLE_3_OPEN` | 12 Bytes |

### 4.4 Door NodeMCU Reception & Command Identification
The Door NodeMCU registers an asynchronous receive callback routine (`esp_now_register_recv_cb`).
1. When an incoming 2.4 GHz RF frame matching the ESP-NOW protocol signature arrives on Channel 1, the hardware MAC interrupt fires.
2. The callback extracts the sender's MAC address and byte payload.
3. The payload is checked against known command signatures:
   ```cpp
   if (strcmp(incomingPayload, "TABLE_1_OPEN") == 0) {
       triggerDoorRelease("TABLE_1");
   } else if (strcmp(incomingPayload, "TABLE_2_OPEN") == 0) {
       triggerDoorRelease("TABLE_2");
   } else if (strcmp(incomingPayload, "TABLE_3_OPEN") == 0) {
       triggerDoorRelease("TABLE_3");
   }
   ```
4. If the payload does not strictly match one of the three recognized commands, the frame is discarded immediately.

### 4.5 Momentary Relay Activation Behavior
- Upon command validation, the Door NodeMCU transitions its relay control GPIO from its idle state to its active state.
- A non-blocking timer (using `millis()` or an asynchronous timer ticker) is initiated for the configured pulse duration of approximately **500 ms**.
- While active, the relay dry contacts (Normally Open to Common) close, completing the laboratory door-release circuit.
- After 500 ms, the timer expires and the GPIO transitions back to the idle de-energized state, breaking the contact.
- Subsequent button presses received while the relay is actively pulsing are ignored or re-trigger the pulse window without causing relay contact jitter.

---

## 5. Remote Communication & Temporary Access Architecture

The remote communication architecture enables authenticated laboratory personnel and authorized temporary guests to trigger door release and inspect device telemetry without being physically present at a workbench.

> [!NOTE]
> **Implementation Status:** The remote communication path and temporary access subsystem are currently in the **planned / development stage**. The software specifications, authorization models, and database schemas below define the authoritative architectural baseline.

### 5.1 Remote Architecture & Standard Ingress Flow

```
+-------------------------------------------------------------------------+
|                  Step 1: User Identity Authentication                   |
|                                                                         |
|  [ User on Mobile / Web ] ---> Firebase Authentication (Email/Password)  |
|                           <--- Returns Signed JSON Web Token (JWT)      |
+------------------------------------+------------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------------+
|                 Step 2: Dual-Layer Ingress Authorization                |
|                                                                         |
|  - Validates user role and authorization permissions                     |
|  - If temporary user: evaluates time-bounded grant validity             |
|  - [ Client Enters Door PIN ] ---> Independent numeric PIN verified     |
|    against secure reference / backend authorization rules               |
+------------------------------------+------------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------------+
|                  Step 3: Database Command Dispatch                      |
|                                                                         |
|  Client writes release command to Firebase Realtime Database:           |
|  PATH: /commands/doorRelease                                            |
|  PAYLOAD: { "uid": "...", "timestamp": ServerValue.TIMESTAMP, ... }     |
+------------------------------------+------------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------------+
|                 Step 4: Door NodeMCU Wi-Fi Ingestion                    |
|                                                                         |
|  Door NodeMCU (Connected to Lab Wi-Fi on Channel 1)                     |
|  Monitors /commands/doorRelease (transport finalized during development)|
|  - Validates /systemControl/remoteAccessEnabled == true                 |
|  - Validates timestamp freshness (command age < 10.0s)                  |
+------------------------------------+------------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------------+
|                 Step 5: Actuation & Command Settlement                  |
|                                                                         |
|  - Energizes Relay GPIO for approximately 500 ms momentary pulse        |
|  - Relay dry contacts momentarily trigger door-release circuit          |
|  - Acknowledges command record in /commands/doorRelease                 |
|  - If operating under temporary grant: appends record to                |
|    /temporaryAccessActivity/{accessGrantId}                             |
+-------------------------------------------------------------------------+
```

### 5.2 Dedicated Temporary-Access Authorization Flow

Temporary access provides controlled, time-bounded remote door-release permissions to non-permanent users (such as visiting researchers, delivery staff, or contractors like Rahul).

```
[ Authenticated User (e.g., Rahul) ]
                 |
                 v
[ 1. Temporary Access Grant Exists? ]
  - Checks /temporaryAccess for valid record matching recipientUid
                 | YES
                 v
[ 2. Server-Authoritative Time Check ]
  - Evaluates: serverTime >= startTime AND serverTime < endTime
                 | WITHIN WINDOW
                 v
[ 3. Grant Status Active & Not Revoked? ]
  - Evaluates: status != "REVOKED"
                 | NOT REVOKED
                 v
[ 4. Master Remote Access Enabled? ]
  - Evaluates: /systemControl/remoteAccessEnabled == true
                 | ENABLED
                 v
[ 5. Secondary Door PIN Verification ]
  - User enters numeric Door PIN -> Verified against secure reference
                 | VALID PIN
                 v
[ 6. Remote Release Request Dispatched ]
  - Writes to /commands/doorRelease
  - Records request under /temporaryAccessActivity/{accessGrantId}
                 |
                 v
[ 7. Door NodeMCU Wi-Fi Ingestion & Relay Pulse ]
  - Edge controller receives command over Channel 1 Wi-Fi link
  - Verifies command freshness (< 10s)
  - Energizes relay for approximately 500 ms momentary pulse
  - Release circuit momentarily triggered
```

### 5.3 Temporary Access Grant Model & Lifecycle

#### Permission Model
Temporary access is engineered as a time-bounded permission grant attached to a user account, rather than as a permanent global user role. A user retains their base account role (for example, Administrator or Standard User), while authorization for remote door-release requests is additionally governed by the validity of an associated temporary access grant.

#### Conceptual Grant Data Schema
Each temporary access grant is defined conceptually with the following fields:

| Field Name | Data Type | Description |
|---|---|---|
| `accessGrantId` | String | Unique identifier for the temporary access grant (e.g., `grant_abc123`). |
| `recipientUid` | String | Firebase Authentication UID of the user receiving access (e.g., Rahul). |
| `recipientEmail` | String | Email address of the recipient for human-readable administrative identification. |
| `grantedBy` | String | Firebase Authentication UID of the administrator who created the grant. |
| `doorId` | String | Target door identifier for access control scoping (e.g., `door_main_lab`). |
| `startTime` | Integer | UTC millisecond timestamp defining when the access window begins. |
| `endTime` | Integer | UTC millisecond timestamp defining when the access window terminates. |
| `status` | String | Lifecycle state: `PENDING`, `ACTIVE`, `EXPIRED`, or `REVOKED`. |
| `createdAt` | Integer | UTC millisecond timestamp when the grant was created. |

#### Lifecycle State Machine
```
   [ Administrator Creates Grant ]
                  |
                  v
             ( PENDING )  --- startTime > serverTime (Request denied)
                  |
                  | When serverTime >= startTime
                  v
             ( ACTIVE )   --- startTime <= serverTime < endTime (Request permitted)
                  |
        +---------+---------+
        |                   |
        | serverTime >=     | Administrator executes
        | endTime           | early revocation
        v                   v
   ( EXPIRED )         ( REVOKED )
   [Permanently]       [Permanently]
   [  Denied   ]       [  Denied   ]
```

- **`PENDING`:** Grant created with a future `startTime`. The user cannot execute release requests until `startTime` arrives.
- **`ACTIVE`:** Authoritative server time is within the window `[startTime, endTime)` and the grant has not been revoked. Ingress release requests are authorized.
- **`EXPIRED`:** Authoritative server time has reached or passed `endTime`. The grant is permanently invalid; requests are denied.
- **`REVOKED`:** Manually terminated early by an administrator. Authorization terminates immediately and permanently.

#### Server-Authoritative Time Enforcement vs. Display Status
1. **No Client Clock Reliance:** Authorization expiry must be evaluated using **server-authoritative time** (e.g., Firebase Realtime Database Security Rules server timestamp variables `now` or backend verification) and must **not** depend on the client phone or browser clock. A user altering their device clock cannot bypass window boundaries.
2. **Dynamic Rule Evaluation:** Firebase Security Rules evaluate `now < data.child('endTime').val()` dynamically at the exact instant a write is attempted. Therefore, authorization denies access **immediately after `endTime`**, even if a stored display `status` string in the database document has not yet been updated from `ACTIVE` to `EXPIRED`. The architecture does not claim an automatic real-time mutation of database strings without a dedicated background trigger.

### 5.4 Activity Boundary & Recording Policy

#### Local vs. Remote Event Boundary
The system establishes a strict separation between physical bench triggers and cloud activity logging:
- **Local ESP-NOW Events (Workstation Tables 1, 2, 3):** Physical button presses transmit directly to the Door NodeMCU over ESP-NOW and trigger the relay purely at the hardware layer. These local bench interactions are **strictly excluded** from Firebase temporary-access activity history.
- **Remote Temporary Ingress Events:** Remote release requests initiated under a temporary access grant are recorded under `/temporaryAccessActivity/{accessGrantId}` to provide full accountability and attribution.

#### Accurate Event Nomenclature (Absence of Door Position Sensor)
The system currently has **no physical door-position sensor** (e.g., magnetic reed switch or contact sensor). The controller energizes the relay dry contacts for ~500 ms to pulse the external release circuit; it has no empirical feedback confirming whether the door latch cleared, the frame physically swung open, or the door was held open.

Therefore, recorded audit events are formally documented as:
`"Remote release command requested/issued"`
rather than `"Door opened"`.

#### Conceptual Temporary Activity Log Schema
Every remote release request submitted under temporary access is logged with the following fields:

| Field Name | Data Type | Description |
|---|---|---|
| `requestId` | String | Unique identifier for the release request event. |
| `timestamp` | Integer | Server-authoritative UTC millisecond timestamp when the request was processed. |
| `user` | String | Identity of the requesting user (`uid` and email/displayName). |
| `accessGrantId` | String | ID of the temporary access grant under which the request was submitted. |
| `doorId` | String | Target door identifier. |
| `action` | String | Action constant: `"REMOTE_RELEASE_REQUESTED"`. |
| `method` | String | Ingress client platform: `"MOBILE_APP"` or `"WEB_DASHBOARD"`. |
| `status` | String | Request status: `"REMOTE_RELEASE_REQUESTED"`, `"REJECTED_EXPIRED"`, `"REJECTED_REVOKED"`, `"REJECTED_INVALID_PIN"`, or `"REJECTED_REMOTE_DISABLED"`. This status reflects command evaluation and does not assert that the physical door opened. |

## 6. Door NodeMCU Responsibilities

The Door NodeMCU functions as the central edge controller and physical bridge between the local RF domain, the cloud broker, and the door release hardware.

### 6.1 Responsibility Breakdown

```
+---------------------------------------------------------------------------+
|                          DOOR NODEMCU CONTROLLER                          |
|                                                                           |
|   +--------------------------+             +--------------------------+   |
|   |   ESP-NOW Radio Stack    |             |    Wi-Fi Radio Stack     |   |
|   |  - Channel 1 Receiver    |             |  - 802.11 b/g/n Station  |   |
|   |  - Table MAC Validation  |             |  - Channel 1 Matched     |   |
|   |  - Packet Identification |             |  - Firebase RTDB Client  |   |
|   +------------+-------------+             +------------+-------------+   |
|                |                                        |                 |
|                +-------------------+--------------------+                 |
|                                    |                                      |
|                                    v                                      |
|                     +-----------------------------+                       |
|                     |     Command Arbitrator &    |                       |
|                     |     Validation Engine       |                       |
|                     | - Safe pulse window         |                       |
|                     | - Remote lockout check      |                       |
|                     | - Timestamp expiry verify   |                       |
|                     +--------------+--------------+                       |
|                                    |                                      |
|                                    v                                      |
|                     +-----------------------------+                       |
|                     |    Isolated GPIO Actuation  |                       |
|                     |  - Glitch-free boot logic   |                       |
|                     |  - 500 ms Momentary Pulse   |                       |
|                     |  - Watchdog monitored       |                       |
|                     +-----------------------------+                       |
+---------------------------------------------------------------------------+
```

### 6.2 Implementation Status of Door Controller Responsibilities

| Responsibility | Architectural Function | Implementation Status |
|---|---|---|
| **ESP-NOW Receiver** | Listens on Channel 1 for incoming frames; triggers MAC interrupt callback. | **Implemented & Validated** |
| **Local Command Identification** | Evaluates payloads (`TABLE_1_OPEN`, `TABLE_2_OPEN`, `TABLE_3_OPEN`). | **Implemented & Validated** |
| **Relay Actuation Control** | Drives relay pin for approximately 500 ms with non-blocking timing. | **Implemented & Validated** |
| **Safe Relay Boot State** | The relay control output remains de-energized during cold boot, reset, and initialization. | **Implemented & Validated** |
| **Wi-Fi Station Association** | Associates with laboratory 2.4 GHz Access Point on Channel 1. | **Planned / In Development** |
| **Firebase Cloud Client** | Subscribes to `/commands` and `/systemControl` endpoints. | **Planned / In Development** |
| **Remote Command Validation** | Checks timestamp validity (< 10s) and lockout flags before firing relay. | **Planned / In Development** |
| **Device Heartbeat Publisher** | Periodically writes RSSI, uptime, and status to `/status/doorController`. | **Planned / In Development** |
| **Temporary Access Command Auditing** | Records remote release requests issued under temporary access grants to `/temporaryAccessActivity`. Local ESP-NOW events are strictly excluded. | **Planned / In Development** |

---

## 7. Table NodeMCU Responsibilities

The three Table NodeMCUs (Table 1, Table 2, Table 3) operate as lightweight, edge-of-workbench transmitters.

### 7.1 Architecture of Table Nodes
- **Dedicated Functionality:** Each table controller has a single deterministic role: sense the physical button depression, debounce the mechanical contacts, package the specific table command, and transmit it over ESP-NOW.
- **Zero Cloud Burden:** Table nodes do not load the Wi-Fi station stack, do not negotiate DHCP IP addresses, do not store cloud credentials, and do not make internet connections.
- **Fast Startup:** Table nodes do not require network association before servicing local button events and are designed for fast recovery after power restoration. Actual startup time will be measured during validation.

### 7.2 Functional Summary

| Table Node | Input Pin | Debounce Mechanism | Dedicated Payload | Destination |
|---|---|---|---|---|
| **Table 1 Controller** | GPIO Input (`INPUT_PULLUP`) | 50 ms software timer | `TABLE_1_OPEN` | Door NodeMCU MAC |
| **Table 2 Controller** | GPIO Input (`INPUT_PULLUP`) | 50 ms software timer | `TABLE_2_OPEN` | Door NodeMCU MAC |
| **Table 3 Controller** | GPIO Input (`INPUT_PULLUP`) | 50 ms software timer | `TABLE_3_OPEN` | Door NodeMCU MAC |

---

## 8. Communication Protocols

| Protocol | Purpose | Direction | Subsystem | Current / Planned Status |
|---|---|---|---|---|
| **ESP-NOW** | Low-latency, connectionless peer-to-peer unicast transmission of button actuation commands over 2.4 GHz raw RF frames. | Unicast from Table 1, 2, 3 to Door Controller Node MAC address. | Local Subsystem | **Currently Validated & Working** |
| **802.11 b/g/n Wi-Fi** | Local area wireless network connection linking Door NodeMCU to the lab router. Operates strictly on Channel 1. | Bi-directional between Door Controller Node and Lab Wi-Fi AP. | Edge-to-Cloud Subsystem | **Planned / Under Development** |
| **HTTPS (TLS 1.2/1.3)** | Secure transport protocol for user authentication, initial token exchange, and REST operations. | Bi-directional between Mobile/Web clients and Firebase Cloud. | Application-to-Cloud Subsystem | **Planned / Under Development** |
| **Firebase Realtime Database** | Bi-directional state synchronization, command queuing, telemetry streaming, temporary access grants, and temporary access activity auditing. The exact ESP8266-to-Firebase communication mechanism will be finalized during implementation. | Bi-directional between Firebase Cloud, Clients, and Door Controller. | Cloud Integration Subsystem | **Planned / Under Development** |
| **Firebase Authentication** | Identity management, credential verification, and JSON Web Token (JWT) issuance. | Inbound from Mobile/Web clients to Firebase Auth servers. | Security & Identity Subsystem | **Planned / Under Development** |

---

## 9. Network Architecture & Radio Coexistence

### 9.1 The Shared-Radio Constraint
The ESP8266 SoC contains a **single physical 2.4 GHz RF transceiver** and baseband processor shared between the Wi-Fi subsystem and the ESP-NOW subsystem. The hardware cannot tune to two different radio frequencies simultaneously.

```
+-------------------------------------------------------------------------+
|                  ESP8266 PHYSICAL RADIO COEXISTENCE                     |
|                                                                         |
|                +---------------------------------------+                |
|                |   Single 2.4 GHz Radio Transceiver    |                |
|                +-------------------+-------------------+                |
|                                    |                                    |
|                     Locked to: RF CHANNEL 1                             |
|                                    |                                    |
|         +--------------------------+--------------------------+         |
|         |                                                     |         |
|         v                                                     v         |
|  +--------------+                                      +--------------+ |
|  |   ESP-NOW    |                                      |  802.11 STA  | |
|  |  Subsystem   |                                      |  Subsystem   | |
|  +-------+------+                                      +-------+------+ |
|          |                                                     |        |
+----------|-----------------------------------------------------|--------+
           |                                                     |
           v                                                     v
+-----------------------+                             +-----------------------+
| Table 1, 2, 3 Nodes   |                             | Lab Wi-Fi Router / AP |
| (Unicast on Ch 1)     |                             | (Operating on Ch 1)   |
+-----------------------+                             +-----------------------+
```

### 9.2 Channel Compatibility & The Channel 1 Imperative
1. **Wi-Fi Infrastructure Dictation:** When an ESP8266 associates with a Wi-Fi Access Point (AP), its internal radio frequency automatically locks to the primary operating channel dictated by that AP's beacon.
2. **ESP-NOW Channel Lock:** While in station (`STA`) mode, any ESP-NOW packet sent or received by that ESP8266 will be processed **only** on the channel to which the radio is currently locked.
3. **Architectural Solution:** The laboratory Wi-Fi router is statically assigned to **Wi-Fi Channel 1** (2412 MHz). Consequently:
   - Table 1 NodeMCU is fixed to transmit ESP-NOW on **Channel 1**.
   - Table 2 NodeMCU is fixed to transmit ESP-NOW on **Channel 1**.
   - Table 3 NodeMCU is fixed to transmit ESP-NOW on **Channel 1**.
   - Door NodeMCU associates with the lab AP on **Channel 1**, allowing its ESP-NOW receiver to capture table frames concurrently without radio re-tuning or frame dropping.

---

## 10. Cloud Architecture

The cloud architecture utilizes Google Firebase to provide a serverless, highly responsive state machine and authentication barrier without requiring self-hosted web servers or complex database maintenance.

### 10.1 Realtime Database Schema Structure

```json
{
  "systemControl": {
    "remoteAccessEnabled": true,
    "pulseDurationMs": 500,
    "doorPinHash": "e2fc714c4727ee9395f324cd2e7f331f"
  },
  "status": {
    "doorController": {
      "online": true,
      "lastSeen": 1718000000000,
      "relayState": "IDLE",
      "uptimeSeconds": 86400,
      "wifiRSSI": -58,
      "channel": 1
    }
  },
  "commands": {
    "doorRelease": {
      "commandId": "cmd-8823-9912",
      "requestedBy": "user_uid_rahul_67890",
      "accessGrantId": "grant-rahul-20261015",
      "clientType": "MOBILE_APP",
      "timestamp": 1718030400000,
      "status": "PENDING"
    }
  },
  "temporaryAccess": {
    "grant-rahul-20261015": {
      "accessGrantId": "grant-rahul-20261015",
      "recipientUid": "user_uid_rahul_67890",
      "recipientEmail": "rahul@vajrax.lab",
      "grantedBy": "user_uid_12345",
      "doorId": "door_main_lab",
      "startTime": 1718028000000,
      "endTime": 1718042400000,
      "status": "ACTIVE",
      "createdAt": 1718025000000
    }
  },
  "temporaryAccessActivity": {
    "grant-rahul-20261015": {
      "req-9912": {
        "requestId": "req-9912",
        "timestamp": 1718030400000,
        "user": "user_uid_rahul_67890",
        "accessGrantId": "grant-rahul-20261015",
        "doorId": "door_main_lab",
        "action": "REMOTE_RELEASE_REQUESTED",
        "method": "MOBILE_APP",
        "status": "REMOTE_RELEASE_REQUESTED"
      }
    }
  },
  "users": {
    "user_uid_12345": {
      "email": "researcher@vajrax.lab",
      "displayName": "Lead Roboticist",
      "role": "admin",
      "isActive": true
    },
    "user_uid_rahul_67890": {
      "email": "rahul@vajrax.lab",
      "displayName": "Rahul (Guest Researcher)",
      "role": "user",
      "isActive": true
    }
  }
}
```

### 10.2 Architectural Cloud Boundaries
- **No Cloud Functions Required for Core Authorization:** The system core does not strictly require custom backend cloud functions. Authorization is enforced directly through Firebase Realtime Database Security Rules evaluating server-authoritative time (`now`), client token authentication, and data validation rules alongside edge microcontroller firmware logic.
- **Server-Authoritative Evaluation:** Access window validity and timestamp comparisons are evaluated strictly using server-side authoritative time (`now`). Authorization is completely independent of the client mobile or web device clock.
- **Immediate Expiry Without Stored Status Mutation:** Firebase authorization denies access immediately once `now >= endTime` dynamically, even if the stored display status string in the database has not yet been changed to `EXPIRED`.
- **Targeted Temporary Activity Logging Boundary:** Physical table-button events (Table 1, Table 2, Table 3 via ESP-NOW) are completely offline and are **never** stored in Firebase activity history. The `/temporaryAccessActivity` collection is dedicated specifically and exclusively to recording remote release requests issued under temporary access grants.
- **Command Lifecycle:** Commands in `/commands/doorRelease` follow an atomic lifecycle: `PENDING` -> `PROCESSING` -> `COMPLETED` (or pruned), preventing duplicate executions.

---

## 11. Security Architecture

The system enforces defense-in-depth across both physical radio interfaces and internet endpoints.

### 11.1 Security Controls Matrix

```
+---------------------------------------------------------------------------+
|                        LAYERED SECURITY ARCHITECTURE                      |
|                                                                           |
|   [ Layer 1: Physical Perimeter ]                                         |
|   - Existing fingerprint scanner & manual deadbolts remain authoritative  |
|   - System relay acts only as parallel momentary exit / release signal    |
|                                                                           |
|   [ Layer 2: Local Radio Security (ESP-NOW) ]                             |
|   - MAC address whitelisting on Door NodeMCU                              |
|   - Exact payload string validation (rejects invalid byte lengths)        |
|                                                                           |
|   [ Layer 3: Remote Identity Authentication (Firebase Auth) ]             |
|   - Email / Password credential verification with signed JWT session      |
|                                                                           |
|   [ Layer 4: Secondary Door PIN Verification ]                            |
|   - Independent 4-to-6 digit numeric PIN required before command dispatch |
|   - Client-side 3-strike rate-limiting lockout                            |
|                                                                           |
|   [ Layer 5: Role-Based & Time-Bounded Authorization ]                    |
|   - Admin vs. Standard User separation                                    |
|   - Temporary access grants evaluated against server-authoritative time   |
|   - Security Rules reject writes if now >= endTime or status == REVOKED   |
|   - Database rules block unauthorized writes to /systemControl            |
|                                                                           |
|   [ Layer 6: Administrative Emergency Lockout ]                           |
|   - Master remoteAccessEnabled toggle immediately freezes remote triggers |
|                                                                           |
|   [ Layer 7: Replay & Stale Command Protection ]                          |
|   - Microcontroller discards cloud commands older than 10.0 seconds       |
+---------------------------------------------------------------------------+
```

### 11.2 Separation of Access Paths
Local table access and remote cloud access are logically separated. If a security anomaly or denial-of-service attack compromises the cloud credentials or internet link, setting `remoteAccessEnabled: false` isolates the door controller from remote commands while maintaining local bench button functionality.

---

## 12. Failure and Resilience Architecture

A fundamental architectural requirement of VAJRAX Smart Door is that **local operation must never fail when external dependencies fail**.

| Failure Event | Subsystem Impact | Expected Architectural Behavior | Recovery Mechanism |
|---|---|---|---|
| **Internet Outage** | Cloud unreachable; Mobile/Web release offline. | **Local table-button functionality remains available as long as the Door NodeMCU and ESP-NOW path remain powered and operational.** Door NodeMCU continues servicing ESP-NOW frames on Channel 1. | Planned behavior: the Door NodeMCU will retry the Firebase connection in the background with controlled backoff. |
| **Firebase Service Outage** | Remote commands cannot be read or dispatched. | Local table buttons continue normal operation. Client apps display a service degradation alert. | Planned behavior: firmware will re-establish the Firebase connection as soon as endpoints become reachable. |
| **Laboratory Wi-Fi AP Offline** | Door NodeMCU loses Wi-Fi station link. | Local ESP-NOW operates normally. Since AP is down, Door NodeMCU maintains Channel 1 reception. | Planned behavior: firmware will execute an auto-reconnect routine and reconnect to the AP as soon as the SSID beacon returns. |
| **Table NodeMCU Power Loss** | Specific table button non-functional. | Other two table nodes and all remote paths remain fully operational. | Hardware recovery upon power restoration. Actual startup time will be measured during validation. |
| **Door NodeMCU Reboot / Crash** | Momentary edge controller downtime during reboot. | Relay remains de-energized during boot (SAFE-001). Hardware Watchdog Timer reboots MCU. | Full re-initialization of ESP-NOW and Wi-Fi occurs upon reboot. Actual recovery time will be measured during validation. |
| **Stale / Delayed Cloud Command** | Queued command arrives minutes after network lag. | Door NodeMCU compares command timestamp against current time; commands > 10s old are silently discarded. | Prevents ghost unlock events after network restores. |
| **Relay Module Hardware Failure** | Contacts open or fail to switch. | Door release circuit is not intentionally energized by the failed relay. | Manual key or existing primary fingerprint scanner used for entry. |
| **Complete Lab AC Power Loss** | All controllers and door power drop. | The relay de-energizes. The resulting physical door state depends on the existing laboratory access-control hardware and must be validated separately. | Normal system boot sequence resumes automatically upon power restoration. |

---

## 13. Safety Architecture

1. **Controlled Momentary Release:** The currently validated Door NodeMCU firmware activates the relay for approximately 500 ms. Under no circumstances does the firmware leave the relay pin continuously energized.
2. **Safe Default Relay State:** The relay control output is assigned to a non-boot-strapping GPIO (e.g., `GPIO4` / `D2`). Upon power-up, brownout, reset, or unhandled crash, this pin remains high-impedance / low, keeping the relay de-energized and preventing an intentional trigger of the door-release circuit during reset, startup, brownout, or crash conditions.
3. **No Direct Motor Control:** The system does not supply raw drive currents or PWM waveforms to DC motors, linear actuators, or lock servos. The relay interfaces as a low-current dry contact closure to an existing access control power unit or authorized door-release input/circuit.
4. **Relay Electrical Interface:** The 5V single-channel relay board provides a controlled electrical contact interface between the ESP8266 logic output and the existing door-release circuit. Appropriate isolation between the microcontroller logic and the door release circuit should be verified during hardware integration testing.
5. **Preservation of Existing Security:** The system acts strictly in parallel with existing door security. It cannot override physical mechanical locks, deadbolts, or external biometric/fingerprint mechanisms required by lab safety policies.
6. **Bench Testing Protocol:** All firmware changes, radio channel configurations, and relay timing logic must be rigorously bench-tested on breadboards and test jigs before any electrical interface is connected to the physical laboratory door.

---

## 14. Data Flow

### 14.1 Flow A: Table 1 Button Press (Implemented & Validated)
```
[User at Table 1]
       |
       | Presses physical button
       v
[Table 1 NodeMCU]
       |
       | Debounces input (50 ms stable LOW)
       | Constructs ESP-NOW packet: "TABLE_1_OPEN"
       | Unicast to Door NodeMCU on Channel 1
       v
[Door NodeMCU]
       |
       | ESP-NOW receive interrupt fires on Channel 1
       | Verifies sender MAC address
       | Validates payload == "TABLE_1_OPEN"
       | Drives Relay GPIO HIGH
       | Starts 500 ms non-blocking timer
       v
[Relay Module]
       |
       | Closes Normally Open contact to Common
       v
[Laboratory Door Release Circuit]
       |
       | Momentary dry contact closure signals door release circuit
       v
[Door NodeMCU Timer Expiry]
       |
       | 500 ms elapsed -> Drives Relay GPIO LOW
       | Contact opens -> Relay returns to de-energized safe state
       | (Local event complete; zero records written to Firebase activity history)
```

### 14.2 Flow B: Table 2 Button Press (Implemented & Validated)
Identical to Flow A, with Table 2 NodeMCU detecting button depression and transmitting payload `TABLE_2_OPEN`.

### 14.3 Flow C: Table 3 Button Press (Implemented & Validated)
Identical to Flow A, with Table 3 NodeMCU detecting button depression and transmitting payload `TABLE_3_OPEN`.

### 14.4 Flow D: Remote Mobile Request (Planned Cloud Flow)
```
[Authorized User]
       |
       | Opens Expo Mobile App
       | Authenticates with Firebase Auth (JWT received)
       | Presses "Unlock Door" -> Prompts for Door PIN
       | Inputs 4-digit Door PIN
       v
[Mobile Application]
       |
       | Validates Door PIN against hash / rules
       | Issues authenticated write to /commands/doorRelease
       | { uid, timestamp: ServerValue.TIMESTAMP, status: "PENDING" }
       v
[Firebase Realtime Database]
       |
       | Evaluates database security rules (auth != null)
       | Commits record to /commands/doorRelease
       v
[Door NodeMCU]
       |
       | Receives update via Wi-Fi (Channel 1) stream
       | Checks /systemControl/remoteAccessEnabled == true
       | Checks (currentTime - timestamp) < 10000 ms
       | Drives Relay GPIO HIGH for approximately 500 ms
       | Settles command status to "COMPLETED"
       v
[Relay Module -> Door Release Circuit]
       |
       | Relay momentary dry contact closes for ~500 ms
       | Relay returns to de-energized safe state
```

### 14.5 Flow E: Remote Web Request (Planned Cloud Flow)
Identical to Flow D, originating from the React/Vite Web Dashboard running in a desktop browser.

### 14.6 Flow F: Temporary Access Remote Ingress Request (Planned Cloud Flow)
```
[Guest Researcher (e.g., Rahul)]
       |
       | Opens Expo Mobile App or Web Dashboard
       | Authenticates with Firebase Auth (JWT received)
       | App verifies active temporary access grant exists
       | Inputs 4-digit Door PIN
       v
[Client Application]
       |
       | Issues release request with accessGrantId: "grant-rahul-20261015"
       v
[Firebase Realtime Database & Security Rules]
       |
       | Evaluates server-authoritative time against grant:
       |   - now >= startTime (e.g., 14:00)
       |   - now < endTime (e.g., 18:00)
       |   - grant status != "REVOKED"
       |   - /systemControl/remoteAccessEnabled == true
       |   - Door PIN verification passes
       | Commits command to /commands/doorRelease
       | Immutably records audit entry under /temporaryAccessActivity/grant-rahul-20261015/req-9912:
       |   { requestId, timestamp, user, accessGrantId, doorId,
       |     action: "REMOTE_RELEASE_REQUESTED", method: "MOBILE_APP", status: "REMOTE_RELEASE_REQUESTED" }
       v
[Door NodeMCU]
       |
       | Receives command update via Wi-Fi (Channel 1)
       | Checks (currentTime - timestamp) < 10000 ms
       | Drives Relay GPIO HIGH for approximately 500 ms
       | Settles command status to "COMPLETED"
       v
[Relay Module -> Door Release Circuit]
       |
       | Relay momentary dry contact closes for ~500 ms
       | Relay returns to de-energized safe state
```

---

## 15. Component Responsibilities

| Component | Responsibility | Technology | Architectural Status |
|---|---|---|---|
| **Table 1 Controller** | Monitors physical push button, debounces input, transmits `TABLE_1_OPEN` via ESP-NOW. | ESP8266 NodeMCU, C++, `espnow.h` | **IMPLEMENTED & VALIDATED** |
| **Table 2 Controller** | Monitors physical push button, debounces input, transmits `TABLE_2_OPEN` via ESP-NOW. | ESP8266 NodeMCU, C++, `espnow.h` | **IMPLEMENTED & VALIDATED** |
| **Table 3 Controller** | Monitors physical push button, debounces input, transmits `TABLE_3_OPEN` via ESP-NOW. | ESP8266 NodeMCU, C++, `espnow.h` | **IMPLEMENTED & VALIDATED** |
| **Door Controller Node** | Receives ESP-NOW commands, validates payloads, manages Wi-Fi link, controls relay timing. | ESP8266 NodeMCU, C++, `espnow.h`, `ESP8266WiFi.h` | **IMPLEMENTED & VALIDATED (Local)**<br/>**PLANNED (Cloud Gateway)** |
| **Relay Module** | Provides controlled momentary dry contact closure for door release circuit. | 5V Relay Module | **IMPLEMENTED & VALIDATED** |
| **Door Release Circuit** | External commercial access control power supply or authorized door-release input/circuit. | Existing Lab Hardware | **IMPLEMENTED (Existing Lab Infrastructure)** |
| **Firebase Auth** | Manages user accounts, enforces password policies, issues signed JWT session tokens. | Google Firebase Authentication | **PLANNED** |
| **Firebase RTDB** | Synchronizes command queues, device status telemetry, temporary access grants, and temporary access activity auditing. | Google Firebase Realtime Database | **PLANNED** |
| **Mobile Application** | Provides authenticated remote door trigger, Door PIN input, and status monitoring on mobile devices. | React Native, Expo, TypeScript | **PLANNED** |
| **Web Dashboard** | Provides administrative management, temporary access grant management, audit inspection of temporary access activity, remote access lockout toggle, and telemetry. | React 18, Vite, TypeScript, Vanilla CSS | **PLANNED** |

---

## 16. Architectural Decisions

### ADR-01: ESP-NOW for Local Low-Latency Communication
- **Decision:** Use Espressif's proprietary ESP-NOW protocol instead of local Wi-Fi UDP/TCP sockets, BLE, or Zigbee for table-to-door communication.
- **Rationale:** ESP-NOW operates connectionless directly at the MAC layer. It requires no Wi-Fi handshake, no DHCP lease, and negligible transmission overhead. This provides fast, consistent button response times and completely decouples local bench operations from router availability.

### ADR-02: Wi-Fi and Firebase for Remote Access
- **Decision:** Utilize Wi-Fi (802.11 b/g/n) on the Door NodeMCU connecting to Google Firebase Realtime Database for the remote access pathway.
- **Rationale:** Firebase provides real-time database synchronization, built-in identity management, and automatic reconnection management. This eliminates the need to configure dynamic DNS, port forwarding, or self-hosted public VPS brokers on the laboratory router.

### ADR-03: Door NodeMCU as Unified Edge Gateway
- **Decision:** Consolidate both the ESP-NOW receiver and the cloud Wi-Fi client onto a single ESP8266 Door NodeMCU controller.
- **Rationale:** Minimizes hardware footprint and cost. Having the edge controller directly drive the relay eliminates the latency and failure modes of intermediate serial bridges or secondary microcontrollers.

### ADR-04: Distributed Standalone Table Controllers
- **Decision:** Deploy dedicated NodeMCUs at each table instead of running long physical wires from workbench buttons to the door.
- **Rationale:** Avoids hazardous physical cable runs across robotics testing floors, enables rapid workbench repositioning, and isolates electrical faults to individual benches.

### ADR-05: Complete Cloud Decoupling for Local Operation
- **Decision:** Local ESP-NOW actuation must never await, depend on, or block on cloud acknowledgment or network connectivity.
- **Rationale:** Safety and operational continuity. A network partition or internet service provider failure must never trap researchers or prevent intra-lab ingress.

### ADR-06: Controlled Momentary Relay Dry Contact
- **Decision:** Interface with the door mechanism purely as a momentary dry contact closure (approximately 500 ms), avoiding direct motor/solenoid power driving.
- **Rationale:** Safety, compliance, and hardware protection. Direct motor driving exposes microcontrollers to high back-EMF spikes and risks violating building fire safety codes. Interfacing to the existing authorized door-release circuit delegates lock mechanics to certified hardware.

### ADR-07: Preservation of Authoritative Security
- **Decision:** System acts strictly as an auxiliary release; existing biometric fingerprint scanners and manual key cylinders remain authoritative.
- **Rationale:** Guarantees that adding smart convenience features does not compromise the physical perimeter security posture of the research facility.

---

## 17. Current vs. Planned Architecture

```
+-------------------------------------------------------------------------------+
|                      VAJRAX SMART DOOR IMPLEMENTATION STATUS                   |
+---------------------------------------+---------------------------------------+
|          CURRENTLY VALIDATED          |                PLANNED                |
|         (Physical Bench Test)         |         (Under Active Dev)            |
+---------------------------------------+---------------------------------------+
| [x] Table 1 NodeMCU button sensing    | [ ] Firebase Authentication (Email/PW)|
| [x] Table 2 NodeMCU button sensing    | [ ] Firebase Realtime Database bridge |
| [x] Table 3 NodeMCU button sensing    | [ ] React Native / Expo mobile app    |
| [x] ESP-NOW transmission (Ch 1)       | [ ] React / Vite web dashboard        |
| [x] Confirmed payloads:               | [ ] Secondary Door PIN verification   |
|     - "TABLE_1_OPEN"                  | [ ] Remote unlock command handling    |
|     - "TABLE_2_OPEN"                  | [ ] Administrative remote lockout     |
|     - "TABLE_3_OPEN"                  | [ ] Temporary access grant management |
| [x] Door NodeMCU ESP-NOW reception    | [ ] Temporary access activity audit   |
| [x] 500 ms Momentary relay pulse      | [ ] Periodic device telemetry & RSSI  |
| [x] Safe relay boot state             | [ ] Web-based user management         |
| [x] Offline local execution           |                                       |
+---------------------------------------+---------------------------------------+
```

---

## 18. Future Scalability

While the current system architecture strictly focuses on 3 table nodes, 1 door controller, and Firebase integration, the architecture is designed to accommodate future expansion:

- **Additional Table Nodes:** The architecture can be extended to additional authorized table controllers, subject to ESP-NOW peer limits, RF conditions, firmware configuration, and validation testing.
- **Multiple Door Zones:** The Firebase schema is structured hierarchically (`/commands/{doorId}`, `/status/{doorId}`), allowing future deployment of multiple door controllers managed by a single web dashboard.
- **Over-the-Air (OTA) Updates:** Inclusion of signed Arduino OTA routines on the Door NodeMCU to allow firmware updates over the local Wi-Fi network without requiring physical USB access.
- **Richer Telemetry & Diagnostics:** Expansion of the edge heartbeat to report packet drop counts, average loop cycle times, and free heap memory.

---

## 19. Architecture Acceptance Criteria

The following acceptance criteria define verifiable, testable conditions for validating the technical architecture:

- **ARCH-001 (Local RF Communication):** The three workstation controllers (Table 1, Table 2, Table 3) shall reliably deliver their respective command frames (`TABLE_1_OPEN`, `TABLE_2_OPEN`, `TABLE_3_OPEN`) to the Door NodeMCU over ESP-NOW on Channel 1 across physical lab distances.
- **ARCH-002 (Momentary Actuation):** Reception of any valid table command by the Door NodeMCU shall trigger an active relay state for a duration of approximately 500 ms, after which the relay must return to the de-energized state.
- **ARCH-003 (Local WAN Independence):** The local communication path (Table Nodes -> Door NodeMCU -> Relay) shall function with zero measurable increase in latency when the Door NodeMCU's Wi-Fi connection and the laboratory's internet uplink are entirely disconnected.
- **ARCH-004 (Dual-Radio Coexistence):** The Door NodeMCU shall maintain uninterrupted ESP-NOW reception on Channel 1 while simultaneously maintaining an active 802.11 b/g/n Wi-Fi station connection to the laboratory AP on Channel 1.
- **ARCH-005 (Authenticated Remote Access):** Remote door-release commands dispatched through Firebase shall require both a valid authenticated user session (Firebase Auth) and correct Door PIN authorization before the Door NodeMCU energizes the relay.
- **ARCH-006 (Controlled Interface):** The relay output shall operate strictly as a controlled momentary dry contact closure interface for existing door release circuits. Appropriate electrical isolation between logic and door power circuits should be verified during hardware integration.
- **ARCH-007 (Remote Lockout Enforcement):** When `/systemControl/remoteAccessEnabled` is set to `false`, the Door NodeMCU shall reject all remote unlock commands while continuing to process local table button commands normally.
- **ARCH-008 (Safe Power Recovery):** During power-on, reset, or brownout conditions, the Door NodeMCU relay control pin shall remain de-energized without intentionally generating an active relay pulse.
- **ARCH-009 (Server-Authoritative Temporary Access Expiry):** Remote door-release requests submitted under a temporary access grant shall be evaluated against server-authoritative time (`now`). Any request submitted when `now < startTime` or `now >= endTime`, or when the grant status is `REVOKED`, shall be rejected immediately by Firebase Security Rules regardless of client device clock time or whether a stored display status field has transitioned to `EXPIRED`.
- **ARCH-010 (Activity Boundary & Local Isolation):** Remote release requests executed under a temporary access grant shall be immutably recorded in `/temporaryAccessActivity/{accessGrantId}` with caller identity, grant ID, and action `"REMOTE_RELEASE_REQUESTED"`. Local physical button presses (Table 1, Table 2, Table 3 via ESP-NOW) shall remain completely independent and shall never generate records in Firebase activity history.
