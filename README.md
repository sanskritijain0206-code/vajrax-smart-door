# VAJRAX Smart Door

Secure Multi-Modal IoT Access Control System

> [!WARNING]
> **Project Status: Active Development**  
> This project is currently in the initial design and active development phase. It is **not** completed, deployed, production-ready, or fully secure. Hardware schematics, firmware implementations, and cloud configurations are currently being built and tested on bench prototypes.

---

## 1. Overview

**VAJRAX Smart Door** is an IoT-based access control system designed specifically for a robotics laboratory environment. 

In a robotics laboratory setting, team members stationed at different workbenches require fast, convenient door access without having to walk across the room or depend on continuous internet connectivity. At the same time, lab managers require remote visibility, user management, and cloud-enabled access controls.

The system addresses these needs through a dual-mode communication model:
- **Local Control:** Workstation table nodes communicate directly with the door controller node using the low-latency **ESP-NOW** wireless protocol, allowing local physical button presses to unlock the door even if the internet or local Wi-Fi router is offline.
- **Cloud Control:** The door controller connects to **Firebase** over Wi-Fi, enabling authorized users to monitor status and unlock the door remotely through dedicated mobile and web applications.

---

## 2. Features

The system is being designed and implemented around the following core features:

- **Local Wireless Door Control:** Direct peer-to-peer door actuation from workstation push buttons using ESP-NOW without internet dependency.
- **Remote Door Control Over the Internet:** Remote door unlocking and state synchronization mediated by Firebase cloud services.
- **Multiple Authorized Users:** Account-based access management supporting multiple laboratory researchers, technicians, and administrators.
- **User Authentication:** Identity validation via Firebase Authentication prior to granting access to client dashboards.
- **Separate Door PIN Verification:** A secondary security step requiring a designated Door PIN before an unlock command can be executed from mobile or web interfaces.
- **Admin and User Roles:** Role-based access control (RBAC) to differentiate administrative privileges (user management, logs review, remote lockout) from standard operator permissions.
- **Activity Logs:** Audit trail recording unlock events, timestamps, trigger sources (Table 1, Table 2, Table 3, Mobile App, or Web Dashboard), and user identity.
- **Door / Device Status Monitoring:** Real-time telemetry monitoring door state (locked/unlocked), relay state, and device connectivity.
- **Remote Access Enable / Disable:** Administrative control to remotely enable or disable external access or lock out specific trigger inputs.
- **Mobile Application:** Dedicated cross-platform mobile app built with React Native and Expo for on-the-go door control.
- **Web Dashboard:** Desktop-optimized management dashboard built with React and Vite for lab monitoring and administration.

---

## 3. System Architecture

The architecture separates local real-time actuation from remote cloud management.

### Architecture Diagram

```text
Mobile App / Web Dashboard
        |
        v
     Firebase
        |
        v
  Door NodeMCU
      /   \
 ESP-NOW  Relay
   / | \
 T1 T2 T3
```

### Communication Channels

1. **Local Channel (ESP-NOW):**
   - Table NodeMCUs (**T1**, **T2**, **T3**) detect physical button presses.
   - Upon detection, an ESP-NOW packet is transmitted peer-to-peer directly to the Door NodeMCU.
   - The Door NodeMCU validates the transmitter and momentarily energizes the relay to unlock the door.

2. **Remote Channel (Wi-Fi + Firebase):**
   - The user authenticates via the Mobile Application or Web Dashboard.
   - The user inputs the secondary Door PIN to request an unlock action.
   - The request is written to Firebase Realtime Database.
   - The Door NodeMCU, connected to the local Wi-Fi network, listens for database events, validates the request, triggers the relay, and writes back the updated status and activity log.

---

## 4. Hardware

The current hardware setup consists of the following components:

- **1 × Door NodeMCU (ESP8266):**  
  Acts as the central controller. It is connected to the door relay, listens for incoming ESP-NOW messages from table nodes, connects to Wi-Fi, and syncs state with Firebase.
- **3 × Table NodeMCUs (ESP8266):**  
  Configured as dedicated wireless transmitters placed at individual laboratory tables (Table 1, Table 2, Table 3).
- **Physical Push Buttons:**  
  Momentary tactile push buttons wired to GPIO pins on Table 1, Table 2, and Table 3 NodeMCUs for physical trigger inputs.
- **Relay Module:**  
  Controlled by the Door NodeMCU to switch power to the electronic door locking mechanism (solenoid / magnetic lock / strike).
- **Power Supplies:**  
  Regulated power supply for the NodeMCU microcontrollers and a dedicated supply for the door lock actuator.

---

## 5. Software Stack

The software components and technologies used across the project:

- **Microcontroller Hardware:** ESP8266 NodeMCU
- **Local Wireless Protocol:** ESP-NOW (connectionless 2.4 GHz peer-to-peer protocol)
- **Network Connectivity:** Wi-Fi (802.11 b/g/n)
- **User Authentication:** Firebase Authentication
- **Cloud Database:** Firebase Realtime Database
- **Mobile Application:** React Native, Expo
- **Web Dashboard:** React, Vite
- **Version Control:** Git, GitHub

