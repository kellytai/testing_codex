# Pixhawk 6C + Teensy 4.1 (UART/MAVLink, ArduPilot) Plan and Example Code

This document provides a concrete plan for connecting a Pixhawk 6C running ArduPilot to a Teensy 4.1 over UART/MAVLink, along with example code for both sides.

## 1) Hardware + Wiring Plan (UART/MAVLink)

**Primary link:** Pixhawk 6C TELEM port ↔ Teensy 4.1 UART (Serial).

**Key points**
- **Logic levels:** Pixhawk 6C and Teensy 4.1 are 3.3V logic. No level shifting needed.
- **Baud rate:** Start at **57600** or **115200**.
- **Ground:** Always connect grounds (GND ↔ GND).
- **Power:** Power Teensy via its own 5V/USB or regulated 5V; avoid drawing from Pixhawk unless the port is rated for it.

**Example wiring (TELEM1):**
- Pixhawk TELEM1 **TX** → Teensy **RX** (e.g., Serial1 RX pin)
- Pixhawk TELEM1 **RX** ← Teensy **TX** (e.g., Serial1 TX pin)
- Pixhawk **GND** ↔ Teensy **GND**

## 2) ArduPilot Firmware Configuration (Pixhawk)

Use Mission Planner or MAVProxy to set:
- `SERIAL1_PROTOCOL = 2` (MAVLink2)
- `SERIAL1_BAUD = 57` or `115` (57,600 or 115,200)

Confirm telemetry on the selected TELEM port and reboot.

## 3) MAVLink Message Plan

Minimal message set for control:
- **HEARTBEAT** (Pixhawk ↔ Teensy) for link status
- **SET_MODE** (Teensy → Pixhawk) to arm and set flight mode
- **COMMAND_LONG** for arming and other commands
- **RC_CHANNELS_OVERRIDE** or **MANUAL_CONTROL** to send control inputs
- **ATTITUDE / LOCAL_POSITION_NED** telemetry for feedback

## 4) Teensy 4.1 Example Code (UART MAVLink)

This is a concise example showing how to:
- Receive MAVLink messages from Pixhawk
- Send a HEARTBEAT
- Arm/disarm and send a simple roll/pitch/yaw/throttle command

> **Note:** This uses the MAVLink C library generated for ArduPilot. You can use `pymavlink` to generate C headers or include a pre-generated MAVLink library.

```cpp
// Teensy 4.1 MAVLink UART example (Serial1)
#include <Arduino.h>
#include "mavlink/common/mavlink.h"

static const uint8_t SYS_ID = 200;    // Teensy system ID
static const uint8_t COMP_ID = 190;   // Companion computer
static const uint8_t TARGET_SYS = 1;  // Pixhawk system ID
static const uint8_t TARGET_COMP = 1; // Pixhawk component ID

void sendHeartbeat() {
  mavlink_message_t msg;
  uint8_t buf[MAVLINK_MAX_PACKET_LEN];
  mavlink_msg_heartbeat_pack(
      SYS_ID, COMP_ID, &msg,
      MAV_TYPE_GCS,
      MAV_AUTOPILOT_INVALID,
      0, 0, MAV_STATE_ACTIVE);

  uint16_t len = mavlink_msg_to_send_buffer(buf, &msg);
  Serial1.write(buf, len);
}

void sendArmCommand(bool arm) {
  mavlink_message_t msg;
  uint8_t buf[MAVLINK_MAX_PACKET_LEN];
  mavlink_msg_command_long_pack(
      SYS_ID, COMP_ID, &msg,
      TARGET_SYS, TARGET_COMP,
      MAV_CMD_COMPONENT_ARM_DISARM,
      0,                    // confirmation
      arm ? 1.0f : 0.0f,     // param1: 1 to arm, 0 to disarm
      0, 0, 0, 0, 0, 0);     // param2-7

  uint16_t len = mavlink_msg_to_send_buffer(buf, &msg);
  Serial1.write(buf, len);
}

void sendManualControl(int16_t x, int16_t y, int16_t z, int16_t r) {
  mavlink_message_t msg;
  uint8_t buf[MAVLINK_MAX_PACKET_LEN];
  mavlink_msg_manual_control_pack(
      SYS_ID, COMP_ID, &msg,
      TARGET_SYS,
      x, y, z, r,
      0); // buttons bitmask

  uint16_t len = mavlink_msg_to_send_buffer(buf, &msg);
  Serial1.write(buf, len);
}

void setup() {
  Serial1.begin(115200);
  delay(1000);
  sendHeartbeat();
}

void loop() {
  // Read MAVLink messages
  mavlink_message_t msg;
  mavlink_status_t status;
  while (Serial1.available()) {
    uint8_t c = Serial1.read();
    if (mavlink_parse_char(MAVLINK_COMM_0, c, &msg, &status)) {
      if (msg.msgid == MAVLINK_MSG_ID_HEARTBEAT) {
        // Link OK; add your logic here
      }
    }
  }

  // Example: send manual control (range -1000..1000)
  sendManualControl(0, 0, 500, 0); // throttle mid
  delay(50);
}
```

## 5) Pixhawk Side: ArduPilot Companion Control Notes

ArduPilot accepts companion computer control via MAVLink:
- **Arming:** `MAV_CMD_COMPONENT_ARM_DISARM`
- **Mode change:** `MAV_CMD_DO_SET_MODE` or `SET_MODE`
- **RC override:** `RC_CHANNELS_OVERRIDE`
- **Manual control:** `MANUAL_CONTROL`

Ensure safety:
- Disable arming checks only for bench tests.
- Use a safety switch and hardware failsafe.

## 6) Integration Checklist

1. Verify wiring, grounds, and baud rate.
2. Confirm heartbeat received both ways.
3. Validate arming and mode change on a bench.
4. Test simple manual control with props removed.
5. Add closed-loop control using telemetry feedback.
