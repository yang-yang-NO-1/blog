# 树莓派系统镜像备份实录：压缩、恢复、校验与USB硬盘安全卸载

最近把树莓派上的系统、服务、自启动和项目环境都调好了，就想着赶紧做一份完整镜像备份。  
这样以后不管是 SD 卡损坏、系统崩掉，还是想快速迁移到新卡，都能直接恢复，不用再从头折腾一遍。

这篇就把整套流程完整记下来，内容包括：

- 修改代理到 `192.168.0.195`
- 下载 `PiShrink`
- 给脚本加执行权限
- 把系统盘备份到 USB 机械硬盘
- 压缩和裁剪镜像
- 生成 SHA256 校验文件
- 恢复镜像到新 SD 卡
- 安全卸载 USB 机械硬盘

整篇偏实战记录，按步骤做就可以。

---

## 当前环境

先看一下当前磁盘情况：

```bash
lsblk
```

我这里对应关系是这样的：

```text
系统盘：/dev/mmcblk0
机械硬盘分区：/dev/sda1
机械硬盘挂载点：/media/pi/机械硬盘
```

也就是说，这次备份的思路非常简单：

- 从 `/dev/mmcblk0` 读取整张系统卡
- 把镜像文件保存到 `/media/pi/机械硬盘`

---

## 一、先把代理改到 192.168.0.195

如果树莓派访问 GitHub 需要走局域网代理，可以先改一下代理地址。

### 临时设置当前终端代理

```bash
export http_proxy="http://192.168.0.195:7890"
export https_proxy="http://192.168.0.195:7890"
export HTTP_PROXY="http://192.168.0.195:7890"
export HTTPS_PROXY="http://192.168.0.195:7890"
export all_proxy="socks5://192.168.0.195:7890"
export ALL_PROXY="socks5://192.168.0.195:7890"
```

检查是否生效：

```bash
env | grep -i proxy
```

### 如果想长期生效，写进 `~/.bashrc`

```bash
cat >> ~/.bashrc <<'EOF'
export http_proxy="http://192.168.0.195:7890"
export https_proxy="http://192.168.0.195:7890"
export HTTP_PROXY="http://192.168.0.195:7890"
export HTTPS_PROXY="http://192.168.0.195:7890"
export all_proxy="socks5://192.168.0.195:7890"
export ALL_PROXY="socks5://192.168.0.195:7890"
EOF

source ~/.bashrc
```

### 如果原来用的是 192.168.0.196，直接替换

```bash
sed -i 's/192\.168\.0\.196/192.168.0.195/g' ~/.bashrc
source ~/.bashrc
```

