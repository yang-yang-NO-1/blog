# Raspberry Pi 镜像恢复说明

本文档用于说明如何将 `raspi-control-panel-final.img.gz` 恢复到新的 SD 卡中。

## 1. 当前镜像文件

镜像文件：

```text
raspi-control-panel-final.img.gz
```

校验文件：

```text
raspi-control-panel-final.img.gz.sha256
```

示例保存位置：

```text
/media/pi/机械硬盘/
```

该镜像是树莓派系统盘 `/dev/mmcblk0` 的完整备份，已经通过 PiShrink 裁剪并压缩。

镜像中包含：

```text
树莓派系统
启动分区
root 文件系统
raspi-control-panel 程序
/etc/raspi-control-panel.conf 配置文件
systemd 自启动服务
OLED 显示相关配置
已安装依赖和运行环境
```

恢复后，树莓派应可以直接启动并运行 OLED 控制面板程序。

---

## 2. 恢复前准备

需要准备：

```text
1. 一张新的 SD 卡
2. 一台 Linux 电脑，或树莓派本机
3. 备份镜像文件 raspi-control-panel-final.img.gz
4. 校验文件 raspi-control-panel-final.img.gz.sha256
```

注意：

```text
目标 SD 卡容量需要大于或等于镜像实际展开后的系统容量。
建议使用 16GB 或更大的 SD 卡。
```

---

## 3. 校验镜像完整性

进入镜像所在目录：

```bash
cd "/media/pi/机械硬盘"
```

执行校验：

```bash
sha256sum -c raspi-control-panel-final.img.gz.sha256
```

如果显示：

```text
raspi-control-panel-final.img.gz: OK
```

说明镜像文件没有损坏。

---

## 4. 解压镜像

执行：

```bash
gunzip -k raspi-control-panel-final.img.gz
```

说明：

```text
-k 表示保留原始 .gz 压缩包
```

解压后会得到：

```text
raspi-control-panel-final.img
```

---

## 5. 插入目标 SD 卡并确认设备名

插入新的 SD 卡后，执行：

```bash
lsblk
```

你会看到类似：

```text
sda      14.8G
├─sda1
└─sda2
```

或者：

```text
sdb      14.8G
├─sdb1
└─sdb2
```

假设目标 SD 卡是：

```text
/dev/sda
```

重要提醒：

```text
一定要确认目标设备名。
不要写成系统盘。
不要写成机械硬盘。
不要写成 /dev/mmcblk0，除非你明确知道它就是目标卡。
```

---

## 6. 写入镜像到 SD 卡

假设目标 SD 卡是 `/dev/sda`，执行：

```bash
sudo dd if=raspi-control-panel-final.img of=/dev/sda bs=4M status=progress conv=fsync
```

写入完成后执行：

```bash
sync
```

---

## 7. 安全拔出 SD 卡

写入完成后，建议先卸载分区：

```bash
sudo umount /dev/sda1 2>/dev/null
sudo umount /dev/sda2 2>/dev/null
```

然后拔出 SD 卡，插入树莓派启动。

---

## 8. 恢复后检查服务状态

树莓派启动后，执行：

```bash
systemctl status raspi-control-panel.service --no-pager
```

如果服务正常，应看到：

```text
active (running)
```

也可以查看进程：

```bash
ps -ef | grep raspi-control-panel | grep -v grep
```

检查程序文件是否一致：

```bash
cmp ~/raspi-control-panel/raspi-control-panel /usr/local/bin/raspi-control-panel && echo "program same"
```

检查配置文件是否一致：

```bash
cmp ~/raspi-control-panel/config/panel.conf /etc/raspi-control-panel.conf && echo "config same"
```

---

## 9. 如果恢复后服务没有启动

手动重启服务：

```bash
sudo systemctl restart raspi-control-panel.service
```

再次查看状态：

```bash
systemctl status raspi-control-panel.service --no-pager
```

查看日志：

```bash
journalctl -u raspi-control-panel.service -n 100 --no-pager
```

---

## 10. Windows 恢复方法

Windows 上可以使用以下工具：

```text
Raspberry Pi Imager
balenaEtcher
Win32 Disk Imager
```

步骤：

```text
1. 将 raspi-control-panel-final.img.gz 解压成 raspi-control-panel-final.img
2. 插入新的 SD 卡
3. 使用 Raspberry Pi Imager / balenaEtcher 选择该 img 文件
4. 写入 SD 卡
5. 写入完成后插回树莓派启动
```

---

## 11. 常用命令汇总

### 校验镜像

```bash
cd "/media/pi/机械硬盘"
sha256sum -c raspi-control-panel-final.img.gz.sha256
```

### 解压镜像

```bash
gunzip -k raspi-control-panel-final.img.gz
```

### 查看 SD 卡设备

```bash
lsblk
```

### 写入镜像

```bash
sudo dd if=raspi-control-panel-final.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

注意：

```text
/dev/sdX 需要替换成真实目标 SD 卡设备，例如 /dev/sda 或 /dev/sdb。
```

### 查看服务状态

```bash
systemctl status raspi-control-panel.service --no-pager
```

---

## 12. 重要提醒

```text
dd 命令会覆盖目标磁盘全部数据。
执行前必须确认 of= 后面的设备名是目标 SD 卡。
如果写错设备，可能会清空系统盘或机械硬盘。
```

推荐恢复前再次确认：

```bash
lsblk
```
---

## 13. USB 机械硬盘安全卸载流程

如果镜像文件保存在外接 USB 机械硬盘中，例如挂载点为：

```text
/media/pi/机械硬盘
```

设备分区为：

```text
/dev/sda1
```

在拔出硬盘前，必须先安全卸载。

### 13.1 写入缓存到硬盘

先执行：

```bash
sync
```

该命令会把系统缓存中的数据真正写入硬盘，避免镜像文件或校验文件损坏。

### 13.2 退出硬盘目录

如果当前终端正在 `/media/pi/机械硬盘` 目录下，需要先退出：

```bash
cd ~
```

### 13.3 卸载 USB 硬盘

推荐使用挂载路径卸载：

```bash
sudo umount "/media/pi/机械硬盘"
```

也可以使用设备名卸载：

```bash
sudo umount /dev/sda1
```

### 13.4 确认卸载成功

执行：

```bash
lsblk
```

如果看到 `/dev/sda1` 后面已经没有挂载路径，说明卸载成功。

卸载成功示例：

```text
sda           8:0    0 465.8G  0 disk
└─sda1        8:1    0 465.8G  0 part
```

此时可以安全拔出 USB 机械硬盘。

### 13.5 如果提示 target is busy

如果卸载时报错：

```text
target is busy
```

说明有程序正在占用硬盘目录。

先执行：

```bash
cd ~
sync
sudo umount "/media/pi/机械硬盘"
```

如果仍然失败，查看占用进程：

```bash
sudo lsof +f -- "/media/pi/机械硬盘"
```

或者：

```bash
sudo fuser -vm "/media/pi/机械硬盘"
```

确认不是正在备份、压缩或校验镜像后，再结束占用进程，然后重新卸载。

### 13.6 常用卸载命令汇总

```bash
cd ~
sync
sudo umount "/media/pi/机械硬盘"
lsblk
```

看到挂载点消失后，再拔出 USB 硬盘。
