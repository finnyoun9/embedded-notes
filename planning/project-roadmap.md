# Embedded Engineering Project Roadmap
## Transitioning from TSE (Dreame Robot Vacuums) to Embedded Engineer — Shenzhen, September-October 2026

---

## Hardware Inventory

| Item | Notes |
|------|-------|
| Raspberry Pi 5 | Linux brain, ROS2 host, CV processing |
| STM32 (F103C8 "Blue Pill") | Bare-metal / RTOS target |
| Arduino | Quick prototyping, CAN shield host |
| Logic Analyzer | Protocol debugging (I2C/SPI/UART/CAN) |
| DP100 Power Supply | Programmable, power profiling |
| VL53L0X ToF Sensor | I2C distance sensing |
| OV7670 Camera Module | Parallel camera (FIFO buffer recommended) |
| Misc Sensors | Assumed: temp/humidity, buttons, LEDs, pots |
| BLDC Motor + Driver Board | Motor control projects, BLDC learning | New (June 2026) |

### Recommended Purchases (~300-500 RMB total)

| Item | Purpose | Est. Cost |
|------|---------|-----------|
| MPU6050 or ICM-42688 IMU | SPI sensor for Project 2 | 15-30 RMB |
| L298N or TB6612 motor driver | Motor control Projects 3-4 | 15-25 RMB |
| 2x DC motors + wheels + chassis | Robot platform Project 3 | 60-100 RMB |
| SN65HVD230 CAN transceiver (x2) | CAN bus Project 4 | 10-20 RMB each |
| SD card SPI module | Data logging | 5-10 RMB |
| STM32F103C8 (second unit) | Second CAN node, spare | 15-20 RMB |
| AS5600 magnetic encoder | BLDC position feedback | 10-15 RMB |
| OV7670 FIFO buffer module | Easier camera capture | 30-50 RMB |

---

## Project 1: Bare-Metal STM32 Sensor Driver Development
### "Own the Peripheral"

**Difficulty:** Beginner → Intermediate
**Timeline:** 2-3 weeks (evenings/weekends)

### Hardware Used
- STM32F103C8 (Blue Pill)
- VL53L0X ToF sensor (I2C)
- Logic analyzer
- DP100 power supply
- ST-Link programmer
- Breadboard + jumpers

### What You Build
A bare-metal (no HAL, no RTOS) firmware that:
- Implements I2C bit-banging AND hardware I2C drivers from scratch
- Reads VL53L0X distance data (register-level, no library)
- Streams formatted data over UART at 115200 baud
- Uses DMA for UART TX (non-blocking)
- Implements a simple command parser over UART (e.g., "read", "calibrate", "continuous on/off")
- Measures execution timing with SysTick

### Key Skills Demonstrated

| Skill | How |
|-------|-----|
| **Bare-metal register programming** | Writing directly to STM32 RCC, GPIO, I2C, UART, DMA registers. No HAL/CMSIS-driver crutches. |
| **I2C protocol mastery** | Implement from scratch: start/stop conditions, ACK/NACK handling, clock stretching, multi-byte reads. Debug with logic analyzer. |
| **Datasheet fluency** | VL53L0X datasheet is ~40 pages of registers. You'll write the driver by reading it. Same for STM32 reference manual. |
| **Logic analyzer debugging** | Capture I2C transactions, decode them, verify against expected waveforms. This is a daily skill in embedded roles. |
| **Power profiling** | Use DP100 to measure current draw in different modes (active sensing vs idle). Understand sleep modes. |
| **DMA** | Set up DMA for UART TX to free the CPU. Understand circular vs normal mode. |
| **ARM Cortex-M fundamentals** | SysTick timer, NVIC interrupts, vector table, linker scripts. |

### Deliverables
- GitHub repo with clean, commented bare-metal driver code
- README with logic analyzer screenshots showing I2C waveforms
- Power consumption table (DP100 data)
- Video demo: sensor responding to UART commands

