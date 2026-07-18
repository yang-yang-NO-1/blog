## 一、目标

本次目标是在 **中兴 F50** 上实现自动温控散热：

```text
F50 读取自身温度
↓
温度高于设定阈值
↓
通过 USB CDC 串口向 ESP32-S3 发送命令
↓
ESP32-S3 控制 GPIO 输出
↓
GPIO 控制 MOS 管
↓
打开散热风扇电源

温度低于设定阈值
↓
F50 发送关闭命令
↓
ESP32-S3 关闭 MOS
↓
风扇断电
```

最终结构：

```text
F50 ttyd/root shell
↓
/dev/ttyACM0
↓
USB CDC 串口
↓
ESP32-S3 SuperMini
↓
GPIO48
↓
MOS 管
↓
风扇
```

---

## 二、设备与环境

### 1. 主设备

```text
设备：中兴 F50
状态：已通过 UFI-TOOLS / ttyd 获得 root shell
管理入口：ttyd
ttyd 端口：1146
官方后台端口：8080
UFI-TOOLS 后台端口：2333
```

### 2. 控制板

```text
开发板：ESP32-S3 SuperMini
芯片：ESP32-S3FH4R2
Flash：4MB
USB：原生 USB CDC / USB Serial-JTAG
串口设备：/dev/ttyACM0
```

该 ESP32-S3 SuperMini 商品页标注：

```text
内置 4M flash
板载 RGB、蓝色 LED：公用 GPIO48 引脚
WS2818 RGB LED 占用 GPIO48
```

本次测试中 GPIO48 可用于控制验证。

---

## 三、F50 无线 ADB 与 UFI-TOOLS 状态

使用 `aaadddbbb.exe` 临时打开无线 ADB 时，输出类似：

```text
[1/6] 发送数据中...
[2/6] 发送数据中...
[3/6] 重启设备...
重启命令发送成功，等待设备重启...

[4/6] 等待设备...
找到设备。

[5/6] 配置设备，发送数据中...
开始配置设备...
配置成功!

[6/6] 等待设备...
无线adb已打开,如果你想手动连接，请确保配置了adb环境，通过以下方式连接设备,设备重启失效:
   adb connect 192.168.0.1

正在尝试自动连接adb....
adb连接成功！，结果：connected to 192.168.0.1:5555
```

该工具明确提示：

```text
设备重启失效
```

说明它只是临时打开无线 ADB，不是永久保持。

重启后曾测试端口：

```powershell
Test-NetConnection 192.168.0.1 -Port 2333
Test-NetConnection 192.168.0.1 -Port 8080
Test-NetConnection 192.168.0.1 -Port 5555
```

状态判断：

```text
8080 通：F50 官方后台正常
2333 通：UFI-TOOLS WebService 已运行
5555 不通：无线 ADB 没有被拉起
```

结论：

```text
UFI-TOOLS 自身服务可以启动
但 ADB 自启功能在当前固件上没有稳定打开 5555
```

---

## 四、F50 识别 ESP32-S3 USB CDC

在 F50 ttyd root shell 中执行：

```sh
lsusb
```

输出中出现：

```text
Bus 002 Device 008: ID 303a:1001
```

`303a:1001` 为 Espressif USB Serial/JTAG 设备，说明 F50 已经识别到 ESP32-S3。

查看串口节点：

```sh
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

输出：

```text
crw------- 1 root root 166,   0 2026-07-18 13:16 /dev/ttyACM0
```

说明 ESP32-S3 在 F50 上枚举为：

```text
/dev/ttyACM0
```

---

## 五、ESP32-S3 固件问题与修复

ESP32-S3 最初启动失败，串口日志反复输出：

```text
E (80) spi_flash: Detected size(4096k) smaller than the size in the binary image header(8192k). Probe failed.

