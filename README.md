# Sentinel

**ESP32 Fault Recovery Engine — a standalone firmware recovery and OTA engine designed to keep ESP32 applications recoverable without physical access to the device.**

Sentinel is an embedded reliability system designed for ESP32 applications where firmware failures, crash loops, and failed deployments can otherwise leave a device requiring physical recovery.

The goal is simple:

> **If an ESP32 application breaks, the device should still have a way to recover itself.**

Sentinel runs as a background recovery engine alongside the user's application. It monitors boot/crash state, provides a recovery environment, stores persistent diagnostic information, provisions networking credentials, and can perform OTA firmware updates even when the main application cannot boot successfully.

The project was designed as a reusable foundation that can eventually be integrated into other ESP32-based IoT applications rather than being tied to one specific application.

---

## Overview

A typical embedded IoT device might look like:

```text
                 Device Boot
                      │
                      ▼
              ┌───────────────┐
              │    Sentinel   │
              │ Recovery Engine│
              └───────┬───────┘
                      │
              Boot/Crash Check
                      │
            ┌─────────┴─────────┐
            │                   │
       Normal Boot          Crash Loop
            │                   │
            ▼                   ▼
     User Application       Safe Mode
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
           Diagnostics        CLI             Forced OTA
              │                 │                 │
              └─────────────────┴─────────────────┘
                                │
                                ▼
                         Recovered Device
```

Sentinel is designed so that the recovery system remains available even when the primary application is not.

---

## Why Sentinel?

Firmware failures are fundamentally different from failures in ordinary software.

If a deployed web application crashes, you can usually redeploy it remotely.

If an embedded device enters a permanent boot loop, the device may become inaccessible.

That can mean:

- Physical retrieval
- Serial connection
- Manual reflashing
- Replacing the device
- An unhappy customer

Sentinel addresses this problem by putting a recovery mechanism below the application layer.

The application can fail.

This is where sentinel comes in.

---
### Key Features
- Autonomous boot-loop detection
- Persistent crash and boot counters
- Safe-mode recovery environment
- Forced OTA recovery
- HTTPS firmware updates
- SHA-256 firmware hashing
- A/B OTA partitioning
- NVS-based persistent configuration
- NimBLE BLE provisioning
- Custom BLE GATT profile
- Wi-Fi provisioning
- Custom recovery CLI / REPL
- Core-dump diagnostics
- Reset/reboot controls
- OTA force command
- Custom ESP-IDF partition layout
- Unity test suite
- OTA backend built with Spring Boot
- Spring Security
- Binary firmware streaming
- Docker-based backend deployment

---

## Architecture

Sentinel is divided into several major components.
```
┌───────────────────────────────────────────────────────┐
│                     Sentinel                          │
│                                                       │
│  ┌─────────────┐       ┌──────────────────────────┐  │
│  │ Boot Monitor│──────▶│    Recovery Manager      │  │
│  └─────────────┘       └────────────┬─────────────┘  │
│                                     │                │
│              ┌──────────────────────┼────────────┐   │
│              │                      │            │   │
│              ▼                      ▼            ▼   │
│          Safe Mode              OTA Engine      CLI  │
│              │                      │            │   │
│              ▼                      ▼            ▼   │
│             NVS                  HTTPS        Diagnostics
│                                     │                │
│                                     ▼                │
│                               Spring Boot Server     │
└───────────────────────────────────────────────────────┘
```
### Boot Loop Detection

Sentinel checks persistent boot state immediately after startup.

Boot and crash counters are stored in NVS so the information survives resets and power cycles.

Conceptually:

```Boot
 │
 ▼
Read boot count
 │
 ├── Normal boot history ──────▶ Start application
 │
 └── Crash/boot-loop threshold ▶ Safe Mode

```
This allows Sentinel to distinguish between an ordinary reboot and a device that repeatedly fails to start.

Persistent state includes:

- Boot count
- Crash count
- Wi-Fi credentials
- Server address

This information remains available across device resets.

---
### Safe Mode

Safe Mode is the primary recovery environment.

If Sentinel detects that the main application is repeatedly failing, it can prevent the device from continuously restarting the broken application and instead expose a recovery interface.

The Safe Mode environment provides:

- Device diagnostics
- Boot/reset information
- Crash information
- Recovery commands
- Forced OTA updates

The objective is to make the device recoverable even when the main application is not.

---
### Safe Mode Main Menu

The recovery menu displays information useful for diagnosing the device.

Example information includes:
```
Wake Reason
Reset Reason
Boot Count
Crash Count
```
This allows the user to determine why the device entered recovery.

---
### Recovery CLI

Sentinel includes a custom CLI/REPL for recovery operations.

