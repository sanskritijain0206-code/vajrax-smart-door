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
- The temporary access management lifecycle and targeted remote-activity auditing subsystem.

This document is intended for embedded systems developers, software engineers, lab administrators, security auditors, and portfolio evaluators reviewing the engineering decisions, safety protocols, and implementation roadmap of the system.

---

## 2. Scope

### 2.1 In-Scope Subsystems
- **Door Controller Node:** A single ESP8266 NodeMCU interfacing with a 5V relay module to generate a controlled, momentary door-release signal (~500 ms). The controller concurrently executes an ESP-NOW receiver and an 802.11 b/g/n Wi-Fi client connected to Firebase Realtime Database.
- **Workstation Table Nodes:** Three independent ESP8266 NodeMCU transmitters positioned at Table 1, Table 2, and Table 3, each wired to a momentary physical push button.
- **Local Peer-to-Peer Bus:** Direct, connectionless ESP-NOW protocol operating on radio Channel 1 between the three table nodes and the door controller.
- **Cloud Gateway & State Synchronization:** Bi-directional synchronization between the Door NodeMCU and Firebase Realtime Database over secured Wi-Fi (HTTPS/TLS).
- **User Authentication & Multi-User Support:** Email and password credential validation managed via Firebase Authentication supporting multiple authorized laboratory personnel and administrative accounts.
- **Temporary Remote Access Management:** Administrator-managed time-bounded remote door-release access granted to specific users (e.g., visiting researchers or contractors) for defined time windows with server-authoritative enforcement.
- **Client Applications:**
  - Cross-platform mobile application (React Native / Expo) for authorized handheld door release and status monitoring.
  - Desktop-optimized web dashboard (React / Vite) for laboratory telemetry, temporary access management, and audit inspection.
- **Dual-Layer Remote Access Control:** App-level authentication coupled with an independent numeric Door PIN required before any remote release command dispatch.
- **Device Telemetry & Targeted Auditing:** Periodic controller heartbeat telemetry and targeted recording of remote release requests performed under temporary access grants.

### 2.2 Out-of-Scope / Explicit System Boundaries
- **Direct Motor / Lock Actuation:** The system does **not** directly power or regulate motor brushes, lock servos, or raw electromagnetic coils. It provides a momentary dry contact closure (via relay) to signal existing commercial access control hardware or door release circuits. The system does not assume a specific interface type (such as a REX input) until electrically verified.
- **Bypassing Existing Perimeter Security:** The system acts strictly as an authorized parallel or auxiliary release mechanism. It must **not** bypass, short-circuit, or disable the laboratory's existing external security systems (e.g., biometric fingerprint readers, keycard panels, or physical keyed deadbolts).
- **Physical Door Position Sensing:** The system currently has **no physical door-position sensor** (such as a magnetic reed switch). The system commands and monitors the electrical relay state; it does **not** detect whether the physical door was opened, closed, propped open, or remained latched.
- **General Logging of Local Events:** Physical table button presses (Table 1, Table 2, Table 3 via ESP-NOW) are purely local events and must **not** be stored in Firebase activity history. Local operation remains strictly decoupled from cloud logging.
- **Internet Connectivity on Table Nodes:** Table controllers do not connect to the local Wi-Fi router, cloud services, or external internet gateways. They operate exclusively via local ESP-NOW peer-to-peer radio.
- **Intercom / Audio-Video Streaming:** Surveillance cameras and two-way voice communication are not handled within this system.

---

## 3. Problem Statement

Modern robotics research laboratories operate in high-friction, dynamic environments characterized by distinct access bottlenecks:

1. **Intra-Laboratory Operational Disruption:** Researchers and robotics engineers stationed at hardware assembly benches (Table 1, Table 2, Table 3) frequently handle delicate electronics, running power supplies, test instruments, or tethered robotic platforms. Walking across the room to physically reach a door handle or wall switch whenever a recognized colleague or delivery arrives causes significant workflow interruption and safety risks.
2. **Lack of Controlled Remote Ingress:** Lab directors and authorized personnel arriving at the laboratory or assisting visitors from another part of the facility lack a secure, audited mechanism to release the door remotely without walking to the door.
3. **Fragility of Cloud-Only Solutions:** Commercial off-the-shelf smart locks and cloud-centric IoT boards fail completely when internet access is disrupted or local Wi-Fi router coverage degrades. A mission-critical lab door release cannot depend on external WAN availability for intra-room button actuation.
4. **Accountability for Remote and Guest Ingress:** Sharing master credentials, physical keys, or uncontrolled entry codes provides zero attribution and creates security vulnerabilities. Without structured temporary access grants, administrators cannot restrict guest access to defined time windows or audit remote release requests.

VAJRAX Smart Door resolves these challenges through a dual-path architecture: ultra-resilient, offline-capable local wireless triggers (ESP-NOW) paired with an audited, role-restricted, cloud-managed remote gateway (Firebase) supporting time-bounded temporary access.

---

## 4. System Objectives

- **Low-Latency Local Actuation:** Provide fast, consistent door release upon a physical table button press via direct peer-to-peer RF communication (to be empirically measured during validation).
- **Guaranteed WAN-Independence for Local Triggers:** Ensure physical table buttons remain 100% functional even during total Wi-Fi router disconnection or internet outages.
- **Defense-in-Depth Cloud Security:** Enforce dual-layer authorization (Identity Authentication + Independent Door PIN) and strict Role-Based Access Control (RBAC) on all remote trigger pathways.
- **Time-Bounded Temporary Access:** Empower administrators to grant temporary remote access to specific users for defined time windows with automated expiration, immediate revocation, and server-authoritative time enforcement.
- **Targeted Audit Trail for Temporary Ingress:** Record and attribute all remote release requests initiated under temporary access grants, without flooding cloud databases with local intra-lab button events.
- **Immediate Administrative Control:** Empower administrators to instantly disable remote unlock capabilities globally (lockout mode) while maintaining local safety.
- **Safe Default Relay State:** Ensure power interruption, reset, startup, brownout, or firmware crash leaves the relay de-energized without intentionally triggering the door-release circuit. Physical door lock behavior depends on existing facility hardware and must be validated separately.

