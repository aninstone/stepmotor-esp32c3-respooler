# ESP32-C3 移植 + BLE 调试经验总结

项目：Respooler 步进电机控制板（ESP32-S3 → 合宙 CORE-ESP32-C3 移植）
日期：2026-10-13

---

## 1. 合宙 CORE-ESP32-C3 引脚铁律（血泪教训）

### ❌ 绝对不能用的脚（连着 SPI Flash，一配置就无限重启）

| GPIO | 用途 |
|---|---|
| GPIO11 | Flash 供电 VDD_SPI（特殊，需配置才能当 GPIO） |
| GPIO14 | Flash 片选 CS |
| GPIO15 | Flash 时钟 CLK |
| GPIO16 | Flash 数据 D0 |
| GPIO17 | Flash 数据 D1 |

**症状**：串口只打出第一行日志就重启（`[BOOT]1` 无限循环）。
因为 `pinMode()`/`ledc` 一碰这些脚 → Flash 总线断 → 取指失败 → 秒重启。

### ⚠️ Strap 脚（能用但别在上电时拉错电平）

| GPIO | 注意事项 |
|---|---|
| GPIO2 | 上电须为高（下载模式判断） |
| GPIO8 | 上电别外部下拉（影响烧录） |
| GPIO9 | BOOT 按键脚，上电别拉低 |

### ✅ 完全可用

- GPIO0~10（除 2/8/9 strap 注意事项）、GPIO12、GPIO13
- GPIO12/13 仅 QIO 模式被 Flash 占用；**板子是 DIO 模式 → 空闲**
- GPIO18/19（原生 USB D-/D+，不用原生 USB 就空闲）、GPIO20/21（UART0，连 CH340 调试口，别抢）

### 官方资料
- 引脚表：https://wiki-zh.luatos.org/chips/esp32c3/board.html
- 板载 LED：**D4=GPIO12、D5=GPIO13**（高电平亮）；按键 BOOT=GPIO9
- 原理图：`D:\WorkPRJ\工作同事\合宙ESP32\CORE-ESP32-C3设计原理图_V1.2.pdf`

### 本项目最终引脚分配（已实测跑通 Web Bluetooth 全链路）

| GPIO | 功能 | | GPIO | 功能 |
|---|---|---|---|---|
| 2 | 断丝检测 | | 9 | NeoPixel |
| 4 | 步进使能 EN | | 10 | 按钮 |
| 5 | 连接状态灯 | | 12/13 | TMC UART RX/TX（=板载D4/D5脚，DIO模式空闲） |
| 6/7 | STEP/DIR | | 19 | 风扇 |
| 8 | 舵机 | | 3 / 1 | **OLED SDA / SCL**（注意顺序！实物是 SDA→GPIO3、SCL→GPIO1） |
| 0 | 编码器 A (CLK) | | 18 | 编码器 B (DT)（v1.3.0 新增） |

---

## 2. BLE 扫不到的排查清单（按踩坑顺序）

1. **先确认固件真的活着**：串口心跳 `[LOOP] alive` 每 5 秒打一次。没心跳 = 先修崩溃，别怀疑蓝牙。
2. **广播包 31 字节上限**：名字 + UUID + 厂商数据全塞一个包 → 超限 → NimBLE 静默发畸形包 → 所有扫描器丢弃。
   - 标准做法：**广播包** = Flags + 名字 + 服务UUID（3+8+18=29 ✅）；**扫描响应** = 厂商数据
   - 想加 TX power 就得砍别的，`addTxPower()` 占 3 字节
3. **`NimBLEDevice::setMTU(512)` 别在 C3 上用**：内存分配失败 → 协议栈半初始化 → 不广播也不报错。默认 247 够用。
4. **nRF 能扫到 ≠ 网页能扫到**：
   - nRF Connect 无过滤，显示一切广播
   - 系统蓝牙设置（Win/Android/iOS）只列标准 profile（耳机/键鼠），自定义 GATT 设备被隐藏——**正常现象**
   - **Web Bluetooth（control.lts-design.com）用 `filters:[{services:[UUID]}]` 按 UUID 匹配，广播包里必须带服务 UUID**，否则列表永远空白
