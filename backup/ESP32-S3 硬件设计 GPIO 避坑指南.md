在使用 ESP32-S3 画自制板时，GPIO 不能只看“能不能配置成普通 IO”。有些引脚虽然软件上可以 `pinMode()`，但它们在上电启动、Flash/PSRAM、USB、JTAG、UART 下载等阶段有特殊作用。

如果把这些引脚拿去做 `LCD_CS`、`LCD_RESET`、背光控制、继电器、MOS 使能、I2C 上拉线等关键功能，就容易出现：

```text
程序正常运行，但屏幕白屏
串口异常
无法下载
上电不稳定
反复复位
外设偶发失效
```

本文总结 ESP32-S3 画板时需要避开的特殊 GPIO，并给出推荐用法。

---

## 1. 先记住一个总原则

ESP32-S3 的 GPIO 并不是从 `GPIO0 ~ GPIO48` 全部连续可用。

ESP32-S3 常见可用 GPIO 范围为：

```text
GPIO0 ~ GPIO21
GPIO26 ~ GPIO48
```

也就是说：

```text
GPIO22 ~ GPIO25 不存在 / 不应使用
```

ESP32-S3 通过 GPIO Matrix 可以把很多外设信号映射到不同 GPIO，但这不代表每个 GPIO 都适合做关键硬件控制脚。

---

## 2. 强烈不建议用于关键外设的 GPIO

这些 GPIO 最容易踩坑：

```text
GPIO0
GPIO3
GPIO45
GPIO46
```

它们是 ESP32-S3 的 Strapping Pins，也就是启动配置采样脚。芯片上电或复位时，会读取这些引脚的电平，用来决定启动模式、Flash 电压等关键行为。

### 避免用途

不要把这些脚用于：

```text
LCD_CS
LCD_RESET
LCD_RD
LCD_BLK / 背光控制
电源使能
MOS Gate
继电器控制
蜂鸣器
I2C SDA / SCL
外设片选 CS
外设复位 RESET
外部强上拉 / 强下拉网络
```

### 建议用途

| GPIO | 建议 |
|---:|---|
| GPIO0 | 只做 BOOT 按键，默认 10k 上拉到 3.3V，按键到 GND |
| GPIO3 | 尽量不用；如果必须用，只接高阻输入，不接关键控制 |
| GPIO45 | 尽量不用；不要做背光、CS、RESET、MOS 控制 |
| GPIO46 | 尽量不用；不要做 LCD_CS、RESET、片选等关键输出 |

---

## 3. Flash / PSRAM 相关 GPIO

这些 GPIO 通常连接到模组内部的 Flash / PSRAM：

```text
GPIO26 ~ GPIO32
```

如果使用 Octal Flash 或 Octal PSRAM，下面这些也会被内部 Flash / PSRAM 占用：

```text
GPIO33 ~ GPIO37
```

对于常见的 ESP32-S3-N8R8、ESP32-S3-N16R8 这类带 OPI PSRAM 的模组，这一组引脚默认不要当普通 GPIO 用。

### 避免用途

```text
GPIO26 ~ GPIO37 不建议外接任何普通外设
```

不要拿来接：

```text
LCD 数据线
I2C
SPI
按键
LED
继电器
背光
外设中断
```

否则可能导致：

```text
无法启动
PSRAM 初始化失败
Flash 读写异常
程序随机崩溃
烧录后不运行
```

---

## 4. USB 相关 GPIO

ESP32-S3 支持原生 USB CDC / USB Serial-JTAG。常用引脚是：

```text
GPIO19 = USB D-
GPIO20 = USB D+
```

如果你想保留 ESP32-S3 原生 USB 下载、调试、CDC 串口，那么：

```text
GPIO19 / GPIO20 不要再接其他外设
```

### 推荐设计

```text
GPIO19 -> USB D-
GPIO20 -> USB D+
```

同时 USB-C 作为设备端时，CC1 / CC2 要有下拉电阻：

