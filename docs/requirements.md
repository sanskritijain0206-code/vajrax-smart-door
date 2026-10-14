# VAJRAX Smart Door — Software & System Requirements Specification (SRS)

**Document Version:** 1.0.0  
**Project:** VAJRAX Smart Door — Robotics Laboratory Access Control System  
**Status:** Active Development / System Specification  
**Target Repository:** `vajrax-smart-door`  

---

## 1. Purpose

The purpose of this document is to define the definitive Software and System Requirements Specification (SRS) for the **VAJRAX Smart Door** access-control system. 

This specification establishes the functional, non-functional, hardware, software, security, safety, and architectural requirements for:
- The firmware operating on the distributed **ESP8266 NodeMCU** microcontrollers.
- The cloud backend infrastructure configured on **Google Firebase** (Authentication and Realtime Database).
- The cross-platform mobile client application built with **React Native and Expo**.
- The centralized administrative web management dashboard built with **React and Vite**.

This document is intended for embedded systems developers, software engineers, lab administrators, security auditors, and portfolio evaluators reviewing the engineering decisions, safety protocols, and implementation roadmap of the system.

---

## 2. Scope

### 2.1 In-Scope Subsystems
- **Door Controller Node:** A single ESP8266 NodeMCU interfacing with a 5V relay module to generate a controlled, momentary door-release signal. The controller concurrently executes an ESP-NOW receiver and an 802.11 b/g/n Wi-Fi client connected to Firebase Realtime Database.
- **Workstation Table Nodes:** Three independent ESP8266 NodeMCU transmitters positioned at Table 1, Table 2, and Table 3, each wired to a momentary physical push button.
- **Local Peer-to-Peer Bus:** Low-latency, connectionless ESP-NOW protocol operating on a synchronized 2.4 GHz channel between the three table nodes and the door controller.
- **Cloud Gateway & State Synchronization:** Bi-directional synchronization between the Door NodeMCU and Firebase Realtime Database over secured Wi-Fi (HTTPS/WSS).
- **User Authentication:** Email and password credential validation managed via Firebase Authentication.
- **Client Applications:** 
  - Cross-platform mobile application (React Native / Expo) for authorized handheld door release and status monitoring.
  - Desktop-optimized web dashboard (React / Vite) for laboratory telemetry, audit log inspection, and administrative configuration.
- **Dual-Layer Access Control:** App-level authentication coupled with an independent numeric Door PIN required before any remote unlock dispatch.
- **Telemetry and Auditing:** Real-time logging of actuation events (source, user, timestamp, status) and periodic controller heartbeat telemetry.

### 2.2 Out-of-Scope / Explicit System Boundaries
- **Direct Motor / Lock Actuation:** The system does **not** directly power or regulate motor brushes, lock servos, or raw electromagnetic coils. It provides an electrically isolated, momentary dry contact closure (via relay) to signal existing commercial access control hardware or door release circuits (e.g., Request-to-Exit / REX loop).
- **Bypassing Existing Perimeter Security:** The system acts strictly as an authorized parallel or auxiliary release mechanism. It must **not** bypass, short-circuit, or disable the laboratory's existing external security systems (e.g., biometric fingerprint readers, keycard panels, or physical keyed deadbolts).
- **Internet Connectivity on Table Nodes:** Table controllers do not connect to the local Wi-Fi router, cloud services, or external internet gateways. They operate exclusively via local ESP-NOW peer-to-peer radio.
- **Intercom / Audio-Video Streaming:** Surveillance cameras and two-way voice communication are not handled within this system.

---

## 3. Problem Statement

Modern robotics research laboratories operate in high-friction, dynamic environments characterized by distinct access bottlenecks:

1. **Intra-Laboratory Operational Disruption:** Researchers and robotics engineers stationed at hardware assembly benches (Table 1, Table 2, Table 3) frequently handle delicate electronics, running power supplies, test instruments, or tethered robotic platforms. Walking across the room to physically reach a wall switch or door handle whenever a recognized colleague or delivery arrives causes significant workflow interruption and safety risks.
2. **Lack of Remote Ingress Authorization:** Lab directors and authorized personnel arriving at the laboratory or assisting visitors from another part of the facility lack a secure, audited mechanism to release the door remotely without walking to the door.
3. **Fragility of Cloud-Only Solutions:** Commercial off-the-shelf smart locks and cloud-centric IoT boards fail completely when internet access is disrupted or local Wi-Fi router coverage degrades. A mission-critical lab door release cannot depend on external WAN availability for intra-room button actuation.
4. **Security and Access Accountability:** Uncontrolled wall buttons or generic keyfobs provide zero attribution. When an incident occurs, administrators cannot determine who triggered an unlock, which table initiated the signal, or whether unauthorized individuals accessed the space after hours.