### Job Mapping
Directly maps to Shenzhen embedded job requirements:
- "熟悉STM32裸机开发" (familiar with STM32 bare-metal development) — this is on literally every JD
- "熟悉I2C/SPI/UART等通信协议" — you'll have deep I2C understanding, not just library calls
- Logic analyzer proficiency is asked for in senior roles
- Power-aware programming is valued in battery-operated devices (robots, wearables)

### Bridge to Dreame Experience
Robot vacuums have multiple ToF/distance sensors for cliff detection, wall following, and obstacle avoidance. You already understand the application — now you understand the silicon-level implementation.

---

## Project 2: FreeRTOS Multi-Sensor Hub with Protocol Debugging
### "The Real-Time Backbone"

**Difficulty:** Intermediate
**Timeline:** 3-4 weeks

### Hardware Used
- STM32F103C8
- VL53L0X (I2C)
- MPU6050 or ICM-42688 IMU (SPI) — *purchase needed*
- SD card SPI module — *purchase needed*
- Logic analyzer
- DP100 power supply
- Optional: second UART sensor or Bluetooth module

### What You Build
A FreeRTOS-based firmware that:
- Runs 4+ concurrent tasks:
  1. **VL53L0X Task**: Reads distance at 50Hz via I2C
  2. **IMU Task**: Reads accelerometer/gyro at 200Hz via SPI
  3. **Logger Task**: Writes timestamped sensor data to SD card (FatFS)
  4. **Command Task**: UART CLI for start/stop/log-dump/status
  5. **Watchdog Task**: Monitors task health, triggers recovery
- Uses FreeRTOS queues for inter-task communication
- Uses mutexes for I2C bus sharing (if multiple I2C devices)
- Uses binary semaphores for task synchronization
- Implements software timers for periodic sensor reads
- Monitors stack usage per task (stack high-water mark)

### Key Skills Demonstrated

| Skill | How |
|-------|-----|
| **RTOS architecture** | Task priority assignment, stack sizing, preemption behavior. You'll explain *why* each task has its priority. |
| **FreeRTOS primitives** | Queues, mutexes, semaphores, task notifications, software timers. Used correctly, not just present. |
| **SPI protocol** | High-speed sensor reading over SPI. Debug with logic analyzer — verify clock polarity, phase, data validity. |
| **Multi-protocol concurrent** | I2C + SPI + UART + SDIO(SPI) running simultaneously without conflicts. |
| **FatFS integration** | Embedded filesystem on SD card. Understand wear leveling concerns, write buffering. |
| **Task monitoring** | Stack overflow detection, task watchdog, CPU usage calculation. Production-grade reliability. |
| **Logic analyzer advanced** | Multi-channel capture: I2C + SPI + UART simultaneously. Protocol decode stacks. |

### Deliverables
- GitHub repo with FreeRTOS project (CubeMX project file + hand-written task code)
- System architecture diagram (task relationships, data flow)
- Logic analyzer capture showing all three buses operating concurrently
- CPU utilization analysis under different sensor rates
- Video: CLI interaction, SD card data plot (distance + IMU data graphed)

### Job Mapping
- "熟悉FreeRTOS或其他RTOS" — appears on 70%+ of Shenzhen embedded JDs
- Multi-tasking architecture design is asked about in interviews
- SD card / filesystem experience is common in data-logging applications
- This project gives you strong answers for "tell me about a time you used an RTOS"

### Bridge to Dreame Experience
Robot vacuums run RTOS on their MCUs (often FreeRTOS or ThreadX). The sensor hub you build mirrors the MCU inside a Dreame bot that reads cliff sensors, IMU, wheel encoders, and bumper switches concurrently.

---

## Project 3: ROS2 Mini Differential-Drive Robot
### "The Capstone — From Sensor to SLAM"

**Difficulty:** Advanced
**Timeline:** 6-8 weeks

This is your flagship project. It directly mirrors the architecture of a robot vacuum and is the single most impressive demo for Shenzhen robotics/embedded interviews.

