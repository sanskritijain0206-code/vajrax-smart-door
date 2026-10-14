# VAJRAX Smart Door — Hardware & Communication Specification

**Document Version:** 1.0.0
**Project:** VAJRAX Smart Door
**Status:** Active Development

This specification defines the hardware interfaces, microcontroller pin assignments (GPIO), wireless radio protocols, relay control interface, electrical boundaries, power architecture, and communication pathways for the VAJRAX Smart Door system. It establishes explicit boundaries between the **Currently Validated** local hardware implementation and the **Planned** remote cloud infrastructure.

---

## 1. Document Scope & Boundary Principles

The VAJRAX Smart Door system is an auxiliary access-control subsystem engineered for a robotics research laboratory. Its primary function is to provide rapid, localized workbench-to-door actuation alongside a planned authenticated remote release mechanism.

To maintain engineering rigor and safety:
- **Verified Electrical Behavior vs. Physical Door State:** Microcontroller relay control outputs and logic states have been bench-tested and verified. However, physical door release behavior depends on the existing laboratory access-control infrastructure and requires separate administrative authorization and physical integration validation.
- **Local Autonomy:** The local hardware path operates completely independent of external routers, Wide Area Networks (WAN), or cloud backends.
- **Separation of Concerns:** Hardware specifications herein document confirmed bench-test configurations only. Unvalidated parameters (e.g., precise current draw, battery lifetimes, wireless ranges across concrete barriers) are deferred to empirical testing phases.

---

## 2. System Hardware Inventory

The hardware architecture consists of four edge microcontrollers, their associated peripheral interfaces, and the external laboratory door infrastructure:

| Component | Quantity | Designation | Role / Function | Implementation Status |
|---|---|---|---|---|
| **ESP8266 NodeMCU** | 1 | Door NodeMCU | Central Edge Gateway, ESP-NOW receiver, relay controller, Wi-Fi client | **Bench Tested & Validated** (Local / Relay) |
| **ESP8266 NodeMCU** | 3 | Table NodeMCUs | Workbench push-button transmitters (Table 1, Table 2, Table 3) | **Bench Tested & Validated** |
| **5V Single-Channel Relay Module** | 1 | Door Relay Module | Momentary dry-contact trigger for existing door-release circuit | **Bench Tested & Validated** (Dry bench logic) |
| **Momentary Push Buttons** | 3 | Physical Buttons | Normally Open (NO) tactile switches connected to Table NodeMCUs | **Bench Tested & Validated** |
| **Laboratory Access Control Loop** | 1 | Existing Door Circuit | Existing authorized door-release input/circuit | **External Infrastructure** (Separate Validation Required) |

---

## 3. Door NodeMCU Specification

The Door NodeMCU functions as the primary edge controller and hardware actuator for the access-control interface.

### 3.1 Responsibilities
- **ESP-NOW Frame Reception:** Listens continuously on radio Channel 1 for unicast frames transmitted by the table controllers.
- **Payload Inspection & Validation:** Matches received byte sequences against recognized command strings (`TABLE_1_OPEN`, `TABLE_2_OPEN`, `TABLE_3_OPEN`). Discards unrecognized or malformed payloads.
- **Relay Actuation:** Drives the relay control pin (GPIO `D1`) to trigger a momentary contact closure pulse of approximately 500 ms upon command validation.
- **Wi-Fi Connectivity (Planned):** Connects to the laboratory 2.4 GHz Access Point on Channel 1 to interface with cloud services.
- **Cloud Gateway Bridge (Planned):** Evaluates incoming remote release commands dispatched via Firebase Realtime Database.

### 3.2 Radio & Network Configuration
- **Hardware MAC Address:** `D8:BF:C0:D2:A9:32`
- **ESP-NOW Channel:** 1
- **Credential Hygiene:** No Wi-Fi network passphrases, static private keys, or Firebase database secrets are stored or exposed in this hardware specification or client-facing configuration files.

---

## 4. Table NodeMCU Specifications

