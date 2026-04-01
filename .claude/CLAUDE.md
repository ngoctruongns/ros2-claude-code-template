# ROS2 Clean Architecture Project

This project is set up with a comprehensive set of **Claude Skills** designed to facilitate ROS2 development following **Clean Architecture** principles.

## Available Skills

The following skills are available in `.claude/skills` and can be used to guide development:

| Skill Name              | Description                               | Key Components                                                                        |
| ----------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- |
| **ros2_node_creation**  | Create Clean Architecture compliant Nodes | `BaseNode` template, Dependency Injection, QoS profiles (Python & C++)                |
| **ros2_launch_config**  | Modular Launch files                      | Composition, `IncludeLaunchDescription`, Parameter management, C++ executable support |
| **ros2_service_action** | Services and Actions                      | Server/Client wrappers, Domain Use Case integration (Python & C++)                    |
| **ros2_messaging**      | Pub/Sub Patterns                          | Domain-driven publishers, Generic subscribers, Thread-safe buffers, Synchronization   |
| **ros2_testing**        | Testing Strategy                          | Unit (Domain), Integration (Node), E2E (Launch), GTest/GMock support                  |
| **ros2_lifecycle**      | Managed Nodes                             | Lifecycle Node templates, State transition management, Lifecycle Clients              |
| **ros2_transforms**     | TF2 Management                            | TF2 Wrappers avoiding domain dependency on `geometry_msgs`                            |
| **ros2_diagnostics**    | Health Monitoring                         | `diagnostic_updater` integration, Health entities, Frequency monitoring               |
| **ros2_bag**            | Data Recording                            | Programmatic bag recording and replay utilities (rosbag2)                             |
| **ros2_embedded_bridge** | STM32, Pi, FastDDS, ROS2 bridge systems  | Raw transport boundaries, packet ownership, ROS2 mapping, diagnostics, testing        |

## Project Structure

The project follows a strict separation of concerns:

- **src/domain/**: Pure business logic, entities, and use cases. No ROS2 dependencies.
- **src/application/**: Application services and interfaces. Orchestrates logic.
- **src/infrastructure/**: ROS2 specific implementations (Nodes, Publishers, Subscribers).

## Embedded Bridge Profile

For robots built around an MCU, a Linux transport bridge, and a ROS2 workstation, use the embedded bridge profile defined in:

- `.claude/rules/embedded_bridge_architecture.md`
- `.claude/skills/ros2_embedded_bridge/SKILL.md`

This profile assumes:

- **STM32 or equivalent MCU** owns hard real-time hardware behavior
- **Raspberry Pi or equivalent SBC** forwards raw transport payloads only
- **Laptop ROS2 workspace** owns packet decoding, ROS2 topic mapping, diagnostics, and application policy

## Getting Started

To use a skill, reference the skill file (e.g., `.claude/skills/ros2_node_creation/SKILL.md`) for templates and best practices. For UART/FastDDS bridge robots, start with `.claude/skills/ros2_embedded_bridge/SKILL.md` before applying the generic ROS2 skills.

## Common Commands

For a comprehensive list of ROS2 commands, build instructions, and debugging tools, please refer to:

- **[ROS2 Commands Reference](.claude/commands/ros2.md)**: `colcon`, `ros2`, `rqt`, etc.

### Quick Reference

- **Build**: `colcon build --symlink-install`
- **Test**: `colcon test`
- **Source**: `source install/setup.bash`