### Hardware Used
- Raspberry Pi 5 (ROS2 brain + SLAM)
- STM32F103C8 (motor controller + sensor aggregator)
- 2x DC motors + wheels + chassis — *purchase needed*
- L298N or TB6612 motor driver — *purchase needed*
- VL53L0X (forward-facing obstacle detection)
- MPU6050/ICM-42688 (IMU for odometry correction)
- 2x wheel encoders (Hall effect or optical) — *comes with most motor kits*
- DP100 power supply (bench testing before battery)
- Logic analyzer (UART protocol debug between Pi and STM32)
- Optional: OV7670 camera (see Project 4 extension)

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Raspberry Pi 5 (Ubuntu + ROS2 Jazzy)                   │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐    │
│  │ USB Cam  │  │  SLAM    │  │ Nav2 Stack         │    │
│  │ (or      │  │ (slam    │  │ (planner,          │    │
│  │ OV7670)  │  │  toolbox)│  │  controller)       │    │
│  └──────────┘  └──────────┘  └────────┬───────────┘    │
│                                       │                 │
│                          ┌────────────▼───────────┐     │
│                          │ Serial Bridge Node     │     │
│                          │ (custom binary protocol)│     │
│                          └────────────┬───────────┘     │
└───────────────────────────────────────┼─────────────────┘
                                        │ UART (921600 baud)