Three identical ESP8266 NodeMCU units serve as dedicated edge transmitters deployed across laboratory workbenches:
- **Table 1 NodeMCU**
- **Table 2 NodeMCU**
- **Table 3 NodeMCU**

### 4.1 Electrical & Input Characteristics
- **Input Pin:** GPIO `D5`
- **Internal Configuration:** Configured with internal pull-up resistor (`INPUT_PULLUP`).
- **Mechanical Interface:** Normally Open (NO) tactile momentary push button wired directly between `D5` and `GND`.
- **Logic Thresholds:**
  - Button Released: Logic `HIGH` (~3.3V via internal pull-up).
  - Button Depressed: Logic `LOW` (0V via closed contact to GND).

### 4.2 Firmware Input Filtering
- **Software Debounce Window:** Minimum stable `LOW` duration of approximately 50 ms required before qualifying a mechanical closure as a valid press. This prevents spurious triggers from mechanical switch contact bounce.
- **Safety Re-Arm Lockout:** Enforces an intentional 2-second rate-limiting lockout following every accepted button actuation. Any subsequent button presses occurring within 2000 ms of an accepted trigger are discarded by firmware to prevent repeated rapid cycling of the relay.

### 4.3 Command Payload Mapping
Each table controller transmits a distinct command payload formatted as a null-terminated ASCII string:
- **Table 1:** `TABLE_1_OPEN`
- **Table 2:** `TABLE_2_OPEN`
- **Table 3:** `TABLE_3_OPEN`

### 4.4 Transmission Protocol
- **Transport Mode:** ESP-NOW Unicast (Peer-to-Peer frame transmission addressed specifically to destination MAC `D8:BF:C0:D2:A9:32`).
- **ESP-NOW Channel:** 1
- **Communication Topology Note:** Transmission is strictly peer-to-peer unicast. It does **not** employ unaddressed broadcast frames.

---

## 5. Microcontroller GPIO Pin Assignment

| Microcontroller Unit | NodeMCU Board Pin | ESP8266 Native GPIO | Assigned Peripheral | Electrical Configuration | Active State | Validated Function |
|---|---|---|---|---|---|---|
| **Door NodeMCU** | `D1` | `GPIO5` | 5V Relay Module (Control Signal `S`) | Push-Pull Digital Output | Active Trigger Level | Drives relay for ~500 ms momentary pulse |
| **Table 1 NodeMCU** | `D5` | `GPIO14` | Momentary Push Button | Digital Input with `INPUT_PULLUP` | Active `LOW` | Generates `TABLE_1_OPEN` unicast packet |
| **Table 2 NodeMCU** | `D5` | `GPIO14` | Momentary Push Button | Digital Input with `INPUT_PULLUP` | Active `LOW` | Generates `TABLE_2_OPEN` unicast packet |
| **Table 3 NodeMCU** | `D5` | `GPIO14` | Momentary Push Button | Digital Input with `INPUT_PULLUP` | Active `LOW` | Generates `TABLE_3_OPEN` unicast packet |

*Note: No additional GPIO pins are utilized in the validated bench baseline. Unused pins remain unconfigured or in standard floating/boot-strapping states.*

---

## 6. Relay Module Interface & Electrical Boundaries

The door actuation interface utilizes an electromechanical single-channel 5V relay module.

### 6.1 Control Wiring (Low-Voltage DC Logic)
```
[ Door NodeMCU ]                    [ 5V Relay Module ]
   ( VIN ) ----------------------------> ( + / VCC )
   ( GND ) ----------------------------> ( - / GND )
   ( D1 / GPIO5 ) ---------------------> ( S / IN )
```

- **Power Feed:** The relay module coil is powered from NodeMCU `VIN` (5V derived from the USB host/adapter bus).
- **Common Ground:** NodeMCU `GND` is tied directly to the relay module ground pin (`-`).
- **Control Signal:** NodeMCU pin `D1` (`GPIO5`) interfaces to relay signal pin `S` (or `IN`).