5. **NimBLE 1.x API 坑**：
   - `NimBLEAdvertisementData` 没有 `addServiceUUID()`，要用 **`setCompleteServices(NimBLEUUID(...))`**
   - 自定义包（`setAdvertisementData`/`setScanResponseData`）会**完全覆盖** `pAdvertising->addServiceUUID()` 加的内容
   - `setManufacturerData()` 会自动加 0xFF 类型字节，payload 里别再带 0xFF
6. **连上后广播自动停**：nRF 里点了 CONNECT，其他设备就扫不到了。断开恢复。

### Web Bluetooth 客户端注意
- 仅 Chrome/Edge（桌面 + Android）；**iOS Safari 不支持**
- 网站过滤条件（webble-manager.js）：service `9E05D06D-...AC101`，特征 `7CB2F1B4-...B1A8`（WRITE+NOTIFY）

---

## 3. 排查方法论

- **崩溃循环**：先看串口停在哪个 `[BOOT] n` → 卡在哪一步 → 对照该步初始化的外设查引脚冲突
- **移植项目**：先把所有 `#define xx_PIN` 列出来对照目标芯片 datasheet 过一遍再编译，别等运行时炸
- **验证链路**：把可观测的输出（板载 LED）挂到被测数据（亮度滑条）上，一眼确认端到端通没通

---

## 4. 本次修改记录

| 文件 | 修改 |
|---|---|
| main.cpp | STEP 14→6、DIR 16→7（Flash 脚）；TMC UART 12/13→1/3（腾出板载 LED）；删 `setMTU(512)`；广播包重排（名字+UUID 主包，厂商数据扫描响应）；删除误用的 `addTxPower` |
| main.cpp | 新增 D4 亮度镜像：GPIO12 走 LEDC CH4/Timer0，在 `calculatePWM()` 里统一更新，网页滑条 → BLE → pwmValue → D4 逐级可验证 |

**改板接线**：TMC-TX→GPIO13、TMC-RX→GPIO12、STEP→GPIO6、DIR→GPIO7。
（验证技巧：临时把输出镜像到板载 D4=GPIO12 拖滑条看渐变，即可端到端确认链路，验证完撤掉。）

---

## 5. OLED（SSD1306 / I2C）黑屏排障 —— 2026-10-18，烧了两个小时的教训

### 🎯 最终真凶：屏的 SDA / SCL 两根线实物接反了

| | 固件原定义 | 实物实际 |
|---|---|---|
| SDA | GPIO1 | **GPIO3** |
| SCL | GPIO3 | **GPIO1** |

I2C 两线反过来 → 主机发什么从机都听不见 → 全总线 0 应答。
**修复**：`src/main.cpp` 改为 `#define OLED_SDA 3 / #define OLED_SCL 1`（已加注释锁死，别再"修正"回 1/3）。

### 🕳️ 三个假象，把排查带偏了两小时

| # | 假象 | 真相 |
|---|---|---|
| 1 | 串口一直打 `[BOOT] 7 oled ok`，以为初始化成功 | **`Adafruit_SSD1306::begin()` 根本不探测设备**，只管把命令发出去，屏没接也返回 true。这个"ok"是纯假象 |
| 2 | 诊断固件能亮 → 判定"正式固件有问题" | 诊断固件里写了**自动对调试探**，它悄悄换成 (3,1) 才亮的，而且串口里早就打印了 `>>> YOUR SDA/SCL WIRES ARE SWAPPED! <<<` —— 当时只看屏幕、没抓串口日志，线索就这么丢了 |
| 3 | "BLE 能扫到，说明系统没毛病" | 蓝牙是**射频**，走芯片内部，跟 GPIO 半点关系没有。I2C 死了蓝牙照样广播。一好一坏完全不矛盾 |

在错误层级上依次试错过（**全部无效，别再走**）：把 OLED init 挪到 BLE 之前/之后、加 `oled.dim()` 降电流、砍掉开机全屏白测试、`Wire.setTimeOut()` 调参、怀疑 3.3V 欠压要改 5V 供电、怀疑框架/系统选型。

### ✅ 真正定位只用了 2 次烧录