assert failed: do_core_init startup.c:328 (flash_ret == ESP_OK)
```

含义：

```text
实际 Flash：4MB
固件声明：8MB
结果：启动失败，循环重启
```

修正 PlatformIO 配置为 4MB Flash：

```ini
[env:esp32s3]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200
upload_speed = 921600

board_upload.flash_size = 4MB
board_build.flash_size = 4MB
board_build.partitions = default.csv

build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1
```

重新编译烧录：

```sh
pio run -t clean
pio run -t upload
```

---

## 六、F50 与 ESP32-S3 串口通信测试

F50 端初始化串口：

```sh
chmod 666 /dev/ttyACM0
stty -F /dev/ttyACM0 115200 raw -echo -icanon min 0 time 10
```

通信测试：

```sh
timeout 5 cat /dev/ttyACM0 &
sleep 1
printf "ping\n" > /dev/ttyACM0
sleep 5
```

输出：

```text
recv: ping
pong
```

结论：

```text
F50 ↔ ESP32-S3 USB CDC 串口双向通信成功
```

---

## 七、ESP32-S3 GPIO 控制方案

本次使用：

```text
GPIO48
```

原因：

```text
该 ESP32-S3 SuperMini 板载普通 LED / RGB 与 GPIO48 相关
测试时可以直接观察 GPIO48 状态
```

> 注意：正式产品中，如果 GPIO48 与板载 RGB 或启动状态有冲突，可以改用 GPIO4、GPIO5、GPIO6、GPIO7 等普通 IO。

ESP32-S3 控制代码：

```cpp
#include <Arduino.h>

static const int FAN_MOS_GPIO = 48;

// 大多数低边 N-MOS：高电平导通
static const bool FAN_ON_LEVEL = true;

static bool fanState = false;

void setFan(bool on) {
  fanState = on;

  digitalWrite(FAN_MOS_GPIO, on ? FAN_ON_LEVEL : !FAN_ON_LEVEL);

  Serial.print("OK FAN ");
  Serial.println(on ? "ON" : "OFF");
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  pinMode(FAN_MOS_GPIO, OUTPUT);

  // 上电默认关闭风扇
  setFan(false);

  Serial.println("READY F50 FAN CTRL GPIO48");
}

void loop() {
  if (Serial.available()) {
    String cmd = Serial.readStringUntil('\n');
    cmd.trim();
    cmd.toUpperCase();

    if (cmd == "PING") {
      Serial.println("OK PONG");
    } else if (cmd == "FAN_ON") {
      setFan(true);
    } else if (cmd == "FAN_OFF") {
      setFan(false);
    } else if (cmd == "STATUS") {
      Serial.print("OK STATUS FAN=");
      Serial.println(fanState ? "ON" : "OFF");
    } else {
      Serial.print("ERR UNKNOWN ");
      Serial.println(cmd);
    }
  }

  delay(10);
}
```

F50 手动控制测试：

```sh
printf "FAN_ON\n" > /dev/ttyACM0
printf "FAN_OFF\n" > /dev/ttyACM0
```

测试结果：

```text
FAN_ON / FAN_OFF 控制正常
GPIO48 状态正常
```

---

## 八、F50 温度节点查找

初次执行：

```sh
find /sys/class/thermal -type f -name temp -exec sh -c 'echo -n "$1 = "; cat "$1"' _ {} \;
```

没有输出。

原因：

```text
/sys/class/thermal/thermal_zone* 是符号链接
find 默认没有跟随符号链接
```

查看 thermal 目录：

```sh
ls -l /sys/class/thermal
```

发现存在大量 thermal zone：

```text
thermal_zone0
thermal_zone1
...
thermal_zone25
```

读取所有 thermal zone：

```sh
for z in /sys/class/thermal/thermal_zone*; do
  type=$(cat "$z/type" 2>/dev/null)
  temp=$(cat "$z/temp" 2>/dev/null)
  echo "$z | $type | $temp"
