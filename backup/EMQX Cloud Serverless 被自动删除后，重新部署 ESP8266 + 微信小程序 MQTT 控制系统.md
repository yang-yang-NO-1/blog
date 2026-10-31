> 记录时间：2026-10-31  
> 环境：EMQX Cloud Serverless、ESP8266（懒人开关）、微信小程序、WiFiManager、MQTT.js  
> 本文是一次实际恢复部署记录。示例域名和密码均为占位符，请替换成自己的配置。

## 一、背景：好久没用，原来的 EMQX 部署没了

之前做过一个基于 ESP8266 + EMQX + 微信小程序的个人 IoT 项目：ESP8266 通过 MQTT 接收开关命令，控制 SG90 舵机；微信小程序负责设备管理和可视化控制。相关代码在 GitHub：

- [lanrenkaiguan（ESP8266 懒人开关固件）](https://github.com/yang-yang-NO-1/lanrenkaiguan)
- [iot-miniapp-emqx-starter（微信小程序 IoT 控制台）](https://github.com/yang-yang-NO-1/iot-miniapp-emqx-starter)

这次准备重新使用时，发现原来的 EMQX Serverless 部署已经不在了，设备和小程序都无法连接。

查阅 [EMQX 官方说明](https://docs.emqx.com/zh/cloud/latest/create/serverless.html)：Serverless 连续 **30 天没有活跃客户端连接**会被系统停止，停止后 **30 天内没有重新启用**则可能被删除。已经删除的部署及其中的数据不能直接恢复。因此决定新建部署、重新创建 MQTT 账号，并把设备端和小程序迁移过去。

## 二、新建 EMQX Serverless 部署

1. 打开 [EMQX Cloud 控制台](https://cloud.emqx.com/)，创建 **Serverless** 部署。
2. 选择合适的区域，按需设置名称。个人自用可将每月消费限额保持为 **0**，使用免费额度（额度和规则以 EMQX 当时的控制台为准）。
3. 等待部署运行，在“概览”中找到新部署的 **连接地址（Host）**，复制备用。

Serverless 默认提供两种加密连接：

| 客户端 | 协议 | 端口 |
| --- | --- | --- |
| ESP8266 / ESP32 等设备 | MQTT over TLS | `8883` |
| 微信小程序 | MQTT over WSS | `8084` |

**注意：Host 只填服务器域名，不要把 `https://`、`wxs://` 或 `/mqtt` 一起填入 ESP8266 的 MQTT Host。**

## 三、重新创建 MQTT 认证账号

进入部署的 **访问控制 → 客户端认证 → 添加**，本次只创建两个账号：

| 用户名 | 用途 |
| --- | --- |
| `miniapp_home` | 微信小程序管理员 |
| `device` | ESP8266 懒人开关 |

两个账号分别设置新的强密码。这是 **MQTT 客户端认证用户**，不是登录 EMQX Cloud 网页的账号。

### 需要手动新建 MQTT Topic 吗？

**不需要。** 一开始误以为要在 EMQX 中新建多条主题，其实 MQTT Topic 由程序发布/订阅时直接使用，通常不需要事先创建。

本项目使用的设备主题前缀为：

```text
u/home/devices/lanrenkaiguan
```

主要通道如下：

| Topic 后缀 | 作用 |
| --- | --- |
| `/cmd` | 小程序下发命令 |
| `/state` | 设备状态回传 |
| `/status` | 在线/离线状态 |
| `/ack` | 命令执行回执 |
| `/meta` | 设备元信息 |
| `/event` | 设备事件/调试信息 |

这次是个人自用，沿用 EMQX 默认的**黑名单授权模式**：未被明确 Deny 的发布和订阅操作默认允许，因此没有额外配置逐条 ACL。这个方案配置简单，但意味着**任何拿到有效 MQTT 账号密码的客户端，可能访问其他设备的 Topic**。若以后增加多用户或开放给别人，应该配置最小权限 ACL。参考 [EMQX 默认授权说明](https://docs.emqx.com/zh/cloud/latest/deployments/default_authz.html)。

## 四、ESP8266 设备端修改 MQTT 连接

`lanrenkaiguan` 固件已有 WiFiManager，可以通过配网页修改保存的 Wi-Fi、MQTT 连接参数，不必为了换服务器而整片擦除 Flash。

进入配网页后，保持设备身份不变，只将服务器地址、用户名和密码更新为新部署的信息：

```ini
MQTT Host   = <新 EMQX 部署的 Host>
MQTT Port   = 8883
MQTT TLS    = 1
MQTT User   = device
MQTT Pass   = <新设置的设备密码>
Account ID  = device
Device ID   = lanrenkaiguan
Topic Base  = u/home/devices
```

默认配网热点格式：

```text
SSID：IoT-Config-lanrenkaiguan
配网页：http://192.168.4.1
```

配网热点密码请以自己固件中的 `IOT_PORTAL_AP_PASSWORD` 为准；公开发布时不建议继续沿用简单的默认密码。

### 关于配网按键

这份固件原本的配网键设置为 `D7 / GPIO13`，在设备运行时长按约 **5 秒**进入配网。我这块自制板的 D7 不方便使用，所以排查时还考虑过改用现有的 **FLASH 按键（GPIO0）**：

```cpp
#define IOT_PORTAL_BUTTON_PIN         0
#define IOT_PORTAL_BUTTON_ACTIVE_LEVEL LOW
#define IOT_FORCE_PORTAL_HOLD_MS      5000UL
```

若采用该改动，需要**重新编译烧录**；且 GPIO0 是 ESP8266 的启动配置脚，**不能在上电/复位时按住**，否则可能进入串口下载模式。已能通过其他方式进入配网页的设备，无需为本次迁移专门修改按键。

如果确实需要**同时清除 Wi-Fi 和 MQTT 保存配置**，固件中已提供 `Iot.resetSettings(true)`：它会清除 WiFiManager 保存的网络信息及 LittleFS 中的 MQTT 配置，并重启设备。注意：这是清除配置，不是普通的“重新进入配网页”；本次只需要换服务器，直接在配网页覆盖参数即可，无须清空。

## 五、微信小程序修改连接地址

小程序项目使用 MQTT.js，通过 WebSocket 连接 EMQX。登录页填入：

```ini
MQTT WebSocket 地址 = wxs://<新 EMQX Host>:8084/mqtt
EMQX 用户名          = miniapp_home
EMQX 密码            = <新设置的小程序密码>
ClientId 前缀        = miniapp_home
Topic Base          = u/home
```

这里有一个很容易写错的地方：

```text
正确：wxs://<Host>:8084/mqtt
错误：wxs://<Host>/mqtt:8084/mqtt
```

**`8084` 必须写在域名后面，`/mqtt` 只写一次。** 微信小程序代码中使用 `wxs://`，而微信平台管理后台的 socket 域名以 `wss://` 形式配置。具体连接端口以 EMQX 控制台显示为准。参考 [EMQX 微信小程序连接文档](https://docs.emqx.com/zh/cloud/latest/connect_to_deployments/wechat_miniprogram.html)。

### 微信公众平台后台也要更新

因为更换了 EMQX 部署，连接域名也变了。进入：

**微信公众平台 → 开发 → 开发管理 → 开发设置 → 服务器域名 → socket 合法域名**

把旧部署的域名替换成**新部署域名**，按微信后台对协议及端口的校验要求保存。这里是 `wss://`，不要填小程序代码使用的 `wxs://`；是否需要显式填写 `:8084`，应以微信后台当前规则和实际请求端口为准。

另外，项目的 `miniprogram/pages/access-login/access-login.js` 中也有旧服务器的默认地址常量，后续重新发布版本时应同步更新，避免新手机首次进入时自动出现失效域名。不要把真实密码写入公开仓库。

## 六、在小程序中重新添加懒人开关

开发者工具登录成功后，打开 **添加设备** 页面，填写：

| 字段 | 内容 |
| --- | --- |
| 设备 ID | `lanrenkaiguan` |
| 设备名称 | `懒人开关` |
| 设备类型 | `servo_switch` |
| Topic Prefix | **留空** |

Topic Prefix 留空后，小程序会按照 `Topic Base + /devices/ + 设备 ID` 自动生成：

```text
u/home/devices/lanrenkaiguan
```

点击保存后，再确保 ESP8266 已通过新 MQTT 地址成功上线。**“添加设备成功”只是登记设备信息，不代表设备已经在线。**

如果要手动测试控制命令，可以通过 MQTTX 发布到：

```text
Topic：u/home/devices/lanrenkaiguan/cmd
Retain：false
```

示例 Payload：

```json
{"msgId":"cmd_test_on","action":"set","params":{"relay":1},"widgetId":"w_lanrenkaiguan_relay"}
```

正常情况下设备会执行舵机动作，并在 `/ack`、`/state` 等主题返回消息。

## 七、遇到的问题：开发者工具正常，安卓手机最初超时

这次迁移最容易让人误判的地方，是手机端第一次连接失败，弹窗提示：

> 连接超时，请检查 EMQX 用户名、密码、WSS 地址、socket 合法域名和 ACL。

排查过程是：

1. 新 EMQX 部署已经运行，两个 MQTT 账号已创建。
2. 微信小程序改用了新地址，微信后台的 socket 合法域名也更新了。
3. **微信开发者工具能成功登录**，添加懒人开关后也能正常控制。
4. **安卓手机第一次打开小程序时超时**。
5. 后来进行真机调试，没有出现问题；再重新打开手机小程序，**没有改 MQTT 密码**，登录和控制也都恢复正常。

因此，这次最终 **不需要额外搭建 443 反向代理，也不需要修改 MQTT 用户密码**。首次超时可能与后台域名配置生效、手机端缓存/旧连接状态或临时 WebSocket 握手失败有关，但当时没有记录到足以定位根因的底层网络错误，不能确定是哪一个。

这里也不能得出“所有安卓微信都支持 8084”的结论，只能说**我这台手机和这次部署的组合已经实测可用**。

## 八、最终验证结果

| 验证项 | 本次结果 |
| --- | --- |
| 新 EMQX Serverless 部署 | 正常 |
| `miniapp_home` MQTT 登录 | 正常 |
| `device` 设备账号连接 | 正常 |
| ESP8266 懒人开关在线 | 正常 |
| 微信开发者工具控制 | 正常 |
| 安卓手机小程序登录 | 正常 |
| 安卓手机实际控制设备 | 正常 |

原有的 ESP8266 + EMQX + 微信小程序控制功能已经恢复。

## 九、这次迁移之后的注意事项

### 1. 及时备份控制台配置

微信小程序有 **维护工具 → 导出完整配置**，也能将设备列表、面板布局、场景、定时等同步为 EMQX retained 消息。如果旧部署被删，旧 broker 内的 retained 不会自动出现在新部署。若原手机仍保留本地设备和面板数据，可以在新 broker 上重新同步。

**注意：导出的备份可能包含 MQTT 登录信息，不应公开上传。**

### 2. 避免下次又因长期不用被删除

Serverless 长时间无活跃连接可能被系统停止，停止后长期不恢复可能被删除。建议定期检查部署运行状态，并保留设备/小程序的本地备份。仅“设备以前配置过自动重连”不意味着设备当前一定在线。

### 3. 保护账号和源码

设备和小程序使用不同的强密码；即使是自用，最好也不要把真实 MQTT 密码、Wi-Fi 密码写进公开 GitHub。后续可进一步补上 ACL 最小权限和设备端 TLS 服务器证书验证。

## 十、总结

这次恢复其实比最初想的简单得多：

**1 个 EMQX Serverless 部署 + 2 个 MQTT 认证账号 + 更新 ESP8266 和微信小程序连接地址 + 更新微信后台 socket 合法域名，即可恢复原有控制功能。**

不需要手动创建每一个 MQTT Topic。个人自用可以先利用默认授权模式恢复服务，但正式面向多人使用时仍应配置 ACL。

---

**参考资料**

- [EMQX Cloud：创建 Serverless 部署](https://docs.emqx.com/zh/cloud/latest/create/serverless.html)
- [EMQX Cloud：Serverless 连接指引](https://docs.emqx.com/zh/cloud/latest/deployments/port_guide_serverless.html)
- [EMQX Cloud：微信小程序连接教程](https://docs.emqx.com/zh/cloud/latest/connect_to_deployments/wechat_miniprogram.html)
- [EMQX Cloud：默认授权](https://docs.emqx.com/zh/cloud/latest/deployments/default_authz.html)