VAJRAX Smart Door resolves these challenges through a dual-path architecture: ultra-resilient, offline-capable local wireless triggers (ESP-NOW) paired with an audited, role-restricted, cloud-managed remote gateway (Firebase).

---

## 4. System Objectives

- **Sub-200ms Local Latency:** Provide near-instantaneous door release upon a physical table button press via direct peer-to-peer RF communication.
- **Guaranteed WAN-Independence for Local Triggers:** Ensure physical table buttons remain 100% functional even during total Wi-Fi router disconnection or internet outages.
- **Defense-in-Depth Cloud Security:** Enforce dual-layer authorization (Identity Authentication + Independent Door PIN) and strict Role-Based Access Control (RBAC) on all remote trigger pathways.
- **Comprehensive Audit Trail:** Automatically capture and log all access events across all four release origins (Table 1, Table 2, Table 3, and Remote Clients) with millisecond-grade server timestamps and user attribution.
- **Immediate Administrative Control:** Empower administrators to instantly disable remote unlock capabilities globally (lockout mode) while maintaining local safety.
- **Fail-Secure Safety Architecture:** Guarantee that electrical faults, brownouts, firmware crashes, or reboots never leave the door latch energized or unlocked.

---

## 5. Stakeholders

| Stakeholder Role | Description | Core Interest |
|---|---|---|
| **Robotics Researchers / Lab Members** | Primary daily occupants working at Table 1, Table 2, and Table 3. | Fast, reliable, zero-effort door release from workbenches without interrupting active experiments. |
| **Lab Directors / Managers** | Responsible for lab security, equipment integrity, and safety compliance. | Granular access control, administrative override capabilities, and detailed historical audit logs. |
| **System / IT Administrators** | Maintain lab network infrastructure, Firebase tenancy, and user accounts. | Secure authentication, secret hygiene, role enforcement, and low network bandwidth overhead. |
| **Hardware & Firmware Engineers** | Maintain ESP8266 firmware, wiring schematics, and relay isolation. | Modular, well-tested C++ code, safe GPIO state management, and reliable RF channel synchronization. |
| **Software Developers** | Maintain the React Native mobile app and React/Vite dashboard. | Clean REST/WebSocket APIs, reactive state synchronization, and intuitive cross-platform UX. |

---

## 6. User Roles

### 6.1 Standard User
- **Description:** Authorized laboratory researchers, student engineers, and technicians granted routine door access privileges.
- **Capabilities:**
  - Authenticate to the mobile application or web dashboard via email and password.
  - View real-time door controller telemetry (Online / Offline status, relay pulse state).
  - Submit remote door-release requests by supplying the verified secondary Door PIN.
  - View their own personal remote unlock history.
- **Restrictions:**
  - Cannot access the administrative settings panel.
  - Cannot alter the global Remote Access Enable/Disable state.
  - Cannot modify or reset the Door PIN.
  - Cannot view, modify, or delete user accounts or system-wide audit logs.

### 6.2 Administrator
- **Description:** Laboratory managers, Principal Investigators (PIs), and designated security leads.
- **Capabilities:**
  - All Standard User privileges.
  - Toggle the global **Remote Access Enable / Disable** master switch (emergency remote lockout).
  - Configure, update, and rotate the shared secondary Door PIN.
  - Inspect, filter, and export the complete system-wide activity audit trail across all origins (Table 1, 2, 3, Mobile, Web).
  - Manage user accounts (invite new users, modify roles, suspend access, revoke tokens).
  - Monitor low-level device health telemetry (Wi-Fi RSSI, controller uptime, memory allocation, ESP-NOW channel status).

---

## 7. System Overview

### 7.1 Architecture Diagram

```
+-----------------------------------------------------------------------------+
|                               REMOTE PATH                                   |
|                                                                             |
|   +-----------------------+           +----------------------+              |
|   |   Mobile App (Expo)   |           |  Web Dashboard (Vite)|              |
|   +-----------+-----------+           +----------+-----------+              |
|               |                                  |                          |
|               +-----------------+----------------+                          |
|                                 | HTTPS / WSS                               |
|                                 v                                           |
|                   +---------------------------+                             |
|                   |   Firebase Cloud Platform |                             |
|                   |  - Authentication         |                             |
|                   |  - Realtime Database      |                             |
|                   +-------------+-------------+                             |
|                                 | Wi-Fi (2.4 GHz) / WSS                     |
|                                 v                                           |
+---------------------------------+-------------------------------------------+
                                  |
                                  v
+-----------------------------------------------------------------------------+
|                            DOOR CONTROLLER NODE                             |
|                                                                             |
|                      +--------------------+                                 |
|                      |  ESP8266 NodeMCU   |                                 |
|                      |  (Dual Radio Mode) |                                 |
|                      +----+----------+----+                                 |
|                           |          |                                      |
+---------------------------|----------|--------------------------------------+
                            |          |
            Controlled GPIO |          | ESP-NOW (2.4 GHz Direct RF)
             Release Signal |          | (Wi-Fi Channel Matched)
                            v          v
+-----------------------------+      +----------------------------------------+
|   RELAY MODULE & DOOR I/F   |      |               LOCAL PATH               |
|                             |      |                                        |
| +-------------------------+ |      |  +---------------+  +---------------+  |
| | 5V Optoisolated Relay   | |      |  | Table 1 Node  |  | Table 2 Node  |  |
| | (Dry Contact NC/COM/NO) | |      |  | (ESP8266 + SW)|  | (ESP8266 + SW)|  |
| +------------+------------+ |      |  +---------------+  +---------------+  |
|              |              |      |                                        |
|              v              |      |          +---------------+             |
| +-------------------------+ |      |          | Table 3 Node  |             |
| | Existing Lab Access     | |      |          | (ESP8266 + SW)|             |
| | Control / Release Loop  | |      |          +---------------+             |
| +-------------------------+ |      +----------------------------------------+
+-----------------------------+
```

