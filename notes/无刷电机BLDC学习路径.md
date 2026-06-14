# 无刷电机（BLDC）学习路径

> **作者**: Finn / 杨一帆  
> **日期**: 2026-06-11  
> **目标**: 深圳嵌入式面试备战 + 扫地机器人项目实战  
> **前置基础**: STM32 裸机开发、Timer/PWM、中断、ADC  

---

## 目录

1. [BLDC 电机基础理论](#1-bldc-电机基础理论)
2. [驱动方式对比](#2-驱动方式对比)
3. [硬件接线与驱动板](#3-硬件接线与驱动板)
4. [STM32 BLDC 控制实战](#4-stm32-bldc-控制实战)
5. [FOC 入门路径（进阶）](#5-foc-入门路径进阶)
6. [项目建议（面试展示用）](#6-项目建议面试展示用)
7. [面试高频考点](#7-面试高频考点)
8. [学习资源](#8-学习资源)

---

## 1. BLDC 电机基础理论

### 1.1 什么是 BLDC（Brushless DC Motor）

BLDC 是一种用电子换向代替机械电刷的直流电机。相比有刷电机，它没有电刷磨损、无火花、效率更高、寿命更长。

**核心结构三要素**:

```
         ┌──────────────────────────────┐
         │    定子 (Stator)              │
         │    - 硅钢片叠压                │
         │    - 绕有线圈绕组 (Coils)       │
         │    - 通常 3 相 (Phase U/V/W)    │
         └────────────┬─────────────────┘
                      │
         ┌────────────▼─────────────────┐
         │    转子 (Rotor)               │
         │    - 永磁体 (Permanent Magnet) │
         │    - 极对数 (Pole Pairs)       │
         └────────────┬─────────────────┘
                      │
         ┌────────────▼─────────────────┐
         │   霍尔传感器 (Hall Sensors)     │
         │    - 3 个，间隔 120° 电角度     │
         │    - 检测转子磁极位置            │
         └──────────────────────────────┘
```

> **动手验证**: 用万用表电阻档，测量电机三根相线两两之间的电阻，应该相等（典型值几 Ω）。用手转动转子，感受磁阻（cogging torque）。

### 1.2 六步换向（6-Step Commutation / Trapezoidal Control）

BLDC 的基本工作原理：通过依次给三相线圈通电，产生旋转磁场，拖动永磁转子旋转。

**六步换向序列**（以 Hall 传感器为参考）:

| 步 | Hall (H3 H2 H1) | A+ | A- | B+ | B- | C+ | C- | 导通相 |
|---|:---:|:--:|:--:|:--:|:--:|:--:|:--:|:---:|
| 1 | 101 | ON | - | - | ON | - | - | A→B |
| 2 | 100 | ON | - | - | - | - | ON | A→C |
| 3 | 110 | - | - | ON | - | - | ON | B→C |
| 4 | 010 | - | ON | ON | - | - | - | B→A |
| 5 | 011 | - | ON | - | - | ON | - | C→A |
| 6 | 001 | - | - | - | ON | ON | - | C→B |

**关键理解**:
- 每一步只有两个半桥工作（上桥臂一个 MOS 导通、下桥臂另一个 MOS 导通）
- Hall 信号每 60° 电角度变化一次，一圈 = 6 步 × 极对数
- 方波驱动（梯形波）—— 电流波形接近矩形，故名 trapezoidal

> **动手验证**: 接上驱动板和电机后，用示波器看一相的对地波形。你会看到梯形（trapezoidal）电压波形 —— 这就是 "六步换向" 的直观证据。

### 1.3 BLDC vs 有刷直流电机（Brushed DC Motor）

| 特性 | BLDC | 有刷 DC |
|------|------|---------|
| 换向方式 | 电子换向 (MOSFET) | 机械电刷 + 换向器 |
| 寿命 | 长（仅轴承磨损） | 短（电刷磨损） |
| 效率 | 高 (~85-90%) | 中等 (~75-85%) |
| EMI/火花 | 无火花，EMI 可控 | 电刷火花，严重 EMI |
| 控制复杂度 | 高（需要 MCU + 驱动） | 低（给电压就转） |
| 成本 | 较高 | 低 |
| 转矩脉动 | 存在（换向切换） | 存在（电刷切换） |
| 速度范围 | 宽 | 中等 |

### 1.4 BLDC vs PMSM（永磁同步电机）

这是**面试高频问题**，一定要能清晰区分：

| 特性 | BLDC | PMSM |
|------|------|------|
| 反电动势波形 | **梯形波** (Trapezoidal) | **正弦波** (Sinusoidal) |
| 驱动方式 | 六步换向 / 方波 | FOC / 正弦波 |
| 绕组形式 | 集中绕组 (Concentrated) | 分布式绕组 (Distributed) |
| 转矩脉动 | 较大 (换向点脉动) | 小（平滑） |
| 控制算法 | 简单 Hall 换向表 | Clarke + Park + SVPWM |
| 效率 | 好 | 更优 |
| 噪音 | 较高 | 低 |
| 成本 | 低 | 高（但差距在缩小） |

> **关键面试话术**: "BLDC 和 PMSM 的主要区别是反电动势波形不同 —— BLDC 是梯形波，适合方波六步换向；PMSM 是正弦波，适合 FOC 矢量控制。但实际中很多 BLDC 电机也能跑 FOC，只是效果不如真正的 PMSM。"

### 1.5 关键参数解读

拿到你的 BLDC 电机，先看懂这几个参数：

| 参数 | 含义 | 你的电机 |
|------|------|----------|
| **KV 值** | 每伏特电压下的空载转速 (RPM/V) | 看铭牌 |
| **极对数 (Pole Pairs)** | 转子永磁体 N-S 对数 | 数转子磁铁或查手册 |
| **最大电流** | 持续工作电流 / 峰值电流 | 设计过流保护的依据 |
| **额定电压** | 正常工作电压范围 | 选择电源和驱动板 |
| **相电阻** | 单相直流电阻 | 影响电流和转矩常数 |
| **最大转速** | 机械转速上限 | 不能超，否则退磁或损坏 |

> **动手验证**: 
> - 用示波器测一相的反电动势（用手快速转动电机，看波形是梯形还是正弦 —— 判断你的电机类型）
> - 数转子磁铁数（如果能看到转子）÷ 2 = 极对数

### 1.6 反电动势（Back EMF）与过零点检测

反电动势是转子永磁体在线圈中感应出的电压，与转速成正比。

**为什么需要检测反电动势**:
- **无传感器控制 (Sensorless)**: 通过检测未导通相的反电动势过零点（Zero-Crossing），推算转子位置，代替 Hall 传感器
- 过零点发生在每步换向的中间时刻，检测到后再延迟 30° 电角度进行换向

```
时间线（一相的反电动势）:
    ┌──┐
    │  │  +BEMF  ← 磁极接近
────┘  └─────────────── 过零点 (Zero-Crossing) ← 转子磁极正对该相
          ┌──┐
          │  │  -BEMF  ← 磁极远离
```

> **动手验证**: 接上电机，不接驱动板，用示波器两通道同时测两相电压（差分）。用手转动电机，看正弦/梯形波和它们之间的相位差。

---

## 2. 驱动方式对比

### 2.1 三种主流驱动方式

#### (A) 六步方波驱动 (6-Step Trapezoidal) — 入门首选

```
V ↑
  │ ┌──────┐        ┌──────┐
  │ │      │        │      │        ← A 相电压波形（梯形）
  │ │      └────────┘      └────
──┼────────────────────────────────→ 电角度
  │        ┌──────┐        ┌──
  │        │      │        │        ← B 相电压波形
  │ ───────┘      └────────┘
  └────────────────────────────────
```

- **控制方式**: Hall 传感器 → 查表 → 换相 → 调节 PWM 占空比调速
- **优点**: 算法简单，MCU 开销小（STM32F103 完全够用），工程中大量使用
- **缺点**: 转矩脉动大，换向点有噪音，低速不平稳

#### (B) FOC 矢量控制 (Field-Oriented Control) — 进阶目标

- **控制方式**: 测量相电流 → Clarke 变换 → Park 变换 → PI 调节 → SVPWM 输出
- **优点**: 转矩平滑、噪音低、效率高、低速性能好、可做位置/力矩控制
- **缺点**: 算法复杂，需要 MCU 算力（STM32F4/G4 系列），调试难度高

#### (C) 同步整流 (Synchronous Rectification)

- 在方波驱动的续流阶段用 MOS 管代替体二极管续流，减少发热
- 是六步换向的优化，很多现代驱动 IC 自带此功能

### 2.2 传感器 vs 无传感器

| 方式 | 如何获取转子位置 | 优点 | 缺点 |
|------|-----------------|------|------|
| **Hall 传感器** | 3 个 Hall 直接输出 6 种位置编码 | 启动可靠，任何时候都能获取位置 | 需要 5 根信号线，Hall 可能失效 |
| **反电动势检测** | 检测未导通相反电动势过零点 | 无需传感器，减少接线 | 低速/零速无法检测，启动需强制换向 |
| **编码器 (Encoder)** | 光/磁编码器输出精确角度 | FOC 位置环，精度极高 | 贵，接线多 |
| **磁编码器 (AS5600/MT6701)** | I2C/SPI 输出 12-bit 角度 | 便宜 (~¥5-15)，精度好 | 需要磁铁对准安装 |

> **面试话术**: "无传感器启动一般采用三段式启动 —— 预定位 (Alignment) → 强制同步加速 (Open-loop ramp) → 切换到闭环 (Switch to BEMF detection)。启动扭矩比较弱，适合风扇、无人机等轻载场景，不适合需要低速大扭矩的场合。"

### 2.3 快速对比总结

| 特性 | 6-Step (方波) | FOC (正弦波) |
|------|:---:|:---:|
| 实现复杂度 | ★☆☆☆☆ | ★★★★☆ |
| MCU 要求 | STM32F103 | STM32F4/G4 |
| 转矩脉动 | 大 (~15%) | 小 (<5%) |
| 噪音 | 较大 | 安静 |
| 效率 | 85-90% | 90-95% |
| 低速性能 | 差 | 好 |
| 位置/力矩控制 | 难以实现 | 天然支持 |
| 面试分量 | 基础，必考 | 加分项，展示深度 |
| **你的学习优先级** | **先学这个** | **学完六步后再学** |

---

## 3. 硬件接线与驱动板

### 3.1 BLDC 驱动板的典型结构

```
电池/电源
    │
    ▼
┌───────────────────────────────────┐
│       三相逆变桥 (3-Phase Inverter)  │
│                                   │
│   ┌─┬─┬─┐  ┌─┬─┬─┐  ┌─┬─┬─┐      │
│   │Q1││ │  │Q3││ │  │Q5││ │   ← 上桥臂 (High-side)
│   │HS││ │  │HS││ │  │HS││ │       │
│   └─┴─┴─┘  └─┴─┴─┘  └─┴─┴─┘      │
│     │A        │B        │C        │
│   ┌─┴─┴─┐  ┌─┴─┴─┐  ┌─┴─┴─┐      │
│   │Q2││ │  │Q4││ │  │Q6││ │   ← 下桥臂 (Low-side)
│   │LS││ │  │LS││ │  │LS││ │       │
│   └─┬─┬─┘  └─┬─┬─┘  └─┬─┬─┘      │
│     │  │     │  │     │  │        │
├─────┴──┴─────┴──┴─────┴──┴────────┤
│   GND   Shunt 电阻 (电流采样)       │
└───────────────────────────────────┘
   │  │  │
   ▼  ▼  ▼
  BLDC 电机 (三根相线)
```

**6 个功率管的导通规则**:
- 任何时候上下桥臂**不能同时导通** → 短路 (Shoot-through) → 炸管
- 上桥臂用 PWM，下桥臂恒通；或下桥臂 PWM、上桥臂恒通
- 死区时间 (Dead-time): 100 ns ~ 2 μs，确保上下管交替时不会同时导通

### 3.2 常见驱动方案

#### 方案 1: 分立 MOSFET + Gate Driver

```
MCU PWM ──→ Gate Driver (如 IR2101/IR2104/EG2104)
                │
                ├──→ 上桥臂 MOS 栅极 + Bootstrap 电容
                └──→ 下桥臂 MOS 栅极
```

- **优点**: 灵活，功率可大可小，成本低
- **缺点**: 自己布线和调试复杂，容易炸
- **典型应用**: 大功率电机驱动（几十~几百 W）

> **Bootstrap 电容原理**: 上桥臂 N-MOS 导通时需要栅极电压高于源极电压（Vgs > Vth），但源极电压接近母线电压。Bootstrap 电容在下管导通时充电，上管导通时提供"浮地"供电。

#### 方案 2: 集成驱动 IC（推荐入门）

| 芯片 | 最大电压 | 最大电流 | 特点 |
|------|---------|---------|------|
| **DRV8313** | 60V | 2.5A (peak) | 3 半桥，带电流检测，SPI 接口 |
| **L6234** | 52V | 5A | 经典芯片，三半桥 |
| **A4950** | 40V | 3.5A | 双半桥（2 片做 BLDC），简单 |
| **TB6612** | 15V | 1.2A | 双半桥（有刷电机用，**不适合 BLDC**） |

> **你的驱动板是哪一个？先搞清楚**:
> ```bash
> # 看芯片丝印，然后查 datasheet
> ```

#### 方案 3: 全集成 FOC 驱动

- **SimpleFOC 兼容驱动**: L6234 + AS5600 磁编码器 + STM32
- **ODrive**: 开源高性能 FOC 驱动器（双电机，Teensy/STM32F4）
- **VESC**: 开源电调，适合滑板车/电动车

### 3.3 实际接线指南

**标准 BLDC 电机出线**:
```
┌──────────────────┐
│   BLDC 电机      │
│                  │
│   U (A) ─ 黄色   │──→ 驱动板 U / Phase A
│   V (B) ─ 绿色   │──→ 驱动板 V / Phase B
│   W (C) ─ 蓝色   │──→ 驱动板 W / Phase C
│                  │
│   Hall A ─ 白色  │──→ MCU GPIO (如 PB6)
│   Hall B ─ 灰色  │──→ MCU GPIO (如 PB7)
│   Hall C ─ 紫色  │──→ MCU GPIO (如 PB8)
│   Hall VCC ─ 红色│──→ 5V
│   Hall GND ─ 黑色│──→ GND
└──────────────────┘
```

> **动手验证**: 
> 1. 用万用表蜂鸣档确认三根相线两两之间是通的（线圈）
> 2. 给 Hall 传感器供电 (5V)，手动转动电机，用万用表量 Hall A/B/C 输出 —— 应该在 0V 和 5V 之间跳变
> 3. 用手快速转动电机，用示波器看 A-B 两相之间的反电动势波形

### 3.4 电流检测方案

| 方案 | 原理 | 精度 | 成本 | 适用场景 |
|------|------|:---:|:---:|---------|
| **Shunt 电阻 + 运放** | 采样电阻 → 差分放大 → ADC | 高 | 低 | 通用，STM32 自带运放可用 |
| **Hall 电流传感器** | ACS712/ACS758 隔离测量 | 中 | 中 | 隔离需求，高压场景 |
| **驱动 IC 内置** | DRV8313 自带电流反馈 | 中 | 芯片内含 | 省 BOM |

**采样位置**: 下桥臂公共端（Low-side）放一个 shunt 电阻最简单；更好的是三相分别采样（三电阻方案）或直流母线采样（单电阻方案）。

> **动手验证**: 
> - 用 ADC 读取 shunt 电压 → I = V_shunt / R_shunt
> - 观察电机堵转时的最大电流，设置软件过流保护阈值

### 3.5 安全要点

1. **死区时间 (Dead-time)**: STM32 高级定时器自带死区插入。先设置 1 μs → 示波器确认上下管没有同时导通 → 再减小
2. **Bootstrap 电容**: 不能太小（充不满上管栅极）也不能太大（充电慢）。典型 1 μF ~ 10 μF，耐压 ≥ 母线电压
3. **母线电容**: 吸收换向尖峰。100 μF ~ 1000 μF 电解电容，耐压 ≥ 1.5 × 母线电压
4. **电源限流**: 调试时电源设置 0.5A 限流，逐步增加，避免炸 MOS
5. **过温保护**: NTC 贴 MOS 管上 → ADC 测温 → 超过 80°C 降功率或停机

---

## 4. STM32 BLDC 控制实战

### 4.1 硬件准备清单

- [ ] STM32 开发板（推荐 STM32F103C8T6 或 STM32F407）
- [ ] BLDC 驱动板（确认芯片型号）
- [ ] BLDC 电机（带 Hall 传感器）
- [ ] 可调稳压电源（建议 0-30V/3A）
- [ ] 逻辑分析仪（Saleae 兼容，~¥30）
- [ ] 示波器（DSO138 也可以，有更好）

### 4.2 软件架构

```
main.c
├── 系统初始化
│   ├── SystemClock_Config()
│   ├── GPIO_Init()          ← Hall 输入引脚
│   ├── TIM1_PWM_Init()      ← 6 路互补 PWM
│   ├── ADC_Init()           ← 电流/电压采样
│   └── UART_Init()          ← 调试输出
│
├── 中断服务
│   ├── EXTI_IRQ (Hall)      ← Hall 变化 → 换向
│   ├── TIM1_UP_IRQ          ← 速度计算 (可选)
│   └── ADC_IRQ              ← 电流采样完成
│
└── 主循环
    ├── Speed_Control()       ← PID 速度环
    ├── Overcurrent_Check() 
    └── UART_Command()        ← 串口调速命令
```

### 4.3 TIM1 高级定时器配置（6 路互补 PWM）

```c
/*============================================================
 * TIM1 初始化：6 路互补 PWM，死区插入
 * 
 * 通道分配（参考 STM32F103C8T6）：
 *   CH1  (PA8)  →  A 相上桥 (UH)    CH1N (PB13) →  A 相下桥 (UL)
 *   CH2  (PA9)  →  B 相上桥 (VH)    CH2N (PB14) →  B 相下桥 (VL)
 *   CH3  (PA10) →  C 相上桥 (WH)    CH3N (PB15) →  C 相下桥 (WL)
 *============================================================*/
void TIM1_PWM_Init(uint16_t pwm_freq_hz)
{
    // === 1. 使能时钟 ===
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_TIM1 | RCC_APB2Periph_GPIOA | 
                           RCC_APB2Periph_GPIOB, ENABLE);
    
    // === 2. GPIO 配置：PA8/9/10 + PB13/14/15 设为复用推挽 ===
    GPIO_InitTypeDef gpio = {0};
    gpio.GPIO_Mode  = GPIO_Mode_AF_PP;
    gpio.GPIO_Speed = GPIO_Speed_50MHz;
    
    gpio.GPIO_Pin = GPIO_Pin_8 | GPIO_Pin_9 | GPIO_Pin_10;
    GPIO_Init(GPIOA, &gpio);
    
    gpio.GPIO_Pin = GPIO_Pin_13 | GPIO_Pin_14 | GPIO_Pin_15;
    GPIO_Init(GPIOB, &gpio);
    
    // === 3. 时基配置 ===
    TIM_TimeBaseInitTypeDef tim = {0};
    tim.TIM_Prescaler = 72 - 1;                // 72MHz / 72 = 1MHz (1µs 精度)
    tim.TIM_Period    = 1000000 / pwm_freq_hz - 1;  // 如 20kHz → 49
    tim.TIM_CounterMode   = TIM_CounterMode_CenterAligned1;  // 中心对齐
    tim.TIM_ClockDivision = TIM_CKD_DIV1;
    tim.TIM_RepetitionCounter = 0;
    TIM_TimeBaseInit(TIM1, &tim);
    
    // === 4. PWM 输出配置 ===
    TIM_OCInitTypeDef oc = {0};
    oc.TIM_OCMode      = TIM_OCMode_PWM1;
    oc.TIM_OutputState = TIM_OutputState_Enable;
    oc.TIM_OutputNState= TIM_OutputNState_Enable;
    oc.TIM_Pulse       = 0;                     // 初始占空比 0
    oc.TIM_OCPolarity  = TIM_OCPolarity_High;    // 上桥臂高有效
    oc.TIM_OCNPolarity = TIM_OCNPolarity_High;   // 下桥臂高有效
    oc.TIM_OCIdleState = TIM_OCIdleState_Reset;
    oc.TIM_OCNIdleState= TIM_OCNIdleState_Reset;
    
    TIM_OC1Init(TIM1, &oc);  // CH1
    TIM_OC2Init(TIM1, &oc);  // CH2
    TIM_OC3Init(TIM1, &oc);  // CH3
    
    // === 5. 死区配置 (Dead-time) ===
    TIM_BDTRInitTypeDef bdtr = {0};
    bdtr.TIM_OSSRState  = TIM_OSSRState_Enable;
    bdtr.TIM_OSSIState  = TIM_OSSIState_Enable;
    bdtr.TIM_LOCKLevel  = TIM_LOCKLevel_OFF;
    bdtr.TIM_DeadTime   = 72;                    // 72 × (1/72MHz) = 1µs
    // ⚠️ Dead-time 计算公式: DTG × Tdts
    //    72MHz 定时器: Tdts = 1/72MHz ≈ 13.89ns
    //    DTG=72 → deadtime ≈ 1µs
    bdtr.TIM_Break      = TIM_Break_Disable;
    bdtr.TIM_BreakPolarity = TIM_BreakPolarity_High;
    bdtr.TIM_AutomaticOutput = TIM_AutomaticOutput_Enable;
    TIM_BDTRConfig(TIM1, &bdtr);
    
    // === 6. 启动定时器 ===
    TIM_Cmd(TIM1, ENABLE);
    TIM_CtrlPWMOutputs(TIM1, ENABLE);  // MOE 主输出使能
}
```

> **动手验证**: 
> 1. 示波器两通道：一通道接 UH (PA8)，另一通道接 UL (PB13)
> 2. 观察波形：上下管永远不会同时为高
> 3. 测量死区宽度 = 1µs 是否符合预期

### 4.4 Hall 传感器读取与换向表

```c
/*============================================================
 * 全局常量
 *============================================================*/

// 根据实测确定的 Hall → Step 映射表
// 注意：必须先实际转动电机，记录 6 种 Hall 组合对应的电机转向
// 这个表是举例，你的电机可能不同！！！
//
// 格式: [Hall值] = {上管位掩码, 下管位掩码}
//   位: bit0=A上, bit1=B上, bit2=C上, bit3=A下, bit4=B下, bit5=C下
//
typedef struct {
    uint8_t high_side_mask;  // 上桥臂使能 (bit0=A, bit1=B, bit2=C)
    uint8_t low_side_mask;   // 下桥臂使能 (bit0=A, bit1=B, bit2=C)
} CommutationStep;

static const CommutationStep commutation_table[8] = {
    // Hall  = H3 H2 H1
    // {高侧使能, 低侧使能}
    [0] = {0, 0},  // 000 — 无效状态，全关（保护）
    [1] = {0b100, 0b010},  // 001 (H1) → A+ B-
    [2] = {0b100, 0b001},  // 010 (H2) → A+ C-
    [3] = {0b010, 0b001},  // 011 (H1+H2) → B+ C-
    [4] = {0b010, 0b100},  // 100 (H3) → B+ A-
    [5] = {0b001, 0b100},  // 101 (H1+H3) → C+ A-
    [6] = {0b001, 0b010},  // 110 (H2+H3) → C+ B-
    [7] = {0, 0},  // 111 — 无效状态
};

// GPIO 引脚定义（根据你的接线修改）
#define HALL_A_GPIO    GPIOB
#define HALL_A_PIN     GPIO_Pin_6
#define HALL_B_GPIO    GPIOB
#define HALL_B_PIN     GPIO_Pin_7
#define HALL_C_GPIO    GPIOB
#define HALL_C_PIN     GPIO_Pin_8

// 读取 Hall 状态 → 3-bit 编码
static inline uint8_t hall_read(void)
{
    uint8_t val = 0;
    if (GPIO_ReadInputDataBit(HALL_A_GPIO, HALL_A_PIN))  val |= (1 << 0);
    if (GPIO_ReadInputDataBit(HALL_B_GPIO, HALL_B_PIN))  val |= (1 << 1);
    if (GPIO_ReadInputDataBit(HALL_C_GPIO, HALL_C_PIN))  val |= (1 << 2);
    return val;
}
```

### 4.5 换向执行函数（核心）

```c
/*============================================================
 * commutation_execute(hall_code)
 * 
 * 根据 Hall 编码查表，设置 TIM1 各通道的输出状态
 * 关键技巧：不用 GPIO 切换，直接操作 TIM1 的 CCER 寄存器
 *============================================================*/
static void commutation_execute(uint8_t hall_code)
{
    if (hall_code == 0 || hall_code == 7) {
        // 无效 Hall 状态 → 紧急关闭所有输出
        TIM1->CCER = 0;  // A/B/C 三路互补输出全关
        return;
    }
    
    const CommutationStep *step = &commutation_table[hall_code];
    
    // ⚠️ 关键：先关所有输出（避免 shoot-through）
    // 然后在同一个操作中设置新状态
    // 方式 1: 操作 TIM1->CCER 寄存器直接设置
    // 方式 2: 用 TIM_SelectOutputTrigger() + TIM_ForcedOCxConfig()
    
    // --- 方法: 直接写 CCER ---
    uint16_t ccer = 0;
    
    // A 相
    if (step->high_side_mask & 0x01) ccer |= TIM_CCER_CC1E;   // A 上桥 PWM 输出
    if (step->low_side_mask  & 0x01) ccer |= TIM_CCER_CC1NE;  // A 下桥 ON
    
    // B 相
    if (step->high_side_mask & 0x02) ccer |= TIM_CCER_CC2E;
    if (step->low_side_mask  & 0x02) ccer |= TIM_CCER_CC2NE;
    
    // C 相
    if (step->high_side_mask & 0x04) ccer |= TIM_CCER_CC3E;
    if (step->low_side_mask  & 0x04) ccer |= TIM_CCER_CC3NE;
    
    TIM1->CCER = ccer;
}
```

### 4.6 Hall 中断服务函数

```c
/*============================================================
 * EXTI 中断处理
 * 
 * 三个 Hall 传感器都接到同一组 GPIO（如 PB6/7/8）
 * 配置 EXTI6/7/8 → EXTI9_5_IRQHandler
 *============================================================*/
static volatile uint8_t  g_hall_code = 0;
static volatile uint32_t g_hall_tick = 0;  // 记录每次换向的时间戳
static volatile uint32_t g_commutation_period = 0;  // 60° 电角度时间

void EXTI9_5_IRQHandler(void)
{
    // 清除中断标志
    if (EXTI_GetITStatus(EXTI_Line6))  EXTI_ClearITPendingBit(EXTI_Line6);
    if (EXTI_GetITStatus(EXTI_Line7))  EXTI_ClearITPendingBit(EXTI_Line7);
    if (EXTI_GetITStatus(EXTI_Line8))  EXTI_ClearITPendingBit(EXTI_Line8);
    
    // 读取 Hall
    uint8_t hall = hall_read();
    
    // 防抖：只有 Hall 真正变化了才执行换向
    if (hall == g_hall_code) return;
    g_hall_code = hall;
    
    // 计算速度
    uint32_t now = micros();  // 使用 DWT 或 SysTick 的微秒计数器
    g_commutation_period = now - g_hall_tick;
    g_hall_tick = now;
    
    // 执行换向
    commutation_execute(hall);
}
```

### 4.7 速度控制（闭环 PID）

```c
/*============================================================
 * 速度计算
 * 
 * 机械转速 RPM = 60 / (6 × 极对数 × T_commutation)
 * 
 * T_commutation = 60° 电角度时间（两次 Hall 跳变间隔）
 * 6 × T_commutation = 一个电周期
 * 极对数 × 电周期 = 一个机械周期
 *============================================================*/
#define MOTOR_POLE_PAIRS  4   // ⚠️ 根据你的电机修改

static float speed_calculate_rpm(void)
{
    if (g_commutation_period == 0) return 0.0f;
    
    // RPM = 60 / (6 × pole_pairs × T_commutation_seconds)
    float period_sec = (float)g_commutation_period / 1000000.0f;  // µs → s
    return 60.0f / (6.0f * MOTOR_POLE_PAIRS * period_sec);
}

/*============================================================
 * PID 速度环
 * 
 * 每 10ms 运行一次（在 main 循环或定时器中断中）
 *============================================================*/
typedef struct {
    float Kp, Ki, Kd;
    float integral;
    float prev_error;
    float integral_limit;  // 积分限幅
    float output_limit;    // 输出限幅 (0-100% duty)
} PID_Controller;

static PID_Controller speed_pid = {
    .Kp = 0.5f,
    .Ki = 2.0f,
    .Kd = 0.01f,
    .integral_limit = 50.0f,
    .output_limit   = 90.0f,  // 最大占空比 90%，留余量
};

static float pid_update(PID_Controller *pid, float setpoint, float measured, float dt)
{
    float error = setpoint - measured;
    
    // 积分项（带限幅抗饱和）
    pid->integral += error * dt;
    if (pid->integral > pid->integral_limit)  pid->integral = pid->integral_limit;
    if (pid->integral < -pid->integral_limit) pid->integral = -pid->integral_limit;
    
    // 微分项
    float derivative = (error - pid->prev_error) / dt;
    pid->prev_error = error;
    
    // PID 输出
    float output = pid->Kp * error + pid->Ki * pid->integral + pid->Kd * derivative;
    
    // 输出限幅
    if (output > pid->output_limit)  output = pid->output_limit;
    if (output < 0)                  output = 0;  // 不反转
    
    return output;
}

/*============================================================
 * 设置 PWM 占空比
 * 
 * TIM1 的 CCR 寄存器控制占空比
 * Duty% → CCR = TIM_Period × duty / 100
 *============================================================*/
static void pwm_set_duty(float duty_percent)
{
    uint16_t ccr = (uint16_t)((float)TIM1->ARR * duty_percent / 100.0f);
    
    TIM_SetCompare1(TIM1, ccr);
    TIM_SetCompare2(TIM1, ccr);
    TIM_SetCompare3(TIM1, ccr);
}
```

### 4.8 电流限制保护

```c
/*============================================================
 * 电流采样和保护
 * 
 * 假设：10mΩ shunt 电阻，通过运放放大后接入 ADC
 * 满量程 3.3V → 对应电流 = 3.3V / (Rshunt × Gain)
 *============================================================*/
#define SHUNT_RESISTANCE    0.01f     // 10mΩ
#define CURRENT_GAIN        50.0f     // 运放增益
#define CURRENT_LIMIT_AMP   2.0f      // 电流上限 2A
#define ADC_VREF            3.3f
#define ADC_RESOLUTION      4096      // 12-bit ADC

static float current_read_amps(void)
{
    uint16_t adc_val = ADC_GetConversionValue(ADC1);
    float voltage = (float)adc_val * ADC_VREF / ADC_RESOLUTION;
    return voltage / (SHUNT_RESISTANCE * CURRENT_GAIN);
}

static bool overcurrent_check(void)
{
    float current = current_read_amps();
    if (current > CURRENT_LIMIT_AMP) {
        // 紧急停机
        pwm_set_duty(0);
        TIM_CtrlPWMOutputs(TIM1, DISABLE);  // 刹车
        return true;
    }
    return false;
}
```

### 4.9 逻辑分析仪调试（关键步骤！）

> **这是最重要的调试环节。面试官问你"怎么验证你的换向逻辑是对的"，你要能说出这套流程。**

**步骤**:

1. **接线**: 逻辑分析仪 CH0-CH2 → Hall A/B/C，CH3-CH5 → TIM1 CH1/2/3 输出（任一相即可）
2. **上电，低占空比，慢速转动**
3. **抓波形，验证 Hall 跳变 → PWM 变化是否正确**

```
期望看到的波形：

Hall A:  ──┐     ┌─────┐     ┌──
            └─────┘     └─────┘
Hall B:  ────┐     ┌─────┐     ┌
              └─────┘     └─────┘
Hall C:  ──────┐     ┌─────┐
                └─────┘     └───

PWM_AH:  ──████──────████──────
PWM_AL:  ──────████──────████──

观察:
  ✓ Hall 每次跳变，PWM 通道切换
  ✓ 上下管波形互补，有死区
  ✓ 死区宽度 1µs（测量上升/下降沿间距）
  ✓ 没有上下管同时为高的瞬间
```

**如果电机不转**: 逐项排查——
1. Power: 电源电压够不够？限流够不够？
2. Hall 供电: 5V 到了 Hall 没？
3. 换向表: Hall 编码对应的导通相是否正确？（用手转电机，量 Hall 输出，对表）
4. PWM: 用示波器看 PA8 有没有波形
5. MOE: `TIM_CtrlPWMOutputs(TIM1, ENABLE)` 执行了没？
6. 三相线: 有没有接错顺序？互换两相试试

---

## 5. FOC 入门路径（进阶）

> **先学会六步换向，再学 FOC。不要跳步。**

### 5.1 从方波到正弦波 —— 为什么需要 Clarke/Park 变换

六步换向的问题：每一步电流突变 → 转矩脉动 → 振动噪音。

FOC 的思想：
1. 测量三相电流 I_a, I_b, I_c
2. **Clarke 变换**：三相静止坐标 (a,b,c) → 两相静止坐标 (α, β)
3. **Park 变换**：两相静止坐标 (α, β) → 旋转坐标 (d, q)
4. 在 d-q 坐标系中，I_q 控制力矩，I_d 控制磁场（通常设 0）
5. 两个 PI 环分别调节 I_d 和 I_q
6. **逆 Park + 逆 Clarke** → 三相 PWM 占空比（SVPWM）

```
       I_a ──┐
       I_b ──┤ Clarke ──→ I_α, I_β ──┤ Park ──→ I_d, I_q ──┤ PI ──→ V_d, V_q
       I_c ──┘              ↑          │                      │
                      转子角度 θ        │                      │
                                        │   逆Park             │
                                        │  V_α, V_β ←────────┘
                                        │    │
                                        │   逆Clarke + SVPWM
                                        │    │
                                        └────┴──→ 6路PWM → 电机
```

> **不要求面试时推导变换矩阵**，但要能说清楚：*"为什么要变换？——因为三相电流互相耦合难以直接控制，变换到旋转 d-q 坐标系后，力矩电流和励磁电流解耦，可以用简单的 PI 控制。"*

### 5.2 ST Motor Control SDK (X-CUBE-MCSDK)

这是ST官方的电机控制软件包，支持 FOC 和六步换向：

- **工具**: Motor Control Workbench（GUI 配置工具）+ Motor Pilot（实时调参）
- **硬件要求**: STM32F302/F303/G431/G474 系列（带高级定时器和运放）
- **学习路线**:
  1. 下载 X-CUBE-MCSDK → 阅读 Quick Start Guide
  2. 用 Workbench 生成一个 "6-step" 项目 → 跑通，理解工程结构
  3. 再试 "FOC" 项目 → 对比六步和 FOC 的区别
  4. 用 Motor Pilot 实时看 I_d/I_q、转速、角度波形

> **动手验证**: 用 Workbench 生成一个最简单的六步换向工程，烧录，对比你自己写的代码和 ST 生成的代码有什么区别。

### 5.3 SimpleFOC 库

[SimpleFOC](https://docs.simplefoc.com/) 是一个开源的 FOC 库，支持 Arduino/STM32/ESP32：

- **优点**: API 简单、文档好、社区活跃、适合学习 FOC 原理
- **硬件搭配**: L6234 驱动 + AS5600 磁编码器 + STM32
- **入门代码**（伪代码）:
```cpp
#include <SimpleFOC.h>

BLDCMotor motor = BLDCMotor(7);  // 7 pole pairs
BLDCDriver3PWM driver = BLDCDriver3PWM(9, 10, 11); // PWM pins
MagneticSensorI2C sensor = MagneticSensorI2C(AS5600_I2C);

void setup() {
    sensor.init();
    motor.linkSensor(&sensor);
    driver.voltage_power_supply = 12;
    motor.linkDriver(&driver);
    motor.controller = MotionControlType::velocity;
    motor.PID_velocity.P = 0.2;
    motor.init();
    motor.initFOC(); // 自动校准
}

void loop() {
    motor.loopFOC();       // FOC 运算
    motor.move(target_rpm); // 速度控制
}
```

> **你的路径**: 先用 SimpleFOC 快速体验 FOC（1 天跑起来）→ 理解各组件作用 → 再回来看 ST SDK 源码。

### 5.4 什么时候从六步升级到 FOC？

| 六步换向够用 | 需要 FOC |
|-------------|---------|
| 风扇、水泵、螺旋桨 | 机器人关节、伺服 |
| 对噪音不敏感 | 安静环境 |
| 不需要低速控制 | 需要有低速大扭矩 |
| 不关心转矩脉动 | 位置控制 / 力矩控制 |
| 简单调速 | 精密速度/位置/力矩环 |

---

## 6. 项目建议（面试展示用）

### 项目 1: BLDC 开环 + 闭环调速 Demo

**目标**: 展示六步换向、PID 速度环、串口调参能力

| 项目 | 详情 |
|------|------|
| **工时** | 3-5 天 |
| **硬件** | STM32 + BLDC 驱动板 + 带 Hall 的 BLDC 电机 |
| **功能** | ① 开环调速（调占空比）② 闭环 PID 调速 ③ 串口发命令设置目标转速 ④ LCD/OLED 显示实时转速和电流 |
| **代码亮点** | 自己手写换向表（不用 ST SDK）、PID 参数在线调整 |
| **调试输出** | 串口打印 RPM/电流/duty → Python/SerialPlot 画曲线 |
| **面试可讲** | "我手写了一个六步换向的 BLDC 控制器，用 PID 做速度闭环，并且用逻辑分析仪验证了换向时序" |

### 项目 2: BLDC 位置伺服控制

**目标**: 展示编码器、位置环、参数整定能力

| 项目 | 详情 |
|------|------|
| **工时** | 5-7 天 |
| **硬件** | 项目 1 + AS5600 磁编码器模块 (~¥10) |
| **功能** | ① 读取磁编码器角度 ② 位置环 PID（位置模式）③ 串口发送目标角度 ④ 回到零位功能 |
| **调试方法** | 逻辑分析仪抓 SPI/I2C 波形 → 确认角度数据正确 → 手动转轴验证 0-360° |
| **面试可讲** | "我在六步换向基础上加了磁编码器，做了位置伺服控制，用了级联 PID（位置环 → 速度环 → 电流环）" |

### 项目 3: BLDC 集成到差分驱动机器人底盘

**目标**: 展示系统集成能力 —— 两个电机、运动学、通信

| 项目 | 详情 |
|------|------|
| **工时** | 7-10 天 |
| **硬件** | 项目 2 × 2 + 底盘 + 编码器轮子 + ESP32（遥控）/ 树莓派（ROS） |
| **功能** | ① 两个 BLDC 分别控制左右轮 ② 速度匹配标定 ③ UART/SPI 接收运动指令 ④ 里程计计算 |
| **软件架构** | 双电机控制 → 底盘运动学逆解 → 速度命令分解 → 两路 PID |
| **面试可讲** | "我用两个 BLDC 电机做了差速底盘，自己写了里程计融合，能响应串口运动指令" |

---

## 7. 面试高频考点

> 以下问题在深圳嵌入式面试中真实出现过。每一个都要能用语言 + 画图回答。

### Q1: 画出六步换向的导通序列表

**回答要素**:
- 6 个步骤，每一步哪两个 MOS 导通（一个上管 + 一个下管）
- Hall 传感器的 6 种编码（101-100-110-010-011-001）
- 电流方向：例如 Step1: A 进 → B 出（简称 A+B-）

> **白板技巧**: 画三个半桥（6 个 MOS），每步标注哪些管子导通，电流怎么流。

### Q2: BLDC 和 PMSM 有什么区别？

**标准回答**: "核心区别是反电动势波形不同 —— BLDC 是梯形波，PMSM 是正弦波。BLDC 通常用六步方波驱动，PMSM 用 FOC 正弦波驱动。但实际上很多 BLDC 电机反电动势接近正弦，也能跑 FOC。从面试角度，关键要能说出驱动方式为什么不同。"

### Q3: 死区时间 (Dead-time) 是什么？为什么需要？

**标准回答**: "死区时间是上下桥臂交替导通时，两个管子同时关断的间隔时间。原因是 MOS 管关断有延迟，如果没有死区，上管还没完全关断时下管就开始导通，就会发生直通 (Shoot-through)，瞬间短路电流会烧毁 MOS。典型值 500ns ~ 2µs，STM32 高级定时器 BDTR 寄存器可以配置。过短的死区有直通风险，过长会增加转矩脉动。用示波器双通道抓上下管栅极波形来确认死区。"

### Q4: 无传感器启动怎么做？

**标准回答**:
1. **预定位 (Alignment)**: 给某一相通电固定时间，把转子拉到已知位置
2. **强制同步加速 (Open-loop ramp)**: 按预设的换向频率逐步加速，转速从 0 拉到反电动势可检测
3. **切换闭环 (Switching to closed-loop)**: 反电动势过零点可靠检测后，切换到闭环运行

"无传感器适用于轻载启动（风扇、航模电机），重载启动容易失步。"

### Q5: 如何测量电机转速？

**三个层次回答**:

| 方法 | 原理 | 适用场景 |
|------|------|----------|
| Hall 传感器 | 测量两次 Hall 跳变的时间间隔 → 换算出 RPM | 方波驱动，有 Hall |
| 反电动势频率 | 检测反电动势过零点频率 | 无传感器 BLDC |
| 编码器 | AB 相脉冲计数或绝对位置差分 | FOC / 伺服 |

**计算公式**:  
`RPM = 60 / (6 × 极对数 × T_60deg)`  
其中 T_60deg 是两个相邻 Hall 跳变的时间间隔（秒）。

### Q6: PID 参数怎么整定？

**标准回答**:
1. 先用纯比例 Kp：从小到大调，找到出现振荡的 Kp
2. 加入 Ki 消除稳态误差：从 0 开始逐步增大
3. Kd 抑制超调：最后加上，通常值很小
4. 注意积分饱和 (integral windup) → 加积分限幅

**动手能力加分**: "我调试电机 PID 时，用串口把转速实时发给上位机，用 Python matplotlib 画阶跃响应曲线来调参，能看到超调和稳态误差。"

### Q7: 电流采样为什么放在下桥臂而不是上桥臂？

**回答**: 下桥臂采样时，shunt 电阻一端接地，测量电压是单端对地的，运放可以直接采样，不需要差分隔离。上桥臂采样则需要差分运放或隔离运放，电路复杂且成本高。缺点是在续流阶段（下管关断时）无法测量电流。

---

## 8. 学习资源

### 8.1 推荐视频 (B站)

| 视频 / UP主 | 链接关键词 | 说明 |
|------------|-----------|------|
| 硬石科技 YS-F4 BLDC 教程 | bilibili 搜索 "硬石 BLDC" | STM32F4 + 六步换向实战，代码开源 |
| 正点原子 BLDC 教程 | bilibili 搜索 "正点原子 无刷电机" | STM32 + 驱动板配套教程 |
| JDH FOC 系列 | 搜索 "JDH FOC" | FOC 从零实现，极好的数学讲解 |
| ODrive 社区教程 | 搜索 "ODrive 调试" | 开源驱动器调试 |
| ST 官方 Webinar | ST 官网 / YouTube | Motor Control SDK 入门 |

### 8.2 开源项目

| 项目 | 链接 | 特点 |
|------|------|------|
| **SimpleFOC** | github.com/simplefoc | Arduino/STM32 FOC 库，文档极好，入门首选 |
| **ODrive** | github.com/odriverobotics | 高性能双电机 FOC 驱动器，固件开源 |
| **VESC** | github.com/vedderb/bldc | 经典开源电调，支持 FOC 和方波 |
| **MESC** | github.com/davidmolony/MESC_FOC | STM32G4 FOC，代码精悍 |

### 8.3 ST 官方文档（必须读的）

| 文档 | 内容 |
|------|------|
| **AN1946** | Sensorless BLDC 控制理论 + 代码实现 |
| **UM2392** | ST Motor Control SDK 用户手册 |
| **AN5464** | STM32 电机控制 SDK v6.x 入门 |
| **STM32 TIM 高级定时器参考手册** | 死区、互补 PWM、刹车功能 |
| **你的驱动板芯片 Datasheet** | 典型应用电路、死区建议 |

### 8.4 学习路线建议（分阶段）

```
阶段 1: 理论入门（2-3 天）
├── 看 B 站 BLDC 科普视频，理解六步换向
├── 阅读本文档第 1-2 章节
├── 手动画出六步换向序列表
└── 用万用表和示波器测量你的电机

阶段 2: 代码实战（3-5 天）
├── 搭建硬件（电机 + 驱动板 + STM32）
├── 写 TIM1 互补 PWM 配置，用示波器验证死区
├── 写 Hall 中断 + 换向表，开环转起来
├── 用逻辑分析仪验证换向时序
└── 加 PID 速度闭环

阶段 3: 功能完善（2-3 天）
├── 电流采样 + 过流保护
├── 串口调参命令（PID 参数在线修改）
├── 上位机显示速度/电流曲线
└── 整理代码，准备面试展示

阶段 4: FOC 入门（5-7 天，可选）
├── 看 JDH FOC 系列视频，理解 Clarke/Park
├── 跑通 SimpleFOC 例程
├── 尝试 ST MCSDK 生成 FOC 工程
└── 对比六步 vs FOC 的噪音/效率差异
```

### 8.5 动手实验检查清单

| 序号 | 实验 | 工具 | 掌握标志 |
|:---:|------|------|----------|
| 1 | 测量三相电阻 | 万用表 | 三相等值 |
| 2 | 观察反电动势波形 | 示波器 | 看到梯形/正弦波 |
| 3 | 验证 Hall 传感器输出 | 万用表/LA | 转轴时 0V/5V 跳变 |
| 4 | 配置 TIM1 互补 PWM + 死区 | 示波器 | 上下管波形互补，死区 1µs |
| 5 | 六步换向开环运转 | 电机转 | 电机平稳旋转 |
| 6 | 逻辑分析仪验证换向 | LA | Hall 跳变 → PWM 切换正确 |
| 7 | PID 闭环调速 | 串口/上位机 | 阶跃响应 <200ms 稳定 |
| 8 | 电流采样 + 过流保护 | 堵转测试 | 堵转自动停机 |

---

> **最后的话**: BLDC 是嵌入式面试的"硬通货"—— 几乎每家中大型公司做电机控制、机器人、无人机的都会问。六步换向是底线，FOC 是加分项。但更重要的是你能**实际动手**把电机转起来，能用示波器和逻辑分析仪验证你的逻辑。代码可以背，但波形不能骗人。

---

*文档持续更新中。完成一个阶段后回来更新你的实验记录。*