```text
CC1 -> 5.1k -> GND
CC2 -> 5.1k -> GND
```

如果同时设计 CH340 和原生 USB CDC，要注意两路 USB 的 5V 不要直接硬并联，避免反灌。

---

## 5. UART0 / CH340 相关 GPIO

ESP32-S3 默认 UART0 常用：

```text
GPIO43 = U0TXD
GPIO44 = U0RXD
```

如果板子上有 CH340 / CP2102 / USB-TTL，用于传统串口下载和日志输出，建议保留：

```text
CH340_TXD -> ESP32_RXD0 / GPIO44
CH340_RXD -> ESP32_TXD0 / GPIO43
```

不要让 GPIO43 / GPIO44 再接其他大负载。

### 自动下载电路注意

CH340 的 DTR / RTS 自动下载电路会控制：

```text
EN
GPIO0
```

如果三极管方向、DTR/RTS 逻辑接错，会出现：

```text
打开串口就复位
进入下载模式
串口打不开
设备管理器 COM 口异常
```

建议自动下载电路预留电阻，方便断开调试。

---

## 6. JTAG 相关 GPIO

ESP32-S3 还有一组 JTAG 相关引脚：

```text
GPIO39
GPIO40
GPIO41
GPIO42
```

如果不使用外部 JTAG，这些脚可以作为普通 GPIO 使用。但如果后续需要 JTAG 调试，就不要占用它们。

### 建议

| GPIO | 可以使用吗 | 注意 |
|---:|---|---|
| GPIO39 | 可以 | 会占用 JTAG 功能 |
| GPIO40 | 可以 | 会占用 JTAG 功能 |
| GPIO41 | 可以 | 会占用 JTAG 功能 |
| GPIO42 | 可以 | 会占用 JTAG 功能 |

如果项目不需要 JTAG，比如普通量产设备，这组可以用作 I2C、按键、低速控制线等。

---

## 7. 推荐优先使用的普通 GPIO

如果没有被你的外设占用，优先选择这些 GPIO 做普通控制：

```text
GPIO1
GPIO2
GPIO4 ~ GPIO18
GPIO21
GPIO38
GPIO47
GPIO48
```

但是仍然要结合具体功能判断：

```text
高速 LCD 并口：优先选连续、走线短、旁边有地参考的 GPIO
I2C：任意普通 GPIO 可用，但必须有 4.7k 上拉
PWM 背光：选普通 GPIO，不要用 Strapping Pin
RESET / CS / EN：必须选上电状态可控、没有启动副作用的 GPIO
```

---

## 8. LCD 屏幕设计重点避坑

如果使用 ILI9488 / ST7796 / ILI9341 等 8080 并口屏，LCD 控制线非常关键：

```text
LCD_CS
LCD_RD
LCD_WR
LCD_RS / DC
LCD_RESET
LCD_BLK
D0 ~ D15
```

### 8.1 LCD_CS 推荐固定接地

如果系统里只有一块 LCD，不需要共享总线，最稳方案是：

```text
LCD_CS -> GND
```

或者预留 0R 电阻：

```text
LCD_CS -- 0R -- GND
LCD_CS -- 预留 0R -- ESP32 GPIO
```

默认焊接到 GND。

这样可以避免 LCD_CS 上电悬空、被特殊 GPIO 影响、初始化阶段没有选中屏幕等问题。

---

### 8.2 LCD_RD 推荐固定拉高

如果不读取 LCD ID 或显存，LCD_RD 没必要接 ESP32。

推荐：

```text
LCD_RD -> 3.3V
```

或者：

```text
LCD_RD -> 10k -> 3.3V
```

如果 LCD_RD 被拉低或悬空，屏幕可能进入读模式，导致写命令失败，典型表现就是：

```text
背光亮
程序正常运行
fillScreen 无效
屏幕一直白屏
```

---

### 8.3 LCD_RESET 不建议悬空