┌───────────────────────────────────────┼─────────────────┐
│  STM32F103C8 (FreeRTOS)              │                  │
│  ┌────────────▼──────────────┐        │                  │
│  │ Command Parser Task       │        │                  │
│  │ (binary protocol handler) │        │                  │
│  └────────────┬──────────────┘        │                  │
│       ┌───────┴────────┐             │                  │
│  ┌────▼────┐    ┌──────▼──────┐      │                  │
│  │ Motor   │    │ Sensor Task │      │                  │
│  │ Control │    │ (I2C: ToF,  │      │                  │
│  │ Task    │    │  SPI: IMU,  │      │                  │
│  │ (PID)   │    │  Encoders)  │      │                  │
│  └────┬────┘    └──────┬──────┘      │                  │
│       │                │              │                  │
│  ┌────▼────┐    ┌──────▼──────┐      │                  │
│  │ PWM     │    │ Encoder     │      │                  │
│  │ Driver  │    │ Counter     │      │                  │
│  └────┬────┘    └──────┬──────┘      │                  │
│       │                │              │                  │
│  ┌────▼────────────────▼──────┐      │                  │
│  │     TB6612 Motor Driver    │      │                  │
│  └────────────┬───────────────┘      │                  │
│               │                       │                  │
│  ┌────────────▼───────────────┐      │                  │
│  │  DC Motors + Encoders      │      │                  │
│  └────────────────────────────┘      │                  │
└─────────────────────────────────────────────────────────┘
```

### What You Build

**STM32 Firmware (FreeRTOS):**
- Custom binary serial protocol (framing, CRC, command/response)
- Motor PID control (speed loop) using encoder feedback
- Odometry computation (integrating encoder ticks → x, y, theta)
- Sensor aggregation: IMU data + ToF distance
- Publishes unified state packet at 50-100Hz
- Accepts velocity commands, returns odometry + sensor data
- Watchdog and failsafe (stop motors on comms loss)

**Raspberry Pi 5 (ROS2 Jazzy):**
- Custom `serial_bridge` ROS2 node (talks binary protocol to STM32)
- `diff_drive_controller` publishing `/odom` and `/tf`
- VL53L0X single-beam distance published as LaserScan
- IMU data published to `/imu` topic
- SLAM Toolbox running online SLAM
- Nav2 for autonomous navigation
- RViz2 visualization
- Optional: Gazebo simulation model of the same robot

### Key Skills Demonstrated

| Skill | How |
|-------|-----|
| **ROS2** | Custom nodes, topics, TF2, launch files, parameters. You'll write a real robot driver, not just tutorials. |
| **Embedded Linux** | Cross-compilation, udev rules, systemd services. Pi boots directly into robot mode. |
| **Serial protocol design** | Custom binary protocol with framing, CRC, versioning. You'll defend this design in interviews. |
| **Motor control + PID** | Closed-loop speed control with encoder feedback. Tuning process documented. |
| **Sensor fusion** | Odometry from encoders + IMU for orientation. Understand covariance. |
| **SLAM + Navigation** | Real SLAM on real hardware. Loop closure, map saving, autonomous navigation. |
| **RTOS + Linux integration** | The exact architecture of commercial robots. This is what Dreame bots do. |
| **System integration** | Bringing together mechanical, electrical, firmware, and software. |
| **Debugging complex systems** | When the robot doesn't move right, is it the PID? The odometry? The protocol? TF tree? You'll debug all layers. |

### Deliverables
- GitHub monorepo: `stm32_firmware/` + `ros2_ws/`
- CAD or hand-drawn chassis design
- Wiring diagram
- Protocol specification document
- Video: robot mapping a room autonomously
- Video: RViz2 showing live SLAM, TF tree, sensor data
- Blog post / portfolio page

### Job Mapping
This is THE project for Shenzhen robotics companies (Dreame, Roborock, DJI, UB Tech, etc.):
- Demonstrates the exact skill stack they use: STM32 + ROS2 + sensor integration
- Proves you can build a complete system, not just write isolated firmware
- Gives you a physical demo to show in interviews (bring the robot!)
- The serial protocol design is a common interview question ("design a protocol between MCU and Linux")

### Bridge to Dreame Experience
You already know what a robot vacuum should do. You've seen the architecture from the TSE side. Now you build a miniature version yourself. In interviews, you can say: "I understood the system from testing it; now I can build it from scratch."

---

## Project 4: CAN Bus Distributed Control System
### "Automotive-Grade Communication"

**Difficulty:** Intermediate → Advanced
**Timeline:** 3-4 weeks (can overlap with Project 3)

### Hardware Used
- 2x STM32F103C8 (two nodes) — *second unit needed*
- 2x SN65HVD230 CAN transceivers — *purchase needed*
- Logic analyzer (CAN decoding)
- Arduino (optional: third CAN node)
- Sensors and actuators per node
- DP100 power supply

### What You Build

A distributed embedded system with 2-3 CAN nodes:

**Node 1 — "Sensor Node" (STM32):**
- Reads VL53L0X distance sensor
- Reads temperature/humidity (DHT22 or similar)
- Publishes sensor data on CAN bus at configurable rates
- Responds to CAN remote frames (request sensor data on demand)

**Node 2 — "Actuator Node" (STM32):**
- Drives a DC motor or servo
- Listens for motor commands on CAN bus
- Reports motor status (RPM, current) back on CAN
- Implements heartbeat messages (node alive at 1Hz)

**Node 3 — "Gateway Node" (Arduino, optional):**
- Bridges CAN to UART/USB
- Acts as CAN bus monitor
- Sends configuration commands

### Protocol Design
- Define CAN ID scheme (node ID, message type, priority)
- Implement CANOpen-like object dictionary (simplified)
- Node guarding / heartbeat
- Emergency messages
- SDO-like configuration (set sensor sample rate over CAN)

### Key Skills Demonstrated

| Skill | How |
|-------|-----|
| **CAN bus** | Physical layer (120Ω termination, transceiver), protocol layer (11/29-bit IDs, remote frames, error handling). Debug with logic analyzer. |
| **Distributed systems** | Multi-node architecture, message scheduling, bus arbitration. |
| **Real-time communication** | CAN is deterministic. You'll analyze bus load and worst-case latency. |
| **CANOpen concepts** | Object dictionary, NMT state machine, heartbeat. These are used in industrial automation and automotive. |
| **Logic analyzer advanced** | CAN protocol decoding, bus load analysis, error frame detection. |
| **Fault tolerance** | Node failure detection, bus-off recovery, graceful degradation. |

### Deliverables
- GitHub repo with CAN node firmware
- CAN protocol specification document (ID scheme, message formats)
- Logic analyzer captures showing multi-node CAN traffic
- Bus load analysis spreadsheet
- Video: demonstrate sensor node publishing data, actuator node responding

### Job Mapping
- CAN bus is essential for automotive, industrial, and many robotics roles
- Many Shenzhen companies (BYD, Huawei automotive, robotics startups) list CAN experience
- Distributed system design demonstrates senior-level thinking
- CANOpen familiarity is a bonus for industrial automation roles

### Bridge to Dreame Experience
Robot vacuums use internal buses (often UART or custom protocols) between modules. CAN is the industrial-grade equivalent. Understanding distributed embedded systems makes you a stronger candidate for any robotics role.

---

## Project 5: BLDC Motor Control — From 6-Step to FOC
### "The Motor Control Arc"

**Difficulty:** Intermediate → Advanced
**Timeline:** 4-6 weeks (can overlap with Project 3)

### Hardware Used
- BLDC motor (any small gimbal/drone motor, e.g., 2204/2205 size)
- BLDC driver board (DRV8313-based or discrete MOSFET bridge)
- STM32F103C8 (Blue Pill)
- AS5600 magnetic encoder (optional, for FOC)
- DP100 power supply
- Logic analyzer

### What You Build

**Phase 1: 6-Step Trapezoidal Control (2 weeks)**
- TIM1 advanced timer configured for 6-step commutation with complementary PWM + dead-time
- Hall sensor reading via EXTI interrupts with commutation lookup table
- Open-loop speed control (vary PWM duty cycle)
- Closed-loop speed PID using Hall sensor frequency as feedback
- Current sensing with ADC + over-current protection
- Logic analyzer verification of commutation sequence and dead-time

**Phase 2: FOC Introduction (2-4 weeks, optional stretch)**
- Install SimpleFOC library or ST X-CUBE-MCSDK
- Add AS5600 magnetic encoder for rotor angle feedback
- Run basic FOC current/torque control
- Compare FOC vs 6-step: torque ripple, noise, efficiency

### Key Skills Demonstrated

| Skill | How |
|-------|-----|
| **Advanced timer configuration** | TIM1 complementary PWM, break input, dead-time insertion |
| **Motor commutation** | 6-step sequence, Hall sensor decoding, commutation timing |
| **PID control** | Speed loop tuning, anti-windup, integral separation |
| **Current sensing** | ADC sampling synchronized with PWM, over-current protection |
| **Power electronics basics** | Bootstrap capacitor, shoot-through prevention, MOSFET switching |
| **FOC concepts** | Clarke/Park transforms, space vector modulation (conceptual) |
| **Logic analyzer verification** | 6 PWM channels + 3 Hall signals simultaneously |

### Deliverables
- GitHub repo with BLDC firmware (6-step + PID)
- Logic analyzer screenshots: 6-phase PWM + Hall signals during commutation
- Speed step response plot (PID tuning result)
- Video: motor speed control demo with UART command interface
- Optional: FOC comparison demo

### Job Mapping
- Motor control is a HIGH-VALUE skill in Shenzhen:
  - DJI (drones/gimbals), Dreame/Roborock (robot vacuums), BYD (EV motors)
  - "熟悉无刷直流电机控制" appears on robotics and automotive JDs
  - PID tuning experience is directly transferable to any control system role
  - FOC knowledge signals senior-level ambition

### Bridge to Dreame Experience
Robot vacuums use BLDC motors for the main suction fan and side brushes. Understanding motor control from the firmware side connects your TSE knowledge of robot behavior to the implementation layer.

---

## Skill Coverage Matrix

| Skill | Project 1 | Project 2 | Project 3 | Project 4 | Project 5 |
|-------|:---------:|:---------:|:---------:|:---------:|:---------:|
| Bare-metal STM32 | **●** | ○ | ○ | ○ | ○ |
| I2C | **●** | **●** | **●** | **●** | ○ |
| SPI | ○ | **●** | **●** | ○ | ○ |
| UART | **●** | **●** | **●** | ○ | ○ |
| CAN | ○ | ○ | ○ | **●** | ○ |
| DMA | **●** | ○ | **●** | ○ | ○ |
| FreeRTOS | ○ | **●** | **●** | **●** | ○ |
| Motor Control / PID | ○ | ○ | **●** | **●** | **●** |
| Advanced Timer / PWM | ○ | ○ | ○ | ○ | **●** |
| FOC Concepts | ○ | ○ | ○ | ○ | **●** |
| Sensor Fusion | ○ | ○ | **●** | ○ | ○ |
| ROS2 | ○ | ○ | **●** | ○ | ○ |
| Linux Embedded | ○ | ○ | **●** | ○ | ○ |
| SLAM / Navigation | ○ | ○ | **●** | ○ | ○ |
| Logic Analyzer | **●** | **●** | **●** | **●** | **●** |
| Power Profiling | **●** | **●** | ○ | ○ | ○ |
| Protocol Design | ○ | ○ | **●** | **●** | ○ |
| System Architecture | ○ | **●** | **●** | **●** | ○ |
| PCB Design (optional) | ○ | **●** | **●** | ○ | ○ |

**●** = Primary focus  
○ = Secondary / practiced

---

## Recommended Execution Order

```
Week 1-3:   Project 1 (Bare-metal sensor drivers)
Week 4-7:   Project 2 (FreeRTOS sensor hub)
Week 8-15:  Project 3 (ROS2 robot) ← overlaps with...
Week 10-14: Project 4 (CAN bus, can start once you have 2nd STM32)
Week 12-17: Project 5 (BLDC motor control, can overlap with Project 3)
Week 16-18: Polish, documentation, portfolio website
```

**Accelerated Schedule (Targeting September-October 2026):**
Given the compressed timeline, focus on completing Projects 1-3 as the core portfolio, with Projects 4-5 as valuable additions you can start in parallel:

```
July 2026:      Project 1 (Bare-metal sensor drivers) — 2-3 weeks
July-Aug 2026:  Project 2 (FreeRTOS sensor hub) — 3-4 weeks
Aug-Sep 2026:   Project 3 (ROS2 robot) — 6-8 weeks, your flagship
Sep-Oct 2026:   Project 4 (CAN bus) + Project 5 (BLDC motor) — overlap, aim for at least Phase 1 of each
Oct 2026:       Polish, documentation, portfolio website
```

Total: ~4 months of focused effort. By October 2026, you'll have:
- 5 GitHub repos demonstrating the exact skills Shenzhen employers want
- A physical robot you can bring to interviews
- BLDC motor control demo (high-value for robotics/automotive roles)
- Deep protocol knowledge verified with a logic analyzer
- The confidence to say "I don't just test robots — I build them"

---

## PCB Design (Bonus Skill)

Once Projects 2-3 are working on breadboard, design a custom PCB:
- STM32F103 + supporting circuitry
- TB6612 motor driver integrated (DC motors)
- BLDC driver integration (DRV8313 or discrete MOSFET bridge — for Project 5)
- CAN transceiver
- I2C/SPI/UART breakout headers
- Power regulation (battery → 5V → 3.3V)
- Encoder inputs

Use KiCad (free). Order from JLCPCB (~$5 for 5 boards, shipping to Shenzhen is fast).

This adds "PCB design" to your skill matrix, which many embedded roles value. The PCB can serve as the mainboard for both Project 3 and Project 4.

---

## Interview Narrative

When asked "why are you transitioning from TSE to embedded engineering?":

> "As a TSE at Dreame, I've been testing and validating the embedded systems inside robot vacuums — the sensor pipelines, motor control loops, and RTOS-based firmware. I understand the system behavior deeply. Over the past months, I've been building these systems from scratch: bare-metal sensor drivers, FreeRTOS sensor hubs, a complete ROS2 differential-drive robot with SLAM, and a CAN bus distributed control system. I've moved from testing embedded systems to designing them. My TSE background gives me a unique perspective: I build firmware with testability and reliability built in from the start."

---

*Roadmap created June 2026. Adjust timelines based on your pace. Focus on depth over speed — one well-documented project beats three half-finished ones.*
