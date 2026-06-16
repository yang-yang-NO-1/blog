<html>
<body>
<!--StartFragment--><html><head></head><body><h1>树莓派 4B 通过 USB 随身 WiFi 上网，并用板载 WiFi 共享热点：从零部署到固定 IP</h1><h2>前言</h2><p>这篇文章记录一次树莓派 4B 网络共享的完整部署过程。</p><p>最终目标是：</p><ul><li><p>树莓派通过 <strong>USB 随身 WiFi / USB 网卡</strong> 上网；</p></li><li><p>使用树莓派 4B 的 <strong>板载 WiFi</strong> 开启热点；</p></li><li><p>手机、电脑等设备连接树莓派热点后可以正常上网；</p></li><li><p>热点网段固定为 <code inline="">192.168.0.x</code>；</p></li><li><p>NAT、转发规则和热点服务均支持 <strong>开机自启</strong>；</p></li><li><p>支持给连接热点的设备按 MAC 地址分配固定 IP。</p></li></ul><p>最终网络拓扑如下：</p><pre><code class="language-text">USB 随身 WiFi / 4G 网卡
        ↓
usb0：树莓派外网入口
        ↓
树莓派 4B
        ↓
wlan0：板载 WiFi 热点
        ↓
手机 / 电脑 / 摄像头等设备
</code></pre><p>本文以如下配置为例：</p><pre><code class="language-text">外网入口接口：usb0
热点接口：wlan0
热点名称：HP-001GR
热点网段：192.168.0.0/24
热点网关：192.168.0.1
热点连接名：PiHotspot
有线管理口：eth0
</code></pre><blockquote><p>注意：本文中所有密码都请替换成你自己的密码，不建议把真实热点密码直接写进公开博客。</p></blockquote><hr><h2>一、适用环境</h2><p>本文适用于：</p><ul><li><p>Raspberry Pi 4B；</p></li><li><p>Raspberry Pi OS Bookworm 或使用 NetworkManager 管理网络的系统；</p></li><li><p>USB 随身 WiFi / 4G 模块在系统中识别为 <code inline="">usb0</code>、<code inline="">eth1</code> 或 <code inline="">enx...</code> 这类以太网接口；</p></li><li><p>使用树莓派板载 <code inline="">wlan0</code> 开热点。</p></li></ul><p>如果你的接口名称不同，请根据 <code inline="">nmcli dev status</code> 的输出替换命令中的接口名。</p><hr><h2>二、先确认网络接口</h2><p>执行：</p><pre><code class="language-bash">nmcli dev status
ip link
ip route
</code></pre><p>正常情况下可以看到类似：</p><pre><code class="language-text">DEVICE  TYPE      STATE      CONNECTION
usb0    ethernet  connected  usb-wifi
eth0    ethernet  connected  netplan-eth0
wlan0   wifi      disconnected  --
lo      loopback  connected  lo
</code></pre><p>这里的含义是：</p>
接口 | 作用
-- | --
usb0 | USB 随身 WiFi / 4G 网卡，上网入口
wlan0 | 树莓派板载 WiFi，用来开热点
eth0 | 有线网口，可作为 SSH 管理口
lo | 本地回环接口，不用管