推荐：

```text
LCD_RESET -> 10k -> 3.3V
LCD_RESET -> 0.1uF -> GND
```

如果需要软件控制：

```text
ESP32 GPIO -> 100R ~ 1k -> LCD_RESET
LCD_RESET -> 10k -> 3.3V
LCD_RESET -> 0.1uF -> GND
```

不要把 LCD_RESET 直接硬接到 ESP32 的 EN，除非明确知道复位时序和负载不会互相影响。

---

### 8.4 背光 BLK 不要用 GPIO45 / GPIO46

背光控制建议使用普通 GPIO。

推荐结构：

```text
ESP32 GPIO -> 100R / 1k -> MOS Gate
MOS Gate -> 100k -> GND
MOS 控制背光电流
```

不要用 GPIO45 / GPIO46 控制背光。

原因是 GPIO45 / GPIO46 属于启动相关脚，上电阶段的外部电路可能影响启动或导致电平异常。

---

## 9. I2C 设计避坑

ESP32-S3 的 I2C 可以映射到很多普通 GPIO，但硬件上必须注意：

```text
SDA -> 4.7k -> 3.3V
SCL -> 4.7k -> 3.3V
```

如果挂多个 I2C 设备，比如触摸芯片、PCF8574、传感器，要注意：

```text
所有设备必须共地
地址不能冲突
SDA/SCL 空闲电平应接近 3.3V
不要让某个设备掉电后反向拖低 SDA/SCL
线长较长时降低 I2C 频率到 100kHz
```

I2C 异常常见表现：

```text
程序卡在 I2C scan
触摸初始化失败
PCF8574 扫出一堆假地址
SDA/SCL 只有 1.xV 或 2.xV
```

---

## 10. EN / BOOT 设计建议

### EN 推荐电路

```text
3.3V
 |
10k
 |
EN
 |
0.1uF ~ 1uF
 |
GND
```

复位按键：

```text
EN -> 按键 -> GND
```

EN 正常运行时应稳定在：

```text
3.3V 左右
```

如果 EN 是 0.4V、1.xV 或跳动，ESP32 会无法启动或反复复位。

---

### BOOT / GPIO0 推荐电路

```text
GPIO0 -> 10k -> 3.3V
GPIO0 -> BOOT 按键 -> GND
```

不要把 GPIO0 接到 LCD、继电器、传感器输出、MOS 等外设上。

GPIO0 被拉低时，ESP32-S3 可能进入下载模式，导致应用程序不启动。

---

## 11. 电源与 USB 供电避坑

如果板子同时有：

```text
USB-C 原生 CDC
CH340 USB
外部 5V 电源
```

必须考虑 5V 反灌。

不要简单把多路 5V 全部硬并联。建议使用：

```text
肖特基二极管
理想二极管
电源选择跳帽
保险丝 / PTC
0R 电阻选择
```

5V 输入端建议加：

```text
100uF ~ 470uF 电解/钽电容
0.1uF 陶瓷电容
```

3.3V 稳压建议：

```text
输出能力 >= 800mA，推荐 1A 以上
ESP32 模组旁边放 10uF + 0.1uF
屏幕、触摸、PCF 等外设旁边各放 0.1uF
```

---

## 12. ESP32-S3 GPIO 风险分级表

| 等级 | GPIO | 建议 |
|---|---|---|
| 禁止普通使用 | GPIO26 ~ GPIO37 | Flash / PSRAM 相关，不要接外设 |
| 强烈不建议 | GPIO0 / GPIO3 / GPIO45 / GPIO46 | Strapping Pins，不要接关键控制 |
| 保留 USB | GPIO19 / GPIO20 | 原生 USB CDC / USB Serial-JTAG |
| 保留 UART0 | GPIO43 / GPIO44 | CH340 / 串口日志 / 下载 |
| 谨慎使用 | GPIO39 ~ GPIO42 | JTAG 相关，不用 JTAG 时可用 |
| 优先普通 IO | GPIO1 / GPIO2 / GPIO4~18 / GPIO21 / GPIO38 / GPIO47 / GPIO48 | 适合普通外设，仍需结合实际功能 |