1. **正式固件加全地址扫描** → 打印 `[OLED] scan: NOTHING on the bus!`
   结论：不是地址错（0x3C/0x3D）、不是 BLE 干扰 —— 是**基础通信压根没建立**
2. **烧诊断固件 + 抓串口日志**：
   ```
   [1] Wire.begin(SDA=1, SCL=3) scan → bus empty
   [2] SWAPPED (SDA=3, SCL=1) → 0x3C ACK   ← 屏在这组引脚上
       >>> YOUR SDA/SCL WIRES ARE SWAPPED! <<<
       begin() = 1 → 棋盘格 + "I2C OK!" 正常显示
   ```

### 🔧 三条铁律（下次直接用）

1. **判断外设在不在，永远自己扫，不信 `begin()` 返回值**
   ```cpp
   Wire.beginTransmission(addr);
   bool alive = (Wire.endTransmission() == 0);
   ```
2. **诊断固件要把结论打到串口，而且必须抓串口日志** —— 屏幕只告诉你"亮/不亮"，线序、地址、ACK 只在日志里
3. **排查顺序固定，不许跳级**：
   `物理层（线序 / 地址 / 上拉 / 接触）→ 电气（电压 / 电流 / 共地）→ 代码 → 框架 / 系统`
   物理层没验过就往上查 = 白烧时间

### 🧰 可复用的诊断工具（工程内已保留）

| 文件 | 作用 |
|---|---|
| `src/oled_diag.cpp` + `platformio.ini` 的 `[env:oled_diag]` | 独立 OLED 诊断固件：全地址扫描 → 自动试 SDA/SCL 对调 → 棋盘格 → `I2C OK! + addr`，全程串口打印 |
| `serial_capture.py`（`D:\workbuddy\2026-10-11-19-03-13\`） | 复位板子并抓 25 秒串口日志 |

用法：`pio run -e oled_diag -t upload` 然后抓串口 —— **5 分钟内出线序/地址结论**。

### 📌 顺便（模块硬件要点，防复发）

- 旺泓 0.96" 七脚模块出厂是 **4 线 SPI**，改 I2C 要动背面电阻（R3→R1、R8 短接）；**本项目最终用的是原生 4 脚 I2C 屏**
- I2C 模式下 `RES→3.3V`、`DC→GND`（地址 0x3C）、`CS→GND` 三个脚必须接固定电平，**悬空 = 黑屏**
- 正式固件现已在启动时运行时扫描地址（0x3C / 0x3D 自适应），不再硬编码

---

## 6. 旋转编码器调速（2026-10-19，固件 v1.3.0）

### 需求

用旋转编码器调速度（SPD 50–100），**快速旋转要能加速调整**，且**编码器优先级高于蓝牙**。
库指定 `igorantolic/Ai Esp32 Rotary Encoder@^1.7`。
参考工程：`D:\WorkPRJ\VscodePRJ\ESP32PRJ\12-28(已经实现spinbox位闪烁)\12-28\esp32_04_key ai encoder_modbus`
（实际实现在 `lib/my_lvgl/src/my_key.cpp`，不在 main.cpp 里，找的时候别只搜 main）。

### 参考工程的关键写法

```cpp
AiEsp32RotaryEncoder rotaryEncoder =
    AiEsp32RotaryEncoder(A_PIN, B_PIN, SW_PIN, /*VCC*/-1, /*STEPS*/4);

rotaryEncoder.begin();
rotaryEncoder.setup(readEncoderISR);                 // ISR 里只调 readEncoder_ISR()
rotaryEncoder.setBoundaries(-1000, 1000, true);      // true = 到边界回绕
rotaryEncoder.setAcceleration(250);                  // 加速度

void IRAM_ATTR readEncoderISR() { rotaryEncoder.readEncoder_ISR(); }