<p>如果 <code inline="">wlan0</code> 显示为 <code inline="">unavailable</code>，先执行：</p><pre><code class="language-bash">sudo nmcli radio wifi on
sudo rfkill unblock wifi
nmcli dev status
</code></pre><p>如果还是不可用，设置 WLAN Country：</p><pre><code class="language-bash">sudo raspi-config
</code></pre><p>进入：</p><pre><code class="language-text">Localisation Options → WLAN Country
</code></pre><p>选择你的实际国家或地区，然后重启：</p><pre><code class="language-bash">sudo reboot
</code></pre><hr><h2>三、安装必要组件</h2><p>NetworkManager 的热点共享模式需要 <code inline="">dnsmasq</code> 组件来给热点客户端分配 DHCP 地址。</p><p>执行：</p><pre><code class="language-bash">sudo apt update
sudo apt install -y dnsmasq-base iptables
</code></pre><p>关闭独立 <code inline="">dnsmasq</code> 服务，避免和 NetworkManager 内部启动的 dnsmasq 冲突：</p><pre><code class="language-bash">sudo systemctl disable --now dnsmasq 2&gt;/dev/null || true
</code></pre><p>确认 dnsmasq 存在：</p><pre><code class="language-bash">command -v dnsmasq
</code></pre><p>正常应输出：</p><pre><code class="language-text">/usr/sbin/dnsmasq
</code></pre><hr><h2>四、配置 USB 随身 WiFi 为默认外网入口</h2><p>如果你的 USB 上网接口已经有 NetworkManager 连接，例如叫 <code inline="">usb-wifi</code>，直接修改即可：</p><pre><code class="language-bash">sudo nmcli connection modify usb-wifi \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  ipv4.method auto \
  ipv4.route-metric 50
</code></pre><p>如果没有这个连接，可以新建：</p><pre><code class="language-bash">sudo nmcli connection add type ethernet ifname usb0 con-name usb-wifi ipv4.method auto

sudo nmcli connection modify usb-wifi \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  ipv4.route-metric 50
</code></pre><p>启动 USB 上网连接：</p><pre><code class="language-bash">sudo nmcli connection up usb-wifi
</code></pre><p>检查路由：</p><pre><code class="language-bash">ip route
</code></pre><p>理想情况下应该看到类似：</p><pre><code class="language-text">default via 192.168.99.1 dev usb0 metric 50
</code></pre><p>这表示树莓派默认上网出口已经优先走 <code inline="">usb0</code>。</p><hr><h2>五、降低 eth0 有线网口优先级</h2><p>如果你还插着网线，<code inline="">eth0</code> 可能会抢默认路由。建议保留 <code inline="">eth0</code> 用于 SSH 管理，但降低它的优先级。</p><p>先看连接名：</p><pre><code class="language-bash">nmcli connection show
</code></pre><p>如果有线连接名是 <code inline="">netplan-eth0</code>，执行：</p><pre><code class="language-bash">sudo nmcli connection modify netplan-eth0 \
  connection.autoconnect yes \
  connection.autoconnect-priority 50 \
  ipv4.route-metric 300
</code></pre><p>这样最终路由优先级为：</p><pre><code class="language-text">usb0：metric 50，优先作为上网出口
eth0：metric 300，只作为备用或管理口
</code></pre><p>再次查看：</p><pre><code class="language-bash">ip route
</code></pre><p>理想结果类似：</p><pre><code class="language-text">default via 192.168.99.1 dev usb0 metric 50
default via 192.168.137.1 dev eth0 metric 300
</code></pre><hr><h2>六、创建板载 WiFi 热点</h2><p>先清理旧热点配置：</p><pre><code class="language-bash">sudo nmcli connection delete PiHotspot 2&gt;/dev/null || true
sudo nmcli device disconnect wlan0 2&gt;/dev/null || true
sudo ip addr flush dev wlan0 2&gt;/dev/null || true
</code></pre><p>设置热点名称和密码变量：</p><pre><code class="language-bash">HOTSPOT_SSID="HP-001GR"
HOTSPOT_PASS="请改成你自己的热点密码"
</code></pre><p>创建热点：</p><pre><code class="language-bash">sudo nmcli dev wifi hotspot ifname wlan0 con-name PiHotspot ssid "$HOTSPOT_SSID" password "$HOTSPOT_PASS"
</code></pre><p>然后固定热点参数：</p><pre><code class="language-bash">sudo nmcli connection modify PiHotspot \
  connection.autoconnect yes \
  connection.autoconnect-priority 90 \
  802-11-wireless.mode ap \
  802-11-wireless.band bg \
  802-11-wireless.channel 6 \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "$HOTSPOT_PASS" \
  ipv4.method shared \
  ipv4.addresses 192.168.0.1/24 \
  ipv4.never-default yes \
  ipv6.method ignore