Example commands include:
```
clear
crash_report
ota_force
reset

```
`clear`

Clears persistent boot and crash counters.

`crash_report`

Triggers/display a core-dump summary for debugging.

`ota_force`

Starts a firmware update directly from Safe Mode.

`reset`

Reboots the device.

The CLI is intentionally lightweight so recovery operations remain accessible even when the main application is unavailable.
---

###Forced OTA Recovery

This is one of Sentinel's most important features.

Normally, an OTA update would be initiated by the running application.

But what happens when the application cannot run?

Sentinel provides a separate recovery path.
```
Application
    │
    │ crash loop
    ▼
Safe Mode
    │
    ▼
Read stored credentials from NVS
    │
    ▼
Initialize Wi-Fi
    │
    ▼
Connect to OTA server
    │
    ▼
Download firmware
    │
    ▼
Write inactive OTA partition
    │
    ▼
Verify firmware
    │
    ▼
Switch boot partition
    │
    ▼
Reboot
    │
    ▼
Recovered application
```
The key property is that this process does not require:

- The main application to boot
- The normal application task system
- Re-provisioning the device
- A physical serial connection

Previously stored credentials allow Sentinel to establish connectivity directly from Safe Mode.

This provides an autonomous recovery path for devices stuck in a crash or boot loop.

--- 

### OTA Engine

Sentinel implements the OTA update path directly on the ESP32 side.

The OTA process:

1. Configures the HTTPS client
2. Connects to the Spring Boot server
3. Requests the latest firmware
4. Streams the firmware binary
5. Writes the binary to the inactive OTA partition
6. Validates the downloaded image
7. Switches the active OTA slot
8. Reboots the device

Conceptually:
```
HTTPS Server
     │
     │ firmware binary
     ▼
┌─────────────┐
│ HTTPS Client│
└──────┬──────┘
       │
       ▼
 Firmware Stream
       │
       ▼
┌───────────────┐
│ Inactive OTA  │
│   Partition   │
└──────┬────────┘
       │
       ▼
 Image Validation
       │
       ▼
 Switch OTA Slot
       │
       ▼
     Reboot
```
---
### Firmware Integrity

Firmware integrity is checked using SHA-256 hashing.

The backend can calculate a hash for the available firmware binary, while the device can use the expected hash to verify the downloaded image.

This provides an additional integrity check before deploying a new firmware image.

The purpose is to ensure that the firmware being written is the expected binary rather than an incomplete or corrupted download.
---
### OTA Partitioning

Sentinel uses a custom partitions.csv configuration.

The partition layout provides dedicated space for:

- Factory application
- OTA application slot 1
- OTA application slot 2
- NVS
- Core dumps
- Additional application/recovery storage

The A/B OTA layout allows the device to maintain an alternate firmware image while updating the inactive slot.

Conceptually:
```
Flash
┌───────────────────────────────┐
│ Bootloader                    │
├───────────────────────────────┤
│ Partition Table               │
├───────────────────────────────┤
│ NVS                           │
├───────────────────────────────┤
│ Factory Application           │
├───────────────────────────────┤
│ OTA Slot 0                    │
├───────────────────────────────┤
│ OTA Slot 1                    │
├───────────────────────────────┤
│ Core Dump / Recovery Storage  │
└───────────────────────────────┘
```
---
### NVS Persistence

Sentinel uses ESP32 NVS for persistent device state.

Stored information includes:

- Wi-Fi SSID
- Wi-Fi password
- OTA server address
- Boot count
- Crash count

Because this information survives resets, Sentinel can use previous configuration to recover a device without requiring the user to provision it again.

This is particularly important for forced OTA recovery.

---

### BLE Provisioning

Initial configuration uses Bluetooth Low Energy provisioning through NimBLE.

Sentinel implements a custom BLE GATT profile for transferring configuration information.

The provisioning flow is:
```
Phone
 │
 │ BLE
 ▼
NimBLE GATT
 │
 ▼
Sentinel Provisioning
 │
 ├── Wi-Fi SSID
 ├── Wi-Fi Password
 └── Server Address
 │
 ▼
NVS
```

The configuration is then persisted so that future recovery sessions can reuse the stored credentials.

The BLE provisioning subsystem was based on experience from ARES-32, allowing the implementation to focus more heavily on integration with the recovery architecture.
---
### Backend

Sentinel includes a Spring Boot backend responsible for firmware delivery.

The backend provides the OTA endpoint used by the ESP32 to retrieve firmware binaries.

The backend includes:

- Spring Boot
- Spring Security
- Firmware discovery
- Binary streaming
- SHA-256 hashing
- HTTP endpoint authorization

