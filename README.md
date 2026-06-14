# Embedded Notes — 嵌入式学习笔记与面试准备

> 📚 从扫地机器人 TSE 到深圳嵌入式工程师的完整知识库
> 
> 纯文档仓库：学习路线、笔记、面试八股、求职调研。代码见其他 repo。

---

## 📂 结构

```
embedded-notes/
├── notes/           ← 学习笔记（STM32/ROS2/BLDC/PCB/Linux）
├── planning/        ← 路线规划 + 求职准备
│   ├── project-roadmap.md          ← 4 阶段项目路线
│   ├── shenzhen-job-market.md      ← 深圳岗位调研
│   ├── interview-prep-八股文.md     ← 面试理论体系
│   ├── cs408-to-embedded-mapping.md ← CS408 知识迁移
│   ├── pcb-jlcpcb-learning-plan.md  ← PCB 打样学习计划
│   └── README.md
├── docs/            ← 参考文档（硬件清单、PCB 指南）
├── tools/           ← 工具使用文档
└── README.md
```

## 🗺 学习路线

```mermaid
graph LR
    A[裸机驱动] --> B[FreeRTOS]
    B --> C[ROS2 机器人]
    C --> D[CAN 总线]
```

| 阶段 | 内容 | 代码仓库 |
|------|------|----------|
| Phase 1 | STM32 裸机 I2C/SPI/UART | [stm32-from-scratch] |
| Phase 2 | FreeRTOS 多任务 | [stm32-from-scratch] |
| Phase 3 | ROS2 差分底盘 + SLAM | [robot-vacuum-lab] |
| Phase 4 | CAN 分布式控制 | [stm32-from-scratch] |

## 🔗 相关仓库

- 💻 [stm32-from-scratch](https://github.com/finnyoun9/stm32-from-scratch) — STM32 裸机 + FreeRTOS
- 🟢 [esp32-playground](https://github.com/finnyoun9/esp32-playground) — ESP32/Arduino 实验
- 🤖 [robot-vacuum-lab](https://github.com/finnyoun9/robot-vacuum-lab) — ROS2 扫地机器人
- 🔧 [hardware-lab](https://github.com/finnyoun9/hardware-lab) — 原理图/仿真/PCB
- 🔬 [theory-lab](https://github.com/finnyoun9/theory-lab) — 数电模电理论

---

> *"Talk is cheap. Show me the code."* — Linus Torvalds