</code></pre><p>说明：</p><pre><code class="language-text">802-11-wireless.band bg     表示 2.4G 热点
802-11-wireless.channel 6   表示使用 2.4G 的 6 信道
ipv4.method shared          表示开启共享网络、DHCP 和 NAT 基础能力
ipv4.addresses 192.168.0.1/24  表示热点网关固定为 192.168.0.1
ipv4.never-default yes      表示热点接口不作为默认上网出口
</code></pre><p>重启热点：</p><pre><code class="language-bash">sudo nmcli connection down PiHotspot 2&gt;/dev/null || true
sudo ip addr flush dev wlan0 2&gt;/dev/null || true
sudo nmcli connection up PiHotspot
</code></pre><p>检查：</p><pre><code class="language-bash">nmcli dev status
ip addr show wlan0
</code></pre><p>正常应该看到：</p><pre><code class="language-text">wlan0  wifi  connected  PiHotspot
</code></pre><p>并且 <code inline="">wlan0</code> 有：</p><pre><code class="language-text">inet 192.168.0.1/24
</code></pre><hr><h2>七、永久开启 IPv4 转发</h2><p>Linux 默认不一定允许不同网卡之间转发流量。热点共享必须开启 IPv4 forwarding。</p><p>执行：</p><pre><code class="language-bash">echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-hotspot-forward.conf
sudo sysctl --system
</code></pre><p>确认：</p><pre><code class="language-bash">cat /proc/sys/net/ipv4/ip_forward
</code></pre><p>正常输出：</p><pre><code class="language-text">1
</code></pre><hr><h2>八、创建 NAT / FORWARD 规则脚本</h2><p>虽然 NetworkManager 的 <code inline="">shared</code> 模式会处理一部分共享逻辑，但如果系统里有 Docker，Docker 可能会影响 <code inline="">FORWARD</code> 链规则。为了稳定，建议手动建立一套热点转发脚本，并通过 systemd 开机自动执行。</p><p>创建脚本：</p><pre><code class="language-bash">sudo tee /usr/local/sbin/hotspot-forward.sh &gt;/dev/null &lt;&lt;'EOF'
#!/usr/bin/env bash
set -u

IPT="/usr/sbin/iptables"
SYSCTL="/usr/sbin/sysctl"

# 开启 IPv4 转发
$SYSCTL -w net.ipv4.ip_forward=1 &gt;/dev/null

# 删除旧热点网段 NAT
while $IPT -w -t nat -D POSTROUTING -s 10.42.0.0/24 -o usb0 -j MASQUERADE 2&gt;/dev/null; do :; done

# 删除可能重复的新 NAT
while $IPT -w -t nat -D POSTROUTING -s 192.168.0.0/24 -o usb0 -j MASQUERADE 2&gt;/dev/null; do :; done

# 删除可能重复的 FORWARD 规则
while $IPT -w -D FORWARD -i wlan0 -o usb0 -j ACCEPT 2&gt;/dev/null; do :; done
while $IPT -w -D FORWARD -i usb0 -o wlan0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 2&gt;/dev/null; do :; done
while $IPT -w -D FORWARD -i usb0 -o wlan0 -j ACCEPT 2&gt;/dev/null; do :; done

# 允许热点客户端通过 usb0 出网
$IPT -w -I FORWARD 1 -i wlan0 -o usb0 -j ACCEPT

# 允许 usb0 回包到热点客户端
$IPT -w -I FORWARD 1 -i usb0 -o wlan0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT || \
$IPT -w -I FORWARD 1 -i usb0 -o wlan0 -j ACCEPT

# NAT：192.168.0.x 通过 usb0 出网
$IPT -w -t nat -A POSTROUTING -s 192.168.0.0/24 -o usb0 -j MASQUERADE

