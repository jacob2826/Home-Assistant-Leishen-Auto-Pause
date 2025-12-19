# Home Assistant Leishen Auto-Pause
# 雷神加速器自动暂停集成 (HA Package)

这是一个用于 Home Assistant 的 Package 配置，旨在实现当你的游戏 PC/主机离线（关机）时，自动暂停雷神加速器时长，防止时长浪费。

## ✨ 主要功能

* **🔄 自动暂停**：监控设备在线状态，离线 5 分钟后自动调用暂停接口。
* **📡 双模检测**：支持 **Ping (ICMP)** 和 **Nmap (ARP)** 两种检测方式，完美适配雷神加速盒等禁 Ping 设备。
* **🔁 智能重试**：如果暂停请求失败（网络波动或 API 错误），系统会自动重试 5 次（间隔 30 秒）。
* **🛡️ Token 保活与监控**：每小时自动刷新账号状态，保持 Token 活跃。
* **🚨 状态报警**：
    * **暂停成功**：发送普通通知。
    * **暂停失败**：重试 5 次后仍失败，发送**高优先级**报警。
    * **Token 失效**：检测到 API 返回 Token 错误时，发送**高优先级**报警。
* **📱 Pushover 集成**：原生支持 Pushover 消息推送。

## 🛠️ 准备工作

在开始之前，请确保你拥有：

1.  **Home Assistant** (已安装并运行)。
2.  **雷神加速器账号 Token** (`account_token`)。
3.  **Pushover 账号** (用于接收通知，可选，也可自行修改为其他通知服务)。
4.  **目标设备的固定 IP 地址** (建议在路由器设置静态 IP)。

### 🔍 如何获取 Token (详细步骤)

1.  在 PC 浏览器中访问 [雷神加速器官网](https://vip.leigod.com/user.html) 并登录。
2.  按下 `F12` 键打开浏览器的开发者工具。
3.  切换到 **Network (网络)** 标签页。
4.  在网页上进行任意操作（例如点击“暂停”或“恢复”按钮），此时 Network 面板会出现新的请求。
5.  在 Network 面板的过滤器/搜索框中输入 `pause` 或 `info`，找到指向 `webapi.leigod.com` 的请求（通常是 `pause` 或 `info` 接口）。
6.  点击该请求，切换到 **Payload (载荷)** 或 **Form Data** 标签页。
7.  找到 `account_token` 字段，复制其后的长字符串（这就是你的 Token）。

## 🚀 安装配置指南

### 第一步：启用 Packages 功能

检查你的 `configuration.yaml` 文件，确保启用了 `packages` 功能：

```yaml
homeassistant:
  packages: !include_dir_named packages
```

如果之前没有 `packages` 文件夹，请在 `configuration.yaml` 同级目录下新建一个名为 `packages` 的文件夹。

### 第二步：导入配置文件

下载本项目中的 `leishen_control.yaml` 文件，将其放入你的 `packages/` 文件夹中。

### 第三步：配置 Secrets (关键)

为了保护隐私，我们不直接将 Token 写在代码里。请打开你的 `secrets.yaml` 文件，添加以下内容：

**⚠️ 注意：格式非常重要！外层必须是单引号 `'`，内部 JSON 属性必须是双引号 `"`。**

```yaml
# secrets.yaml

# 雷神 API 请求体 (替换 YOUR_TOKEN 为你的真实 Token)
leishen_api_payload: '{"account_token": "YOUR_TOKEN", "lang": "zh_CN"}'
```

### 第四步：添加设备检测 (二选一)

根据你的设备类型，选择一种检测方式。

#### 方式 A：Ping 检测 (推荐大多数电脑/主机)
如果你的设备可以被 Ping 通：
1.  进入 HA **配置** -> **设备与服务** -> **添加集成** -> 搜索 **Ping (ICMP)**。
2.  输入设备 IP。
3.  添加后，将实体重命名为 `binary_sensor.leishen_box`。
4.  **YAML 配置**：保持 `leishen_control.yaml` 默认即可。

#### 方式 B：Nmap 检测 (推荐雷神盒子/禁Ping设备)
如果设备 Ping 不通 (如雷神盒子)，使用 ARP 协议检测：
1.  进入 HA **配置** -> **设备与服务** -> **添加集成** -> 搜索 **Nmap Tracker**。
2.  配置扫描 IP 为设备 IP (例如 192.168.1.100)。
3.  勾选 "Home Assistant 负责追踪设备"。
4.  添加后，系统会生成一个 `device_tracker` 实体 (例如 `device_tracker.192_168_1_100`)。建议将其重命名为 `device_tracker.leishen_box`。
5.  **YAML 配置 (必须修改)**：
    打开 `packages/leishen_control.yaml`，找到 `trigger` 部分：
    * 注释掉 **方式一 (Ping)** 的代码。
    * 启用 **方式二 (Nmap)** 的代码 (删除前面的 `#`)。

### 第五步：确认通知服务名称

本项目默认使用 `notify.pushover` 服务。请前往 HA **开发者工具** -> **服务**，检查你的 Pushover 服务名称是否一致。

* 如果你的服务名是 `notify.my_iphone` 或其他，请在 `leishen_control.yaml` 中全局搜索并替换 `notify.pushover`。

### 第六步：重启生效

重启 Home Assistant。

## ✅ 验证是否成功

1.  **检查 Sensor**：
    * 在开发者工具中查看 `sensor.leishen_account_info`。
    * 状态应为 `0` (正常) 或 `400803` (已暂停)。
    * 如果状态是 `400xxx`，说明 Token 填错了。

2.  **测试自动化**：
    * 手动在开发者工具中调用 `automation.trigger`，触发 `automation.leishen_auto_pause_logic`。
    * 如果你收到了 Pushover 推送，说明配置成功！

## 📝 常见问题

**Q: Token 会过期吗？**
A: 本项目包含每小时一次的 API 请求（Sensor），通常可以起到保活作用。如果 Token 意外失效，系统会通过 Pushover 发送高优先级报警提醒你更新。

**Q: 为什么设备关机了还没暂停？**
A: 请检查 HA 中的设备状态。
* 如果是 Ping 方式，`binary_sensor` 应该变 `off`。
* 如果是 Nmap 方式，`device_tracker` 应该变 `not_home`。
* 确认配置中的 `for: "00:05:00"` 延时是否已过。

**Q: 我不想用 Pushover，用微信/Telegram 可以吗？**
A: 可以。修改 yaml 文件中的 `action` 部分，将 `service: notify.pushover` 替换为你使用的通知服务（如 `notify.wechat`），并调整 `data` 下的内容格式即可。

## 📄 License

GNU General Public License v3.0
