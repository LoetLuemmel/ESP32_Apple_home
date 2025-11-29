# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an ESP32-C3 Matter Light implementation that enables control of an onboard LED through Apple Home via the Matter protocol over WiFi. The device acts as an On/Off Light (Matter Device Type 0x0100) using BLE for commissioning.

## Build System & Environment

### Required Environment Setup

This project requires **ESP-IDF v5.1.2** and **ESP-Matter SDK** to be installed and activated:

```bash
# Activate ESP-IDF
source ~/esp/esp-idf/export.sh

# Activate ESP-Matter
source ~/esp/esp-matter/export.sh
```

**CRITICAL**: Both environments must be active before any build commands. The build will fail without these.

### Build Commands

```bash
# Set target (first time only)
idf.py set-target esp32c3

# Build project
idf.py build

# Or use automation script
./build.sh

# Clean build
rm -rf build && idf.py build

# Complete erase (factory reset)
idf.py erase-flash
```

### Flash & Monitor

```bash
# Flash and monitor (auto-detect port on macOS/Linux)
./flash.sh

# Manual flash with specific port
idf.py -p /dev/cu.usbserial-XXXX flash monitor

# Monitor only
idf.py -p /dev/cu.usbserial-XXXX monitor

# Exit monitor: Ctrl+]
```

Common serial ports:
- macOS: `/dev/cu.usbserial-*` or `/dev/cu.SLAB_USBtoUART`
- Linux: `/dev/ttyUSB0`

## Architecture

### Matter Device Hierarchy

```
Matter Node (root)
└── On/Off Light Endpoint (endpoint_id: 1)
    ├── Basic Information Cluster (device metadata)
    ├── On/Off Cluster (0x0006)
    │   └── OnOff Attribute (Boolean)
    └── Identify Cluster (for pairing identification)
```

### Code Flow

1. **Initialization** (`app_main()`):
   - Initialize NVS flash (stores WiFi credentials and Matter data)
   - Initialize LED GPIO driver
   - Create Matter node with callbacks
   - Create On/Off Light endpoint
   - Start Matter stack
   - Open commissioning window (BLE)

2. **Attribute Updates** (`app_attribute_update_cb()`):
   - PRE_UPDATE: Called before attribute changes
   - POST_UPDATE: Called after attribute changes - **this is where LED control happens**
   - Checks for OnOff Cluster (0x0006) and OnOff Attribute
   - Directly controls GPIO based on boolean value

3. **Event Handling** (`app_event_cb()`):
   - Monitors Matter stack events (IP changes, commissioning status, etc.)
   - Logs important state transitions

### Key Constants

- **LED GPIO**: `GPIO_NUM_8` (defined in `main/app_main.cpp:27`)
  - Common for ESP32-C3-DevKitM-1
  - May need adjustment for different boards (GPIO 2, 18, or 19)

- **Matter Clusters**:
  - OnOff Cluster ID: `0x0006`
  - OnOff Attribute ID: `0x0000`

## Configuration Files

- **`sdkconfig.defaults`**: ESP32-C3 specific config (Bluetooth, WiFi, partitions)
- **`partitions.csv`**: Flash layout with dual-OTA support
  - nvs: 24KB (credentials storage)
  - app0/app1: 1.5MB each (dual OTA partitions)
  - fctry: 24KB (factory settings)

## Apple Home Commissioning

After flashing, the device automatically opens a commissioning window. The serial monitor will display:

```
I CHIP: [SVR] Manual pairing code: 34970112332
```

This 11-digit code is used in the Home App to pair the device. The commissioning flow:
1. BLE discovery and connection
2. Passcode verification
3. WiFi credential provisioning
4. Matter operational certificate exchange
5. Device control via WiFi

## Common Modifications

### Changing LED GPIO

Edit `main/app_main.cpp:27`:
```cpp
#define LED_GPIO GPIO_NUM_X  // Replace X with your GPIO number
```

### Adding Device Metadata

Before `node::create()` in `app_main()`:
```cpp
node_config.root_node.basic_information.vendor_name = "YourBrand";
node_config.root_node.basic_information.product_name = "Your Product";
```

### Upgrading to Dimmable Light

Change from `on_off_light::create()` to `dimmable_light::create()` and implement PWM control in the attribute callback for Level Control Cluster (0x0008).

## Important Notes

- **No state persistence**: LED state is not saved to NVS. After reboot, the device starts with LED off. Matter controller (Apple Home) will re-sync state.
- **Active-HIGH LED**: `gpio_set_level(LED_GPIO, 1)` turns LED on
- **BLE + WiFi**: Device uses BLE only for initial commissioning, then operates over WiFi
- **OTA Ready**: Dual-partition layout supports OTA updates via Matter controller

## File Reference

- **`main/app_main.cpp`**: Core application logic (206 lines)
- **`CMakeLists.txt`**: Project build configuration with Matter integration
- **`sdkconfig.defaults`**: ESP32-C3 hardware configuration
- **`partitions.csv`**: Flash memory layout
- **`build.sh`**: Automated build with environment checks
- **`flash.sh`**: Automated flash with port auto-detection
- **`PROJECT_STATUS.md`**: Complete project status and implementation checklist

## Resources

- ESP-IDF v5.1.2 Documentation: https://docs.espressif.com/projects/esp-idf/
- ESP-Matter SDK: https://github.com/espressif/esp-matter
- Matter Specification: https://csa-iot.org/all-solutions/matter/