---

## 5. Stakeholders

| Stakeholder Role | Description | Core Interest |
|---|---|---|
| **Robotics Researchers / Lab Members** | Primary daily occupants working at Table 1, Table 2, and Table 3. | Fast, reliable, zero-effort door release from workbenches without interrupting active experiments. |
| **Lab Directors / Managers** | Responsible for lab security, equipment integrity, and safety compliance. | Granular access control, administrative override capabilities, and detailed temporary-access audit logs. |
| **Temporary / Guest Users** | Visiting researchers, external collaborators (e.g., Rahul), or contractors. | Time-limited remote access during approved working hours via mobile or web client. |
| **System / IT Administrators** | Maintain lab network infrastructure, Firebase tenancy, and user accounts. | Secure authentication, secret hygiene, role enforcement, and minimal cloud bandwidth overhead. |
| **Hardware & Firmware Engineers** | Maintain ESP8266 firmware, wiring schematics, and relay isolation. | Modular, well-tested C++ code, safe GPIO state management, and reliable RF channel synchronization. |
| **Software Developers** | Maintain the React Native mobile app and React/Vite dashboard. | Clean REST/real-time APIs, reactive state synchronization, and intuitive cross-platform UX. |

---

## 6. User Roles

### 6.1 Standard User
- **Description:** Authorized permanent laboratory researchers, student engineers, and technicians granted routine door access privileges.
- **Capabilities:**
  - Authenticate to the mobile application or web dashboard via email and password.
  - View real-time door controller telemetry (Online / Offline status, relay pulse state).
  - Submit remote door-release requests by supplying the verified secondary Door PIN.
- **Restrictions:**
  - Cannot access the administrative settings panel.
  - Cannot alter the global Remote Access Enable/Disable state.
  - Cannot modify or reset the Door PIN.
  - Cannot create, modify, or revoke temporary access grants.
  - Cannot view administrative audit logs.

### 6.2 Administrator
- **Description:** Laboratory managers, Principal Investigators (PIs), and designated security leads.
- **Capabilities:**
  - All Standard User privileges.
  - Toggle the global **Remote Access Enable / Disable** master switch (emergency remote lockout).
  - Configure, update, and rotate the shared secondary Door PIN.
  - Create, view, manage, and revoke time-bounded temporary access grants.
  - Inspect, filter, and audit remote release requests submitted under temporary access grants.
  - Manage user accounts (invite new users, modify roles, suspend access, revoke tokens).
  - Monitor low-level device health telemetry (Wi-Fi RSSI, controller uptime, memory allocation, ESP-NOW channel status).

### 6.3 Temporary User
- **Description:** Visiting researchers, service contractors, or temporary collaborators (e.g., Rahul) granted time-bounded remote door access by an Administrator.
- **Capabilities:**
  - Authenticate to the mobile application or web dashboard during the granted access window.
  - Submit remote door-release requests within the configured start and end timestamps, subject to Door PIN verification.
  - View active grant status and remaining time window.
- **Restrictions:**
  - Strictly restricted to the administrator-configured time window (`startTime` to `endTime`).
  - Cannot request remote release prior to `startTime`, after `endTime`, or if revoked by an administrator.
  - Cannot access administrative controls or grant access to other individuals.
  - Every remote release request is immutably logged with attribution to their Temporary Access ID.

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
|                                 | HTTPS / TLS                               |
|                                 v                                           |
|                   +---------------------------+                             |
|                   |   Firebase Cloud Platform |                             |
|                   |  - Authentication         |                             |
|                   |  - Realtime Database      |                             |
|                   |    * /commands            |                             |
|                   |    * /temporaryAccess     |                             |
|                   |    * /temporaryActivity   |                             |
|                   |    * /systemControl       |                             |
|                   +-------------+-------------+                             |
|                                 | Wi-Fi (Channel 1) / TLS                   |
|                                 v                                           |
|                   +---------------------------+                             |
|                   | Laboratory Wi-Fi Router   |                             |
|                   +-------------+-------------+                             |
|                                 | 802.11 b/g/n (Channel 1)                  |
|                                 v                                           |
+---------------------------------+-------------------------------------------+
                                  |
                                  v
+-----------------------------------------------------------------------------+
|                            DOOR CONTROLLER NODE                             |
|                                                                             |
|                      +--------------------+                                 |
|                      |  ESP8266 NodeMCU   |                                 |
|                      |  (MAC Confirmed)   |                                 |
|                      +----+----------+----+                                 |
|                           |          |                                      |
+---------------------------|----------|--------------------------------------+
                            |          |
         GPIO D1 (~500 ms)  |          | ESP-NOW Unicast (Channel 1)
         Momentary Trigger  |          | (Local Workbench RF Link)
                            v          v