### 7.2 Access Path Comparison

| Attribute | Local Access Path | Remote Access Path |
|---|---|---|
| **Trigger Origin** | Table 1, Table 2, Table 3 push buttons | React Native Mobile App or React Web Dashboard |
| **Transmission Protocol** | ESP-NOW (Peer-to-peer 2.4 GHz RF) | HTTPS / WebSockets via Firebase Realtime Database |
| **Intermediary Gateway** | None (direct microcontroller to microcontroller) | Internet router + Google Firebase cloud infrastructure |
| **WAN Dependency** | **Zero.** Operates completely offline. | Requires active internet and Wi-Fi connectivity. |
| **Authentication** | Hardware MAC whitelisting & fixed command string | Firebase Auth (User session) + Secondary Door PIN |
| **Average Latency** | < 100 ms | 600 ms – 1500 ms (network dependent) |
| **Current Status** | **Implemented and Bench-Validated** | **Planned Cloud Implementation** |

---

## 8. Functional Requirements

### 8.1 Local Button Access (Implemented & Validated)
- **FR-001:** Table 1 NodeMCU shall continuously sample the physical button input with hardware/software debouncing (minimum 50 ms debounce threshold).
- **FR-002:** Table 2 NodeMCU shall continuously sample the physical button input with hardware/software debouncing (minimum 50 ms debounce threshold).
- **FR-003:** Table 3 NodeMCU shall continuously sample the physical button input with hardware/software debouncing (minimum 50 ms debounce threshold).
- **FR-004:** Upon a validated button press, Table 1 NodeMCU shall transmit the exact ASCII payload string `TABLE_1_OPEN` via ESP-NOW directly to the Door NodeMCU's registered MAC address.
- **FR-005:** Upon a validated button press, Table 2 NodeMCU shall transmit the exact ASCII payload string `TABLE_2_OPEN` via ESP-NOW directly to the Door NodeMCU's registered MAC address.
- **FR-006:** Upon a validated button press, Table 3 NodeMCU shall transmit the exact ASCII payload string `TABLE_3_OPEN` via ESP-NOW directly to the Door NodeMCU's registered MAC address.
- **FR-007:** The Door NodeMCU shall operate its ESP-NOW receiver on the specific 2.4 GHz RF channel shared by the local network access point and table transmitters.
- **FR-008:** Upon receiving a valid payload (`TABLE_1_OPEN`, `TABLE_2_OPEN`, or `TABLE_3_OPEN`), the Door NodeMCU shall energize the relay pin for a momentary release duration of 2.0 seconds (± 200 ms), then automatically de-energize the pin.
- **FR-009:** The local button access sequence (FR-001 through FR-008) shall execute autonomously without requiring an active Wi-Fi association, router IP assignment, or internet connection.

### 8.2 Remote Access (Planned Cloud Functionality)
- **FR-010:** The Mobile and Web applications shall provide an authorized interactive door-release button accessible only after user login.
- **FR-011:** The Door NodeMCU shall maintain a real-time subscription or long-poll connection to the Firebase Realtime Database path `/commands/doorRelease`.
- **FR-012:** Upon receipt of a valid, authorized remote unlock command, the Door NodeMCU shall verify that the global remote access switch is enabled prior to actuation.
- **FR-013:** If verified, the Door NodeMCU shall actuate the relay for the designated momentary release duration (2.0 seconds) and immediately clear or mark the remote command as acknowledged.