done
```

关键输出：

```text
/sys/class/thermal/thermal_zone4 | soc-thmzone | 57555
/sys/class/thermal/thermal_zone16 | gpu-thmzone | 57555
/sys/class/thermal/thermal_zone11 | apcpu0-thmzone | 57305
/sys/class/thermal/thermal_zone17 | apcpu1-thmzone | 57305
/sys/class/thermal/thermal_zone21 | lte-thmzone | 56006
/sys/class/thermal/thermal_zone22 | nr0-thmzone | 56006
```

最终选择温度节点：

```text
/sys/class/thermal/thermal_zone4/temp
```

对应：

```text
soc-thmzone
```

理由：

```text
soc-thmzone 代表 SoC 温度
比 chg/battery/board 更适合作为主动散热控制依据
```

温度单位：

```text
57555 = 57.555℃
```

---

## 九、F50 温控脚本

推荐阈值：

```text
≥ 65℃：开启风扇
≤ 55℃：关闭风扇
检测间隔：5 秒
```

采用双阈值回差控制，避免风扇频繁启停。

脚本路径：

```text
/data/local/tmp/f50_fan_control.sh
```

脚本内容：

```sh
#!/system/bin/sh

DEV="/dev/ttyACM0"
BAUD="115200"

# 温度阈值，单位：毫摄氏度
# 65000 = 65℃
# 55000 = 55℃
ON_TEMP=65000
OFF_TEMP=55000

INTERVAL=5

SOC_TEMP="/sys/class/thermal/thermal_zone4/temp"
STATE_FILE="/data/local/tmp/fan_state"
LOG_FILE="/data/local/tmp/f50_fan_control.log"

init_uart() {
  if [ ! -e "$DEV" ]; then
    echo "$(date '+%F %T') ERROR: $DEV not found" >> "$LOG_FILE"
    return 1
  fi

  chmod 666 "$DEV" 2>/dev/null
  stty -F "$DEV" "$BAUD" raw -echo -icanon min 0 time 10 2>/dev/null
  return 0
}

get_temp() {
  if [ -f "$SOC_TEMP" ]; then
    t=$(cat "$SOC_TEMP" 2>/dev/null)
    case "$t" in
      ''|*[!0-9]*) echo 0 ;;
      *) echo "$t" ;;
    esac
  else
    echo 0
  fi
}

send_cmd() {
  cmd="$1"

  init_uart || return 1

  printf "%s\n" "$cmd" > "$DEV"
  echo "$(date '+%F %T') send=$cmd" >> "$LOG_FILE"
  return 0
}

[ -f "$STATE_FILE" ] || echo "OFF" > "$STATE_FILE"

echo "$(date '+%F %T') fan control started ON=${ON_TEMP} OFF=${OFF_TEMP}" >> "$LOG_FILE"

while true; do
  temp=$(get_temp)
  state=$(cat "$STATE_FILE" 2>/dev/null)

  temp_c=$((temp / 1000))

  echo "$(date '+%F %T') temp=${temp_c}C raw=$temp state=$state" >> "$LOG_FILE"

  if [ "$state" = "OFF" ] && [ "$temp" -ge "$ON_TEMP" ]; then
    if send_cmd "FAN_ON"; then
      echo "ON" > "$STATE_FILE"
    fi
  elif [ "$state" = "ON" ] && [ "$temp" -le "$OFF_TEMP" ]; then
    if send_cmd "FAN_OFF"; then
      echo "OFF" > "$STATE_FILE"
    fi
  fi

  sleep "$INTERVAL"
