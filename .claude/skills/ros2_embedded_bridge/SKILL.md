---
name: ROS2 Embedded Bridge
description: Standards and templates for STM32, Raspberry Pi, FastDDS, and ROS2 bridge systems
---

# ROS2 Embedded Bridge Skill

Use this skill when the robot architecture is split across:

- an embedded controller that owns hardware and real-time control
- a Linux bridge device that forwards bytes or frames
- a ROS2 workstation that exposes application-facing topics and control APIs

This skill is optimized for the following topology:

```text
STM32F407 <-> UART <-> Raspberry Pi <-> FastDDS <-> Laptop ROS2
```

## Design Goal

Keep transport concerns, protocol concerns, and ROS2 concerns separated so that:

- the Raspberry Pi remains a transparent bridge
- the Laptop owns ROS2 semantics
- the MCU owns deterministic hardware control

## Recommended Workspace Layout

```text
velocity_control/
├── library/
│   ├── serial_comm/
│   │   ├── include/ or serial_comm/
│   │   ├── packet_types
│   │   ├── encoder_decoder
│   │   └── crc
│   ├── fastdds_comm/
│   │   ├── publisher_subscriber wrappers
│   │   └── transport session management
│   └── linux_comm_driver/
│       ├── uart driver
│       └── reconnect and buffering utilities
├── velocity_server/
│   └── raw transport bridge for Raspberry Pi
└── ros2_bridge/
    ├── domain/
    ├── application/
    └── infrastructure/ros2/
```

## Boundary Rules

### serial_comm

- Define packet headers, payload layouts, message identifiers, CRC, and decode errors.
- Stay independent from ROS2 and FastDDS.

### velocity_server

- Subscribe to FastDDS raw payload channel.
- Write bytes to UART.
- Read bytes from UART.
- Publish bytes to FastDDS raw payload channel.
- Expose transport state separately.
- Do not decode application payloads.

### ros2_bridge

- Decode telemetry packets.
- Publish ROS2 topics.
- Subscribe ROS2 control topics.
- Encode outgoing command packets.
- Apply application validation and safety logic.

## Canonical Data Paths

### Telemetry Path

```text
STM32 telemetry packet
-> UART
-> Raspberry Pi forwards bytes
-> FastDDS raw payload
-> Laptop decodes packet
-> ROS2 topics
```

### Command Path

```text
ROS2 control topic
-> Laptop validates and encodes packet
-> FastDDS raw payload
-> Raspberry Pi forwards bytes
-> UART
-> STM32 decodes command
```

## Suggested Interfaces

### Domain Entities

Use pure domain models for data the rest of the ROS2 application cares about.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class WheelOdometry:
    left_velocity_mps: float
    right_velocity_mps: float
    left_ticks: int
    right_ticks: int
    stamp_sec: float


@dataclass(frozen=True)
class ImuSample:
    angular_velocity_xyz: tuple[float, float, float]
    linear_acceleration_xyz: tuple[float, float, float]
    stamp_sec: float
```

### Protocol Models

Keep protocol objects close to `serial_comm`.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class PacketEnvelope:
    version: int
    message_type: int
    sequence: int
    payload: bytes
```

### Application Ports

```python
from abc import ABC, abstractmethod


class PacketTransport(ABC):
    @abstractmethod
    def send(self, packet: bytes) -> None:
        raise NotImplementedError


class TelemetryPublisher(ABC):
    @abstractmethod
    def publish_odometry(self, odometry) -> None:
        raise NotImplementedError
```

## Recommended ROS2 Interfaces

Prefer a narrow public ROS2 surface:

- `/robot/control/cmd_vel`
- `/robot/control/buzzer`
- `/robot/control/led`
- `/robot/state/odometry`
- `/robot/state/joint_states`
- `/robot/sensors/imu/data`
- `/robot/diagnostics/transport`
- `/robot/debug/raw_rx`
- `/robot/debug/raw_tx`

Debug topics should be optional and disabled by default in production.

## Topic, Service, and Action Policy

- Use topics for streaming control and telemetry.
- Use services for request-response features that are not time-critical streams.
- Use actions for long-running Laptop-side behaviors that need feedback and cancellation.
- Convert ROS2 services and actions into normal protocol packets before transport.
- Do not implement ROS2 services or actions inside the Raspberry Pi bridge.

## Adapter Pattern

### Incoming Telemetry Adapter

1. Receive raw payload from transport.
2. Decode `PacketEnvelope`.
3. Validate CRC, version, and type.
4. Convert payload to a protocol DTO.
5. Convert DTO to a domain entity.
6. Publish ROS2 messages.

### Outgoing Command Adapter

1. Receive ROS2 command.
2. Validate ranges and safety state.
3. Convert command to protocol DTO.
4. Encode DTO into packet bytes.
5. Send raw payload through transport.

## Transport Diagnostics

Track these states explicitly:

- UART connected or disconnected
- FastDDS connected or disconnected
- last RX timestamp
- last TX timestamp
- CRC failure count
- decode error count
- dropped packet count if measurable

Publish them as a dedicated diagnostics topic instead of overloading functional topics.

## Rules For New Features

When adding a new feature such as horn, LED, or mode switching:

1. Add packet definitions to `serial_comm` first.
2. Add encode and decode tests.
3. Update STM32 firmware handler.
4. Keep Raspberry Pi bridge unchanged unless transport framing changes.
5. Add ROS2 adapter mapping in `ros2_bridge`.
6. Add integration tests for the command and telemetry path.

## Example Feature Ownership

### `cmd_vel`

- ROS2 topic exists only on Laptop.
- Packet encoding exists in Laptop and STM32.
- Raspberry Pi forwards bytes only.
- Wheel kinematics enforcement belongs to STM32 or Laptop application logic, but not the raw bridge.

### IMU telemetry

- Raw sensor capture belongs to STM32.
- Packet structure belongs to `serial_comm`.
- ROS2 `sensor_msgs/msg/Imu` publication belongs to Laptop.
- Frame ID assignment belongs to Laptop ROS2 configuration.

## Testing Matrix

### Protocol

- valid frame decode
- CRC failure rejection
- unsupported version rejection
- partial frame buffering
- multi-frame stream decode

### Bridge

- UART to DDS byte preservation
- DDS to UART byte preservation
- reconnect without process restart
- no payload mutation

### ROS2 Adapter

- packet to topic mapping
- topic to packet encoding
- timeout diagnostics
- invalid command rejection

### End-to-End

- recorded command replay
- fake MCU telemetry playback
- link interruption recovery

## What To Avoid

- Putting packet decode logic into the Raspberry Pi bridge
- Publishing ROS2 messages from transport-only packages
- Hiding unit conversions inside callback glue code
- Coupling command safety rules to UART driver classes
- Mixing firmware packet constants into domain entities