### 8.3 Authentication (Planned Cloud Functionality)
- **FR-014:** The system shall authenticate all remote users via Firebase Authentication utilizing valid email and password credentials.
- **FR-015:** Client applications shall securely persist JSON Web Tokens (JWT) using platform-standard secure storage (Expo SecureStore on mobile, secure local storage/memory on web).
- **FR-016:** Client applications shall automatically reject unauthenticated API or database transactions and route unauthenticated users to the login screen.
- **FR-017:** The system shall support administrative password reset dispatch through standard Firebase Auth verification emails.

### 8.4 Door PIN Verification (Planned Cloud Functionality)
- **FR-018:** The client applications shall require the user to input a secondary 4-to-6 digit numeric Door PIN prior to submitting any remote door release request.
- **FR-019:** The Door PIN verification shall be evaluated independently of user account login credentials.
- **FR-020:** The system shall enforce PIN validation against an encrypted or cryptographically hashed reference stored in Firebase, or verified server-side/controller-side via database security rules.
- **FR-021:** The client application shall lock out further PIN entry attempts for 60 seconds after three consecutive incorrect PIN submissions.

### 8.5 User Management (Planned Cloud Functionality)
- **FR-022:** Administrators shall be able to invite, activate, deactivate, and remove user accounts via the administrative web dashboard.
- **FR-023:** Each user record in the database shall contain: `uid`, `email`, `displayName`, `role` (`standard` or `admin`), `createdAt`, and `isActive` status.
- **FR-024:** Deactivating a user record shall immediately prevent that user from successfully dispatching door-release commands.

### 8.6 Role-Based Permissions (Planned Cloud Functionality)
- **FR-025:** Firebase Realtime Database Security Rules shall enforce that only users with `role: "admin"` possess write authority to `/config`, `/systemControl`, and `/users`.
- **FR-026:** Standard users shall possess write authority exclusively to submit momentary commands under `/commands/doorRelease` appended with their own authenticated `uid`.
- **FR-027:** Client applications shall dynamically adapt navigation views, hiding administrative controls (user management, global lockout, PIN update) from Standard Users.

### 8.7 Activity Logging (Planned Cloud Functionality)
- **FR-028:** The Door NodeMCU and client applications shall record an immutable log record in Firebase under `/logs` for every release attempt.
- **FR-029:** Each log entry shall contain:
  - `id`: Unique push identifier.
  - `timestamp`: UTC millisecond server timestamp.
  - `source`: `TABLE_1`, `TABLE_2`, `TABLE_3`, `MOBILE_APP`, or `WEB_DASHBOARD`.
  - `triggeredBy`: User `uid` or table identifier string.
  - `action`: `DOOR_RELEASE`.
  - `status`: `SUCCESS`, `DENIED_INVALID_PIN`, `DENIED_REMOTE_DISABLED`, or `REJECTED_UNAUTHORIZED`.
- **FR-030:** The web dashboard shall allow administrators to view, search, and paginate through the historical activity log.

### 8.8 Device Status & Telemetry (Planned Cloud Functionality)
- **FR-031:** The Door NodeMCU shall publish a periodic heartbeat message to Firebase under `/status/doorController` every 30 seconds.
- **FR-032:** The heartbeat telemetry payload shall include:
  - `online`: Boolean indicator.
  - `lastSeen`: Unix epoch timestamp.
  - `relayState`: `IDLE` or `PULSING`.
  - `uptimeSeconds`: Cumulative microcontroller uptime.
  - `wifiRSSI`: Wi-Fi signal strength in dBm.
  - `channel`: Operational 2.4 GHz RF channel.
- **FR-033:** Client dashboards shall display an `OFFLINE` badge if the Door NodeMCU heartbeat is older than 90 seconds.

### 8.9 Remote Access Enable / Disable (Planned Cloud Functionality)
- **FR-034:** The system shall maintain a persistent boolean flag in Firebase under `/systemControl/remoteAccessEnabled`.
- **FR-035:** Only users with `role: "admin"` shall be permitted to toggle `remoteAccessEnabled`.
- **FR-036:** When `remoteAccessEnabled` is set to `false`, the Door NodeMCU shall immediately ignore and discard any remote unlock command.
- **FR-037:** Toggling `remoteAccessEnabled` to `false` shall **not** disable, impede, or alter the operation of the local table buttons (Table 1, 2, and 3 via ESP-NOW).

### 8.10 Command Handling & Idempotency (Planned Cloud Functionality)
- **FR-038:** Remote unlock requests shall be written with a client timestamp and a unique transaction ID.
- **FR-039:** The Door NodeMCU shall reject any command received with a timestamp older than 10.0 seconds to prevent replay attacks or stale command execution after network reconnect.
- **FR-040:** Upon processing a command, the Door NodeMCU shall atomically transition the command status to `PROCESSED` or delete the command node to guarantee single-execution idempotency.

---

## 9. Non-Functional Requirements

