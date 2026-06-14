# 📋 学习规划 & 求职调研

> 从 TSE（测试工程师）到嵌入式工程师的完整转型路线图
> 目标时间：2026 年 4 月深圳

---

## 📄 文件说明

| 文件 | 内容 |
|------|------|
| [project-roadmap.md](./project-roadmap.md) | **4 阶段实战项目路线** — 基于现有硬件的 4 个递进项目，覆盖裸机→RTOS→ROS2→CAN 总线 |
| [shenzhen-job-market.md](./shenzhen-job-market.md) | **深圳嵌入式岗位调研** — 技能要求、薪资、公司名单、面试题、学习路径 |

---

## 🗺 四阶段路线

```
Phase 1: 裸机驱动（2-3 周）
  └─ VL53L0X ToF 传感器 I2C 驱动 + UART/DMA + 逻辑分析仪波形

Phase 2: RTOS 多传感器（3-4 周）
  └─ FreeRTOS 5 任务并行采集 + I2C/SPI/UART + FatFS SD 卡记录

Phase 3: ROS2 差分底盘（6-8 周）⭐ 旗舰项目
  └─ Pi 5 ROS2 + STM32 PID 电机控制 + 串口协议 + SLAM/Nav2

Phase 4: CAN 分布式控制（3-4 周）
  └─ CAN 总线双节点 + CANOpen 对象字典 + 心跳/紧急帧
```

---

## 🎯 技能映射

| 岗位要求 | 对应项目 |
|---------|---------|
| "熟悉 STM32 裸机开发" | Phase 1 |
| "熟悉 I2C/SPI/UART/CAN" | Phase 1, 2, 4 |
| "熟悉 FreeRTOS" | Phase 2 |
| "熟悉 Linux 嵌入式开发" | Phase 3 (Pi 5 + ROS2) |
| "有机器人项目经验" | Phase 3 (完整差分底盘) |
| "会用逻辑分析仪/示波器" | Phase 1, 2, 4 |
| "低功耗设计经验" | Phase 1 (DP100 功耗分析) |
