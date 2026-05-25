# WPT_Railway — 无线充电桥接服务器 (历史备选方案)

[![Deploy](https://img.shields.io/badge/Deploy-Railway-0B0D0E)]()
[![Runtime](https://img.shields.io/badge/Runtime-Node.js-339933)]()

WPT 无线充电系统的 MQTT-HTTP 桥接服务器。**V5.0 起已废弃**，微信小程序改为 OneNET API 直连，不再需要中间桥接。

该仓库仅保留作为备选方案和历史参考。

## 历史背景

V5.0 之前，微信小程序无法直连 OneNET API（域名白名单问题），需要通过桥接服务器中转：

```
小程序 ──HTTP── 桥接服务器 ──MQTT── EMQX ── ESP8266 ── STM32
```

现改为：

```
小程序 ──HTTPS── OneNET API ──MQTT── ESP8266 ── STM32
```

## 备用场景

如果未来 OneNET API 域名在小程序白名单中失效，可重新部署此方案。

### 部署到 Railway

1. Fork 此仓库
2. Railway → New Project → Deploy from GitHub
3. 设置环境变量 (通过 Railway `.env` 或 Dashboard)
4. 自动部署

### 本地测试

```bash
node bridge.mjs
```

## 关联项目

- 主项目: [Ran-sh/WPT_PWM](https://github.com/Ran-sh/WPT_PWM) (分支 `ONENET`)
- 网页控制台: [Ran-sh/WPT_Onenet_IoT](https://github.com/Ran-sh/WPT_Onenet_IoT)