### 6.2 Output Contacts (High-Voltage / Facility Side)
The relay module terminal block provides three screw-terminal dry contacts:
- **`COM` (Common):** Moving contact terminal.
- **`NO` (Normally Open):** Open circuit relative to `COM` when relay coil is de-energized.
- **`NC` (Normally Closed):** Closed circuit relative to `COM` when relay coil is de-energized.

### 6.3 Operating Parameters & Validated Timing
- **Pulse Duration:** The currently validated Door NodeMCU firmware drives an actuation pulse of approximately **500 ms**.
- **Contact Action:** The relay energizes momentarily for ~500 ms and automatically de-energizes, providing a momentary contact closure across the existing authorized door-release input/circuit.
- **Bench-Test Boundary:** During current bench testing and logic verification, **the physical laboratory door-release circuit is NOT connected to the relay screw terminals**. The relay operates in dry-bench mode to verify timing and command sequencing without switching live external loads.

### 6.4 Safety & Boundary Disclaimers
- **No Direct Motor/Solenoid Drive:** The system is strictly a low-power dry-contact signaling interface. The NodeMCU and relay do **not** supply raw DC power, high-amperage holding currents, or PWM signals to lock motors or magnetic lock coils.
- **No Unverified Isolation Claims:** The relay module is documented as a standard electromechanical dry-contact interface. No claims of specialized optoisolation or galvanic barrier ratings are made without empirical component qualification.
- **No Physical Door Lock Guarantee:** De-energizing the relay coil simply ensures the dry contacts remain in their normal rest position. It does **not** guarantee that the physical facility door remains mechanically locked; lock status depends on the external access control system and power supply architecture.

---

## 7. Local Communication Flow Architecture

The local workbench-to-door communication path is **Currently Validated & Working**:

```
+--------------------------+
|  User Presses Table Btn  |
+--------------------------+
             |
             v
+--------------------------+
|      Table NodeMCU       |
|   (D5 / ~50 ms Debounce) |
+--------------------------+
             |
             v
+--------------------------+
|     ESP-NOW Unicast      |
|   (ESP-NOW Channel 1)    |
+--------------------------+
             |
             v
+--------------------------+
|       Door NodeMCU       |
|    (MAC D8:BF:C0:D2:A9:32)|
+--------------------------+
             |
             v
+--------------------------+
| Command String Validated |
|  (TABLE_X_OPEN Match)    |
+--------------------------+
             |
             v
+--------------------------+
|     D1 Relay Output      |
|   (Energizes for ~500ms) |
+--------------------------+
             |
             v
+--------------------------+
|    5V Relay Dry Contact  |
|  (Momentary Closure)     |
+--------------------------+
             |
             v
+--------------------------+
| Existing Authorized Door |
|  Release Input / Circuit |
+--------------------------+
```

---

## 8. ESP-NOW Command Protocol Specification

### 8.1 Command Payload Table

| Transmitting Node | Hardware Role | Recognized Command Payload | Target MAC Address | Action on Door NodeMCU |
|---|---|---|---|---|
| **Table 1 NodeMCU** | Workbench 1 Controller | `TABLE_1_OPEN` | `D8:BF:C0:D2:A9:32` | Fires ~500 ms relay pulse |
| **Table 2 NodeMCU** | Workbench 2 Controller | `TABLE_2_OPEN` | `D8:BF:C0:D2:A9:32` | Fires ~500 ms relay pulse |
| **Table 3 NodeMCU** | Workbench 3 Controller | `TABLE_3_OPEN` | `D8:BF:C0:D2:A9:32` | Fires ~500 ms relay pulse |

### 8.2 Ingress Filtering Rules
1. **Command Payload Matching:** Incoming ESP-NOW packet buffers are inspected upon the MAC receive callback. The received payload must match one of the three recognized command strings.
2. **Rejection of Unrecognized or Malformed Payloads:** Any incoming frame containing unrecognized command payloads or malformed data is immediately dropped without activating the relay pin.
3. **No Intermediate Cloud Relay:** The ESP-NOW packet travels point-to-point over the local radio link with no reliance on Wi-Fi access points, router routing tables, or internet connectivity.

