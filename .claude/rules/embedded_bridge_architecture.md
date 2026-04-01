---
description: Embedded Bridge Architecture Standards for STM32, Raspberry Pi, FastDDS, and ROS2
---

# Embedded Bridge Architecture Standards

This rule file defines the preferred architecture for a differential-drive robot built around:

- **STM32F407** for real-time motor and sensor control
- **Raspberry Pi** as a transparent transport bridge
- **Laptop ROS2 workspace** for decoding packets and exposing ROS2 APIs
- **FastDDS** as the network transport between Raspberry Pi and Laptop

## Canonical Topology

```text
STM32F407 <-> UART <-> Raspberry Pi bridge <-> FastDDS <-> Laptop ROS2
```

## Responsibility Boundaries

### STM32

- Owns hard real-time control loops.
- Owns encoder acquisition, IMU acquisition, motor actuation, buzzer, LED, and other MCU peripherals.
- Encodes and decodes the binary packet protocol.
- Must not depend on ROS2 concepts, ROS message schemas, or FastDDS-specific types.

### Raspberry Pi Bridge

- Forwards raw bytes between UART and FastDDS.
- May manage transport reliability concerns such as reconnect, buffering, and heartbeat forwarding.
- Must not decode packet semantics into business fields.
- Must not publish ROS2 domain topics derived from packet contents.
- Must not re-map commands, clamp motion values, or alter packet payload meaning.

### Laptop ROS2 Layer

- Owns packet decoding and encoding at the application boundary.
- Maps protocol messages to ROS2 topics, services, or actions.
- Performs validation, unit conversion, safety policy, and application orchestration.
- Is the only layer allowed to understand both the binary protocol and ROS2 domain semantics.

## Package Roles

The preferred workspace structure for this architecture is:

```text
velocity_control/
├── library/
│   ├── serial_comm/         # Packet schema, encoder, decoder, CRC, protocol constants
│   ├── fastdds_comm/        # FastDDS transport abstraction used by bridge and laptop
│   └── linux_comm_driver/   # Linux UART driver wrapper
├── velocity_server/         # Raspberry Pi byte-forwarding bridge
└── ros2_bridge/             # Laptop ROS2 adapter between packets and topics
```

### Ownership Rules

- `serial_comm` is the source of truth for packet formats.
- `fastdds_comm` transports opaque payloads and transport metadata only.
- `linux_comm_driver` exposes byte streams and port state, not protocol semantics.
- `velocity_server` is infrastructure only.
- `ros2_bridge` contains ROS2 adapters and packet-to-topic mapping.

## Clean Architecture Mapping

Within the Laptop ROS2 side, prefer this dependency direction:

```text
ROS2 node adapters -> application services -> domain models/interfaces
                                     |
                                     -> protocol adapters using serial_comm
```

### Rules

1. Domain models must not import ROS2 message types, FastDDS APIs, or UART drivers.
2. Application services may depend on domain abstractions and protocol translation ports.
3. Infrastructure adapters may depend on ROS2, FastDDS, UART, and generated interfaces.
4. Packet DTOs and ROS2 messages must be translated at boundaries, never leaked inward.

## Protocol Contract Standards

### Packet Schema

Every packet should define at least:

- protocol version
- message type
- sequence number
- payload length
- payload bytes
- CRC or checksum
- timestamp source expectations if timing matters

### Versioning

- Increment protocol version when payload meaning changes.
- Additive payload changes should keep backward compatibility when possible.
- Reject unsupported protocol versions explicitly and log the reason.

### Message Categories

Preferred categories:

- telemetry: encoder ticks, wheel speed, IMU, battery, board state
- command: velocity, buzzer, LED, mode switching
- lifecycle: heartbeat, boot, ready, error, firmware version
- diagnostics: CRC failure count, timeout count, transport state

## ROS2 Mapping Standards

### Topic Naming

Use descriptive ROS2 topics on the Laptop side:

```text
/robot/control/cmd_vel
/robot/control/buzzer
/robot/control/led
/robot/state/odometry
/robot/state/joint_states
/robot/state/battery
/robot/sensors/imu/data
/robot/diagnostics/transport
```

### Message Mapping Rules

- Packet decoding must happen before ROS2 publication.
- Unit conversion must be explicit and centralized.
- Encoder ticks should not be published directly when a physical unit topic is expected.
- Raw packet topics are allowed only for debugging and recording.
- If both raw and decoded topics exist, name them clearly and document ownership.

### QoS Guidance

- Use `RELIABLE` for commands and state required by control loops.
- Use `BEST_EFFORT` for high-rate sensor streams when loss is acceptable.
- Use `TRANSIENT_LOCAL` only for latched configuration or static state.
- Do not mirror UART or FastDDS buffering assumptions directly into ROS2 QoS without justification.

### Topic vs Service vs Action

- Use topics for continuous command streams such as `cmd_vel`.
- Use services for discrete request-response operations such as LED mode set, buzzer pattern set, or firmware query.
- Use actions only on the Laptop ROS2 side for long-running behaviors such as rotate, dock, or navigate.
- Do not tunnel ROS2 service or action semantics through the Raspberry Pi bridge as a special transport mode.
- Convert services or actions into protocol packets at the Laptop boundary.

## Behavioral Rules

### Raspberry Pi Bridge Behavior

- Forward bytes as received.
- Preserve ordering.
- Surface transport status separately from payload data.
- Reconnect UART and DDS endpoints without changing packet contents.
- Never deserialize payload into robot state objects.

### Laptop ROS2 Bridge Behavior

- Decode raw packets into typed protocol messages.
- Validate CRC, version, payload size, and message type.
- Publish decoded ROS2 topics.
- Subscribe ROS2 control topics and encode them back into protocol packets.
- Expose services or actions only when the behavior is meaningful at the ROS2 application boundary.
- Centralize safety checks before command encoding.

### STM32 Behavior

- Treat encoded command packets as the control contract.
- Publish telemetry at deterministic rates.
- Surface explicit fault packets instead of silent failure when feasible.

## Error Handling Standards

- Count and expose CRC failures.
- Count and expose packet decode failures.
- Detect stale telemetry with explicit timeout thresholds.
- Differentiate transport loss from payload corruption.
- Publish transport diagnostics without blocking the forwarding path.

## Testing Requirements

### serial_comm

- Unit test framing, CRC, decode, malformed packet rejection, and version compatibility.

### velocity_server

- Integration test UART to FastDDS forwarding with fake endpoints.
- Verify bytes are preserved exactly.
- Verify reconnect behavior and backpressure handling.

### ros2_bridge

- Unit test packet-to-topic mapping and topic-to-packet encoding.
- Integration test ROS2 publications from recorded packets.
- Integration test command path from subscribed topics to encoded packets.

### End-to-End

- Verify `cmd_vel` reaches STM32 as the correct packet type and units.
- Verify encoder and IMU packets reach ROS2 topics with correct frame IDs and units.
- Verify timeout and disconnection diagnostics are published.

## Anti-Patterns

- Decoding packet business fields on Raspberry Pi.
- Importing ROS2 messages into protocol libraries.
- Reusing ROS2 message definitions as the binary packet schema.
- Letting ROS2 callback code directly manipulate UART framing details.
- Mixing safety policy into the raw byte-forwarding bridge.
- Hiding packet version mismatches behind generic transport errors.

## Implementation Checklist

Before adding a new feature, define:

1. Which component owns the feature: STM32, Raspberry Pi, or Laptop.
2. The packet contract and versioning impact.
3. The ROS2 interface exposed on the Laptop side.
4. Validation and timeout behavior.
5. Unit and integration tests for each affected layer.