done
```

赋予执行权限：

```sh
chmod +x /data/local/tmp/f50_fan_control.sh
```

手动启动：

```sh
/data/local/tmp/f50_fan_control.sh &
```

查看日志：

```sh
tail -f /data/local/tmp/f50_fan_control.log
```

日志示例：

```text
2026-07-18 14:28:04 fan control started ON=65000 OFF=55000
2026-07-18 14:28:04 temp=53C raw=53417 state=OFF
2026-07-18 14:28:09 temp=52C raw=52938 state=OFF
2026-07-18 14:28:14 temp=53C raw=53424 state=OFF
```

此时温度约 52～53℃，低于 65℃，风扇保持关闭。

---

## 十、低阈值测试自动触发

为了验证自动触发逻辑，可以临时改低阈值。

改成 55℃ 开 / 45℃ 关：

```sh
sed -i 's/ON_TEMP=65000/ON_TEMP=55000/' /data/local/tmp/f50_fan_control.sh
sed -i 's/OFF_TEMP=55000/OFF_TEMP=45000/' /data/local/tmp/f50_fan_control.sh
```

但当时温度只有 52℃，因此不会触发：

```text
fan control started ON=55000 OFF=45000
temp=52C raw=52695 state=OFF
```

这是正常的，因为：

```text
52695 < 55000
```

如需强制测试，可改成 50℃ 开 / 40℃ 关：

```sh
sed -i 's/ON_TEMP=55000/ON_TEMP=50000/' /data/local/tmp/f50_fan_control.sh
sed -i 's/OFF_TEMP=45000/OFF_TEMP=40000/' /data/local/tmp/f50_fan_control.sh
echo OFF > /data/local/tmp/fan_state
```

重新启动脚本后，当前 52℃ 会立即触发：

```text
send=FAN_ON
```

---

## 十一、开机自启方案

手动测试稳定后，可以写入 UFI-TOOLS 高级功能开机脚本。

路径：

```text
/sdcard/ufi_tools_boot.sh
```

内容：

```sh
#!/system/bin/sh

sleep 60

# 等待 ESP32-S3 USB CDC 串口出现
for i in $(seq 1 30); do
  if [ -e /dev/ttyACM0 ]; then
    break
  fi
  sleep 2
done

chmod 666 /dev/ttyACM0 2>/dev/null
stty -F /dev/ttyACM0 115200 raw -echo -icanon min 0 time 10 2>/dev/null

# 避免重复启动温控脚本
if ! ps -A | grep -q f50_fan_control.sh; then
  nohup /data/local/tmp/f50_fan_control.sh >/data/local/tmp/f50_fan_control.out 2>&1 &