---

## 9. Wi-Fi & ESP-NOW Channel Coexistence Relationship

The ESP8266 contains a single 2.4 GHz radio transceiver shared between the Wi-Fi 802.11 b/g/n physical layer and the ESP-NOW MAC protocol.

### 9.1 Radio Channel Alignment
- **Laboratory Wi-Fi Channel:** 1
- **ESP-NOW Channel:** 1
- **Coexistence Mechanism:** When an ESP8266 associates as a Wi-Fi station (`STA`), its physical radio frequency locks to the channel of the connected Access Point. Because both Wi-Fi and ESP-NOW are fixed to Channel 1, the Door NodeMCU can receive local ESP-NOW frames on Channel 1 without requiring radio frequency hopping.

### 9.2 Validation Status
While the channel alignment principle is sound and configured, **concurrent dual-stack operation (simultaneous active Wi-Fi data streaming while servicing ESP-NOW interrupts) is a Planned Validation task**. The local ESP-NOW path has been bench-tested independently in standalone station mode.

---

## 10. Remote Communication Path (Planned / In Development)

The remote ingress path provides authorized cloud-based door release for mobile and desktop workstations.

```
+-----------------------------+
|   React Native / Expo App   |
|    or React / Vite Web      |
+-----------------------------+
               |
               v
+-----------------------------+
|   Firebase Authentication   |
|   (User Identity & Token)   |
+-----------------------------+
               |
               v
+-----------------------------+
| Firebase Realtime Database  |
|  (/commands/doorController) |
+-----------------------------+
               |
               v  [Over Internet / WAN]
+-----------------------------+
|     Laboratory Wi-Fi AP     |
|         (Channel 1)         |
+-----------------------------+
               |
               v  [802.11 b/g/n Station]
+-----------------------------+
|        Door NodeMCU         |
|  (Evaluates Cloud Command)  |
+-----------------------------+
               |
               v
+-----------------------------+
|     D1 Relay Actuation      |
|         (~500 ms)           |
+-----------------------------+
```

### 10.1 Planned Technologies
- **Client Applications:** Cross-platform mobile client (React Native / Expo) and browser dashboard (React 18 / Vite / Vanilla CSS).
- **Identity & Access Management:** Firebase Authentication (Email/Password with JSON Web Tokens).
- **Message Exchange:** Firebase Realtime Database.
- **Secondary Authorization:** Secondary numeric Door PIN verification to guard against unauthorized release from unlocked devices.

### 10.2 Development Status
- Mobile application, web dashboard, and Firebase remote control pathways are **Planned / Under Development**.
- The specific transport layer on the ESP8266 for Firebase Realtime Database communication will be finalized during remote implementation.
- Cloud-to-ESP8266 actuation is **not currently validated**.

---

## 11. Power Architecture & Electrical Considerations

### 11.1 Current Bench Implementation
- **Power Source:** During bench development, the NodeMCU boards are powered through their USB/VIN power interfaces using suitable regulated sources.
- **Relay Power Supply:** The 5V relay module coil draws power from the NodeMCU `VIN` pin.

### 11.2 Design Boundaries & Future Power Work
- **No Unsubstantiated Power Claims:** This specification does not assert unverified current consumption figures, sleep current measurements, or regulator/thermal specifications.
- **Future Battery Operation:** While battery-powered deployment for table nodes may be investigated in subsequent engineering phases, it is **not validated** in the current revision. All nodes currently require continuous USB power.

---

## 12. Safety, Authority & Physical Boundary Constraints