---

## 6. Repository Structure

The codebase is organized into modular directories representing each subsystem:

```text
vajrax-smart-door/
├── docs/                     # Documentation, wiring diagrams, and architecture notes
├── firmware/                 # C++/Arduino firmware for ESP8266 NodeMCUs
│   ├── door-controller/     # Door NodeMCU code (Relay control, ESP-NOW receiver, Firebase client)
│   ├── table-1/              # Workstation Table 1 NodeMCU transmitter firmware
│   ├── table-2/              # Workstation Table 2 NodeMCU transmitter firmware
│   └── table-3/              # Workstation Table 3 NodeMCU transmitter firmware
├── mobile/                   # React Native + Expo mobile application
├── web/                      # React + Vite web management dashboard
├── firebase/                 # Firebase security rules, database structure, and config templates
├── hardware/                 # Hardware design assets
│   ├── diagrams/             # High-level system diagrams and schematics
│   └── wiring/               # Pinout and breadboard wiring documentation
├── tests/                    # Firmware and software test scripts
├── screenshots/              # Hardware photos, schematics, and interface previews
├── .env.example              # Placeholder environment variable template
├── .gitignore                # Git ignore configuration
├── LICENSE                   # Project license
└── README.md                 # Project documentation
```

---

## 7. Security

Security in VAJRAX Smart Door is approached through layered verification and defense-in-depth planning:

- **Dual-Layer Access Verification:**  
  Authentication at the app/dashboard level is separated from door actuation. An independent Door PIN must be provided before unlock commands can be dispatched.
- **Role-Based Access Control:**  
  User roles (Admin vs. Standard User) determine authorization levels. Administrative actions, such as remote access enable/disable and log inspection, require elevated privileges enforced through Firebase Security Rules.
- **Local Node Identification:**  
  ESP-NOW communication will pair table nodes to the door controller using MAC address whitelisting, with planned packet counter and payload verification to safeguard against accidental or unauthorized broadcasts.
- **Administrative Override:**  
  Administrators can remotely toggle access permissions or disable remote unlocks in case of security anomalies.
- **Secret & Key Hygiene:**  
  No production keys, passwords, or Firebase service account credentials are stored in version control. All configuration values rely on `.env` files tracked via `.env.example`.

---

## 8. Development Status

> [!NOTE]
> The VAJRAX Smart Door project is actively under development. Features listed below represent the current implementation plan and milestones.

- [x] Initial repository setup and directory structure
- [x] Architecture definition and environment templates
- [ ] ESP-NOW communication prototype between Table nodes and Door controller
- [ ] Relay control logic with safe power-on default state
- [ ] Firebase Authentication and Realtime Database integration
- [ ] React Native + Expo mobile app interface
- [ ] React + Vite web administrative dashboard
- [ ] Door PIN verification logic
- [ ] Activity logging and telemetry synchronization
- [ ] Laboratory bench testing and integration

---

## 9. Testing

The testing strategy is designed to validate system stability, safety, and responsiveness across the planned phases:

- **ESP-NOW Peer-to-Peer Communication Testing:**  
  Testing packet delivery, range, and latency between Table 1, 2, 3 NodeMCUs and the Door NodeMCU across physical laboratory distances.
- **Relay Logic & Safety Testing:**  
  Verifying that the relay trigger operates reliably on valid signals, does not pulse on microcontroller reboot, and returns to a safe locked state after a configured pulse duration.
- **Wi-Fi Disconnection & Fallback Testing:**  
  Verifying that local table buttons via ESP-NOW remain functional when the Door NodeMCU loses Wi-Fi or internet connectivity.
- **Cloud Synchronization & Latency Testing:**  
  Validating real-time updates and measuring latency for remote unlock commands dispatched via Firebase.
- **Authorization & PIN Enforcement Testing:**  
  Testing that invalid PIN attempts and unauthorized user roles are rejected without triggering the door relay.

---

## 10. Future Improvements

Planned future enhancements once core development and bench testing are complete:

- **Captive Portal Wi-Fi Setup:**  
  On-device configuration interface to connect the Door NodeMCU to local Wi-Fi without hardcoding credentials in firmware.
- **Encrypted Local Messaging:**  
  Implementation of encrypted payloads for ESP-NOW communication between Table nodes and the Door controller.
- **Offline Event Caching:**  
  Local timestamped buffer on the Door NodeMCU to log local button unlock events during network outages and sync when reconnected.
- **Biometric / RFID Integration:**  
  Hardware extension supporting external RFID cards or fingerprint sensors at the physical door.
- **Battery Backup & Power Monitoring:**  
  Hardware support for battery backup on the door controller with alerts on power disconnects.

---

## 11. License

This project is licensed under the terms described in the [LICENSE](file:///Users/sanskritijain/Desktop/vajrax-smart-door/LICENSE) file in the root directory of this repository.
