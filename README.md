# Home Assistant Leishen Auto Pause (ARP Edition)

这是一个 Home Assistant 的自动化配置方案，用于实现 **雷神加速器盒子（或 PC）** 的自动暂停功能。

相比于传统的 Ping 或 Nmap 扫描方案，本项目采用了 **ARP（物理层）检测机制**。

这完美解决了部分加速器盒子为了安全而设置的 **“禁 Ping” 和 “关闭所有 TCP 端口”** 导致 Home Assistant 无法判断设备在线状态的问题。

---

## ✨ 特性

- 🚀 **极速响应**：设备关机 / 拔线后，约 **1 分钟内** 即可自动触发暂停（传统 Nmap 方案通常需要 5–10 分钟）。
- 🛡️ **穿透防火墙**：利用 Linux 底层 `ip neigh` 命令读取 ARP 表，无需设备回应 Ping，只要设备物理连接了网线即可被检测。
- 🤖 **自动重试**：网络波动导致 API 请求失败会自动重试 5 次。
- 📩 **状态通知**：集成通知推送（可配置），暂停成功或 Token 失效时及时提醒。
- 🔒 **隐私安全**：使用 `secrets.yaml` 存储 Token，敏感信息不直接暴露在代码中。

---

## ⚙️ 前置要求

- 已安装 **Home Assistant**（推荐运行在 Linux / Docker 环境，因为需要使用 `ip neigh` 命令）。
- 拥有 **雷神加速器账号**。
- 知道你的 **雷神加速器设备的内网 IP 地址**。

---

## 🚀 安装与配置

### 第一步：获取 API Payload

1. 在浏览器中登录雷神加速器官网。
2. 按 **F12** 打开开发者工具，点击 **“暂停时长”** 按钮。
3. 在 **Network（网络）** 标签页中找到 `pause` 请求。
4. 复制 **Request Payload（JSON 格式）**，它看起来像这样：

```json
{"account_token":"你的Token","lang":"zh_CN"}
```

---

### 第二步：配置 `secrets.yaml`

在你的 Home Assistant `secrets.yaml` 文件中添加以下内容：

```yaml
# 注意：这里需要填入完整的 JSON 字符串
leishen_api_payload: '{"account_token":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx","lang":"zh_CN"}'
```

---

### 第三步：部署 Package 文件

- 下载本项目中的 `leishen_control.yaml` 文件。
- 将其放入 Home Assistant 配置目录下的 `packages` 文件夹中（如果你启用了 Packages 功能）。

或者：

- 直接将代码内容复制到你的 `configuration.yaml` 及 `automations.yaml` 中对应的位置。

---

### 第四步：修改配置（重要！）

打开 `leishen_control.yaml`，找到以下两处进行修改：

#### 1️⃣ 修改 IP 地址

搜索 `command_line` 部分，将 `192.168.x.x` 替换为你设备的真实 IP（注意：一行命令里有两处需要替换）：

```bash
command: "ping -c 1 -W 1 192.168.1.100 > /dev/null 2>&1; ip neigh show 192.168.1.100 | grep -q 'lladdr' && echo on || echo off"
```

#### 2️⃣ 修改通知服务

搜索 `notify.pushover`，将其替换为你正在使用的通知服务实体，例如：

- `notify.mobile_app_iphone`
- `notify.wechat`

---

### 第五步：重启 Home Assistant

配置修改完成后，**完全重启 Home Assistant** 以加载 Command Line 实体。

---

## 🛠️ 原理说明

### 为什么使用 ARP 检测？

- 传统的 Ping 检测依赖 **ICMP 协议**，而很多硬件加速器盒子防火墙级别很高，会直接丢弃 ICMP 包，导致 HA 认为设备一直离线。
- 使用 Nmap 扫描端口虽然可行，但 **速度慢且资源消耗大**。

本项目利用以下组合命令：

```bash
ping -c 1 -W 1 IP ... ; ip neigh show IP | grep -q 'lladdr'
```

执行逻辑：

1. 先发送一个 Ping 包（不管对方回不回应）。
2. 该操作会强制 Linux 内核更新 ARP 缓存表。
3. 通过 `ip neigh` 查看底层 ARP 表：
   - 如果存在 MAC 地址（`lladdr`），说明设备 **物理连接正常**，判定为在线。

---

## 🤝 贡献

欢迎提交 **Issue** 或 **Pull Request** 来改进代码。