Firmware binaries are streamed in chunks rather than loading the entire firmware image into memory at once.

Firmware Streaming

The backend locates the latest firmware binary and streams it through the OTA endpoint.

Conceptually:
```
Firmware Directory
       │
       ▼
Latest Binary
       │
       ▼
8 KB Chunks
       │
       ▼
HTTP Response Stream
       │
       ▼
ESP32 HTTPS Client
```
This allows large firmware images to be transferred without requiring the backend to load the entire binary into memory.

---
### Security

Sentinel's OTA communication uses HTTPS.

The backend also includes Spring Security configuration for controlling access to the OTA endpoint.

Firmware integrity is additionally checked using SHA-256 hashing.

The security architecture is intended to prevent the OTA system from becoming an unauthenticated firmware distribution endpoint.
---

### Testing

Sentinel includes a Unity-based firmware test suite for testing important engine behavior.

Testing focuses on both normal and failure paths.

Examples include:

- Boot-loop detection
- Recovery behavior
- Configuration handling
- OTA behavior
- Error handling
- Recovery-state transitions

Additional mocking and test coverage are planned for future versions.

---

### Development Roadmap

Sentinel is being developed incrementally.

v1.0.0

Initial recovery engine:

- Boot-loop detection
- NVS persistence
- Safe Mode
- Recovery CLI
- BLE provisioning
- Wi-Fi configuration
- HTTPS OTA
- Forced OTA recovery
- Spring Boot OTA backend
- SHA-256 firmware validation
- v1.1.0

Backend and deployment improvements:

- Docker
- Improved server deployment
- CI/CD
- Additional reliability work

v1.2.0

Testing improvements:

- Expanded Unity test coverage
- More edge-case testing
- Additional recovery-path validation

v1.3.0

Firmware quality improvements:

- CMock
- More extensive component isolation
- Expanded failure-path testing
- Additional edge cases

The roadmap intentionally prioritizes reliability and maintainability over continuously adding new features.

---

###Example Recovery Flow

A device experiencing repeated crashes might follow this sequence:
```
1. Device boots
        │
2. Sentinel checks persistent boot state
        │
3. Boot loop detected
        │
4. Main application is not started
        │
5. Safe Mode launches
        │
6. User selects `sentinel ota force`
        │
7. Sentinel retrieves stored credentials
        │
8. Wi-Fi starts
        │
9. HTTPS connection established
        │
10. Firmware downloaded
        │
11. Firmware written to inactive OTA slot
        │
12. Firmware validated
        │
13. OTA slot switched
        │
14. Device reboots
        │
15. New firmware starts
```
The important property is that the broken application never needs to participate in its own recovery.
---
### Why This Project Exists

Sentinel was designed around a problem common to remotely deployed embedded systems:

What happens when the firmware responsible for running the device is the exact firmware that has failed?

The solution is to move recovery functionality into a separate layer that can survive application failures.

Sentinel therefore treats recovery as a first-class embedded system rather than as an afterthought.
---
### Technology
| Area               | Technology               |
| ------------------ | ------------------------ |
| MCU                | ESP32-S3                 |
| Language           | C                        |
| Framework          | ESP-IDF                  |
| RTOS               | FreeRTOS                 |
| Provisioning       | NimBLE / BLE             |
| Networking         | Wi-Fi                    |
| OTA Transport      | HTTPS                    |
| Persistence        | NVS                      |
| Firmware Integrity | SHA-256                  |
| Testing            | Unity                    |
| Backend            | Java / Spring Boot       |
| Backend Security   | Spring Security          |
| Firmware Streaming | HTTP chunked streaming   |
| Deployment         | Docker / CI/CD (roadmap) |

---

### Design Philosophy

Sentinel is intentionally designed as a standalone recovery engine.

The ideal future workflow is:
```
Developer's ESP32 Application
             │
             ▼
        Add Sentinel
             │
             ▼
   Configure device/server
             │
             ▼
      Build Application
             │
             ▼
       Deploy Firmware
```
The application developer should not need to implement their own:

- Boot-loop detection
- Recovery environment
- Persistent crash tracking
- OTA recovery
- Firmware retrieval
- Safe-mode networking
- Recovery CLI

Sentinel should handle those responsibilities independently.

---
### Future Tooling

A future development goal is to simplify firmware deployment with a small command-line or Python utility.

For example:

`python sentinel.py add-firmware build/app.bin`

The tool could automatically:

- Locate the firmware binary
- Copy it into the appropriate server directory
- Calculate the expected hash
- Update firmware metadata
- Prepare the OTA server for deployment

The goal is to make Sentinel usable as a reusable developer tool rather than requiring users to understand its internal firmware-distribution structure.
