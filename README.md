# WPT_Railway — 无线充电 MQTT-HTTP 桥接服务器 (历史备选方案)

[![Deploy](https://img.shields.io/badge/Deploy-Railway-0B0D0E)]()
[![Runtime](https://img.shields.io/badge/Runtime-Node.js-339933)]()
[![MQTT](https://img.shields.io/badge/Broker-EMQX-00E16C)]()
[![Status](https://img.shields.io/badge/Status-Deprecated_(V5.0+)-orange)]()

WPT 无线充电系统的 MQTT-HTTP 桥接服务器。**V5.0 起已废弃** —— 微信小程序改为 OneNET HTTP API 直连，不再需要中间桥接。

该仓库保留作为备选方案和历史参考。如果未来 OneNET API 域名在微信小程序白名单中失效，可重新部署。

## 目录

1. [历史背景](#历史背景)
2. [当前方案 (V5.0+)](#当前方案-v50)
3. [功能](#功能)
4. [架构](#架构)
5. [快速开始](#快速开始)
6. [部署到 Railway](#部署到-railway)
7. [本地测试](#本地测试)
8. [API 参考](#api-参考)
9. [环境变量](#环境变量)
10. [备用场景](#备用场景)

---

## 历史背景

### V1-V4: 桥接方案

V5.0 之前，微信小程序存在两个问题导致无法直连云平台：

1. **EMQX TLS 证书**：微信小程序真机只信任有限的白名单 CA，公共 EMQX broker 的 TLS 证书不被微信认可，WebSocket 连接被拒。
2. **OneNET 域名未备案**：OneNET API 的域名 `iot-api.heclouds.com` 需要在小程序后台添加为 request 合法域名（已在 V5.1 解决）。

因此设计了中间桥接层：

```
微信小程序 ──HTTP── 桥接服务器 ──MQTT── EMQX ── ESP8266 ── STM32
```

桥接服务器运行在 Railway 容器中，监听 HTTP 请求，转发到 EMQX MQTT broker。这是一个典型的"协议适配"中间件。

### V5.0+: 直连方案

微信小程序域名白名单配置成功后，改为直连 OneNET HTTP API：

```
微信小程序 ──HTTPS── OneNET API ──MQTT── ESP8266 ── STM32
```

优点是：
- 零中间层，减少单点故障
- 无需管理服务器/容器
- 与网页控制台共享同一套 API 和 Token
- 免去 Railway 免费额度限制

---

## 当前方案 (V5.0+)

| 组件 | 连接方式 | 说明 |
|:---|:---|:---|
| 微信小程序 | OneNET HTTP API 直连 | 2s 轮询, 在线检测, Switch 验证重发 |
| 网页控制台 | OneNET HTTP API 直连 | Cloudflare Pages 部署 |
| ESP8266 | OneNET MQTT 直连 | `mqtts.heclouds.com:1883` |

---

## 功能

桥接服务器提供两个 HTTP 端点：

| 端点 | 方法 | 说明 |
|:---|:---|:---|
| `/data` | GET | 获取设备最新数据 (从 EMQX 缓存读取) |
| `/control` | POST | 下发控制指令到设备 (发布到 EMQX `wpt/20260001/cmd`) |

同时维护一个内存缓存 (`latestData`)，订阅 EMQX `wpt/20260001/data` 主题保持最新值。

---

## 架构

```
┌──────────────┐   HTTP    ┌──────────────┐   MQTT    ┌──────────┐
│  微信小程序   │ ◄──────► │ Railway 容器  │ ◄──────► │  EMQX    │
│  wx.request  │  GET/POST │ (Node.js)    │  sub/pub │  Broker  │
└──────────────┘          │ bridge.mjs   │          └────┬─────┘
                          │ Express +    │               │ MQTT
                          │ mqtt.js      │          ┌────┴─────┐
                          └──────────────┘          │ ESP8266  │
                                                    └──────────┘
```

---

## 快速开始

### 本地运行

```bash
npm install
node bridge.mjs
```

服务器默认监听 `0.0.0.0:4567`。

### 环境变量

| 变量 | 默认值 | 说明 |
|:---|:---|:---|
| `PORT` | `4567` | HTTP 监听端口 |
| `MQTT_HOST` | `broker.emqx.io` | MQTT Broker 地址 |
| `MQTT_PORT` | `1883` | MQTT 端口 |
| `MQTT_TOPIC` | `wpt/20260001/data` | 数据订阅主题 |
| `CMD_TOPIC` | `wpt/20260001/cmd` | 指令发布主题 |

### Docker

```bash
docker build -t wpt-bridge .
docker run -p 4567:4567 wpt-bridge
```

---

## 部署到 Railway

1. Fork 此仓库到自己的 GitHub
2. [Railway](https://railway.app/) → New Project → Deploy from GitHub → 选择仓库
3. Railway 自动检测 Node.js 项目并部署
4. 在 Dashboard → Variables 设置环境变量 (如有自定义)
5. 获得 URL: `https://<project-name>.up.railway.app`

或者通过 Railway CLI：

```bash
railway login
railway init
railway up
```

### Railway 免费额度

每月 $5.00 免费额度。单实例部署约消耗：
- 512MB RAM: ~$0.40/天 ≈ $12/月 → 超免费额度
- 可以把 `railway.toml` 换成 `numReplicas: 0` + `sleepAfter: 300` 降低消耗

---

## API 参考

### GET /data

获取设备最新遥测数据。

**Response 200**:
```json
{
  "V": 12.50,
  "I": 1.23,
  "F": 100000,
  "timestamp": "2026-05-25T10:30:00.000Z"
}
```

**Response 204**: 暂无缓存数据 (ESP8266 未上报)

### POST /control

下发控制指令。

**Request**:
```json
{
  "cmd": "CMD:ON"
}
```

或:

```json
{
  "cmd": "CMD:SETFREQ:108000"
}
```

**Response 200**:
```json
{
  "success": true,
  "cmd": "CMD:ON"
}
```

---

## 备用场景

以下情况可考虑重新部署桥接方案：

1. OneNET API 域名在小程序白名单中再次失效
2. 需要额外的数据缓存层
3. 需要在指令下发前做额外的权限校验或格式转换
4. OneNET 平台迁移到其他云平台，需要协议适配

---

## 关联项目

- 主项目: [Ran-sh/WPT_PWM](https://github.com/Ran-sh/WPT_PWM) (分支 `ONENET`)
- 网页控制台: [Ran-sh/WPT_Onenet_IoT](https://github.com/Ran-sh/WPT_Onenet_IoT)

## 作者

**Rssss**

## 许可

MIT
