# LTS Respooler 控制板固件（ESP32-C3）

合宙 CORE-ESP32-C3 核心板的 LTS Respooler 控制固件：步进电机收线、TMC2209 驱动、
Web Bluetooth（网页直连）控制、SSD1306 OLED 状态屏、旋转编码器调速。

- 原工程基于 ESP32-S3，已完整移植到 ESP32-C3（引脚、BLE 广播包、内存限制都改过）
- 手机/电脑用 Chrome 或 Edge 打开控制页，通过 Web Bluetooth 连接（iOS Safari 不支持）

---

## 硬件引脚分配（合宙 CORE-ESP32-C3）

| GPIO | 功能 | 备注 |
|---|---|---|
| 2 | 断丝检测 | 上电勿外部下拉（strap） |
| 4 | 步进使能 EN | 低有效 |
| 5 | 连接状态灯 | PWM |
| 6 / 7 | STEP / DIR | |
| 8 | 舵机 | |
| 9 | NeoPixel | BOOT 键同脚，注意 strap |
| 10 | 按钮 | |
| 12 / 13 | TMC2209 UART RX / TX | DIO 模式空闲（= 板载 D4/D5） |
| 19 | 风扇 | 原 USB D+，本板串口走 CH343，故可用 |
| 3 / 1 | **OLED SDA / SCL** | ⚠️ 实物是 SDA→GPIO3、SCL→GPIO1，**不要改成 1/3** |
| 0 / 18 | **编码器 A(CLK) / B(DT)** | v1.3.0 新增，空闲脚 |
| 20 / 21 | UART0（CH343 调试口） | 不要占用 |

**绝对不能用**：GPIO11（VDD_SPI）、GPIO14（Flash CS）、GPIO15（Flash CLK）、
GPIO16/17（Flash D0/D1）——一配置就无限重启。

---

## 旋转编码器调速（v1.3.0）

### 接线

| 编码器 | ESP32-C3 |
|---|---|
| CLK (A) | GPIO0 |
| DT (B) | GPIO18 |
| + / VCC | 3.3V |
| GND | GND |
| SW（按键） | 不接（`ENC_SW_PIN -1`；板载按钮在 GPIO10） |

也可用 VCC=GND 接法（编码器无源），库参数 `-1` 表示不占用引脚供电。

### 行为

| 操作 | 结果 |
|---|---|
| 慢慢转 | 每格 ±1%，SPD 限制在 50–100 |
| 快速猛转 | 库加速度生效，单次最多跳 ±10%（`ENC_MAX_JUMP`） |
| 转完 150 ms | 自动 `sendStatus()` 推给手机，App 滑条同步 |
| 转完 2 秒内 | 手机发来的 `SPD` 命令**被忽略**（编码器优先），串口打印 `[BLE] SPD ignored (encoder has priority)` |
| 手停 800 ms | 才写一次 NVS（保护 flash，断电不丢） |
| 电机运转中 | 随时可调，立刻生效 |

速度换算与 BLE 的 `SPD` 命令完全一致：`间隔(µs) = 170 − (SPD−50) × 77 / 50`。

### 可调参数（`src/main.cpp` 顶部）

```cpp
#define ENC_A_PIN 0
#define ENC_B_PIN 18
#define ENC_SW_PIN -1
#define ENC_STEPS 4            // 转一格加 2% 时改成 2
#define ENC_ACCEL 90           // 0=关闭加速；越大猛转跳得越狠
#define ENC_MAX_JUMP 10        // 单次事件最大 ±10%
#define ENC_BLE_LOCKOUT_MS 2000
#define ENC_NVS_SETTLE_MS 800
```

方向反了 → 交换 `ENC_A_PIN` / `ENC_B_PIN`。

---

## 构建与烧录

**只有一个构建环境**，VSCode 里直接点 Upload 即可（`platformio.ini` 里已设
`default_envs = esp32c3`）。命令行：

```bash
pio run -e esp32c3 -t upload        # 编译 + 烧录
pio device monitor -b 115200        # 串口日志
```

> 工程里曾有一个 `[env:oled_diag]` 诊断固件（屏幕画棋盘格 + `I2C OK!`），
> 因容易被误烧、看起来像"主固件坏了"，已整段注释。需要测屏时去掉 `platformio.ini`
> 末尾的 `;` 恢复。

### 开机串口日志（正常）

```
[BOOT] 1 setup start
[BOOT] 2 gpio done
[BOOT] 2c rotary encoder ok        ← 编码器初始化
[OLED] scan 0x3C ACK               ← 屏在线（自动扫地址，0x3C/0x3D 自适应）
[BOOT] 2b oled ok (pre-BLE draw)
[BOOT] 6 TMC driver configured
[BOOT] NimBLEDevice::init ok
[BLE] name=LTS CB advStarted=1
[LOOP] alive state=I heap=17xxxx   ← 主循环心跳，每 5 秒
```

---

## 变更记录

### v1.3.0 — 2026-10-19
- **新增旋转编码器调速**（`AiEsp32RotaryEncoder` 1.7）：慢转 ±1%、快转加速放大，
  编码器优先级高于蓝牙（2 秒锁定），改动实时回传手机，NVS 写入做 800 ms 防抖
- `platformio.ini`：新增库依赖；设 `default_envs = esp32c3`；注释掉 `[env:oled_diag]`
- 速度换算统一收口到 `applySpeedPercent()`，BLE `SPD` 与编码器共用同一入口

### v1.2.1 — 2026-10-18
- **修复 OLED 黑屏**：SSD1306 的 SDA/SCL 实物接反（固件按 1/3、实物是 3/1），
  已改为 `OLED_SDA 3` / `OLED_SCL 1`；启动时改为 I2C 全地址扫描、地址自适应
- 移除调试探针（每秒 re-begin、t+8s 全总线扫描等）
- 详细排障过程见 [ESP32-C3移植与BLE调试经验总结.md](ESP32-C3移植与BLE调试经验总结.md)

### v1.2.0 — 2026-10-13
- ESP32-C3 移植完成（引脚重排、避开 Flash 脚）
- BLE 广播包重排（31 字节上限）：名字 + 服务 UUID 放广播包，厂商数据放扫描响应
- 删除 C3 上会导致协议栈半初始化的 `setMTU(512)`