### 9.1 Security
- **NFR-001:** All remote communications between client applications, Firebase, and the Door NodeMCU shall be encrypted using Transport Layer Security (TLS 1.2 or TLS 1.3).
- **NFR-002:** Microcontroller firmware source code committed to version control shall contain zero plain-text production passwords, Wi-Fi credentials, Firebase API keys, or database URLs. All credentials shall be injected via untracked configuration headers or `.env` files.
- **NFR-003:** The Door NodeMCU shall filter all incoming ESP-NOW frames by sender MAC address against a hardcoded whitelist of Table 1, Table 2, and Table 3 physical addresses.

### 9.2 Reliability
- **NFR-004:** The Door NodeMCU firmware shall implement a hardware/software Watchdog Timer (ESP8266 WDT) configured to reboot the processor automatically in the event of an infinite loop or unhandled lockup exceeding 3.0 seconds.
- **NFR-005:** The relay control GPIO pin shall initialize to the de-energized state immediately upon boot, and shall remain de-energized during firmware flashing, brownouts, and resets.
- **NFR-006:** The system shall survive power cycles on any combination of nodes without requiring manual re-pairing or firmware intervention.

### 9.3 Availability
- **NFR-007:** Local table button access shall maintain a service availability of 99.9% within the physical bounds of the laboratory, independent of external internet outages.
- **NFR-008:** The Door NodeMCU shall implement an exponential backoff auto-reconnect routine for both Wi-Fi connectivity and Firebase streaming sessions whenever the connection drops.

### 9.4 Performance
- **NFR-009:** The latency from physical table button depression to relay closure shall not exceed 150 milliseconds under standard operating conditions.
- **NFR-010:** The end-to-end latency for remote door release (client button press -> Firebase -> Door NodeMCU relay pulse) shall be less than 1500 milliseconds over broadband Wi-Fi.
- **NFR-011:** The web dashboard and mobile application shall display real-time status updates within 500 milliseconds of state changes in Firebase.

### 9.5 Maintainability
- **NFR-012:** Firmware for Table 1, Table 2, Table 3, and Door Controller shall be structured into modular C++ source files with strict separation between radio protocols, relay control, and cloud synchronization.
- **NFR-013:** Mobile and web applications shall maintain shared data model definitions and TypeScript interfaces matching the Firebase Realtime Database schema.

### 9.6 Scalability
- **NFR-014:** The ESP-NOW peer management structure on the Door NodeMCU shall support expansion up to the ESP8266 hardware protocol limit (up to 20 registered peers).
- **NFR-015:** The Firebase Realtime Database schema shall support concurrent read telemetry connections for up to 50 active laboratory members without schema alterations.

### 9.7 Usability
- **NFR-016:** The mobile and web user interfaces shall present distinct visual states for `LOCKED` (neutral/idle), `UNLOCKING` (active pulse), `REMOTE_DISABLED` (warning), and `OFFLINE` (error).
- **NFR-017:** All interactive door actuation buttons on client applications shall require explicit confirmation (e.g., Door PIN modal or sliding release gesture) to eliminate accidental actuations.

---

## 10. Hardware Requirements

### 10.1 Microcontrollers
- **Door Controller:** 
  - 1 × ESP8266 NodeMCU v2 / v3 development board.
  - Tensilica L106 32-bit RISC core operating at 80 MHz (or 160 MHz).
  - 4 MB onboard SPI Flash memory.
  - Integrated 2.4 GHz PCB trace antenna.
- **Table Controllers:**
  - 3 × ESP8266 NodeMCU development boards (designated Table 1, Table 2, and Table 3).
  - Integrated 2.4 GHz PCB trace antenna.

### 10.2 Relays and Door Release Interface
- **Relay Module:** 1-channel 5V electromechanical relay module equipped with optocoupler isolation.
- **Contact Ratings:** Minimum 10A @ 250V AC / 30V DC SPDT contacts (Common, Normally Open, Normally Closed).
- **Logic Drive:** Driven by a designated Door NodeMCU GPIO (recommended: `GPIO4` / `D2` or `GPIO5` / `D1` to guarantee high-impedance safe boot behavior).
- **Interface Characterization:** The relay shall interface to the laboratory's door release circuit as an isolated, momentary contact closure (Normally Open dry contact). The relay signals the existing door power supply or access controller REX (Request to Exit) input. The system shall **not** directly drive electric strike or magnetic lock motor coils from logic power.

### 10.3 Push Buttons and Wiring
- **Buttons:** 3 × momentary, normally open (NO) mechanical tactile push buttons or industrial industrial dome push buttons.
- **Interface:** Connected between designated GPIO and GND on each respective Table NodeMCU, utilizing configured internal pull-up resistors (`INPUT_PULLUP`) or external 10kΩ pull-up resistors.