+-----------------------------+      +----------------------------------------+
|   RELAY MODULE & DOOR I/F   |      |               LOCAL PATH               |
|                             |      |                                        |
| +-------------------------+ |      |  +---------------+  +---------------+  |
| | 5V Single-Channel Relay | |      |  | Table 1 Node  |  | Table 2 Node  |  |
| | (Dry Contact NC/COM/NO) | |      |  | (D5 Input)    |  | (D5 Input)    |  |
| +------------+------------+ |      |  +---------------+  +---------------+  |
|              |              |      |                                        |
|              v              |      |          +---------------+             |
| +-------------------------+ |      |          | Table 3 Node  |             |
| | Existing Authorized Lab | |      |          | (D5 Input)    |             |
| | Door-Release Circuit    | |      |          +---------------+             |
| +-------------------------+ |      +----------------------------------------+
+-----------------------------+
```

### 7.2 Access Path Comparison

| Attribute | Local Access Path | Remote Access Path |
|---|---|---|
| **Trigger Origin** | Table 1, Table 2, Table 3 push buttons | React Native Mobile App or React Web Dashboard |
| **Transmission Protocol** | ESP-NOW Unicast (Channel 1) | HTTPS / WebSockets via Firebase Realtime Database |
| **Intermediary Gateway** | None (direct microcontroller to microcontroller) | Internet router + Google Firebase cloud infrastructure |
| **WAN Dependency** | **Zero.** Operates completely offline. | Requires active internet and Wi-Fi connectivity. |
| **Authentication** | Direct ESP-NOW unicast to confirmed Door MAC | Firebase Auth session + Secondary Door PIN |
| **Relay Pulse Duration** | ~500 ms momentary pulse (bench validated) | ~500 ms momentary pulse (planned) |
| **Activity Logging** | **None.** Local events are strictly excluded from Firebase. | Remote release requests under temporary access are recorded in Firebase. |
| **Physical Door State** | No physical sensor; relay actuation signal only. | No physical sensor; command issuance status only. |
| **Current Status** | **Implemented & Validated** | **Planned Cloud Implementation** |

---

## 8. Functional Requirements

### 8.1 Local Button Access (Implemented & Validated)
- **FR-001:** Table 1 NodeMCU shall continuously sample the physical button input on GPIO `D5` configured as `INPUT_PULLUP` with software debouncing (minimum 50 ms debounce threshold).
- **FR-002:** Table 2 NodeMCU shall continuously sample the physical button input on GPIO `D5` configured as `INPUT_PULLUP` with software debouncing (minimum 50 ms debounce threshold).
- **FR-003:** Table 3 NodeMCU shall continuously sample the physical button input on GPIO `D5` configured as `INPUT_PULLUP` with software debouncing (minimum 50 ms debounce threshold).
- **FR-004:** Upon a validated button press, Table 1 NodeMCU shall transmit the exact ASCII payload string `TABLE_1_OPEN` via ESP-NOW unicast directly to the Door NodeMCU's confirmed MAC address (`D8:BF:C0:D2:A9:32`) on Channel 1.
- **FR-005:** Upon a validated button press, Table 2 NodeMCU shall transmit the exact ASCII payload string `TABLE_2_OPEN` via ESP-NOW unicast directly to the Door NodeMCU's confirmed MAC address (`D8:BF:C0:D2:A9:32`) on Channel 1.
- **FR-006:** Upon a validated button press, Table 3 NodeMCU shall transmit the exact ASCII payload string `TABLE_3_OPEN` via ESP-NOW unicast directly to the Door NodeMCU's confirmed MAC address (`D8:BF:C0:D2:A9:32`) on Channel 1.
- **FR-007:** Each Table NodeMCU shall enforce a 2-second rate-limiting lockout following every accepted button press before servicing a subsequent trigger.
- **FR-008:** Upon receiving a recognized payload (`TABLE_1_OPEN`, `TABLE_2_OPEN`, or `TABLE_3_OPEN`), the Door NodeMCU shall energize relay pin `D1` for a momentary release duration of approximately 500 ms, then automatically de-energize the pin.
- **FR-009:** The local button access sequence (FR-001 through FR-008) shall execute autonomously without requiring an active Wi-Fi association, router IP assignment, or internet connection.

### 8.2 Remote Access (Planned Cloud Functionality)
- **FR-010:** The Mobile and Web applications shall provide an authorized interactive door-release button accessible only after authenticated login.
- **FR-011:** The Door NodeMCU shall maintain a real-time subscription or long-poll connection to the Firebase Realtime Database path `/commands/doorRelease`.
- **FR-012:** Upon receipt of a remote release command, the Door NodeMCU shall verify that the global remote access switch (`/systemControl/remoteAccessEnabled`) is `true` prior to actuation.
- **FR-013:** If verified, the Door NodeMCU shall actuate relay pin `D1` for the designated momentary release duration (~500 ms) and acknowledge or clear the command record.

### 8.3 Authentication (Planned Cloud Functionality)
- **FR-014:** The system shall authenticate all remote users via Firebase Authentication utilizing valid email and password credentials.
- **FR-015:** Client applications shall securely persist JSON Web Tokens (JWT) using platform-standard secure storage (`Expo SecureStore` on mobile, secure browser storage/memory on web).
- **FR-016:** Client applications shall automatically reject unauthenticated API or database transactions and route unauthenticated users to the login screen.
- **FR-017:** The system shall support administrative password reset dispatch through standard Firebase Auth verification workflows.

### 8.4 Door PIN Verification (Planned Cloud Functionality)
- **FR-018:** The client applications shall require the user to input a secondary 4-to-6 digit numeric Door PIN prior to dispatching any remote door release request.
- **FR-019:** The Door PIN verification shall serve as an additional authorization layer evaluated independently of user account login credentials.
- **FR-020:** The system shall enforce PIN validation against a reference stored securely in Firebase or validated via backend security rules.
- **FR-021:** The client application shall lock out further PIN entry attempts for 60 seconds after three consecutive incorrect PIN submissions.

### 8.5 User Management & Multi-User Support (Planned Cloud Functionality)
- **FR-022:** The system shall support multiple distinct user accounts categorized into roles: `admin`, `standard`, and `temporary`.
- **FR-023:** Administrators shall be able to invite, activate, deactivate, and remove user accounts via the administrative web dashboard.
- **FR-024:** Each user record in the database shall contain: `uid`, `email`, `displayName`, `role`, `createdAt`, and `isActive` status.
- **FR-025:** Deactivating a user record shall immediately prevent that user from successfully dispatching door-release commands.

### 8.6 Role-Based Permissions (Planned Cloud Functionality)
- **FR-026:** Firebase Realtime Database Security Rules shall enforce that only users with `role: "admin"` possess write authority to `/config`, `/systemControl`, `/temporaryAccess`, and `/users`.
- **FR-027:** Standard and Temporary users shall possess write authority restricted to their authorized command and request pathways.
- **FR-028:** Client applications shall dynamically adapt navigation views, hiding administrative controls from Standard and Temporary users.

### 8.7 Temporary Access Management (Planned Cloud Functionality)
- **FR-029 (Temporary Access Creation):** An Administrator shall be able to grant temporary remote door-release access to a designated user for a defined time window by creating a record with:
  - `accessGrantId`: Unique identifier for the temporary access grant.
  - `recipientUid`: Target user identifier (e.g., user account for Rahul).
  - `recipientEmail`: Email address of the recipient user.
  - `grantedBy`: Administrator user identifier (`adminUid`).
  - `doorId`: Identifier of the authorized door controller (e.g., `door_main_lab`).
  - `startTime`: UTC millisecond timestamp defining when access begins.
  - `endTime`: UTC millisecond timestamp defining when access terminates.
  - `status`: Current grant state (`PENDING`, `ACTIVE`, `EXPIRED`, or `REVOKED`).
  - `createdAt`: UTC timestamp of grant creation.
- **FR-030 (Time-Bounded Remote Ingress Window):** During the valid time window (`serverTime >= startTime` and `serverTime < endTime`), the designated temporary user shall be authorized to use the mobile or web application to submit remote door-release requests, subject to Door PIN verification.
  - *Example Scenario:* User Rahul is granted temporary access with `startTime: 14:00` (2:00 PM) and `endTime: 18:00` (6:00 PM). Rahul is authorized to request remote door release only between 2:00 PM and 6:00 PM.
- **FR-031 (Automatic Expiration):** A temporary access grant shall automatically become invalid and transition to `EXPIRED` once the authoritative server time reaches or exceeds `endTime`.
- **FR-032 (Administrative Early Revocation):** An Administrator shall have the authority to revoke any temporary access grant at any point prior to its scheduled `endTime`.
- **FR-033 (Post-Expiry/Revocation Invalidation):** Once a temporary access grant has expired or been revoked, that user shall no longer be authorized to perform remote door-release requests through that grant. Any subsequent remote release request submitted under that grant ID shall be rejected.

### 8.8 Temporary Access Lifecycle (Planned Cloud Functionality)
- **FR-034 (Deterministic State Machine):** Each temporary access grant shall follow a deterministic lifecycle:
  - `PENDING`: Created with `startTime > currentServerTime`. The user cannot execute release requests until `startTime` arrives.
  - `ACTIVE`: `startTime <= currentServerTime < endTime` and grant has not been revoked. The user is authorized to submit remote release requests.
  - `EXPIRED`: `currentServerTime >= endTime`. The grant is permanently invalid; requests are rejected.
  - `REVOKED`: Manually terminated early by an Administrator. The grant is permanently invalid; requests are rejected immediately.

### 8.9 Server-Authoritative Time & Authorization Rules (Planned Cloud Functionality)
- **FR-035 (Server-Authoritative Evaluation):** Access window validity and timestamp comparisons shall be evaluated using server-side / backend-authoritative time (e.g., Firebase Realtime Database Security Rules evaluating `now` or backend Cloud Functions).
- **FR-036 (Client Clock Independence):** The system shall **not** depend on the client mobile or web device's local clock to enforce temporary access authorization. If a client device clock is altered or out of synchronization, the server-side evaluation shall enforce authorization strictly against authoritative server time.

### 8.10 Remote Activity Recording Policy for Temporary Access (Planned Cloud Functionality)
- **FR-037 (Targeted Logging Boundary):** The system shall **not** create a general activity/history stream for every door opening.
  - Physical table-button events (Table 1, Table 2, Table 3 via ESP-NOW) are purely local events and shall **not** be stored in Firebase activity history.
  - The temporary-access activity history is dedicated specifically and exclusively to recording remote activity performed under temporary authorization.
- **FR-038 (Event Description Policy — No Physical Sensor):** Because the system currently lacks a physical door-position sensor, the recorded event shall be formally described as:
  `"Remote release command requested/issued"`
  rather than:
  `"Door opened"`.
- **FR-039 (Mandatory Activity Log Schema):** Every remote release request initiated under temporary access shall be immutably recorded in Firebase under `/temporaryAccessActivity/{accessGrantId}/{requestId}` with the following mandatory fields:
  - `requestId`: Unique push identifier.
  - `timestamp`: Authoritative UTC millisecond timestamp of the request.
  - `user`: Recipient user identifier (`uid` and email/displayName).
  - `accessGrantId`: Associated temporary access ID.
  - `doorId`: Target door identifier.
  - `action`: `"REMOTE_RELEASE_REQUESTED"`.
  - `method`: `"MOBILE_APP"` or `"WEB_DASHBOARD"`.
  - `status`: Result of the request (`PROCESSED`, `REJECTED_EXPIRED`, `REJECTED_REVOKED`, `REJECTED_INVALID_PIN`, or `REJECTED_REMOTE_DISABLED`).

### 8.11 Administrative Inspection of Temporary Access (Planned Cloud Functionality)
- **FR-040 (Administrator Audit Capabilities):** The administrative web dashboard shall provide a dedicated view allowing administrators to inspect the complete remote-access activity for any temporary access grant.
- **FR-041 (Required Inspection Summary Data):** For any selected temporary access grant, the administrative view shall clearly display:
  - Who received access (`recipientEmail` / `displayName`).
  - Who granted the access (`grantorEmail` / `displayName`).
  - Access start time (`startTime` formatted in human-readable local and UTC time).
  - Access end time (`endTime` formatted in human-readable local and UTC time).
  - Current status (`PENDING`, `ACTIVE`, `EXPIRED`, `REVOKED`).
  - Total number of remote release requests submitted under that grant.
  - Timestamps of all remote release requests submitted under that grant.

### 8.12 Local vs. Remote Event Handling Policy
- **FR-042 (Local Event Isolation):** Physical button presses at Table 1, Table 2, and Table 3 trigger local ESP-NOW unicast transmission directly to the Door NodeMCU and actuate the relay purely at the edge. These events shall remain completely offline and isolated from Firebase activity storage.
- **FR-043 (Decoupled Fault Domain):** Any failure of the cloud connection, database rate limits, or internet routing shall have zero impact on the local table button door-release operation.

### 8.13 Device Status & Telemetry (Planned Cloud Functionality)
- **FR-044:** The Door NodeMCU shall publish a periodic heartbeat message to Firebase under `/status/doorController` every 30 seconds.
- **FR-045:** The heartbeat telemetry payload shall include:
  - `online`: Boolean indicator.
  - `lastSeen`: Unix epoch timestamp.
  - `relayState`: `IDLE` or `PULSING`.
  - `uptimeSeconds`: Cumulative microcontroller uptime.
  - `wifiRSSI`: Wi-Fi signal strength in dBm.
  - `channel`: Operational radio channel (Channel 1).
- **FR-046:** Client dashboards shall display an `OFFLINE` badge if the Door NodeMCU heartbeat is older than 90 seconds.

### 8.14 Remote Access Enable / Disable (Planned Cloud Functionality)
- **FR-047:** The system shall maintain a persistent boolean flag in Firebase under `/systemControl/remoteAccessEnabled`.
- **FR-048:** Only users with `role: "admin"` shall be permitted to toggle `remoteAccessEnabled`.
- **FR-049:** When `remoteAccessEnabled` is set to `false`, the Door NodeMCU shall immediately ignore and discard any remote unlock command.
- **FR-050:** Toggling `remoteAccessEnabled` to `false` shall **not** disable, impede, or alter the operation of the local table buttons (Table 1, 2, and 3 via ESP-NOW).

### 8.15 Command Handling & Idempotency (Planned Cloud Functionality)
- **FR-051:** Remote release requests shall be written with a client timestamp and a unique transaction ID.
- **FR-052:** The Door NodeMCU shall reject any command received with a timestamp older than 10.0 seconds to prevent replay attacks or stale command execution after network reconnect.
- **FR-053:** Upon processing a command, the Door NodeMCU shall atomically transition the command status to `PROCESSED` or remove the command node to guarantee single-execution idempotency.

---

## 9. Non-Functional Requirements

### 9.1 Security
- **NFR-001:** All remote communications between client applications, Firebase, and the Door NodeMCU shall be encrypted using Transport Layer Security (TLS 1.2 or TLS 1.3).
- **NFR-002:** Microcontroller firmware source code committed to version control shall contain zero plain-text production passwords, Wi-Fi credentials, Firebase API keys, or database URLs. All credentials shall be injected via untracked configuration headers or `.env` files.
- **NFR-003:** The Door NodeMCU shall validate received ESP-NOW frames by sender MAC address against configured Table 1, Table 2, and Table 3 physical addresses.

### 9.2 Reliability
- **NFR-004:** The Door NodeMCU firmware shall implement a hardware/software Watchdog Timer (ESP8266 WDT) configured to reboot the processor automatically in the event of an unhandled lockup.
- **NFR-005:** The relay control GPIO pin shall initialize to the de-energized state immediately upon boot, and shall remain de-energized during firmware flashing, brownouts, and resets.
- **NFR-006:** The system shall survive power cycles on any combination of nodes without requiring manual re-pairing or firmware intervention.

### 9.3 Availability
- **NFR-007:** Local table button access shall maintain high operational availability within the physical bounds of the laboratory, completely independent of external internet outages.
- **NFR-008:** The Door NodeMCU shall implement an auto-reconnect routine for both Wi-Fi connectivity and Firebase streaming sessions whenever the connection drops.

### 9.4 Performance
- **NFR-009:** Local actuation latency from physical table button depression to relay closure is designed for low latency; actual end-to-end timing will be empirically documented during system validation.
- **NFR-010:** Target latency for remote door release requests dispatched over broadband Wi-Fi shall be documented during cloud integration testing.
- **NFR-011:** Client applications shall display status updates reactively upon state changes in Firebase.

### 9.5 Maintainability
- **NFR-012:** Firmware for Table 1, Table 2, Table 3, and Door Controller shall be structured into modular C++ source files with strict separation between radio protocols, relay control, and cloud synchronization.
- **NFR-013:** Mobile and web applications shall maintain shared data model definitions and TypeScript interfaces matching the Firebase Realtime Database schema.

### 9.6 Scalability
- **NFR-014:** The ESP-NOW peer management structure on the Door NodeMCU shall support expansion up to the ESP8266 hardware protocol limit (up to 20 registered peers).
- **NFR-015:** The Firebase Realtime Database schema shall support multiple concurrent users and temporary access records without requiring schema redesign.

### 9.7 Usability
- **NFR-016:** The mobile and web user interfaces shall present distinct visual states for `SECURED` (neutral/idle), `RELEASE_ACTIVE` (active pulse), `REMOTE_DISABLED` (warning), and `OFFLINE` (error).
- **NFR-017:** All interactive door release buttons on client applications shall require explicit confirmation (Door PIN modal) to eliminate accidental actuations.

---

## 10. Hardware Requirements

### 10.1 Microcontrollers
- **Door Controller:**
  - 1 × ESP8266 NodeMCU development board.
  - Tensilica L106 32-bit RISC core operating at 80 MHz (or 160 MHz).
  - 4 MB onboard SPI Flash memory.
  - Integrated 2.4 GHz PCB trace antenna.
  - Confirmed MAC address: `D8:BF:C0:D2:A9:32`.
- **Table Controllers:**
  - 3 × ESP8266 NodeMCU development boards (designated Table 1, Table 2, and Table 3).
  - Integrated 2.4 GHz PCB trace antenna.

### 10.2 Relays and Door Release Interface
- **Relay Module:** Single-channel 5V electromechanical relay module.
- **Contact Ratings:** SPDT contacts (Common, Normally Open, Normally Closed).
- **Logic Drive:** Driven by Door NodeMCU GPIO `D1` (`GPIO5`).
- **Interface Characterization:** The relay shall interface to the laboratory's door release circuit as a controlled, momentary contact closure (Normally Open dry contact) to signal the existing authorized door-release input/circuit. The system shall **not** directly drive electric strike or magnetic lock motor coils from logic power.
- **Sensor Constraint:** The system incorporates **no physical door-position sensor**; it controls the momentary relay dry-contact state only.

### 10.3 Push Buttons and Wiring
- **Buttons:** 3 × momentary, normally open (NO) mechanical tactile push buttons.
- **Interface:** Connected between GPIO `D5` (`GPIO14`) and GND on each Table NodeMCU, utilizing internal pull-up resistors (`INPUT_PULLUP`).

### 10.4 Power Supplies
- **Logic Power:** During bench development, the NodeMCU boards are powered through their USB/VIN power interfaces using suitable regulated sources.
- **Electrical Isolation:** The logic power supply for the NodeMCU controllers must be isolated from any external power supplies utilized by the existing door lock circuit.

---

## 11. Software Requirements

### 11.1 Firmware Specifications
- **Development Toolchain:** Arduino Core for ESP8266 / PlatformIO.
- **Core Libraries:**
  - `ESP8266WiFi.h`: For 802.11 b/g/n station mode and channel management.
  - `espnow.h`: For connectionless ESP-NOW frame generation, peer registration, and reception callbacks.
  - Firebase client / REST client: For authenticated Firebase Realtime Database stream synchronization.
  - Non-blocking `millis()` timing routines for asynchronous relay pulse resets (~500 ms).

### 11.2 Cloud Infrastructure (Google Firebase)
- **Firebase Authentication:** Configured for Email/Password provider.
- **Firebase Realtime Database:** Low-latency JSON database hosting `/commands`, `/status`, `/temporaryAccess`, `/temporaryAccessActivity`, `/users`, and `/systemControl`.
- **Security Rules (`database.rules.json`):** Enforcing authentication, role checks via user lookup, temporary access window validation using server timestamps, schema validation, and write restrictions.

### 11.3 Mobile Application
- **Framework:** React Native with Expo SDK.
- **Key Modules:**
  - Secure token storage for persistent credentials.
  - Role-protected navigation hierarchies (Home, Activity, Settings).
  - Firebase JavaScript SDK (v9+ modular).

### 11.4 Web Dashboard
- **Framework:** React 18+ bootstrapped with Vite.
- **Language / Styling:** TypeScript / Semantic Vanilla CSS.
- **Routing:** React Router DOM.

---

## 12. Network Requirements

### 12.1 RF Band & Physical Frequencies
- All wireless operations (ESP-NOW and Wi-Fi) shall occur strictly within the **2.4 GHz ISM band**.
- The ESP8266 hardware does not support 5.0 GHz networks; the host laboratory access point must broadcast a dedicated 2.4 GHz SSID.

### 12.2 ESP-NOW Channel Synchronization
- **Constraint:** The ESP8266 possesses a single physical 2.4 GHz radio transceiver. When connected to a Wi-Fi Access Point, its radio frequency is locked to the operating channel of that AP.
- **Requirement:** Table 1, Table 2, and Table 3 NodeMCUs and the Door NodeMCU operate on **Channel 1**, matched to the laboratory Wi-Fi router channel.

### 12.3 Cloud Protocols & Firewall Ports
- Outbound HTTPS/TLS traffic on port `443`.
- Outbound Secure WebSockets (WSS) traffic on port `443` for real-time Firebase listeners.
- No inbound open ports or public port-forwarding rules required on the local lab router.

---

## 13. Security Requirements

- **SEC-001:** The system shall enforce dual-layer access verification for remote actuation: a valid authenticated user session (Firebase Auth) combined with an independent, verified Door PIN.
- **SEC-002:** The system shall operate exclusively as an authorized parallel release mechanism and shall under no circumstances bypass, deactivate, or override existing laboratory fingerprint scanners, card readers, or manual lock cylinders.
- **SEC-003:** The Door NodeMCU shall validate the MAC address of any received ESP-NOW frame against configured table addresses before evaluating payload contents.
- **SEC-004:** All production API keys, service credentials, Wi-Fi passwords, and PIN secrets shall be stored in external environment variables (`.env`) and excluded from Git commits via `.gitignore`.
- **SEC-005:** Firebase Realtime Database Security Rules shall validate that write operations to `/commands/doorRelease` contain valid authenticated user UIDs and are rejected if `/systemControl/remoteAccessEnabled` is `false`.
- **SEC-006:** Release commands dispatched through Firebase shall contain a timestamp and shall be discarded by the Door NodeMCU if the command is stale (> 10.0 seconds).
- **SEC-007:** Failed PIN verification attempts shall be rate-limited and logged to detect brute-force activity.
- **SEC-008:** The Door NodeMCU firmware shall not expose any unauthenticated local web server or configuration interface.
- **SEC-009 (Server-Authoritative Temporal Access Enforcement):** All temporary access permissions shall be validated using server-authoritative time rather than client-reported device time to prevent client clock manipulation attacks.
- **SEC-010 (Immediate Revocation Propagation):** Revoking a temporary access grant in Firebase shall invalidate subsequent command submissions immediately.
- **SEC-011 (Local RF Isolation):** Local ESP-NOW frames shall not be transmitted to cloud databases or exposed to WAN endpoints, protecting bench workflow privacy and preserving network bandwidth.

---

## 14. Safety Requirements

- **SAFE-001 (Safe Relay Default State):** In the event of electrical power interruption, reset, startup, brownout, or firmware crash, the door relay module shall remain in its de-energized state, ensuring the door release circuit is not intentionally energized. Physical door lock behavior depends on the existing facility access-control hardware and must be validated separately.
- **SAFE-002 (Pulse Duration Hard Ceiling):** The firmware shall employ an asynchronous safety timer enforcing an absolute maximum relay energization duration of 3.0 seconds, even if cloud communications hang or crash during an active pulse.
- **SAFE-003 (Boot-up Glitch Immunity):** The Door NodeMCU relay control output shall be assigned to a GPIO pin that remains de-energized during the ESP8266 boot-loader sequence.
- **SAFE-004 (Emergency Egress Preservation):** The installation shall never inhibit or impede interior mechanical egress (e.g., panic exit push bars, mechanical thumb-turns, or manual bypass switches required by life safety standards).
- **SAFE-005 (Relay Electrical Interface Isolation):** Appropriate electrical isolation between microcontroller logic and the external door release circuit should be verified during hardware integration testing.

---

## 15. Failure Scenarios and Mitigations

| # | Failure Mode | Impact | Automated System Mitigation | Recovery Action |
|---|---|---|---|---|
| **FS-01** | **Wi-Fi Router / ISP Outage** | Cloud connectivity lost; remote mobile/web release unavailable. | Door NodeMCU continues executing ESP-NOW receiver loop. Local table buttons continue releasing the door without interruption. | System automatically reconnects to Firebase when Wi-Fi/ISP recovers. |
| **FS-02** | **Door NodeMCU Power Interruption** | Door release circuit inactive; remote status shows offline. | Relay defaults to de-energized state; door release circuit is not energized. When power returns, safe GPIO initialization prevents false trigger. | Microcontroller re-attaches to ESP-NOW and Wi-Fi upon reboot. |
| **FS-03** | **Table NodeMCU Power Loss / Failure** | That specific table's push button ceases to trigger door release. | Other two table nodes and all remote access paths continue normal operation. | Replace 5V power cable or reset button node. |
| **FS-04** | **Stale Cloud Command Execution** | Network lag causes delayed delivery of remote unlock command. | Command handling logic verifies command timestamp. Commands older than 10.0 seconds are discarded without relay actuation. | Client application notifies user of command timeout; prompt to retry. |
| **FS-05** | **Wi-Fi Channel Shift on AP** | AP switches 2.4 GHz channel; ESP-NOW packet reception breaks. | Door NodeMCU follows AP channel; table nodes operating on fixed channel lose contact. | Realign channel configuration across all nodes. |
| **FS-06** | **Relay Module Hardware Failure** | Contacts open or fail to switch. | Door release circuit is not intentionally energized by the failed relay. | Manual key or existing primary fingerprint scanner used for entry; replace relay module. |
| **FS-07** | **Brute-Force PIN Attack** | Unauthorized user attempts multiple PIN guesses on client app. | Client enforces 60-second lockout after 3 consecutive failures. | Administrator reviews security logs and suspends offending account. |
| **FS-08** | **Client Clock Manipulation Attack** | User alters mobile device clock to simulate a valid temporary access window. | Server-authoritative time evaluation (Firebase rules / Cloud backend) evaluates actual server time; unauthorized request is rejected. | Command rejected with `REJECTED_EXPIRED` or `REJECTED_UNAUTHORIZED`. |

---

## 16. Constraints

1. **Microcontroller Resources:** The ESP8266 features a single core running at 80/160 MHz with approximately 80 KB of user RAM. All firmware code must be non-blocking and avoid heavy memory allocations.
2. **Single Shared Radio Hardware:** The ESP8266 radio must service both ESP-NOW frames and Wi-Fi 802.11 b/g/n packets concurrently on Channel 1.
3. **Absence of Physical Door Sensor:** The system monitors relay coil command status; it does not currently incorporate a physical magnetic reed switch to confirm whether the door frame physically separated or remained latched.
4. **Institutional Security Compliance:** All remote operations must remain subservient to the primary laboratory security mechanism (external fingerprint scanner); remote release is treated as an auxiliary convenience release trigger.

---

## 17. Acceptance Criteria

The following testable criteria define the baseline requirements for project milestone verification:

### AC-01: Table 1 Local Trigger Verification (Implemented & Bench-Validated)
- **Given:** Table 1 NodeMCU and Door NodeMCU are powered on and configured to radio Channel 1.
- **When:** The Table 1 physical button is depressed.
- **Then:** Table 1 transmits `TABLE_1_OPEN` via ESP-NOW, the Door NodeMCU receives the packet, energizes the relay for approximately 500 ms, and de-energizes the relay without writing any log to Firebase.

### AC-02: Table 2 Local Trigger Verification (Implemented & Bench-Validated)
- **Given:** Table 2 NodeMCU and Door NodeMCU are powered on and configured to radio Channel 1.
- **When:** The Table 2 physical button is depressed.
- **Then:** Table 2 transmits `TABLE_2_OPEN` via ESP-NOW, the Door NodeMCU receives the packet, energizes the relay for approximately 500 ms, and de-energizes the relay without writing any log to Firebase.

### AC-03: Table 3 Local Trigger Verification (Implemented & Bench-Validated)
- **Given:** Table 3 NodeMCU and Door NodeMCU are powered on and configured to radio Channel 1.
- **When:** The Table 3 physical button is depressed.
- **Then:** Table 3 transmits `TABLE_3_OPEN` via ESP-NOW, the Door NodeMCU receives the packet, energizes the relay for approximately 500 ms, and de-energizes the relay without writing any log to Firebase.

### AC-04: Offline Local Autonomy (Implemented & Bench-Validated)
- **Given:** The laboratory Wi-Fi router is completely disconnected or powered down.
- **When:** Any table button (Table 1, Table 2, or Table 3) is pressed.
- **Then:** The Door NodeMCU successfully actuates the relay for approximately 500 ms without delay or error.

### AC-05: Unauthorized Payload Rejection (Implemented & Bench-Validated)
- **Given:** An ESP8266 transmits an unrecognized payload string to the Door NodeMCU.
- **When:** The frame is received by the Door NodeMCU.
- **Then:** The Door NodeMCU rejects the packet and the relay remains completely de-energized.

### AC-06: Remote Release with Valid PIN (Planned Milestone)
- **Given:** An authenticated user with role `standard` is logged into the Mobile or Web application, and `remoteAccessEnabled` is `true`.
- **When:** The user triggers a door release and enters the correct Door PIN.
- **Then:** A command is dispatched to Firebase, received by the Door NodeMCU, and the relay energizes for approximately 500 ms.

### AC-07: Remote Release with Invalid PIN (Planned Milestone)
- **Given:** An authenticated user is logged into the client application.
- **When:** The user enters an incorrect Door PIN.
- **Then:** The client rejects the command, no release request is dispatched to `/commands/doorRelease`, and the relay does not actuate.

### AC-08: Administrative Remote Lockout (Planned Milestone)
- **Given:** An administrator has set `/systemControl/remoteAccessEnabled` to `false`.
- **When:** A user attempts a remote door release with a valid PIN.
- **Then:** The Door NodeMCU ignores the remote command and the relay remains de-energized, while Table 1, 2, and 3 physical buttons continue to function normally.

### AC-09: Power-On Glitch Immunity (Safety Verification)
- **Given:** The Door NodeMCU is connected to the relay module and an oscilloscope / logic analyzer is attached to relay pin `D1`.
- **When:** The Door NodeMCU is cold-booted, reset via hardware reset pin, or flashed over serial.
- **Then:** The relay control pin shall remain in the de-energized state at all times during boot and reset.

### AC-10: Temporary Access Creation & Authorized Remote Release Window (Planned Milestone)
- **Given:** An administrator creates a temporary access grant for user Rahul with a window from 2:00 PM to 6:00 PM (`status: ACTIVE`).
- **When:** Rahul logs into the mobile or web application at 3:15 PM, submits a remote release request, and enters the correct Door PIN.
- **Then:** The command is authorized via server-authoritative time, the Door NodeMCU energizes the relay for approximately 500 ms, and an immutable record is logged under `/temporaryAccessActivity/{accessGrantId}` containing `timestamp`, `user: "Rahul"`, `accessGrantId`, `doorId`, `action: "REMOTE_RELEASE_REQUESTED"`, and `method`.

### AC-11: Temporary Access Automatic Expiration (Planned Milestone)
- **Given:** User Rahul was granted temporary access with an `endTime` of 6:00 PM.
- **When:** Authoritative server time reaches or exceeds 6:00 PM (e.g., 6:01 PM) and Rahul attempts a remote release request.
- **Then:** The system rejects the request as expired (`status: EXPIRED`), the relay remains de-energized, and Rahul is no longer authorized to trigger remote release.

### AC-12: Immediate Early Revocation of Temporary Access (Planned Milestone)
- **Given:** User Rahul has an active temporary access grant scheduled to end at 6:00 PM.
- **When:** An administrator revokes the grant at 4:30 PM (`status: REVOKED`) and Rahul attempts a remote release request at 4:31 PM.
- **Then:** The system immediately rejects the command, the relay remains de-energized, and no release signal is issued.

### AC-13: Local ESP-NOW Exclusion from Firebase Activity History (Implemented & Bench-Validated)
- **Given:** All table controllers and Door NodeMCU are operating on Channel 1, and the Door NodeMCU is connected to Wi-Fi and Firebase.
- **When:** A user presses the physical button at Table 1, Table 2, or Table 3.
- **Then:** The relay actuates locally for approximately 500 ms, and **zero records** are written to Firebase `/temporaryAccessActivity` or any cloud activity history.

### AC-14: Server-Authoritative Time Enforcement Against Client Clock Manipulation (Planned Milestone)
- **Given:** User Rahul's temporary access window expired at 6:00 PM, and the actual server time is 7:15 PM.
- **When:** Rahul manually adjusts the local clock on his mobile device backward to 3:00 PM and attempts a remote release request.
- **Then:** The server-side authorization check evaluates the true server timestamp, identifies that the actual time is past `endTime`, and rejects the request without actuating the relay.

### AC-15: Administrator Temporary Access Inspection & Request Metrics (Planned Milestone)
- **Given:** User Rahul has executed three remote release requests during his approved temporary access window.
- **When:** An administrator navigates to the temporary access management section of the web dashboard.
- **Then:** The dashboard displays:
  - Recipient: Rahul.
  - Granted by: Administrator.
  - Start time and End time (2:00 PM to 6:00 PM).
  - Current status (`ACTIVE` or `EXPIRED`).
  - Total number of remote release requests: 3.
  - The exact timestamps for all 3 remote release requests.

---

## 18. Future Requirements

The following requirements represent planned evolutionary enhancements scheduled for subsequent system revisions once core cloud bench integration is complete:

- **FUT-001 (Captive Portal Provisioning):** The Door NodeMCU shall support an on-demand Wi-Fi captive portal (e.g., using `WiFiManager`) to allow laboratory administrators to configure local SSID and password credentials without re-flashing firmware.
- **FUT-002 (Cryptographic ESP-NOW Encryption):** Implement AES-128 payload encryption and rolling packet sequence counters for all table-to-door ESP-NOW transmissions to prevent local RF eavesdropping or replay packets.
- **FUT-003 (Device Telemetry Buffering):** The Door NodeMCU may allocate a ring buffer in onboard Flash to store system health and diagnostic telemetry during internet outages, synchronizing telemetry once connectivity is restored.
- **FUT-004 (Door Status Magnetic Sensor):** Integrate a physical magnetic reed switch on an available Door NodeMCU GPIO to detect and publish actual physical door position (`CLOSED`, `OPEN`, `HELD_OPEN_ALERT`).
- **FUT-005 (Biometric & RFID Panel Ingress Integration):** Support hardware UART / Wiegand protocol input from an external physical RFID badge reader or keypad mounted directly beside the door frame.
- **FUT-006 (Battery Backup & Brownout Telemetry):** Incorporate an uninterruptible lithium-ion battery backup circuit with ADC voltage monitoring to detect main power failures and alert administrators via push notification.