exit 0
EOF
</code></pre><p>赋予执行权限：</p><pre><code class="language-bash">sudo chmod +x /usr/local/sbin/hotspot-forward.sh
</code></pre><p>手动测试脚本：</p><pre><code class="language-bash">sudo /usr/local/sbin/hotspot-forward.sh
echo $?
</code></pre><p>正常输出：</p><pre><code class="language-text">0
</code></pre><hr><h2>九、创建 systemd 开机自启服务</h2><p>创建服务文件：</p><pre><code class="language-bash">sudo tee /etc/systemd/system/hotspot-forward.service &gt;/dev/null &lt;&lt;'EOF'
[Unit]
Description=Hotspot NAT and Forwarding Rules
After=NetworkManager.service docker.service
Wants=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/hotspot-forward.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
</code></pre><p>启用并立即启动：</p><pre><code class="language-bash">sudo systemctl daemon-reload
sudo systemctl enable hotspot-forward.service
sudo systemctl restart hotspot-forward.service
</code></pre><p>检查服务状态：</p><pre><code class="language-bash">systemctl is-active hotspot-forward.service
systemctl is-enabled hotspot-forward.service
</code></pre><p>正常应该输出：</p><pre><code class="language-text">active
enabled
</code></pre><p>如果查看完整状态：</p><pre><code class="language-bash">sudo systemctl status hotspot-forward.service --no-pager
</code></pre><p>正常应看到：</p><pre><code class="language-text">active (exited)
</code></pre><p><code inline="">oneshot</code> 类型服务运行完脚本后退出是正常的，因为规则已经写入内核网络栈。</p><hr><h2>十、最终验证</h2><p>执行：</p><pre><code class="language-bash">nmcli dev status
ip addr show wlan0
ip route
cat /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -S | grep 192.168.0
sudo iptables -S FORWARD | grep -E "wlan0|usb0"
</code></pre><p>理想结果应该包含：</p><pre><code class="language-text">wlan0 connected PiHotspot
usb0 connected usb-wifi
inet 192.168.0.1/24
default via 192.168.99.1 dev usb0 metric 50
1
-A POSTROUTING -s 192.168.0.0/24 -o usb0 -j MASQUERADE
-A FORWARD -i usb0 -o wlan0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i wlan0 -o usb0 -j ACCEPT
</code></pre><p>此时手机连接热点后，应获得：</p><pre><code class="language-text">IP：192.168.0.x
网关：192.168.0.1
DNS：192.168.0.1
</code></pre><p>并可以正常上网。</p><hr><h2>十一、重启验证</h2><p>执行：</p><pre><code class="language-bash">sudo reboot
</code></pre><p>重启后再次检查：</p><pre><code class="language-bash">nmcli dev status
ip addr show wlan0
ip route
systemctl is-active hotspot-forward.service
systemctl is-enabled hotspot-forward.service
sudo iptables -t nat -S | grep 192.168.0
sudo iptables -S FORWARD | grep -E "wlan0|usb0"
</code></pre><p>正常应看到：</p><pre><code class="language-text">wlan0 connected PiHotspot
usb0 connected usb-wifi
wlan0 inet 192.168.0.1/24
default route 走 usb0
hotspot-forward.service active
hotspot-forward.service enabled
NAT 和 FORWARD 规则存在
</code></pre><hr><h2>十二、给连接热点的设备分配固定 IP</h2><p>前面已经让热点网段固定为：</p><pre><code class="language-text">192.168.0.0/24
</code></pre><p>树莓派热点网关为：</p><pre><code class="language-text">192.168.0.1
</code></pre><p>接下来可以通过 DHCP 静态绑定，让指定设备每次连接热点时都获得固定 IP。</p><h3>1. 查看客户端 MAC 地址</h3><p>让手机、电脑或摄像头先连接热点，然后在树莓派上执行：</p><pre><code class="language-bash">ip neigh show dev wlan0
</code></pre><p>输出类似：</p><pre><code class="language-text">192.168.0.23 dev wlan0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
</code></pre><p>其中：</p><pre><code class="language-text">aa:bb:cc:dd:ee:ff
</code></pre><p>就是客户端 MAC 地址。</p><p>也可以用：</p><pre><code class="language-bash">sudo arp -n -i wlan0
</code></pre><hr><h3>2. 创建 dnsmasq 静态租约配置目录</h3><pre><code class="language-bash">sudo mkdir -p /etc/NetworkManager/dnsmasq-shared.d
</code></pre><p>创建静态 IP 配置文件：</p><pre><code class="language-bash">sudo nano /etc/NetworkManager/dnsmasq-shared.d/hotspot-static-leases.conf
</code></pre><p>写入固定 IP 规则。</p><p>示例：</p><pre><code class="language-conf"># 格式：dhcp-host=MAC地址,固定IP,主机名,租期
dhcp-host=aa:bb:cc:dd:ee:ff,192.168.0.20,phone01,infinite
dhcp-host=11:22:33:44:55:66,192.168.0.21,laptop01,infinite
dhcp-host=66:55:44:33:22:11,192.168.0.22,camera01,infinite
</code></pre><p>建议规划：</p><pre><code class="language-text">192.168.0.1      树莓派热点网关，不要分配给客户端
192.168.0.2-19   保留
192.168.0.20-99  固定 IP 设备
192.168.0.100+   临时 DHCP 设备
</code></pre><hr><h3>3. 重启热点让静态绑定生效</h3><pre><code class="language-bash">sudo nmcli connection down PiHotspot
sudo nmcli connection up PiHotspot
</code></pre><p>或者重启 NetworkManager：</p><pre><code class="language-bash">sudo systemctl restart NetworkManager
</code></pre><p>然后让客户端断开 WiFi，再重新连接热点。</p><hr><h3>4. 验证固定 IP</h3><p>执行：</p><pre><code class="language-bash">ip neigh show dev wlan0
</code></pre><p>如果配置成功，可以看到类似：</p><pre><code class="language-text">192.168.0.20 dev wlan0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
</code></pre><p>也可以在手机 WiFi 详情里查看：</p><pre><code class="language-text">IP 地址：192.168.0.20
网关：192.168.0.1
DNS：192.168.0.1
</code></pre><hr><h2>十三、手机固定 IP 注意事项：关闭随机 MAC</h2><p>如果是手机连接热点，很多 Android / iOS 设备默认启用“随机 MAC”或“私有 WiFi 地址”。</p><p>如果开启随机 MAC，树莓派看到的 MAC 地址可能会变化，导致 DHCP 静态绑定失效。</p><p>建议进入手机的 WiFi 设置：</p><pre><code class="language-text">WiFi → HP-001GR → 隐私 / MAC 地址类型
</code></pre><p>改成：</p><pre><code class="language-text">使用设备 MAC
</code></pre><p>或者关闭：</p><pre><code class="language-text">随机 MAC / 私有 WiFi 地址
</code></pre><p>然后重新连接热点。</p><hr><h2>十四、常用排错命令</h2><h3>1. 查看接口状态</h3><pre><code class="language-bash">nmcli dev status
ip addr show wlan0
ip route
</code></pre><p>重点看：</p><pre><code class="language-text">wlan0 是否 connected PiHotspot
usb0 是否 connected usb-wifi
wlan0 是否是 192.168.0.1/24
默认路由是否走 usb0
</code></pre><hr><h3>2. 查看热点密码</h3><pre><code class="language-bash">nmcli dev wifi show-password
</code></pre><hr><h3>3. 查看热点客户端</h3><pre><code class="language-bash">ip neigh show dev wlan0
sudo arp -n -i wlan0
</code></pre><hr><h3>4. 查看 IPv4 转发</h3><pre><code class="language-bash">cat /proc/sys/net/ipv4/ip_forward
</code></pre><p>正常应输出：</p><pre><code class="language-text">1
</code></pre><hr><h3>5. 查看 NAT 规则</h3><pre><code class="language-bash">sudo iptables -t nat -S | grep 192.168.0
</code></pre><p>正常应有：</p><pre><code class="language-text">-A POSTROUTING -s 192.168.0.0/24 -o usb0 -j MASQUERADE
</code></pre><hr><h3>6. 查看转发规则</h3><pre><code class="language-bash">sudo iptables -S FORWARD | grep -E "wlan0|usb0"
</code></pre><p>正常应有：</p><pre><code class="language-text">-A FORWARD -i usb0 -o wlan0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i wlan0 -o usb0 -j ACCEPT
</code></pre><hr><h3>7. 手动恢复热点</h3><p>如果开机后热点异常，可以执行：</p><pre><code class="language-bash">sudo nmcli radio wifi on
sudo rfkill unblock wifi
sudo nmcli connection up usb-wifi
sudo nmcli connection up PiHotspot
sudo systemctl restart hotspot-forward.service
</code></pre><hr><h2>十五、完整配置总结</h2><p>最终配置如下：</p><pre><code class="language-text">usb0：
  作用：USB 随身 WiFi / 4G 网卡，上网入口
  路由优先级：metric 50
  开机自动连接：是