### 10.4 Power Supplies
- **Logic Power:** Regulated 5.0V DC (minimum 1.0A capacity per node) delivered via micro-USB or `Vin` pin on each NodeMCU board.
- **Electrical Isolation:** The 5V logic power supply for the NodeMCU controllers must be electrically isolated from any external 12V/24V power supply utilized by the laboratory door strike or magnetic lock.

---

## 11. Software Requirements

### 11.1 Firmware Specifications
- **Development Toolchain:** Arduino Core for ESP8266 (v3.0.0 or higher) / PlatformIO.
- **Core Libraries:**
  - `ESP8266WiFi.h`: For 802.11 b/g/n station mode and channel management.
  - `espnow.h`: For connectionless ESP-NOW frame generation, peer registration, and reception callbacks.
  - `FirebaseESP8266.h` (or lightweight REST client): For authenticated Firebase Realtime Database stream synchronization.
  - `ArduinoJson.h` (v6.x / v7.x): For structured telemetry serialization.
  - `Ticker.h` / Non-blocking `millis()` timing routines for asynchronous relay pulse resets.

### 11.2 Cloud Infrastructure (Google Firebase)
- **Firebase Authentication:** Configured for Email/Password provider.
- **Firebase Realtime Database:** Low-latency JSON database hosting `/commands`, `/status`, `/logs`, `/users`, and `/systemControl`.
- **Security Rules (`database.rules.json`):** Enforcing authentication, role checks via user lookup, schema validation, and write restrictions.

### 11.3 Mobile Application
- **Framework:** React Native (v0.72+) with Expo SDK (v49+).
- **Key Modules:**
  - `expo-secure-store` for encrypted token storage.
  - `@react-navigation` for role-protected navigation hierarchies.
  - Firebase JavaScript SDK (v9+ / v10+ modular) or REST/WebSocket clients.

### 11.4 Web Dashboard
- **Framework:** React 18+ bootstrapped with Vite.
- **Language / Styling:** TypeScript / Modern Semantic Vanilla CSS with design token variables (glassmorphic dark mode).
- **Routing:** React Router DOM (v6+).

---

## 12. Network Requirements

### 12.1 RF Band & Physical Frequencies
- All wireless operations (ESP-NOW and Wi-Fi) shall occur strictly within the **2.4 GHz ISM band** (2412 MHz – 2484 MHz).
- The ESP8266 hardware does not support 5.0 GHz networks; the host laboratory access point must broadcast a dedicated 2.4 GHz SSID.

### 12.2 ESP-NOW Channel Synchronization
- **Constraint:** The ESP8266 possesses a single physical 2.4 GHz radio transceiver. When connected to a Wi-Fi Access Point, its radio frequency is locked to the operating channel of that AP.
- **Requirement:** Table 1, Table 2, and Table 3 NodeMCUs must be configured to broadcast ESP-NOW packets on the **identical channel** assigned to the laboratory's 2.4 GHz Wi-Fi router (e.g., Channel 1, Channel 6, or Channel 11).

### 12.3 Cloud Protocols & Firewall Ports
- Outbound HTTPS traffic on port `443`.
- Outbound Secure WebSockets (WSS) traffic on port `443` for real-time Firebase listeners.
- No inbound open ports or public port-forwarding rules required on the local lab router.

---

## 13. Security Requirements

- **SEC-001:** The system shall enforce dual-layer access verification for remote actuation: a valid authenticated user session (Firebase Auth) combined with an independent, verified Door PIN.
- **SEC-002:** The system shall operate exclusively as an authorized parallel release mechanism and shall under no circumstances bypass, deactivate, or override existing laboratory fingerprint scanners, card readers, or manual lock cylinders.
- **SEC-003:** The Door NodeMCU shall validate the MAC address of any received ESP-NOW frame against an internal authorization list before evaluating payload contents.
- **SEC-004:** All production API keys, service credentials, Wi-Fi passwords, and PIN seeds shall be stored in external environment variables (`.env`) and excluded from Git commits via `.gitignore`.
- **SEC-005:** Firebase Realtime Database Security Rules shall validate that write operations to `/commands/doorRelease` contain valid authenticated user UIDs and are rejected if `/systemControl/remoteAccessEnabled` is `false`.
- **SEC-006:** Unlock commands dispatched through Firebase shall contain an epoch timestamp and shall be discarded by the Door NodeMCU if the difference between current time and command timestamp exceeds 10.0 seconds.
- **SEC-007:** Failed PIN verification attempts shall be logged to the audit trail with the associated user ID and client IP to detect brute-force activity.
- **SEC-008:** The Door NodeMCU firmware shall not expose any unauthenticated local web server, Telnet interface, or over-the-air (OTA) update endpoint without cryptographic signature verification.

---

## 14. Safety Requirements