fi
```

赋权：

```sh
chmod +x /sdcard/ufi_tools_boot.sh
```

重启后验证：

```sh
ps -A | grep f50_fan_control
tail -n 50 /data/local/tmp/f50_fan_control.log
```

---

## 十二、MOS 接线建议

使用低边 N 沟道 MOS 控制风扇。

```text
外部电源正极  ───── 风扇正极
风扇负极      ───── MOS D 极
MOS S 极       ───── GND
ESP32 GPIO48   ───── 100Ω ───── MOS G 极
MOS G 极       ───── 100kΩ ───── GND
ESP32 GND      ───── 外部电源 GND
```

关键点：

```text
ESP32 GND、风扇电源 GND、MOS S 极必须共地
GPIO 只控制 MOS，不直接带风扇
```

如果是电机风扇，建议风扇两端并联续流二极管：

```text
二极管阴极接风扇正极
二极管阳极接风扇负极 / MOS D 极
```

推荐 MOS：

```text
AO3400
IRLML2502
SI2302
```

---

## 十三、其他开发板选择

本项目不一定必须用 ESP32-S3。

| 开发板 | 是否适合 | 说明 |
|---|---|---|
| ESP32-S3 SuperMini | 适合 | 已实测可用，继续用最稳 |
| ESP32-C3 SuperMini | 推荐 | 便宜，原生 USB Serial/JTAG，控制风扇足够 |
| ESP32-C6 SuperMini | 可用 | 支持 USB Serial/JTAG，但功能过剩 |
| ESP32-H2 SuperMini | 不推荐 | 没 WiFi，对本项目优势不大 |
| 老 ESP32 / ESP8266 + CH340 | 不推荐 | F50 不一定有 CH340 驱动 |

优先选择能枚举为：

```text
ID 303a:1001
/dev/ttyACM0
```

的板子。

尽量避免依赖 CH340 / CP2102 的 USB 转串口板，因为 F50 内核不一定内置对应驱动。

---

## 十四、当前最终状态

截至本次记录，已完成：

```text
1. F50 可通过 ttyd root shell 操作
2. ESP32-S3 被 F50 识别为 /dev/ttyACM0
3. F50 ↔ ESP32-S3 USB CDC 串口通信成功
4. ESP32-S3 可接收 FAN_ON / FAN_OFF
5. GPIO48 控制测试正常
6. F50 已找到 soc-thmzone 温度节点
7. F50 温控脚本已创建并运行
8. 日志正常记录温度和状态
```

当前推荐正式参数：

```text
温度节点：/sys/class/thermal/thermal_zone4/temp
节点名称：soc-thmzone
开启风扇：65℃
关闭风扇：55℃
检测间隔：5 秒
串口设备：/dev/ttyACM0
控制命令：FAN_ON / FAN_OFF
```

---

## 十五、后续优化方向

### 1. 增加异常保护

可增加：

```text
/dev/ttyACM0 消失时重试
ESP32 无响应时重新初始化串口
连续高温时强制重复发送 FAN_ON
日志大小限制
```

### 2. 增加 ESP32 状态反馈

F50 可定时发送：

```text
STATUS
```

ESP32 返回：

```text
OK STATUS FAN=ON
```

用于确认风扇状态。

### 3. 增加日志轮转

避免 `/data/local/tmp/f50_fan_control.log` 长期增长过大。

### 4. 改成二进制或 JSON 协议

后期可以扩展为：

```json
{"cmd":"fan","value":1}
{"cmd":"status"}
```

ESP32 返回：

```json
{"ok":true,"fan":1}
```

### 5. 改用更合适 GPIO

正式接 MOS 时，如果 GPIO48 与板载灯或 RGB 逻辑冲突，可改用：

```text
GPIO4
GPIO5
GPIO6
GPIO7
```

---

## 十六、常用命令汇总

查看 ESP32 串口：

```sh
lsusb
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

初始化串口：

```sh
chmod 666 /dev/ttyACM0
stty -F /dev/ttyACM0 115200 raw -echo -icanon min 0 time 10
```

测试 ESP32 通信：

```sh
timeout 5 cat /dev/ttyACM0 &
sleep 1
printf "PING\n" > /dev/ttyACM0
sleep 3
```

手动开关风扇：

```sh
printf "FAN_ON\n" > /dev/ttyACM0
printf "FAN_OFF\n" > /dev/ttyACM0
```

查看温度：

```sh
cat /sys/class/thermal/thermal_zone4/temp
```

查看所有温度节点：

```sh
for z in /sys/class/thermal/thermal_zone*; do
  type=$(cat "$z/type" 2>/dev/null)
  temp=$(cat "$z/temp" 2>/dev/null)
  echo "$z | $type | $temp"
done
```

启动温控脚本：

```sh
/data/local/tmp/f50_fan_control.sh &
```

查看温控日志：

```sh
tail -f /data/local/tmp/f50_fan_control.log
```

重置风扇状态：

```sh
echo OFF > /data/local/tmp/fan_state
```

---

## 十七、结论

本方案已经证明可行：

```text
中兴 F50 可以通过 USB CDC 串口控制 ESP32-S3
ESP32-S3 可以通过 GPIO 控制 MOS 管
F50 可以读取自身 SoC 温度
F50 可以根据温度阈值自动发送 FAN_ON / FAN_OFF
```

最终实现路径：

```text
F50 温度节点
/sys/class/thermal/thermal_zone4/temp
↓
F50 Shell 温控脚本
/data/local/tmp/f50_fan_control.sh
↓
USB CDC 串口
/dev/ttyACM0
↓
ESP32-S3
GPIO48
↓
MOS 管
↓
散热风扇
```