1. **Auxiliary Role:** VAJRAX Smart Door functions strictly as an auxiliary convenience release trigger. It operates in parallel with existing door access control hardware.
2. **Preservation of Existing Authoritative Security:** The system must not bypass, disable, or alter primary laboratory security measures, including physical keyed locks, deadbolts, master panic crash bars, or external biometric/fingerprint terminals.
3. **No Direct Load Switching:** The relay must interface exclusively with the laboratory's existing authorized door-release input/circuit rather than directly powering electromagnets, strike coils, or drive motors.
4. **Prerequisites for Physical Connection:** Physical wiring of the relay contacts to the laboratory door circuit shall take place **only after**:
   - Complete bench testing of all local and remote firmware.
   - Comprehensive electrical verification of voltage levels and contact ratings.
   - Formal administrative authorization from the laboratory director and facility manager.
   - Validation that the existing door system's fail-safe or fail-secure electrical behavior is compatible with the relay interface.

---

## 13. Implementation Status: Currently Validated vs. Planned

### 13.1 Currently Validated Architecture (Bench-Tested)

| Capability / Subsystem | Verification Scope | Current Operational State |
|---|---|---|
| **Table 1 Button Input** | Physical tactile button on GPIO `D5` | Validated |
| **Table 2 Button Input** | Physical tactile button on GPIO `D5` | Validated |
| **Table 3 Button Input** | Physical tactile button on GPIO `D5` | Validated |
| **`INPUT_PULLUP` Sensing** | Active `LOW` input sensing on GPIO `D5` | Validated |
| **Software Debounce** | ~50 ms stable contact filter on table buttons | Validated |
| **Safety Rate Lockout** | 2-second rate-limiting re-arm window between presses | Validated |
| **ESP-NOW RF Link** | Direct unicast transmission to MAC `D8:BF:C0:D2:A9:32` | Validated |
| **Radio Channel Alignment** | Unicast operation locked to RF Channel 1 | Validated |
| **Door NodeMCU Reception** | Ingress frame processing via MAC callback | Validated |
| **Command String Validation** | Exact matching of `TABLE_1_OPEN`, `TABLE_2_OPEN`, `TABLE_3_OPEN` | Validated |
| **Relay Output Driver** | Digital output control on GPIO `D1` (`GPIO5`) | Validated |
| **Relay Pulse Timing** | ~500 ms momentary pulse with automatic reset | Validated |
| **Relay boot/reset state** | Relay configured to remain de-energized during normal startup/reset | Validated |
| **Dry Bench Signaling** | Isolated bench validation without physical door load | Validated |

### 13.2 Planned / Under Development Architecture

| Capability / Subsystem | Target Subsystem | Implementation Status |
|---|---|---|
| **Wi-Fi Station Connection** | Door NodeMCU association to lab AP | Planned / In Development |
| **Concurrent Radio Coexistence** | Active Wi-Fi STA + ESP-NOW interrupt handling | Planned Validation |
| **Firebase Cloud Bridge** | NodeMCU subscription to Firebase Realtime Database | Planned / In Development |
| **User Authentication** | Firebase Auth (Email/Password) token management | Planned / In Development |
| **Mobile Application** | React Native / Expo cross-platform mobile client | Planned / In Development |
| **Web Dashboard** | React / Vite administrative management console | Planned / In Development |
| **Secondary Door PIN Authorization**| Additional authorization layer for remote release | Planned / In Development |
| **Multi-User Role-Based Access** | Role separation (Standard User vs. Admin) | Planned / In Development |
| **Audit Logging & History** | Cloud-synchronized door actuation activity log | Planned / In Development |
| **Device Telemetry & Heartbeat** | Periodic RSSI, uptime, and status reporting | Planned / In Development |
| **Administrative Remote Lockout** | Cloud toggle to disable remote unlock triggers | Planned / In Development |
| **Physical Facility Integration** | Interfacing relay contacts to existing authorized door-release input/circuit | Planned (Requires Authorization) |
| **Battery Power Architecture** | Low-power sleep modes and battery regulation | Future Investigation |
| **OTA Firmware Updates** | Secure network firmware flashing over Wi-Fi | Future Roadmap |

---

## 14. Testing & Verification Matrix