### 不想用了就取消代理

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY all_proxy ALL_PROXY
```

---

## 二、下载 PiShrink

`PiShrink` 很适合做树莓派镜像瘦身。  
它会把镜像里没用到的空间裁掉，然后还能顺手再压缩一遍，最后得到一个更适合长期保存的 `.img.gz` 文件。

先进入临时目录：

```bash
cd /tmp
```

下载脚本：

```bash
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
```

如果不想走代理，也可以这样：

```bash
wget --no-proxy https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
```

下载完成后，加执行权限：

```bash
chmod +x /tmp/pishrink.sh
```

或者当前目录下执行：

```bash
chmod +x pishrink.sh
```

可以顺手确认一下文件是否真的在：

```bash
ls -lh /tmp/pishrink.sh
```

---

## 三、备份前先看看机械硬盘空间够不够

先检查挂载的 USB 机械硬盘：

```bash
df -h "/media/pi/机械硬盘"
```

我这边系统盘是 `14.8G`，所以机械硬盘最好至少留出 `15G` 以上可用空间。  
因为在压缩前，`dd` 生成的是完整整卡镜像，体积会比较大。

---

## 四、备份前顺手清理一下系统垃圾

不是必须，但建议做。  
这样后面压缩出来的镜像往往会更小一些。

```bash
sudo apt clean
sudo journalctl --vacuum-time=3d
rm -rf ~/.cache/*
```

如果你系统里还有一些临时文件、旧日志、编译缓存，也可以提前手动删掉。

---

## 五、开始做整卡镜像备份

核心命令就是这一条：

```bash
sudo dd if=/dev/mmcblk0 of="/media/pi/机械硬盘/raspi-control-panel-final.img" bs=4M status=progress conv=fsync
```

这几个参数的意思：

```text
if=/dev/mmcblk0   输入源，也就是当前树莓派系统盘
of=...img         输出镜像文件到机械硬盘
bs=4M             每次读写 4MB，效率更高
status=progress   显示实时进度
conv=fsync        确保数据真正写入磁盘
```

跑完以后，机械硬盘里会生成一个完整镜像：

```text
/media/pi/机械硬盘/raspi-control-panel-final.img
```

这个文件通常会接近原始 SD 卡容量，所以看到它很大是正常的。

---

## 六、用 PiShrink 裁剪并压缩镜像

整卡镜像虽然完整，但太占空间。  
这时候就轮到 `PiShrink` 上场了。

执行：

```bash
sudo /tmp/pishrink.sh -z "/media/pi/机械硬盘/raspi-control-panel-final.img"
```

这里的 `-z` 参数表示：

```text
裁剪完成后自动再压缩成 .gz
```

最终一般会得到：

```text
/media/pi/机械硬盘/raspi-control-panel-final.img.gz
```

这个文件才是最适合长期保存的版本。

---

## 七、再补一个 SHA256 校验文件

镜像这种东西，最怕以后拷来拷去出了损坏但自己不知道。  
所以我建议顺手再生成一个校验文件。

```bash
sha256sum "/media/pi/机械硬盘/raspi-control-panel-final.img.gz" > "/media/pi/机械硬盘/raspi-control-panel-final.img.gz.sha256"
```

以后要验证镜像完整性，就执行：

```bash
cd "/media/pi/机械硬盘"
sha256sum -c raspi-control-panel-final.img.gz.sha256
```

如果输出：

```text
raspi-control-panel-final.img.gz: OK
```

那就说明文件没坏。

---

## 八、最后建议保留哪些文件

备份完成后，我建议至少保留这两个：

```text
/media/pi/机械硬盘/raspi-control-panel-final.img.gz
/media/pi/机械硬盘/raspi-control-panel-final.img.gz.sha256
```

一个是镜像本体，一个是校验文件。  
这样以后恢复前先校验一下，心里会踏实很多。

---

## 九、整套备份流程命令放一起

为了以后自己查方便，也把完整命令放一遍：

```bash
export http_proxy="http://192.168.0.195:7890"
export https_proxy="http://192.168.0.195:7890"
export HTTP_PROXY="http://192.168.0.195:7890"
export HTTPS_PROXY="http://192.168.0.195:7890"
export all_proxy="socks5://192.168.0.195:7890"
export ALL_PROXY="socks5://192.168.0.195:7890"

cd /tmp
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
chmod +x /tmp/pishrink.sh

df -h "/media/pi/机械硬盘"

sudo apt clean
sudo journalctl --vacuum-time=3d
rm -rf ~/.cache/*

sudo dd if=/dev/mmcblk0 of="/media/pi/机械硬盘/raspi-control-panel-final.img" bs=4M status=progress conv=fsync

sudo /tmp/pishrink.sh -z "/media/pi/机械硬盘/raspi-control-panel-final.img"

sha256sum "/media/pi/机械硬盘/raspi-control-panel-final.img.gz" > "/media/pi/机械硬盘/raspi-control-panel-final.img.gz.sha256"
```

---

## 十、以后怎么恢复镜像

恢复前需要准备：

- 一张新的 SD 卡
- 镜像文件 `raspi-control-panel-final.img.gz`
- 校验文件 `raspi-control-panel-final.img.gz.sha256`

建议目标卡容量不要小于原系统卡容量，稳妥一点就上 `16GB` 或更大。

### 1）先校验镜像完整性

```bash
cd "/media/pi/机械硬盘"
sha256sum -c raspi-control-panel-final.img.gz.sha256
```

如果输出 `OK`，说明镜像正常。

### 2）解压镜像

```bash
gunzip -k raspi-control-panel-final.img.gz
```

这里的 `-k` 表示保留原来的 `.gz` 压缩文件。

解压后会得到：

```text
raspi-control-panel-final.img
```

### 3）插入目标 SD 卡，确认设备名

```bash
lsblk
```

这一步一定要仔细看。  
假设目标 SD 卡是：

```text
/dev/sda
```

千万别把目标写成系统盘或者机械硬盘。

### 4）开始写入镜像

```bash
sudo dd if=raspi-control-panel-final.img of=/dev/sda bs=4M status=progress conv=fsync
sync
```

写完以后，就可以把卡插回树莓派启动了。

---

## 十一、恢复后是否还要手动 resize

通常不需要。  
`PiShrink` 处理过的镜像，一般第一次启动时就会自动扩展根分区。

启动后可以先检查一下：

```bash
df -h
```

如果发现根分区 `/` 没有自动扩展，再手动执行：

```bash
sudo raspi-config --expand-rootfs
sudo reboot
```

---

## 十二、恢复后检查服务是不是正常

树莓派启动后，我一般会先检查服务状态：

```bash
systemctl status raspi-control-panel.service --no-pager
```

如果正常，应该能看到：

```text
active (running)
```

也可以继续看进程：

```bash
ps -ef | grep raspi-control-panel | grep -v grep
```

如果你想更严谨一点，也可以比较程序文件和配置文件：

```bash
cmp ~/raspi-control-panel/raspi-control-panel /usr/local/bin/raspi-control-panel && echo "program same"
cmp ~/raspi-control-panel/config/panel.conf /etc/raspi-control-panel.conf && echo "config same"
```

---

## 十三、如果恢复后服务没起来

那就直接手动重启：

```bash
sudo systemctl restart raspi-control-panel.service
```

然后再看状态：

```bash
systemctl status raspi-control-panel.service --no-pager
```

想进一步查问题，就看日志：

```bash
journalctl -u raspi-control-panel.service -n 100 --no-pager
```

---

## 十四、Windows 下也能恢复

如果不想在 Linux 下操作，Windows 也可以恢复镜像。

常见工具有：

- Raspberry Pi Imager
- balenaEtcher
- Win32 Disk Imager

大概流程就是：

```text
1. 先把 .img.gz 解压成 .img
2. 插入新的 SD 卡
3. 在烧录工具里选择 .img 文件
4. 写入 SD 卡
5. 写入完成后插回树莓派
```

---

## 十五、备份完成后，别忘了安全卸载 USB 机械硬盘

镜像文件通常很大，所以拔硬盘之前一定别偷懒，最好走一遍安全卸载流程。

假设挂载点是：

```text
/media/pi/机械硬盘
```

设备分区是：

```text
/dev/sda1
```

### 1）先写入缓存

```bash
sync
```

### 2）如果当前就在硬盘目录下，先退出来

```bash
cd ~
```

### 3）卸载硬盘

```bash
sudo umount "/media/pi/机械硬盘"
```

或者：

```bash
sudo umount /dev/sda1
```

### 4）确认是否已经卸载成功

```bash
lsblk
```

如果 `sda1` 后面已经没有挂载路径，就说明可以安全拔掉了。

成功示例大概像这样：

```text
sda           8:0    0 465.8G  0 disk
└─sda1        8:1    0 465.8G  0 part
```

### 5）如果提示 `target is busy`

那说明还有进程占着这个目录。

可以先试一遍：

```bash
cd ~
sync
sudo umount "/media/pi/机械硬盘"
```

如果还不行，再查占用：

```bash
sudo lsof +f -- "/media/pi/机械硬盘"
```

或者：

```bash
sudo fuser -vm "/media/pi/机械硬盘"
```

确认不是备份、压缩或校验过程还没结束，再把相关进程处理掉。

---

## 十六、常用命令总汇总

### 设置代理

```bash
export http_proxy="http://192.168.0.195:7890"
export https_proxy="http://192.168.0.195:7890"
export HTTP_PROXY="http://192.168.0.195:7890"
export HTTPS_PROXY="http://192.168.0.195:7890"
export all_proxy="socks5://192.168.0.195:7890"
export ALL_PROXY="socks5://192.168.0.195:7890"
```

### 下载 PiShrink 并授权

```bash
cd /tmp
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
chmod +x /tmp/pishrink.sh
```

### 校验镜像

```bash
cd "/media/pi/机械硬盘"
sha256sum -c raspi-control-panel-final.img.gz.sha256
```

### 解压镜像

```bash
gunzip -k raspi-control-panel-final.img.gz
```

### 查看设备

```bash
lsblk
```

### 写入镜像

```bash
sudo dd if=raspi-control-panel-final.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

其中 `/dev/sdX` 要替换成真实目标设备。

### 查看服务状态

```bash
systemctl status raspi-control-panel.service --no-pager
```

---

## 十七、最后提醒

`dd` 命令非常强，但也非常“直”。  
你一旦把 `of=` 写错，它不会提醒你“你是不是选错盘了”，它会直接开写。

所以每次恢复前，我都建议先再看一眼：

```bash
lsblk
```

确认目标设备到底是谁，再执行写入命令。

---

## 结语

这套流程总结下来，其实就四步：

1. `dd` 做整卡备份  
2. `PiShrink` 裁剪压缩  
3. `sha256sum` 做校验  
4. 需要时再写回 SD 卡恢复  

优点很明显：

- 备份完整
- 恢复直接
- 镜像体积更小
- 适合长期保存

如果你的树莓派系统已经调试到一个稳定状态，真的很建议尽早做一份。  
等系统出问题之后再想起来备份，通常就已经晚了。