---

## 13. 推荐硬件设计清单

画 ESP32-S3 自制板时，可以直接按下面原则检查：

```text
[ ] GPIO0 只做 BOOT，不接其他外设
[ ] GPIO45 / GPIO46 不做关键控制脚
[ ] GPIO26~37 不接外设，留给 Flash / PSRAM
[ ] GPIO19 / GPIO20 留给原生 USB
[ ] GPIO43 / GPIO44 留给 CH340 或测试串口
[ ] LCD_CS 默认接 GND，或通过 0R 选择
[ ] LCD_RD 默认接 3.3V，或通过 10k 上拉
[ ] LCD_RESET 有 10k 上拉和 0.1uF 电容
[ ] 背光控制用普通 GPIO + MOS，不用 Strapping Pin
[ ] I2C SDA/SCL 有 4.7k 上拉
[ ] EN 有 10k 上拉和 0.1uF~1uF 电容
[ ] USB-C CC1/CC2 有 5.1k 下拉
[ ] 多路 5V 供电之间有隔离或选择方案
[ ] LCD、触摸、I2C、UART、USB 都预留测试点
```

---

## 14. 针对 LCD 白屏的快速排查法

如果遇到：

```text
串口正常
程序 alive 正常
背光亮
屏幕一直白屏
fillScreen 无效
```

优先检查：

```text
1. LCD_CS 是否真正为低电平
2. LCD_RD 是否真正为高电平
3. LCD_RESET 是否释放到 3.3V
4. LCD_WR 是否有写脉冲
5. LCD_RS/DC 是否能翻转
6. D0~D15 是否错线、虚焊、短路
7. LCD 接口模式 IM0~IM3 是否为 8080 并口模式
8. LovyanGFX 的 pin 配置是否与原理图一致
9. freq_write 是否过高，必要时降到 2MHz / 5MHz 测试
10. 换屏交叉验证
```

实际经验中，如果 LCD_CS 或 LCD_RD 没有默认固定电平，即使程序和电源都正常，也可能表现为白屏。

---

## 15. 推荐的 LCD 连接策略

对于单 LCD 设备，推荐这样设计：

```text
LCD_CS    -> GND
LCD_RD    -> 3.3V
LCD_RESET -> 10k 上拉 + 0.1uF 到 GND，可选 GPIO 控制
LCD_BLK   -> 普通 GPIO 控制 MOS
LCD_WR    -> 普通 GPIO
LCD_RS/DC -> 普通 GPIO
D0~D15    -> 普通 GPIO，避免 Flash/PSRAM/Strapping/USB/UART0
```

这样能大幅降低屏幕白屏、初始化失败和启动异常的概率。

---

## 16. 总结

ESP32-S3 的 GPIO 很灵活，但画硬件时不能只看软件能不能配置。最容易踩坑的是：

```text
GPIO0 / GPIO3 / GPIO45 / GPIO46
GPIO26 ~ GPIO37
GPIO19 / GPIO20
GPIO43 / GPIO44
GPIO39 ~ GPIO42
```

对于关键外设，尤其是 LCD、触摸、I2C、背光、电源控制和复位控制，建议优先使用普通 GPIO，并给所有关键控制脚设计明确的默认电平。

最实用的一句话：

```text
启动脚不做关键控制，Flash/PSRAM 脚不碰，USB/UART 调试脚保留，LCD_CS 固定低，LCD_RD 固定高。
```

---

## 参考资料

- Espressif ESP-IDF GPIO & RTC GPIO 文档：`https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/gpio.html`
- Espressif ESP32-S3 技术参考与 Datasheet：`https://www.espressif.com/en/support/documents/technical-documents`