- **SAFE-001 (Fail-Secure Default):** In the event of total electrical power loss, the door relay module shall revert to its Normally Open (NO) de-energized state, ensuring the door release signal is broken and the door remains secured by default.
- **SAFE-002 (Pulse Duration Hard Ceiling):** The firmware shall employ an asynchronous hardware/software safety timer enforcing an absolute maximum relay energization duration of 3.0 seconds, even if cloud communications hang or crash during an active pulse.
- **SAFE-003 (Boot-up Glitch Immunity):** The Door NodeMCU relay control output shall be assigned to a GPIO pin that does not output a high pulse or active state during the ESP8266 boot-loader sequence (recommended: `GPIO4` / Pin `D2`).
- **SAFE-004 (Emergency Egress Preservation):** The installation shall never inhibit or impede interior mechanical egress (e.g., panic exit push bars, mechanical thumb-turns, or manual bypass switches required by NFPA/OSHA life safety standards).
- **SAFE-005 (Galvanic / Optoelectronic Isolation):** The relay module must maintain complete optocoupler galvanic isolation between the low-voltage ESP8266 logic ground and the external door power supply conductors to prevent voltage spikes, ground loops, or inductive kickback.

---

## 15. Failure Scenarios and Mitigations

| # | Failure Mode | Impact | Automated System Mitigation | Recovery Action |
|---|---|---|---|---|
| **FS-01** | **Wi-Fi Router / ISP Outage** | Cloud connectivity lost; remote mobile/web release unavailable. | Door NodeMCU continues executing ESP-NOW receiver loop. Local table buttons continue releasing the door without interruption. | System automatically reconnects to Firebase when Wi-Fi/ISP recovers. |
| **FS-02** | **Door NodeMCU Power Interruption** | Door release circuit inactive; remote status shows offline. | Relay defaults to Normally Open (de-energized, door locked). When power returns, safe GPIO initialization prevents false trigger. | Microcontroller boots in < 2 seconds, re-attaches to ESP-NOW and Wi-Fi. |
| **FS-03** | **Table NodeMCU Power Loss / Failure** | That specific table's push button ceases to trigger door release. | Other two table nodes and all remote access paths continue normal operation. | Replace 5V power cable or reset button node. |
| **FS-04** | **Stale Cloud Command Execution** | Network lag causes delayed delivery of remote unlock command. | Command handling logic verifies command timestamp. Commands older than 10.0 seconds are discarded without relay actuation. | Client application notifies user of command timeout; prompt to retry. |
| **FS-05** | **Wi-Fi Channel Shift on AP** | AP switches 2.4 GHz channel; ESP-NOW packet reception breaks. | Door NodeMCU follows AP channel; table nodes operating on fixed channel lose contact. | Firmware channel configuration or auto-scan channel alignment recovery routine. |
| **FS-06** | **Relay Module Hardware Failure (Contact Weld)** | Relay remains continuously energized. | Circuit fused to open under excessive current draw; hardware watchdog alarm flags continuous active state. | Replace 1-channel relay module; fail-secure wiring isolates fault. |
| **FS-07** | **Brute-Force PIN Attack** | Unauthorized user attempts multiple PIN guesses on client app. | Client enforces 60-second lockout after 3 consecutive failures. Failed attempts logged to Firebase with user UID. | Administrator reviews security log and suspends offending account. |

---

## 16. Constraints

1. **Microcontroller Architecture:** The ESP8266 features a single core running at 80/160 MHz with approximately 80 KB of user RAM. All firmware code must be non-blocking and avoid heavy memory allocations.
2. **Single Shared Radio Hardware:** The ESP8266 radio must service both ESP-NOW frames and Wi-Fi 802.11 b/g/n packets concurrently. High Wi-Fi network congestion can cause minor ESP-NOW frame drop if packets are not retried.
3. **No Direct Motor Feedback:** The system monitors relay coil command status; it does not currently incorporate a physical magnetic reed switch to confirm whether the door frame physically separated or remained latched.
4. **Institutional Security Compliance:** All remote operations must remain subservient to the primary laboratory security mechanism (external fingerprint scanner); remote release is treated as an internal convenience release trigger.

---

## 17. Acceptance Criteria

The following testable criteria define the baseline requirements for project milestone verification and automated/bench testing:

### AC-01: Table 1 Local Trigger Verification (Implemented & Bench-Validated)
- **Given:** Table 1 NodeMCU and Door NodeMCU are powered on and configured to the same 2.4 GHz channel.
- **When:** The Table 1 physical button is depressed.
- **Then:** Table 1 transmits `TABLE_1_OPEN` via ESP-NOW, the Door NodeMCU receives the packet, energizes the relay within 150 ms, maintains energization for 2.0s ± 200ms, and de-energizes the relay.