| Test ID | Subsystem | Test Objective | Expected Engineering Result | Verification Status |
|---|---|---|---|---|
| **TST-HW-01** | Table Controllers | Physical button press detection | GPIO `D5` transitions from `HIGH` to `LOW`; debounce ignores transients < 50 ms | **Bench Tested & Validated** |
| **TST-HW-02** | Table Controllers | Rate-limiting lockout verification | Rapid secondary button presses within 2000 ms generate no outgoing packets | **Bench Tested & Validated** |
| **TST-HW-03** | Local Wireless | ESP-NOW unicast transmission | Table nodes deliver command frame to MAC `D8:BF:C0:D2:A9:32` on Channel 1 | **Bench Tested & Validated** |
| **TST-HW-04** | Door Controller | Command string validation | Valid commands (`TABLE_X_OPEN`) trigger actuation; arbitrary strings are discarded | **Bench Tested & Validated** |
| **TST-HW-05** | Door Controller | Unrecognized payload rejection | Non-matching ESP-NOW frames produce zero relay transition or state change | **Bench Tested & Validated** |
| **TST-HW-06** | Relay Actuator | Relay output activation | GPIO `D1` switches relay coil on upon receipt of valid table command | **Bench Tested & Validated** |
| **TST-HW-07** | Relay Actuator | Momentary pulse duration | Relay coil de-energizes automatically after approximately 500 ms | **Bench Tested & Validated** |
| **TST-HW-08** | Door Controller | Relay boot/reset state verification | Relay configured to remain de-energized during normal startup/reset | **Bench Tested & Validated** |
| **TST-HW-09** | Wireless Coexistence | Wi-Fi Station association on Channel 1 | Door NodeMCU associates with AP on Channel 1 while continuing ESP-NOW reception | **Planned** |
| **TST-HW-10** | Cloud Gateway | Firebase remote command execution | Authenticated Firebase command triggers ~500 ms relay pulse via Wi-Fi | **Planned** |
| **TST-HW-11** | Facility Interface | Live physical door integration | Relay dry contacts interface to existing authorized door-release input/circuit; door releases correctly | **Planned (Requires Authorization)** |

---

## 15. Hardware Wiring & Connection Summary

### 15.1 Door NodeMCU & Relay Module
- **`VIN` (NodeMCU)** $\rightarrow$ **`VCC` / `+` (Relay Module)**
- **`GND` (NodeMCU)** $\rightarrow$ **`GND` / `-` (Relay Module)**
- **`D1` / `GPIO5` (NodeMCU)** $\rightarrow$ **`IN` / `S` (Relay Module)**
- **Relay Screw Terminals (`NC`, `COM`, `NO`):** Kept completely disconnected during bench testing.

### 15.2 Table NodeMCUs (Tables 1, 2, and 3)
- **`D5` / `GPIO14` (NodeMCU)** $\rightarrow$ **Push Button Terminal A**
- **`GND` (NodeMCU)** $\rightarrow$ **Push Button Terminal B**
- **Configuration:** Internal pull-up (`INPUT_PULLUP`) enabled in firmware.

---

## 16. Engineering Notes & Best Practices

1. **Avoid Unsupported Claims:** Technical documentation must never assert unvalidated characteristics such as galvanic isolation, microsecond latency guarantees, zero-power sleep states, or fail-secure physical perimeter states.
2. **Credential Security:** Wi-Fi credentials, Firebase service keys, and Door PIN secrets must remain in untracked local configuration files and environment variables. Never commit secrets to documentation or public repositories.
3. **Physical Isolation Before Validation:** Never connect development microcontrollers directly to building access control loops, maglocks, or strike plates until bench testing is 100% complete and administrative approval is granted.
4. **WAN Decoupling:** Firmware architecture should preserve the local table-to-door actuation path independently of internet connectivity, router availability, and cloud service availability.
5. **Separate Development Tracks:** The local RF path (ESP-NOW) and remote cloud path (Firebase) must be treated as independent subsystems. Upgrades or refactoring of cloud endpoints must never compromise local hardware safety routines.
6. **Empirical Measurement:** All performance figures (such as RF latency, startup times, or power consumption) must be documented only after formal instrumentation and test measurement in the target laboratory environment.