eth0：
  作用：有线管理口 / SSH 管理口
  路由优先级：metric 300
  不抢默认出口

wlan0：
  作用：树莓派板载 WiFi 热点
  热点名称：HP-001GR
  频段：2.4G
  信道：6
  网关：192.168.0.1
  客户端网段：192.168.0.x
  开机自动启动：是

系统转发：
  net.ipv4.ip_forward = 1

NAT：
  192.168.0.0/24 → usb0 MASQUERADE

systemd：
  hotspot-forward.service 开机自动恢复 NAT / FORWARD 规则

固定 IP：
  通过 /etc/NetworkManager/dnsmasq-shared.d/hotspot-static-leases.conf
  使用 dhcp-host=MAC,IP,hostname,infinite 绑定
</code></pre><hr><h2>十六、需要注意的坑</h2><h3>1. 192.168.0.x 容易和上游路由器冲突</h3><p><code inline="">192.168.0.0/24</code> 是很多家用路由器的默认网段。如果以后 <code inline="">usb0</code> 上游也变成了 <code inline="">192.168.0.x</code>，热点网段就会和上游冲突，客户端可能无法上网。</p><p>如果发生冲突，建议把热点网段改成：</p><pre><code class="language-text">192.168.50.1/24
192.168.88.1/24
10.42.0.1/24
</code></pre><h3>2. 树莓派 4B 板载 WiFi 不能同时开 2.4G 和 5G 热点</h3><p>树莓派 4B 支持 2.4G 和 5G，但板载 WiFi 只有一个无线接口，同一时间只能开一个频段的热点。</p><p>如果要同时开 2.4G 和 5G，需要额外插一个支持 AP 模式的 USB WiFi 网卡。</p><h3>3. 手机随机 MAC 会影响固定 IP</h3><p>如果固定 IP 不生效，优先检查手机是否开启了随机 MAC。</p><h3>4. Docker 可能影响转发规则</h3><p>如果系统里运行 Docker，可能会改写 <code inline="">FORWARD</code> 链。本文使用 <code inline="">hotspot-forward.service</code> 每次开机主动恢复规则，目的就是降低 Docker 对热点转发的影响。</p><hr><h2>结语</h2><p>到这里，树莓派 4B 就可以作为一个小型网络共享设备使用：</p><pre><code class="language-text">USB 随身 WiFi 上网
        ↓
树莓派 4B NAT 转发
        ↓
板载 WiFi 发热点
        ↓
手机、电脑、摄像头等设备联网
</code></pre><p>这套方案的优点是配置简单、成本低、可以开机自启，并且能通过 DHCP 静态绑定给重要设备分配固定 IP。对于临时组网、摄像头接入、IoT 设备调试、小型局域网测试都比较实用。</p></body></html><!--EndFragment-->
</body>
</html>