### AC-02: Table 2 Local Trigger Verification (Implemented & Bench-Validated)
- **Given:** Table 2 NodeMCU and Door NodeMCU are powered on and configured to the same 2.4 GHz channel.
- **When:** The Table 2 physical button is depressed.
- **Then:** Table 2 transmits `TABLE_2_OPEN` via ESP-NOW, the Door NodeMCU receives the packet, energizes the relay within 150 ms, maintains energization for 2.0s ± 200ms, and de-energizes the relay.

### AC-03: Table 3 Local Trigger Verification (Implemented & Bench-Validated)
- **Given:** Table 3 NodeMCU and Door NodeMCU are powered on and configured to the same 2.4 GHz channel.
- **When:** The Table 3 physical button is depressed.
- **Then:** Table 3 transmits `TABLE_3_OPEN` via ESP-NOW, the Door NodeMCU receives the packet, energizes the relay within 150 ms, maintains energization for 2.0s ± 200ms, and de-energizes the relay.

### AC-04: Offline Local Autonomy
- **Given:** The laboratory Wi-Fi router is completely disconnected or powered down.
- **When:** Any table button (Table 1, Table 2, or Table 3) is pressed.
- **Then:** The Door NodeMCU successfully actuates the relay for 2.0 seconds without delay or error.

### AC-05: Unauthorized MAC Filtering
- **Given:** An unauthorized fourth ESP8266 transmits `TABLE_1_OPEN` to the Door NodeMCU.
- **When:** The frame is received by the Door NodeMCU.
- **Then:** The Door NodeMCU rejects the packet based on MAC mismatch and the relay remains completely de-energized.

### AC-06: Remote Unlock with Valid PIN (Planned Milestone)
- **Given:** An authenticated user with role `standard` is logged into the Mobile or Web application, and `remoteAccessEnabled` is `true`.
- **When:** The user triggers a door release and enters the correct Door PIN.
- **Then:** A command is dispatched to Firebase, received by the Door NodeMCU, the relay energizes for 2.0 seconds, and a log entry with `status: SUCCESS` is generated.

### AC-07: Remote Unlock with Invalid PIN (Planned Milestone)
- **Given:** An authenticated user is logged into the client application.
- **When:** The user enters an incorrect Door PIN.
- **Then:** The client rejects the command, no unlock request is dispatched to `/commands/doorRelease`, the relay does not actuate, and a failure record is logged.

### AC-08: Administrative Remote Lockout (Planned Milestone)
- **Given:** An administrator has set `/systemControl/remoteAccessEnabled` to `false`.
- **When:** A user attempts a remote door release with a valid PIN.
- **Then:** The Door NodeMCU ignores the remote command, the relay remains de-energized, while Table 1, 2, and 3 physical buttons continue to function normally.

### AC-09: Power-On Glitch Immunity (Safety Verification)
- **Given:** The Door NodeMCU is connected to the relay module and an oscilloscope / logic analyzer is attached to the relay control line.
- **When:** The Door NodeMCU is cold-booted, reset via hardware reset pin, or flashed over serial.
- **Then:** The relay control pin shall never transition to the active (energized) state at any point during boot.

### AC-10: Audit Log Record Integrity (Planned Milestone)
- **Given:** Any valid actuation occurs (local Table 1-3 or remote).
- **When:** The event completes and the Door NodeMCU is online.
- **Then:** A record appears in Firebase under `/logs` containing non-null values for `timestamp`, `source`, `triggeredBy`, `action`, and `status`.

---

## 18. Future Requirements

The following requirements represent planned evolutionary enhancements scheduled for subsequent system revisions once core cloud bench integration is complete:

- **FUT-001 (Captive Portal Provisioning):** The Door NodeMCU shall support an on-demand Wi-Fi captive portal (e.g., using `WiFiManager`) to allow laboratory administrators to configure local SSID and password credentials without re-flashing firmware.
- **FUT-002 (Cryptographic ESP-NOW Encryption):** Implement AES-128 payload encryption and rolling packet sequence counters for all table-to-door ESP-NOW transmissions to prevent local RF eavesdropping or replay packets.
- **FUT-003 (Offline Event Caching):** The Door NodeMCU shall allocate a ring buffer in onboard Flash (LittleFS) to store up to 500 local button actuation logs during internet outages, synchronizing buffered records with Firebase once connectivity is restored.
- **FUT-004 (Door Status Magnetic Sensor):** Integrate a physical magnetic reed switch on an available Door NodeMCU GPIO to detect and publish actual physical door position (`CLOSED`, `OPEN`, `HELD_OPEN_ALERT`).
- **FUT-005 (Biometric & RFID Panel Ingress Integration):** Support hardware UART / Wiegand protocol input from an external physical RFID badge reader or keypad mounted directly beside the door frame.
- **FUT-006 (Battery Backup & Brownout Telemetry):** Incorporate an uninterruptible lithium-ion battery backup circuit with ADC voltage monitoring to detect main power failures and alert administrators via push notification.