// 主循环取值
if (rotaryEncoder.encoderChanged()) {
  long v = rotaryEncoder.readEncoder();
  long delta = v - lastValue;
  lastValue = v;
}
```

- ⚠️ 参考工程里调了 `disableAcceleration()`（注释写着"默认已启用"）。**要"快转加速"就必须让它启用**，`setAcceleration(0)` 才是关闭。
- 加速度原理（库源码）：相邻两次脉冲间隔 < 200 ms 时，`encoder0Pos += coef / 间隔ms`，所以**转得越快，单次 delta 越大**；慢转就是干净的 1 格 1 计数。

### 本工程实现

引脚（C3 只剩这两个空闲脚；18/19 是原生 USB D-/D+，本板串口走 CH343，所以 19 已给风扇、18 可用）：

| 编码器 | GPIO | 宏 |
|---|---|---|
| CLK (A) | 0 | `ENC_A_PIN` |
| DT (B) | 18 | `ENC_B_PIN` |
| SW | 不接 | `ENC_SW_PIN -1` |

可调宏（都在 `src/main.cpp` 顶部）：

```cpp
#define ENC_STEPS 4              // 转一格跳 2% 就改成 2
#define ENC_ACCEL 90             // 0 = 关加速；越大猛转跳得越狠
#define ENC_MAX_JUMP 10          // 单次事件最多 ±10%
#define ENC_BLE_LOCKOUT_MS 2000  // 编码器优先时长
#define ENC_NVS_SETTLE_MS 800    // 防抖写 flash
```

四个函数的分工：

| 函数 | 职责 |
|---|---|
| `applySpeedPercent(pct)` | **唯一速度换算入口**：`间隔µs = 170 − (SPD−50) × 77 / 50`，再 `applySpeedTarget()`。BLE 的 `SPD` 命令也改成调它 |
| `pollRotaryEncoder()` | 每 loop 调一次。`inc = 1 + |delta|/4`（上限 `ENC_MAX_JUMP`），慢转 1 格 = ±1%，快转自动放大 |
| `flushRotaryEncoder()` | 变速后 150 ms 推一次 `sendStatus(true)` 给手机（节流）；手停 800 ms 后才写 NVS |
| `encoderHasPriority()` | 编码器动过 2 秒内，BLE 的 `SPD` 直接忽略，串口打 `[BLE] SPD ignored (encoder has priority)` |

设计取舍：
- **NVS 必须防抖**：每转一格就 `prefs.putUInt()` 会疯狂磨损 flash，改成"静止 800 ms 才落盘"
- **BLE 回传要节流**：notify 太频繁会挤占 BLE 栈，150 ms 一次足够让 App 滑条跟手
- **优先级只锁 2 秒**：既能挡住 App 滑条回写旧值，又不至于让手机永久失控

### 踩过的坑

1. **前向声明**：`applySpeedPercent()` / `encoderHasPriority()` 定义在 loop 区（文件后部），但 BLE 命令处理在文件前部 → 编译报 `not declared`，必须在顶部前向声明区加两行。
2. **`-1` 按键脚不是所有方法都安全**：库的 `setup()` / `begin()` 里有 `if (encoderButtonPin >= 0)` 保护，但 `isEncoderButtonDown()` / `isEncoderButtonClicked()` **没有**，不接按键时千万别调这两个。
3. **又烧错环境**：工程里曾有 `[env:oled_diag]`（屏幕显示 `I2C OK!`），烧错后看起来像"主固件坏了"，白折腾。已把该 env 整段注释 + `[platformio] default_envs = esp32c3`，现在只有一个环境。
4. **编译一次约 2 分钟**，改完代码交给用户自己编译，不要反复跑 pio。

### 现场调试对照表

| 现象 | 改法 |
|---|---|
| 顺时针速度变小（方向反） | `ENC_A_PIN` / `ENC_B_PIN` 对调 |
| 转一格跳 2% | `ENC_STEPS` 4 → 2 |
| 猛转跳太猛 / 太肉 | `ENC_ACCEL` 调小 / 调大 |
| 手机一拖滑条就把编码器值顶掉 | `ENC_BLE_LOCKOUT_MS` 加大 |
| 变速后手机界面不动 | 检查 `sendStatus(true)` 是否被 `!deviceConnected` 挡掉 |

### 验证方法

```
[BOOT] 2c rotary encoder ok      ← 编码器初始化成功
```

转编码器 → OLED 上 `SPD xx%` 立刻变、电机转速跟着变、手机滑条同步；
2 秒内用手机发 `SPD` 应看到 `[BLE] SPD ignored (encoder has priority)`。
