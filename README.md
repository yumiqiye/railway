<!--
  ══════════════════════════════════════════════════════
    清 江 铁 路  1 2 3 0 6    QINGJIANG RAILWAY
    版权所有，仿冒必究    All Rights Reserved
  ══════════════════════════════════════════════════════
-->
# 清江铁路12306 · 购票演示系统

> 文档版本：2026-11-14  
> 适用代码：`ticket` / `check` / `admin` / `shared` 四目录结构  
> 源码位置：本文件及全部未编译代码位于 `source/`  
> 打包入口：项目根目录 `一键打包.cmd`，产物输出到根目录 `dist/`  
> 运行环境：Windows + Node.js ≥ 18 + pnpm  
> 演示定位：本地铁路购票、检票、后台管理演示系统，不连接真实 12306、不接真实支付

---

## 文档目录

1. 项目简介
2. 本轮升级总览
3. 系统架构
4. 目录结构
5. 运行环境与依赖
6. 快速启动
7. 配置文件说明
8. 默认账号与权限
9. 数据模型
10. 前台购票端
11. 后台管理端
12. 检票终端
13. 动态电子客票二维码
14. WebSocket 实时推送
15. HTTP API 参考
16. 异常处理与人工复核
17. 日志管理
18. 数据重置
19. UI、页脚与横竖屏
20. 本地部署与便携打包
21. 常见问题
22. 安全边界与已知限制
23. 验收清单
24. 开发约定
25. 变更记录

---

## 1. 项目简介

清江铁路12306 是一个可完整跑通的铁路购票演示系统，覆盖：

- 旅客注册、登录、人机验证
- 城市、车站、车次、票额维护
- 车票查询、下单、订单、退票、改签、电子发票
- 乘车人管理、乘车人本人人脸录入
- 检票端扫码、身份证、人脸核验
- 一人一闸、按乘车人逐个核销
- 后台实时监控、人工复核、设备管理
- 公告、版权、ICP 备案、公安备案
- WebSocket 实时刷新
- 动态电子客票二维码

系统由四个目录组成：

| 目录 | 作用 | 默认端口 |
| --- | --- | --- |
| `ticket/` | 前台页面 + 主业务 API + 数据库 | 3000 |
| `check/` | 独立检票服务 + 检票端页面 | 3001 |
| `admin/` | Vue 3 后台管理 UI | 8001 |
| `shared/` | 公共配置读取、二维码签名 | 无 |

根目录还有：

| 文件 | 作用 |
| --- | --- |
| `config.toml` | 三个服务端口、二维码密钥、二维码有效期 |
| `manage.ps1` | PowerShell 启停脚本 |
| `manage.bat` | 双击菜单入口 |
| `README.md` | 本文档 |
| `CHANGELOG.md` | 版本变更记录 |

---

## 2. 本轮升级总览

2026-11-14 这一轮改造覆盖了服务端、后台、前台、检票端和项目结构。

### 2.1 服务端

- 引入 `ws`，从轮询升级为真正的 WebSocket。
- 新增 `shared/config.js`，读取根目录 `config.toml`。
- 新增 `shared/ticket-qr.js`，动态二维码 HMAC-SHA256 签名。
- 数据库新增 `devices`、`logs`、`checkinEvents`。
- 车站数据新增 `status`、`gateCount`、`platformCount`、`address`。
- 设置新增公告策略、页脚备案、维修文案、活体检测、扫码画面、二维码刷新间隔。
- 新增后台监控、设备、人工复核、日志、事件上报、数据重置接口。
- 新增 `/api/accounts/me`，后台每次进入都会刷新权限。
- 修复设备编辑接口 `account is not defined`。
- 删除购票成功后对已废弃 `faceToken` 的引用，避免下单最后一步报错。

### 2.2 后台

- 菜单调整为 10 项：
  总览、城市、车站、车次、车票、检票管理、订单、系统设置、日志、关于。
- 新增「检票管理」独立菜单。
- 新增「日志管理」独立菜单。
- 系统设置移动到订单管理下方。
- 检票管理包含：
  - 实时监控
  - 人工复核
  - 检票设备
  - 车站大屏
- 车站管理新增状态、检票口数、站台数、地址。
- 车站操作列改为「编辑 / 更多」，删除收进下拉菜单。
- 系统设置新增公告、备案、检票终端、数据重置。
- 后台登录态失效时自动回登录页，不再停留在错误页。

### 2.3 前台

- 购票不再要求账号本人人脸核验。
- 仍要求所选乘车人已录入本人人脸。
- 查票改为新标签页打开，并自动回填查询条件。
- 动态二维码按设置间隔刷新。
- 公告支持顶部显示、弹窗显示、启动弹窗、重复弹窗。
- 页脚支持版权、ICP、公安备案、客服电话。
- 支持横屏和竖屏。

### 2.4 检票端

- 首次打开强制设置车站、闸机类型、检票机编号。
- 新增活体检测，要求检测到一次眨眼。
- 扫码画面默认全部隐藏；仅检票闸机可在设置中打开为右下角小窗，入站和出站不显示。
- 停用与维修统一由后台控制，检票端不再提供本机停用开关。
- 停用后显示可后台修改的维修文案。
- 新增版权、ICP、公安备案页脚。
- 支持横屏和竖屏。

### 2.5 工程结构

- 原 `server` 拆为 `ticket` 与 `check`。
- 原 `naive-ui-admin-3.0.0` 改名为 `admin`。
- 公共代码抽到 `shared`。
- 新增 `manage.bat` / `manage.ps1`。
- 端口统一由 `config.toml` 控制。
- 后台 Vite 也读取 `[admin] port`。
- 提供拆分前备份 `backup-20261114`。

### 2.6 异常告警、运营报表与大屏、Windows 客户端

- 告警中心：闸机离线、配置失败、配置超时未确认、检票异常统一入告警池；按“类型 + 车站 + 闸机”去重。
- 告警支持确认、忽略、解决，后台 `/alerts` 页 10 秒刷新，可选声音提醒。
- 运营报表 `/reports`：按乘车日期统计售票、退票、收入、净收入、检票率、客单价，支持车站/车次/席别/渠道下钻和 CSV 导出。
- 候车大屏 `/screen.html`：站点切换、公告、列车状态、检票口、站台、地标色、下一趟开检、今日候车/已检/检票率/在线闸机，5 秒刷新。
- 检票口引导屏 `/guide.html`：大字号显示车次、终到站、开点、检票口、站台和状态，临时改口/改站台显示“临时调整”，10 秒刷新；支持 `?stationId=`、`?date=` 和页面内切换车站。
- Windows 检票客户端 `QingjiangCheckClient.exe`：Edge/Chrome kiosk 启动器，支持 watchdog、托盘、开机自启；不内嵌浏览器内核。
- 打包脚本会把客户端 exe 和说明放进分发包根目录；Android 5 仍直接使用现代 Chrome，不安装该客户端。

---

## 3. 系统架构

整体是「两个 Node 服务 + 一个 Vue 后台 + 一个共享目录」的结构。

```text
┌──────────────────────────────┐
│ 浏览器                          │
│                                │
│  旅客端  localhost:3000         │
│  检票端  localhost:3001         │
│  后台    localhost:8001         │
└──────────────┬─────────────────┘
               │ HTTP / WebSocket
               │
┌──────────────▼─────────────────┐
│ ticket/server.js :3000         │
│ - 静态前台页面                  │
│ - 注册登录 / 城市车站车次票额     │
│ - 订单 / 退票 / 改签 / 发票      │
│ - 动态二维码                     │
│ - 后台 API                       │
│ - WebSocket /ws                  │
│ - 读写 ticket/data/db.json       │
└──────────────┬─────────────────┘
               │ 同一份 db.json
┌──────────────▼─────────────────┐
│ check/checkin-server.js :3001  │
│ - 静态检票页面                  │
│ - 车站 / 车次 / 开检信息         │
│ - 扫码 / 身份证 / 人脸验证       │
│ - 一人一闸                       │
│ - 检票事件上报                   │
│ - 设备状态查询                   │
└──────────────┬─────────────────┘
               │
┌──────────────▼─────────────────┐
│ admin Vue 3 :8001              │
│ - Vite 开发服务                 │
│ - /api 代理到 127.0.0.1:3000    │
│ - Naive UI 管理页面             │
└────────────────────────────────┘
```

### 3.1 数据流

1. 后台维护城市、车站、车次、票额。
2. 前台读取配置、城市、车站、车次。
3. 旅客录入乘车人本人人脸。
4. 旅客下单，系统生成待支付订单；旅客在倒计时内完成模拟支付后转为已支付。
5. 前台显示动态二维码。
6. 检票端扫码，向后端校验订单、车次、开检时间、乘车人。
7. 人脸比对通过后核销单个乘车人。
8. 主服务写 `db.json`，文件监听触发快照刷新。
9. WebSocket 把最新快照推给后台。
10. 后台实时监控与日志同步更新。

### 3.2 为什么要拆 ticket / check

- 检票端可以独立部署在闸机电脑上。
- 检票端不依赖主服务在线也能读取 `db.json`。
- 两端共享同一数据文件，订单和核销状态实时一致。
- 后续可以分别打包成两个可执行程序。

---

## 4. 目录结构

```text
ticket/
├─ admin/                        后台管理 UI
│  ├─ src/
│  │  ├─ api/ticket/index.ts     票务 API 封装
│  │  ├─ router/modules/ticket.ts 后台菜单与权限
│  │  ├─ store/modules/user.ts   登录用户与权限
│  │  ├─ router/guards.ts        路由守卫 / 登录失效处理
│  │  └─ views/ticket/
│  │     ├─ overview/            总览
│  │     ├─ cities/              城市管理
│  │     ├─ stations/            车站管理
│  │     ├─ trains/              车次管理
│  │     ├─ tickets/             车票管理
│  │     ├─ checkin/             检票管理
│  │     ├─ orders/              订单管理
│  │     ├─ settings/            系统设置
│  │     ├─ logs/                日志管理
│  │     └─ about/               关于
│  ├─ vite.config.mts            读取 config.toml 的 admin 端口
│  └─ dist/                      构建产物
│
├─ check/                        独立检票端
│  ├─ checkin-server.js          检票服务
│  ├─ public-checkin/
│  │  ├─ index.html              检票页面
│  │  ├─ app.js                  扫码 / 人脸 / 设备逻辑
│  │  ├─ style.css               横竖屏 / 扫码小窗 / 页脚
│  │  ├─ face-api.js             人脸模型加载
│  │  ├─ jsqr.js                 二维码解析
│  │  └─ face-models/            人脸模型文件
│  ├─ package.json               依赖 qrcode
│  └─ node_modules/
│
├─ shared/
│  ├─ config.js                  TOML 读取
│  └─ ticket-qr.js               二维码签名 / 校验
│
├─ ticket/                       前台 + 主服务
│  ├─ server.js                  主服务
│  ├─ db.js                      数据读写 / 默认值 / 迁移
│  ├─ data/db.json               运行数据
│  ├─ public/
│  │  ├─ index.html              前台页面
│  │  ├─ app.js                  前台交互
│  │  ├─ style.css               前台样式
│  │  └─ ...                     静态资源
│  ├─ hcaptcha.config.json       hCaptcha 配置
│  └─ node_modules/
│
├─ backup-20261114/              拆分前源码备份
├─ config.toml                   端口与共享配置
├─ manage.ps1                    启停脚本
├─ manage.bat                    双击入口
├─ README.md                     本文档
└─ CHANGELOG.md                  变更记录
```

### 4.1 文件职责

| 文件 | 职责 |
| --- | --- |
| `ticket/server.js` | 主 HTTP 服务、静态资源、业务 API、WebSocket |
| `ticket/db.js` | 数据库初始化、默认值、旧数据迁移、原子写入 |
| `ticket/public/app.js` | 前台查票、购票、订单、乘车人、发票、公告 |
| `check/checkin-server.js` | 检票静态资源、检票 API、事件上报 |
| `check/public-checkin/app.js` | 扫码、人脸、活体、设备状态、维修遮罩 |
| `shared/config.js` | 读取 `config.toml` |
| `shared/ticket-qr.js` | 生成和校验签名二维码 |
| `admin/src/router/modules/ticket.ts` | 后台菜单顺序与权限 |
| `admin/src/api/ticket/index.ts` | 后台所有票务接口 |
| `admin/vite.config.mts` | 端口读取、API 代理 |

---

## 5. 运行环境与依赖

### 5.1 必需环境

| 项目 | 要求 |
| --- | --- |
| 操作系统 | Windows 10/11 |
| Node.js | 18 及以上，当前验证版本 22 |
| 浏览器 | Chrome / Edge，支持摄像头与 WebSocket |
| pnpm | 后台开发与构建 |
| 网络 | hCaptcha 与高德/短信等外部能力需要联网 |

### 5.2 依赖说明

`ticket/` 主要依赖：

- `ws`：WebSocket
- `qrcode`：二维码 PNG
- `pdfkit`：电子发票 PDF

`check/` 主要依赖：

- `qrcode`：检票端二维码生成

`admin/` 主要依赖：

- Vue 3
- Naive UI
- Vite
- Alova

### 5.3 首次安装

```powershell
cd C:\Users\zhaoyuxing\Desktop\ticket\ticket
npm install

cd C:\Users\zhaoyuxing\Desktop\ticket\check
npm install

cd C:\Users\zhaoyuxing\Desktop\ticket\admin
pnpm install
```

---

## 6. 快速启动

### 6.1 一键脚本

双击根目录 `manage.bat`，按菜单选择：

```text
1  启动全部服务
2  停止全部服务
3  查看运行状态
```

也可以直接执行：

```powershell
cd C:\Users\zhaoyuxing\Desktop\ticket
.\manage.ps1 start
.\manage.ps1 stop
.\manage.ps1 status
```

### 6.2 手动启动

分别开三个终端：

```powershell
# 主服务 + 前台
cd C:\Users\zhaoyuxing\Desktop\ticket\ticket
node server.js

# 检票服务
cd C:\Users\zhaoyuxing\Desktop\ticket\check
node checkin-server.js

# 后台
cd C:\Users\zhaoyuxing\Desktop\ticket\admin
pnpm dev
```

### 6.3 访问地址

| 端 | 地址 |
| --- | --- |
| 前台 | <http://ticket.localhost:3000> |
| 前台备用 | <http://localhost:3000> |
| 检票 | <http://localhost:3001> |
| 后台 | <http://localhost:8001> |

注意：hCaptcha 不接受 `localhost` 域名，前台会自动跳转到 `ticket.localhost:3000`。

### 6.4 启停脚本行为

`manage.ps1` 包含：

- `services` 列表：名称、端口、工作目录
- `Get-PortPid`：解析 netstat 获取 PID
- `Start-All`：按服务逐个启动
- `Stop-All`：按端口停止进程
- `Show-Status`：输出运行状态

脚本文件必须是 UTF-8 BOM，否则 Windows PowerShell 会把中文当成 ANSI，出现解析错误。

---

## 7. 配置文件说明

根目录 `config.toml`：

```toml
[ticket]
port = 3000
host = "127.0.0.1"

[check]
port = 3001
host = "127.0.0.1"

[admin]
port = 8001
host = "127.0.0.1"

[shared]
qr_secret = "qingjiang-railway-demo-2026"
qr_ttl = 90
```

### 7.1 字段说明

| 字段 | 说明 |
| --- | --- |
| `ticket.port` | 主服务与前台端口 |
| `ticket.host` | 主服务监听地址 |
| `check.port` | 检票服务端口 |
| `check.host` | 检票服务监听地址 |
| `admin.port` | 后台 Vite 端口 |
| `shared.qr_secret` | 动态二维码 HMAC 密钥 |
| `shared.qr_ttl` | 二维码有效期，单位秒 |

### 7.2 修改端口

1. 修改 `config.toml`。
2. 停止全部服务。
3. 重新启动全部服务。
4. 后台 Vite 会在启动时读取 `[admin] port`。

注意：

- 三个端口不能重复。
- 修改后浏览器地址也要相应变化。
- 后台 `/api` 代理会读取 `[ticket] port`，`manage.ps1` 会读取三个端口，前台入口链接通过 `/api/runtime-config` 自动生成；通常只需改 `config.toml` 并重启服务。
- `qr_secret` 在 `ticket` 和 `check` 必须一致，否则检票端会报签名无效。

---
## 8. 默认账号与权限

### 8.1 后台账号

登录接口：`POST /api/accounts/login`

```json
{
  "username": "admin",
  "password": "123456"
}
```

返回重点字段：

```json
{
  "code": 200,
  "result": {
    "token": "后台 token",
    "account": {
      "username": "admin",
      "role": "admin",
      "permissions": []
    }
  }
}
```

后台请求头：

```text
x-token: <token>
```

### 8.2 账号列表

| 用户名 | 密码 | 角色 | 名称 |
| --- | --- | --- | --- |
| `admin` | `123456` | admin | 系统管理员 |
| `operator` | `123456` | operator | 购票员 |
| `inspector` | `123456` | inspector | 检票员 |
| `pwdtest` | `123456` | operator | 测试账号，可能被数据重置清除 |

### 8.3 权限矩阵

| 权限值 | 菜单 | admin | operator | inspector |
| --- | --- | --- | --- | --- |
| `ticket_overview` | 总览 | 是 | 是 | 是 |
| `ticket_cities` | 城市管理 | 是 | 否 | 否 |
| `ticket_stations` | 车站管理 | 是 | 否 | 否 |
| `ticket_trains` | 车次管理 | 是 | 否 | 否 |
| `ticket_tickets` | 车票管理 | 是 | 否 | 否 |
| `ticket_sales` | 售票窗口 | 是 | 是 | 否 |
| `ticket_supplement` | 补票管理 | 是 | 否 | 否 |
| `ticket_checkin` | 检票管理 | 是 | 否 | 是 |
| `ticket_orders` | 订单管理 | 是 | 是 | 是 |
| `ticket_orders_manage` | 订单处理 | 是 | 是 | 否 |
| `ticket_orders_delete` | 删除订单 | 是 | 否 | 否 |
| `ticket_invoices` | 发票管理 | 是 | 否 | 否 |
| `ticket_settings` | 系统设置 | 是 | 否 | 否 |
| `ticket_logs` | 日志管理 | 是 | 否 | 否 |
| `ticket_alerts` | 告警中心 | 是 | 否 | 是 |
| `ticket_reports` | 运营报表 | 是 | 否 | 否 |
| `ticket_about` | 关于 | 是 | 是 | 是 |

### 8.4 账号行为差异

管理员：

- 可以维护全部基础数据。
- 可以新增、编辑、停用检票设备。
- 可以查看日志。
- 可以重置演示数据。
- 可以修改系统设置。

购票员：

- 可以查看总览、售票窗口、订单和退票。
- 可以打印车票，打印操作会写入审计日志。
- 不能补票、开发票、删除订单。
- 不能进入检票管理、设备管理、系统设置、日志。

检票员：

- 可以查看总览。
- 可以进入检票管理。
- 可以查看实时监控、订单检票进度。
- 可以执行人工放行、标记异常。
- 不能修改检票设备。
- 不能管理基础数据。

### 8.5 权限刷新

后台每次进入会请求：

```text
GET /api/accounts/me
```

因此：

- 服务端权限调整后，刷新页面即可生效。
- 角色变更后建议重新登录。
- 服务端重启会使旧 token 失效，后台会自动回登录页。

### 8.6 密码规则

- 后台账号密码使用 SHA-256 + 固定盐 `::qingjiang`。
- 数据库不保存明文密码。
- 演示密码仅用于本地演示，正式部署必须修改。
- 数据重置需要当前管理员密码再次确认。

---

## 9. 数据模型

数据库文件：`ticket/data/db.json`

顶层结构：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `version` | number | 数据版本 |
| `settings` | object | 系统设置 |
| `accounts` | array | 后台账号 |
| `users` | array | 前台旅客账号 |
| `cities` | array | 城市 |
| `stations` | array | 车站 |
| `trains` | array | 车次 |
| `quotas` | array | 每日票额 |
| `passengers` | array | 常用乘车人 |
| `orders` | array | 订单 |
| `devices` | array | 检票设备 |
| `logs` | array | 审计日志 |
| `checkinEvents` | array | 检票事件 |

### 9.1 settings

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `siteName` | `清江铁路12306` | 站点名称 |
| `subtitle` | `网上购票系统` | 副标题 |
| `announcement` | 欢迎语 | 公告内容 |
| `maxPassengersPerOrder` | `5` | 单笔最大乘客数 |
| `presaleDays` | `30` | 预售天数 |
| `paymentTimeoutMinutes` | `15` | 待支付订单超时分钟数（1~120） |
| `refundEnabled` | `true` | 是否允许在线退票 |
| `servicePhone` | `12306` | 客服电话 |
| `refundRules` | 三档规则 | 退票手续费阶梯 |
| `changeFeeRate` | `0.05` | 改签费率 |
| `childDiscount` | `0.5` | 儿童票折扣 |
| `checkinMinutes` | `30` | 开检提前分钟 |
| `entryWelcomeText` | `欢迎光临{station}` | 入站欢迎语 |
| `exitWelcomeText` | 欢送语 | 出站欢送语 |
| `announcementEnabled` | `true` | 公告总开关 |
| `announcementShowMode` | `top` | `top` 或 `modal` |
| `announcementOnStart` | `true` | 启动时是否显示 |
| `announcementRepeat` | `false` | 是否每次进入都弹 |
| `footerCopyright` | `© 2026 清江铁路` | 版权信息 |
| `footerIcp` | 空 | ICP 备案号 |
| `footerPolice` | 空 | 公安备案名称 |
| `footerPoliceNo` | 空 | 公安备案编号 |
| `maintenanceText` | 此闸机正在维修 | 维修提示 |
| `liveDetection` | `true` | 活体检测开关 |
| `showScanPreview` | `false` | 扫码小窗默认值 |
| `ticketQrRefreshSeconds` | `30` | 二维码刷新秒数 |

退票规则默认：

| 距发车时间 | 费率 |
| --- | --- |
| ≥ 24 小时 | 5% |
| ≥ 2 小时 | 10% |
| ≥ 0 小时 | 20% |

### 9.2 accounts

| 字段 | 说明 |
| --- | --- |
| `id` | 账号 ID |
| `username` | 登录名 |
| `name` | 显示名 |
| `role` | `admin` / `operator` / `inspector` |
| `password` | 哈希后的密码 |
| `phone` | 手机号 |
| `enabled` | 是否启用 |
| `createdAt` | 创建时间 |

### 9.3 users

| 字段 | 说明 |
| --- | --- |
| `id` | 旅客 ID |
| `phone` | 手机号 |
| `name` | 姓名 |
| `password` | 哈希密码 |
| `faceDescriptor` | 账号本人人脸，购票已不再强制 |
| `faceEnrolledAt` | 人脸录入时间 |
| `createdAt` | 注册时间 |

### 9.4 cities

| 字段 | 说明 |
| --- | --- |
| `id` | 城市 ID |
| `name` | 城市名 |
| `pinyin` | 拼音，用于排序 |
| `createdAt` | 创建时间 |

### 9.5 stations

| 字段 | 说明 |
| --- | --- |
| `id` | 车站 ID |
| `name` | 车站名 |
| `code` | 电报码 |
| `cityId` | 所属城市 |
| `status` | `normal` / `maintenance` 等 |
| `gateCount` | 检票口数 |
| `platformCount` | 站台数 |
| `address` | 地址 |
| `createdAt` | 创建时间 |

### 9.6 trains

| 字段 | 说明 |
| --- | --- |
| `id` | 车次 ID |
| `trainNo` | 车次号 |
| `fromStationId` | 出发站 |
| `toStationId` | 到达站 |
| `departTime` | 出发时间 `HH:mm` |
| `arriveTime` | 到达时间 `HH:mm` |
| `gates` | 检票口数组，最多 8 个 |
| `gate` | 检票口展示文本，多个用 ` / ` 连接 |
| `status` | 旧版全局运行状态，启动时会迁移到当天 `runtimeByDate`，新逻辑不再使用 |
| `delayMinutes` | 旧版全局晚点分钟，迁移后归零 |
| `tempGates` | 旧版全局临时检票口，迁移后清空 |
| `tempPlatform` | 旧版全局临时站台，迁移后清空 |
| `runtimeNote` | 旧版全局运行公告，迁移后清空 |
| `runtimeByDate` | 按日期运行调整，键为 `YYYY-MM-DD`，值为状态、晚点、临时检票口/站台和公告 |
| `platform` | 站台 |
| `landmarkColor` | 地标颜色 |
| `seats` | 席别数组，含类型、价格、票额 |

### 9.7 quotas

| 字段 | 说明 |
| --- | --- |
| `trainId` | 车次 |
| `date` | 日期 |
| `seatType` | 席别 |
| `total` | 当日票额 |

### 9.8 passengers

| 字段 | 说明 |
| --- | --- |
| `id` | 乘车人 ID |
| `userId` | 所属旅客账号 |
| `name` | 姓名 |
| `idType` | 证件类型 |
| `idNo` | 证件号 |
| `phone` | 手机号 |
| `type` | 成人 / 儿童 |
| `faceDescriptor` | 128 维本人人脸特征 |
| `faceEnrolledAt` | 人脸录入时间 |
| `createdAt` | 创建时间 |

### 9.9 orders

| 字段 | 说明 |
| --- | --- |
| `id` | 订单 ID |
| `orderNo` | 订单号 |
| `userId` | 所属旅客 |
| `trainId` | 车次 ID |
| `trainNo` | 车次号 |
| `date` | 乘车日期 |
| `seatType` | 席别 |
| `price` | 单价 |
| `totalPrice` | 总价 |
| `status` | `paid` / `refunded` 等 |
| `contactPhone` | 联系人手机 |
| `passengers` | 乘车人数组 |
| `checkedIn` | 整单是否已检 |
| `invoiceNo` | 发票号 |
| `invoiceStatus` | 发票状态：`issued` 已开具 / `redFlushed` 已红冲 |
| `invoiceTitle` | 发票抬头 |
| `invoiceIssuedAt` | 开票时间 |
| `invoiceRedFlushedAt` | 红冲时间 |
| `lostReported` / `voided` | 原票已挂失、已作废 |
| `source` | 来源，如 `web` |
| `createdAt` | 创建时间 |

订单内单个乘车人：

| 字段 | 说明 |
| --- | --- |
| `name` | 姓名 |
| `idNo` | 证件号 |
| `type` | 成人 / 儿童 |
| `price` | 实付价格 |
| `faceDescriptor` | 下单时的人脸快照 |
| `checkedIn` | 是否已核销 |
| `checkedInAt` | 核销时间 |
| `checkinMode` | 核销方式 |
| `exception` | 异常标记 |
| `reviewedBy` | 人工复核人 |

### 9.10 devices

| 字段 | 说明 |
| --- | --- |
| `id` | 设备 ID |
| `stationId` | 所属车站 |
| `gateNo` | 检票机编号 |
| `name` | 设备名称 |
| `gateType` | `checkin` / `entry` / `exit` |
| `status` | `normal` / `maintenance` / `disabled` |
| `maintenanceText` | 该设备维修文案 |
| `lastSeenAt` | 最后心跳时间；10 秒心跳，30 秒未上报判离线 |
| `configVersion` | 远程配置版本，每次修改递增 |
| `configUpdatedAt` | 配置下发时间 |
| `configAckVersion` | 闸机已确认的配置版本 |
| `configAckedAt` | 闸机确认时间 |
| `configError` | 配置应用失败原因 |
| `remoteConfig` | 远程配置：模式、摄像头、扫码小窗、欢迎语 |
| `createdAt` | 创建时间 |
| `updatedAt` | 更新时间 |

### 9.11 logs

| 字段 | 说明 |
| --- | --- |
| `id` | 日志 ID |
| `at` | 时间 |
| `action` | 动作，如 `device.update` |
| `actor` | 操作人 |
| `role` | 角色 |
| `target` | 操作对象 |
| `detail` | 详情 |
| `level` | `info` / `warn` / `error` |

日志最多保留 2000 条。

### 9.12 checkinEvents

| 字段 | 说明 |
| --- | --- |
| `id` | 事件 ID |
| `at` | 时间 |
| `stationId` | 车站 |
| `gateNo` | 闸机编号 |
| `type` | 事件类型 |
| `level` | 级别 |
| `detail` | 详情 |

事件最多保留 500 条。

---
## 10. 前台购票端

地址：<http://ticket.localhost:3000>

### 10.1 顶部导航

前台所有主要入口都在顶部导航：

- 车票预订
- 我的订单
- 乘车人
- 电子发票
- 检票入口
- 管理后台
- 登录 / 注册

窄屏时导航自动换行，不横向滚动。

### 10.2 登录与注册

- 未登录可以查询车次。
- 点击席别购票、下单、查订单、开发票时要求登录。
- 登录、注册、查询车票、订单查询前需要 hCaptcha。
- 前端加载官方 hCaptcha。
- 后端调用 hCaptcha siteverify。
- 验证通过后签发一次性 `X-Captcha` 令牌。
- 令牌约 20 分钟有效。
- 用户登录态约 12 小时有效。

hCaptcha 配置：

- Sitekey 和 Secret 在 `ticket/hcaptcha.config.json`。
- Secret 只在服务端使用。
- 也支持环境变量 `HCAPTCHA_SITE_KEY`、`HCAPTCHA_SECRET`。
- 没有配置时验证接口返回 503。
- 不支持本地图形验证码兜底。

### 10.3 车票查询

查询条件：

- 出发城市
- 到达城市
- 出发日期

查询结果：

- 车次号
- 出发站 / 到达站
- 出发时间 / 到达时间
- 席别
- 价格
- 余票
- 检票口
- 站台
- 地标颜色

同城出行允许出发城市与到达城市相同；没有匹配车次时返回空列表。

### 10.4 查票新标签页

查票不再停留在当前页面，而是打开新标签页：

```text
/?search=1&fromCityId=...&toCityId=...&date=...
```

新页面会：

1. 读取查询参数。
2. 回填出发城市、到达城市、日期。
3. 自动调用 `ensureCaptcha(searchNow)`。
4. 验证通过后自动查询。

### 10.5 购票人脸规则

当前规则：

- 购票不再要求账号本人人脸核验。
- 提交订单不再携带 `faceToken`。
- 购票卡片 `bookingFaceCard` 已隐藏。
- 但下单时必须选择已录入本人人脸的乘车人。
- 后端会校验乘车人 `faceDescriptor` 是否为 128 维。
- 未录入人脸的乘车人不能购票。

这样既减少购票步骤，又保证检票端仍可做人脸比对。

### 10.6 下单

请求体主要字段：

| 字段 | 说明 |
| --- | --- |
| `trainId` | 车次 ID |
| `date` | 乘车日期 |
| `seatType` | 席别 |
| `contactPhone` | 联系人手机 |
| `passengers` | 乘车人数组 |

后端校验：

- 车次存在。
- 日期格式正确且不早于今天。
- 席别存在。
- 乘客数量 1 到 `maxPassengersPerOrder`。
- 日期不超过预售天数。
- 联系人手机号合法。
- 每位乘车人都已录入本人人脸。
- 同一订单证件号不重复。
- 余票充足。

订单创建后状态为 `pending`，系统按后台“支付超时时间”（默认 15 分钟，可在 1~120 分钟之间配置）生成 `paymentDeadline`。前台待支付弹窗显示实时倒计时；超时后订单自动变为 `canceled`（`cancelReason=payment_timeout`）并释放座位。旅客点击“立即支付”后由模拟支付回调把订单改为 `paid`。

### 10.7 订单

订单列表支持：

- 按状态筛选。
- 查看详情。
- 退票。
- 改签。
- 下载电子发票。
- 查看动态二维码。

订单状态：

| 状态 | 说明 |
| --- | --- |
| `pending` | 待支付，按 `paymentDeadline` 计时，超时自动取消 |
| `paid` | 已支付 |
| `refunded` | 已退票 |
| `canceled` | 已取消（含支付超时） |

### 10.8 退票

规则：

- 已检票订单不能退票。
- 系统关闭在线退票时不能退。
- 已发车不能退。
- 按距发车时间阶梯收费。
- 退票后写入 `refundFee`、`refundAmount`、`refundRate`、`refundedAt`。

### 10.9 改签

改签会校验：

- 新日期。
- 新车次。
- 新席别。
- 余票。
- 手续费。

改签成功后更新订单车次、日期、席别和价格信息。

### 10.10 电子发票

- 使用 `pdfkit` 生成 PDF，发票接口需要登录。
- 前台“发票”页对已支付订单提供“申请开票”，申请后写入 `invoiceNo`、`invoiceStatus=issued`、`invoiceIssuedAt`、发票抬头；随后可下载 PDF。
- 发票号写入订单，格式为 `FP + 日期 + 订单号后 8 位`。
- 已开具发票的订单退票时，系统自动把发票标记为 `invoiceStatus=redFlushed` 并记录 `invoiceRedFlushedAt`、`invoiceRedFlushReason`；已红冲发票不能再次下载。
- 后台“发票管理”可查看已开具 / 已红冲发票，并对已退票订单手动红冲。
- 下载文件为 PDF。

### 10.11 乘车人

每位旅客只能看到自己的乘车人。

支持：

- 新增
- 编辑
- 删除
- 人脸录入
- 人脸状态显示

新增或编辑乘车人时：

- 必须采集本人人脸。
- 人脸特征保存为 128 维数组。
- 前端不展示人脸原始特征，只展示是否已录入。

### 10.12 动态二维码

- 成功页显示二维码。
- 订单详情显示二维码。
- 图片地址带 `data-order`。
- 每 30 秒刷新一次可见二维码。
- 刷新间隔可由后台 `ticketQrRefreshSeconds` 调整。
- 二维码 URL 带时间戳，避免浏览器缓存。

### 10.13 公告

公告策略：

| 设置 | 行为 |
| --- | --- |
| `announcementEnabled=false` | 完全不显示 |
| `announcementShowMode=top` | 顶部横条 |
| `announcementShowMode=modal` | 弹窗 |
| `announcementOnStart=true` | 启动时显示 |
| `announcementRepeat=true` | 每次进入都弹 |
| `announcementRepeat=false` | 本地记录已看过，不重复弹 |

前台使用 `localStorage` 记录已看过状态。

### 10.14 页脚

页脚显示：

- 版权信息
- ICP 备案号
- 公安备案名称与编号
- 客服电话

全部来自后台系统设置。

### 10.15 横竖屏

前台支持：

- 横屏
- 竖屏
- 窄屏导航换行
- 竖屏页脚压缩

---

## 11. 后台管理端

地址：<http://localhost:8001>

### 11.1 登录

- 登录接口 `/api/accounts/login`
- 请求头使用 `x-token`
- 登录成功保存 token 和当前用户
- 每次进入刷新 `/api/accounts/me`
- 服务端重启后旧 token 失效，自动回登录页

### 11.2 菜单顺序

| 排序 | 菜单 | 路径 | 权限 |
| --- | --- | --- | --- |
| 1 | 总览 | `/overview` | `ticket_overview` |
| 1.5 | 运营报表 | `/reports` | `ticket_reports` |
| 2 | 城市管理 | `/cities` | `ticket_cities` |
| 3 | 车站管理 | `/stations` | `ticket_stations` |
| 4 | 车次管理 | `/trains` | `ticket_trains` |
| 5 | 车票管理 | `/tickets` | `ticket_tickets` |
| 6 | 检票管理 | `/checkin` | `ticket_checkin` |
| 7 | 订单管理 | `/orders` | `ticket_orders` |
| 8.5 | 告警中心 | `/alerts` | `ticket_alerts` |
| 9 | 系统设置 | `/settings` | `ticket_settings` |
| 10 | 日志管理 | `/logs` | `ticket_logs` |
| 11 | 关于 | `/about` | `ticket_about` |

### 11.3 总览

显示：

- 开通城市数
- 车站总数
- 在售车次
- 累计订单
- 票额合计
- 已售车票
- 余票
- 售票率
- 票额明细
- 最新订单

### 11.4 城市管理

支持：

- 城市列表
- 新增城市
- 编辑城市
- 删除城市
- 显示该城市车站数量

城市按拼音排序。

### 11.5 车站管理

字段：

- 车站名
- 所属城市
- 电报码
- 车站状态
- 检票口数
- 站台数
- 地址

操作列：

```text
编辑 / 更多 ⌄
```

删除收进「更多」下拉菜单，减少误点。

打开新增或编辑弹窗时会刷新城市列表，因此新建城市后无需刷新页面即可选择。

### 11.6 车次管理

支持：

- 新增车次
- 编辑车次
- 删除车次
- 配置出发站、到达站、时间
- 配置检票口（支持逗号分隔多个，最多 8 个）、站台、地标颜色
- 按日期设置运行状态：正常 / 晚点 / 停运 / 暂停检票，晚点可填 1-240 分钟，只影响所选日期
- 按日期设置临时检票口、临时站台和运行公告，留空表示不修改；保存“正常”并清空即恢复当天计划
- 配置席别和价格

### 11.7 车票管理

支持：

- 按日期查看票额
- 修改每日票额
- 查看已售、余票
- 查看席别票额明细

### 11.8 检票管理

检票管理包含四个标签页。

#### 11.8.1 实时监控

显示：

- 今日已支付订单数
- 乘客总数
- 已核销人数
- 核销率
- 设备在线情况
- WebSocket 连接状态
- 订单检票进度
- 检票事件时间线
- 最近日志

数据每 15 秒轮询一次，同时接受 WebSocket 推送。

WebSocket 地址由后端快照返回：

```text
ws://localhost:3000/ws
```

#### 11.8.2 人工复核

对指定订单和证件号执行：

- 放行
- 标记异常

放行会：

- 把该乘车人标记为已检。
- 写入核销时间。
- 写入核销方式 `manual-review`。
- 写入复核人。
- 清除异常标记。
- 全部乘车人都已检时，整单标记已检。

标记异常会：

- 写入 `exception`。
- 写日志。
- 广播监控刷新。

#### 11.8.3 检票设备

支持：

- 新增设备
- 编辑设备
- 修改状态
- 修改维修文案
- 查看最后心跳

设备状态：

| 状态 | 检票端表现 |
| --- | --- |
| `normal` | 正常检票 |
| `maintenance` | 显示维修遮罩，停止摄像头 |
| `disabled` | 显示“本机已停用”，停止摄像头 |

只有 admin 可以新增和修改设备。

#### 11.8.4 车站大屏

支持：

- 选择车站
- 显示站名与地址
- 实时时钟
- 今日车次
- 出发/到达
- 检票口
- 站台
- 已检票人数
- 全屏预览

大屏使用 iframe 生成独立 HTML，时钟在 iframe 内自行刷新，避免整块 iframe 每秒重载。

### 11.9 订单管理

支持：

- 订单列表
- 按状态筛选
- 查看详情
- 代客购票
- 退票
- 删除订单
- 导出/查看发票

操作受权限控制：

- operator 可购票、退票。
- 删除仅 admin。

### 11.10 系统设置

包含：

#### 基础设置

- 系统名称
- 副标题
- 公告
- 单笔最大乘客数
- 预售天数
- 客服电话
- 退票开关
- 退票规则
- 改签费率
- 儿童票折扣

#### 公告策略

- 公告开关
- 顶部 / 弹窗
- 启动时显示
- 重复弹窗

#### 版权备案

- 版权信息
- ICP 备案号
- 公安备案名称
- 公安备案编号

#### 检票终端

- 维修文案
- 活体检测
- 默认隐藏扫码画面
- 二维码刷新秒数

#### 后台账号

- 新增账号
- 修改密码
- 启用/停用
- 删除账号

#### 数据重置

- 需要当前管理员密码
- 清空演示数据
- 清空后需要重新登录后台

### 11.11 日志管理

支持：

- 分页
- 按关键字过滤
- 按级别过滤
- 查看动作、操作人、角色、对象、详情

只有 admin 可以访问。

### 11.12 关于

显示系统信息、版本、技术栈和说明。

---

## 12. 检票终端

地址：<http://localhost:3001>

### 12.1 首次设置

第一次打开时强制弹出设置窗口，且不可关闭。

必须设置：

- 所属车站
- 闸机类型
- 检票模式
- 人脸摄像头
- 扫码摄像头
- 检票机编号

保存后写入浏览器本地存储：

| 键 | 说明 |
| --- | --- |
| `qj_station_id` | 车站 ID |
| `qj_gate_type` | 闸机类型 |
| `qj_gate_no` | 检票机编号 |
| `qj_checkin_mode` | 检票模式 |
| `qj_face_camera` | 人脸摄像头 |
| `qj_scan_camera` | 扫码摄像头 |
| `qj_device_configured` | 是否完成首次设置 |
| `qj_show_scan_preview` | 是否显示扫码小窗 |
| `qj_device_disabled` | 旧版本本机停用键，已废弃并自动清零 |

### 12.2 闸机类型

| 类型 | 说明 |
| --- | --- |
| `checkin` | 检票闸机 |
| `entry` | 入站闸机 |
| `exit` | 出站闸机 |

入站和出站闸机显示欢迎语，不执行车票核销。

入站闸机只校验车票是否为「当日、已支付」：符合条件的车票即可放行，不限制是否本站出发、不要求车次仍在车次库中，并允许同一张票重复进入，每次记录进站时间、闸机编号与进站次数。出站闸机仍要求车次存在且到达站为本站。检票闸机维持单次核销，已检票订单或乘车人再次检票会被拒绝。

### 12.3 检票模式

| 模式 | 说明 |
| --- | --- |
| `auto` | 自动检票 |
| `manual` | 人工检票 |

### 12.4 开检逻辑

- 后台设置 `checkinMinutes`。
- 默认开车前 30 分钟开始检票。
- 检票端只显示开检车次。
- 开车前 1 小时内购票会提示时间临近。
- 开检后自动开启摄像头。

### 12.5 扫码

- 使用 `jsQR`。
- 支持 `attemptBoth`。
- 同一个码 3 秒内不重复处理。
- 支持动态签名二维码。
- 兼容旧格式 `QJ|订单号`。
- 支持直接输入订单号。

### 12.6 人脸

- 加载 `face-api.js` 模型。
- 检测人脸。
- 计算 128 维特征。
- 与订单乘车人人脸快照比对。
- 阈值欧氏距离 ≤ 0.58。
- 摄像头不用时自动关闭。
- 开检时自动开启。
- 旅客不需要手动点击开始。

### 12.7 活体检测

- 由后台 `liveDetection` 控制。
- 计算眼部开合比。
- 要求检测到一次眨眼。
- 低阈值 0.18，高阈值 0.24。
- 15 秒未完成则兜底放行，避免卡死。

### 12.7.1 人脸与车票同时采集

检票闸机开检后：

- 人脸摄像头和扫码摄像头同时工作。
- 人脸在后台持续采样，达到 3 帧并完成活体检测后标记为「人脸已就绪」。
- 扫码、IC 卡或手输证件号读取到车票后，服务器先校验车票；人脸已就绪则直接核销，否则等待人脸补齐后再核销。
- 每次检票成功后清空人脸样本，下一位旅客必须重新采集，不复用上一位的人脸。
- IC 卡使用键盘模拟型读卡器（USB 免驱、刷卡后自动输入卡号并回车），卡内写入身份证号或订单号即可识别；检票、入站、出站通用。

### 12.8 扫码画面

默认：

- 所有闸机类型都不显示扫码小窗。
- 入站、出站闸机不显示扫码小窗。
- 入站、出站闸机的设置页不显示「显示扫码画面」开关。
- 入站、出站页面不显示扫码摄像头实时画面；摄像头仍在后台运行用于识别二维码，页面上只保留验证提示和输入框。
- CSS `display:none`。
- 不显示扫码摄像头小窗。

首次升级时会把旧的 `qj_show_scan_preview=1` 重置为 `0`，避免历史设置让扫码画面继续显示。

开启后：

- 写入 `qj_show_scan_preview=1`。
- 仅在检票闸机模式下，`body` 加 `scan-preview-on`。
- 固定右下角小窗。
- 竖屏时会缩小宽度并抬高位置。

说明：入站、出站闸机的主扫码摄像头是通行验证必需区域，不属于“扫码小窗”开关控制范围。

### 12.9 后台停用（等同本机停用）

检票端不再提供「停用此检票机」本地开关，停用和维修统一由后台控制。

后台「检票管理 → 检票设备」把设备状态设为 `disabled`：

- 检票端下一轮询读取到 `disabled`。
- 显示“本机已停用”遮罩。
- 停止摄像头。
- 提示「本机已在后台停用，请联系车站管理员恢复」。
- 后台改回 `normal` 后自动恢复。

旧版本本地键 `qj_device_disabled` 已废弃；检票端启动时会自动清零，不再参与状态判断。

### 12.10 后台维修

检票端每 5 秒请求：

```text
GET /api/checkin/device?stationId=...&gateNo=...
```

后台把设备设为 `maintenance`：

- 显示“此闸机正在维修”遮罩。
- 停止摄像头。
- 使用设备自己的 `maintenanceText`，没有则使用全局维修文案。

后台恢复 `normal` 后，检票端下一轮询自动恢复。

维修遮罩的 `z-index` 低于设置弹窗，设置弹窗仍可正常打开；但停用和维修状态本身只能从后台修改。

### 12.11 一人一闸

- 一次只核销一位乘车人。
- 同一订单多位乘客需要逐个核销。
- 已核销乘客再次扫码会被拒绝。
- 全部核销后整单标记已检。

### 12.12 检票事件

检票端会写入事件：

| 类型 | 说明 |
| --- | --- |
| `verify-failed` | 车票验证失败 |
| `checkin-ok` | 检票成功 |
| `duplicate` | 重复检票 |
| `access-ok` | 入站/出站验证通过 |
| `access-failed` | 入站/出站验证失败 |
| `verify-error` | 服务异常 |

事件通过 `POST /api/checkin_event/report` 上报主服务。

### 12.13 空状态大屏视觉

无列车开检时，只有检票闸机显示车站大屏；入站、出站仍使用通行验证页面。

- 顶部深蓝状态栏：左侧检票机编号和版本号，中间 12306 标识和车站名，右侧实时日期时间与设置入口。
- 页面背景为蓝色渐变。
- 中央白色圆环内展示双柱闸机图标，闸机带红色 X。
- 主文案：`无列车开检` / `No ticket-checking in progress`。
- 等待文案：黄色 `请等待` / `Please wait`。
- 底部叠加城市建筑剪影和列车轮廓。
- 页脚只保留版权、备案信息，缩小显示。
- 无车开检时隐藏人脸摄像头区域。

### 12.14 开检与检票通过

开检后：

- 上方显示人脸摄像头画面。
- 下方显示开检车次和扫码/输入区域。
- 标题显示「请检票」。

检票通过后：

- 底部弹出绿色通行条。
- 黄色圆形行人图标。
- 大字「检票通过」。
- 显示车次，例如 `G1145`。
- 提示「请快速通过闸机」。
- 右侧显示检票口和站台。
- 约 7 秒后自动收起。

### 12.13 页脚与横竖屏

检票端页脚显示：

- 版权
- ICP
- 公安备案

横屏：

- 大屏布局。
- 摄像头、二维码、提示分区显示。

竖屏：

- 单列布局。
- 扫码小窗缩小。
- 页脚压缩。

### 12.15 Windows 检票客户端

适用于 Windows 检票机，源码在 `deploy/check-terminal/QingjiangCheckClient.cs`，打包脚本会自动编译为 `QingjiangCheckClient.exe`。

- 不是内嵌 WebView 的独立浏览器，而是 Edge/Chrome kiosk 启动器。
- 填检票地址后点“启动全屏检票”，用 `--kiosk --app` 打开页面。
- 浏览器退出后每 3 秒 watchdog 自动重启。
- 支持“开机自动启动并进入检票”，写 `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`。
- 关闭窗口最小化到托盘；托盘菜单“退出”才会停止。
- 依赖检票机已安装 Edge 或 Chrome；摄像头需要 HTTPS 或 localhost。
- Android 5 不使用该客户端，直接用现代 Chrome 打开检票页。

---
## 13. 动态电子客票二维码

### 13.1 目标

防止旅客截图后长期复用同一张二维码。

### 13.2 格式

```text
QJ|订单号|秒级时间戳|24位HMAC
```

示例：

```text
QJ|E202611141335264113|1770000000|a1b2c3d4e5f60718293a4b5c
```

### 13.3 签名算法

```text
sig = HMAC-SHA256(qr_secret, `${orderNo}|${ts}`).hex().slice(0, 24)
```

### 13.4 有效期

- 默认 90 秒。
- 由 `config.toml` 的 `qr_ttl` 控制。
- 超时返回「二维码已过期，请刷新电子客票后重试」。
- 时间戳超前超过 30 秒返回「二维码时间异常」。

### 13.5 兼容旧格式

检票端仍接受：

```text
QJ|订单号
```

这是兼容旧数据。正式环境建议关闭旧格式，只接受签名码。

### 13.6 刷新

- 前台成功页和订单详情都会显示二维码。
- 每 30 秒刷新可见图片。
- 后台可调整 `ticketQrRefreshSeconds`，范围 10~180 秒。
- 二维码响应设置 `Cache-Control: no-store`。

### 13.7 安全注意

- `qr_secret` 只存在服务端和 `config.toml`。
- 不要把密钥写进前端 JS。
- 二维码接口当前只校验订单号和状态，不校验登录态；正式环境建议增加短期票据或登录态校验。

---

## 14. WebSocket 实时推送

### 14.1 地址

```text
ws://localhost:3000/ws
```

### 14.2 连接流程

1. 客户端建立连接。
2. 服务端立即发送 `hello`。
3. 服务端立即发送 `snapshot`。
4. 数据变化时广播对应类型。
5. 客户端收到消息后刷新数据。

### 14.3 消息格式

```json
{
  "type": "snapshot",
  "at": "2026-11-14T08:30:00.000Z",
  "payload": {}
}
```

### 14.4 消息类型

| 类型 | 说明 |
| --- | --- |
| `hello` | 连接成功 |
| `snapshot` | 全量快照 |
| `checkin` | 检票变化 |
| `devices` | 设备变化 |
| `reset` | 数据重置 |
| 其他 | 服务端扩展类型 |

### 14.5 快照结构

```json
{
  "today": "2026-11-14",
  "wsUrl": "ws://localhost:3000/ws",
  "devices": [],
  "orders": [],
  "events": [],
  "logs": []
}
```

`orders` 只包含今日已支付订单，最多 60 条。

每条订单包含：

- `orderNo`
- `trainNo`
- `date`
- `seatType`
- `passengers`
- `checkedCount`
- `passengerCount`
- `checkedIn`

### 14.6 文件监听

主服务监听 `db.json`：

```text
fs.watch(DB_PATH)
```

变化后去抖、刷新内存、广播快照。

因此检票端写入 `db.json` 后，后台也能实时看到。

### 14.7 降级

后台实时监控同时每 15 秒轮询一次，WebSocket 断开时仍能更新。

---

## 15. HTTP API 参考

### 15.1 通用约定

- 主服务前缀：`/api`
- 检票服务前缀：`/api/checkin`
- 请求体：JSON
- 响应格式：

```json
{
  "code": 200,
  "result": {},
  "message": "操作成功"
}
```

- 后台认证头：

```text
x-token: <token>
```

- 旅客认证头：

```text
x-token: <token>
```

- 人机验证头：

```text
x-captcha: <captcha token>
```

### 15.2 健康检查

```text
GET /api/health
```

返回：

```json
{
  "status": "up",
  "time": "..."
}
```

### 15.3 数据概览

```text
GET /api/stats?date=YYYY-MM-DD
```

权限：后台账号。

返回：

- 城市数
- 车站数
- 车次数
- 订单数
- 已支付订单数
- 今日订单数
- 车票数
- 今日车票数
- 收入
- 今日收入
- 票额汇总
- 席别汇总
- 车次票额
- 最近订单

### 15.4 城市

#### 城市列表

```text
GET /api/cities
```

权限：后台账号。

#### 新增城市

```text
POST /api/cities
```

请求：

```json
{
  "name": "南京",
  "pinyin": "nanjing"
}
```

#### 编辑城市

```text
PUT /api/cities/:id
```

#### 删除城市

```text
DELETE /api/cities/:id
```

删除前会检查是否仍有关联车站。

### 15.5 车站

#### 车站列表

```text
GET /api/stations?cityId=...
```

权限：后台账号。

#### 新增车站

```text
POST /api/stations
```

请求字段：

- `name`
- `code`
- `cityId`
- `status`
- `gateCount`
- `platformCount`
- `address`

#### 编辑车站

```text
PUT /api/stations/:id
```

#### 删除车站

```text
DELETE /api/stations/:id
```

删除前会检查是否仍有关联车次。

### 15.6 车次

#### 车次列表

```text
GET /api/trains
```

权限：后台账号。

#### 新增车次

```text
POST /api/trains
```

#### 编辑车次

```text
PUT /api/trains/:id
```

#### 删除车次

```text
DELETE /api/trains/:id
```

### 15.7 车票

#### 按日期查看票额

```text
GET /api/tickets?date=YYYY-MM-DD
```

权限：后台账号。

#### 修改票额

```text
PUT /api/tickets/:trainId
```

请求：

```json
{
  "date": "2026-11-14",
  "seatType": "二等座",
  "total": 100
}
```

### 15.8 系统设置

#### 读取设置

```text
GET /api/settings
```

权限：后台账号。

#### 保存设置

```text
PUT /api/settings
```

权限：仅 admin。

可更新字段：

| 字段 | 约束 |
| --- | --- |
| `siteName` | 1~30 字符 |
| `subtitle` | 不超过 30 字符 |
| `announcement` | 不超过 200 字符 |
| `maxPassengersPerOrder` | 1~20 整数 |
| `presaleDays` | 1~90 整数 |
| `paymentTimeoutMinutes` | 1~120 整数 |
| `refundEnabled` | 布尔 |
| `servicePhone` | 不超过 20 字符 |
| `refundRules` | 非空数组 |
| `changeFeeRate` | 0~1 |
| `childDiscount` | 0~1 |
| `checkinMinutes` | 5~120 整数 |
| `entryWelcomeText` | 1~80 字符 |
| `exitWelcomeText` | 1~80 字符 |
| `announcementEnabled` | 布尔 |
| `announcementOnStart` | 布尔 |
| `announcementRepeat` | 布尔 |
| `announcementShowMode` | `top` / `modal` |
| `footerCopyright` | 不超过 60 字符 |
| `footerIcp` | 不超过 80 字符 |
| `footerPolice` | 不超过 80 字符 |
| `footerPoliceNo` | 不超过 30 字符 |
| `maintenanceText` | 1~80 字符 |
| `liveDetection` | 布尔 |
| `showScanPreview` | 布尔 |
| `ticketQrRefreshSeconds` | 10~180 整数 |

### 15.9 hCaptcha

#### 获取配置

```text
GET /api/captcha
```

返回：

```json
{
  "provider": "hcaptcha",
  "siteKey": "...",
  "prompt": "请完成 hCaptcha 人机验证",
  "expiresIn": 1200
}
```

#### 校验

```text
POST /api/captcha/verify
```

请求：

```json
{
  "response": "hcaptcha response token"
}
```

返回：

```json
{
  "token": "capok...",
  "expiresIn": 1200
}
```

### 15.10 动态二维码

```text
GET /api/qrcode/ticket?orderNo=...
```

返回：`image/png`

限制：

- 订单必须存在。
- 订单必须为 `paid`。
- 响应不缓存。

### 15.11 后台账号

#### 当前账号

```text
GET /api/accounts/me
```

权限：任意后台账号。

返回：

- `id`
- `username`
- `name`
- `role`
- `phone`
- `enabled`
- `permissions`

#### 登录

```text
POST /api/accounts/login
```

请求：

```json
{
  "username": "admin",
  "password": "123456"
}
```

#### 账号列表

```text
GET /api/accounts
```

权限：仅 admin。

#### 新增账号

```text
POST /api/accounts
```

#### 修改密码

```text
PUT /api/accounts/:id/password
```

#### 编辑账号

```text
PUT /api/accounts/:id
```

#### 删除账号

```text
DELETE /api/accounts/:id
```

管理员账号至少保留一个。

### 15.12 后台订单

#### 订单列表

```text
GET /api/orders
```

权限：后台账号。

#### 订单详情

```text
GET /api/orders/:id
```

#### 修改订单状态

```text
PUT /api/orders/:id/status
```

#### 删除订单

```text
DELETE /api/orders/:id
```

权限：仅 admin。

### 15.13 旅客接口

#### 注册

```text
POST /api/public/auth/register
```

请求：

```json
{
  "phone": "13800138000",
  "name": "张伟",
  "password": "123456"
}
```

需要 `x-captcha`。

#### 登录

```text
POST /api/public/auth/login
```

请求：

```json
{
  "account": "13800138000",
  "password": "123456"
}
```

需要 `x-captcha`。

#### 当前旅客

```text
GET /api/public/auth/me
```

需要旅客 token。

#### 退出

```text
POST /api/public/auth/logout
```

#### 修改密码

```text
PUT /api/public/auth/password
```

### 15.14 旅客人脸

#### 状态

```text
GET /api/public/face/status
```

返回：

- `enrolled`
- `enrolledAt`
- `required`

#### 录入

```text
POST /api/public/face/enroll
```

请求：

```json
{
  "descriptor": [128 个数字]
}
```

限制：

- 每个账号只能录入一次。
- 已录入后返回错误。

#### 核验

```text
POST /api/public/face/verify
```

请求：

```json
{
  "descriptor": [128 个数字]
}
```

阈值：欧氏距离 ≤ 0.58。

说明：账号本人人脸接口仍保留，但购票流程已不再强制。

### 15.15 旅客公共配置

```text
GET /api/public/settings
```

返回前台需要的公告、页脚、二维码刷新间隔等。

### 15.16 城市与车站

```text
GET /api/public/cities
GET /api/public/stations?cityId=...
```

### 15.17 车票查询

```text
GET /api/public/search?fromCityId=...&toCityId=...&date=YYYY-MM-DD
```

需要 `x-captcha`。

返回：

- 日期
- 出发城市
- 到达城市
- 总数
- 车次列表

### 15.18 常用乘车人

#### 列表

```text
GET /api/public/passengers
```

需要旅客 token，只返回当前账号的乘车人。

#### 新增

```text
POST /api/public/passengers
```

需要 `x-captcha` 和 `faceDescriptor`。

#### 修改

```text
PUT /api/public/passengers/:id
```

#### 删除

```text
DELETE /api/public/passengers/:id
```

### 15.19 旅客订单

#### 创建订单

```text
POST /api/public/orders
```

需要旅客 token 和 `x-captcha`。

#### 订单列表

```text
GET /api/public/orders?status=paid
```

#### 订单详情

```text
GET /api/public/orders/:orderNo
```

#### 退票

```text
POST /api/public/orders/:orderNo/cancel
```

#### 改签

```text
POST /api/public/orders/:orderNo/change
```

#### 电子发票

```text
POST /api/public/orders/:orderNo/invoice   申请开票，body: { title?, taxNo? }
GET  /api/public/orders/:orderNo/invoice   下载发票 PDF
```

后台发票管理：

```text
GET  /api/invoices                         已开具 / 已红冲发票列表，可带 ?status=issued|redFlushed
GET  /api/invoices/:orderNo/pdf            后台下载发票 PDF
POST /api/invoices/:orderNo/red-flush      对已退票订单红冲，body: { reason? }
```

权限：admin、operator（`ticket_invoices`）。

### 15.20 实时监控

```text
GET /api/monitor
```

权限：admin、inspector。

返回 `liveSnapshot`：

- 今日日期
- WebSocket 地址
- 设备
- 今日已支付订单
- 检票事件
- 最近日志

### 15.21 设备

#### 列表

```text
GET /api/devices
```

权限：任意后台账号。

#### 新增

```text
POST /api/devices
```

权限：仅 admin。

请求：

```json
{
  "stationId": "s_xxx",
  "gateNo": "QJ-01",
  "name": "一号检票机",
  "gateType": "checkin",
  "status": "normal",
  "maintenanceText": "此闸机正在维修"
}
```

限制：

- 车站必须存在。
- 同车站下编号不能重复。
- 编号最长 20 字符。

#### 编辑

```text
PUT /api/devices/:id
```

权限：仅 admin。

可修改：

- `name`
- `gateNo`
- `gateType`
- `status`
- `maintenanceText`

### 15.22 人工复核

```text
POST /api/checkin_admin/review
```

权限：admin、inspector。

请求：

```json
{
  "orderNo": "E202611141335264113",
  "idNo": "11010519491231002X",
  "action": "approve",
  "reason": "人工核验通过"
}
```

`action` 取值：

| 值 | 说明 |
| --- | --- |
| `approve` | 放行并核销 |
| `exception` | 标记异常 |

### 15.23 日志

```text
GET /api/logs?page=1&pageSize=30
```

权限：仅 admin。

返回：

- `total`
- `page`
- `pageSize`
- `rows`

`pageSize` 范围 10~100。

### 15.24 检票事件上报

```text
POST /api/checkin_event/report
```

请求：

```json
{
  "secret": "qingjiang-railway-demo-2026",
  "stationId": "s_xxx",
  "gateNo": "QJ-01",
  "type": "checkin-ok",
  "level": "info",
  "detail": "G1145 E2026... 张伟"
}
```

说明：

- `secret` 必须等于 `qr_secret`。
- 事件最多保留 500 条。
- 非 info 事件同时写审计日志。

### 15.25 数据重置

```text
POST /api/admin/reset
```

权限：仅 admin。

请求：

```json
{
  "password": "123456"
}
```

校验当前管理员密码。

成功后：

- 清空演示数据。
- 广播 `reset`。
- 返回「请重新登录后台」。

### 15.26 导入

```text
POST /api/import/:type
```

用于批量导入城市、车站、车次等数据，属于开发辅助能力。

---

## 16. 异常处理与人工复核

### 16.1 常见异常

| 场景 | 表现 | 处理 |
| --- | --- | --- |
| 二维码签名无效 | 检票端拒绝 | 刷新电子客票 |
| 二维码过期 | 提示已过期 | 刷新二维码 |
| 重复检票 | 提示已核销 | 人工复核 |
| 人脸不匹配 | 提示差异值 | 重试或人工复核 |
| 未到开检时间 | 不显示车次 | 等待开检 |
| 已过发车时间 | 拒绝检票 | 人工复核 |
| 设备维修 | 维修遮罩 | 换闸机或后台恢复 |
| 订单不存在 | 404 | 核对订单号 |

### 16.2 人工复核流程

1. 后台进入检票管理。
2. 查看异常事件或订单进度。
3. 输入订单号和证件号。
4. 选择放行、标记异常或取消检票。
5. 放行后写入核销记录。
6. 标记异常后写入异常原因。
7. 取消检票会清除该乘车人的已检状态，订单恢复为未检票，可重新检票。
8. 监控页实时刷新。

### 16.3 复核记录字段

放行：

- `checkedIn=true`
- `checkedInAt`
- `checkinMode=manual-review`
- `reviewedBy`
- `exception=''`

异常：

- `exception=原因`

取消检票：

- `checkedIn=false`
- 清除 `checkedInAt`、`checkinMode`、`reviewedBy`
- 订单 `checkedIn=false`

### 16.4 日志

人工复核会写日志：

| 动作 | 级别 |
| --- | --- |
| `checkin.review.approve` | info |
| `checkin.review.exception` | warn |
| `checkin.review.cancel` | warn |

---

## 17. 日志管理

### 17.1 日志来源

- 后台登录
- 设备新增/修改
- 订单操作
- 人工复核
- 数据重置
- 检票异常事件

### 17.2 日志字段

- 时间
- 动作
- 操作人
- 角色
- 对象
- 详情
- 级别

### 17.3 日志上限

- 最多 2000 条。
- 新日志插入到最前面。

### 17.4 查询

```text
GET /api/logs?page=1&pageSize=30
```

### 17.5 页面

后台「日志管理」支持：

- 分页
- 关键字过滤
- 级别过滤
- 查看详情

---

## 18. 数据重置

### 18.1 入口

后台「系统设置 → 数据重置」。

### 18.2 校验

- 必须是 admin。
- 必须输入当前管理员密码。
- 密码错误返回 403。

### 18.3 结果

- 清空城市、车站、车次、订单、乘车人、设备、日志、事件。
- 恢复默认设置。
- 恢复默认三个后台账号。
- 前台旅客数据清空。
- 广播 `reset`。

### 18.4 重置后

- 后台需要重新登录。
- 前台需要重新注册。
- 检票端本机配置仍保留在浏览器，但设备可能在数据库中被清空；下次心跳会自动重新登记。

---

## 19. UI、页脚与横竖屏

### 19.1 全站页脚

前台、检票端都支持：

- 版权信息
- ICP 备案号
- 公安备案名称
- 公安备案编号
- 客服电话

后台修改后，前台和检票端读取同一份设置。

### 19.2 版权格式

默认：

```text
© 2026 清江铁路
```

后台可自定义，最多 60 字符。

### 19.3 备案格式

- ICP 单独一行。
- 公安备案名称和编号合并显示。
- 小字号、低对比度，放在最下面。

### 19.4 横竖屏

前台：

- 横屏双列查询。
- 竖屏单列。
- 导航换行。

检票端：

- 横屏左右分栏。
- 竖屏单列。
- 扫码小窗缩小。

后台不要求横竖屏适配，按桌面使用。

### 19.5 车站大屏

- 深色背景。
- 大字号站名。
- 实时时钟。
- 今日车次列表。
- 检票口、站台、已检人数。
- 全屏预览。

---

## 20. 本地部署与便携打包

### 20.1 同一台电脑

源码现在全部位于 `source/`，启动脚本也在该目录：

```powershell
cd C:\Users\zhaoyuxing\Desktop\ticket\source
.\manage.ps1 start
```

### 20.2 一键打包 Windows 单文件版

项目根目录只保留一个一键打包入口：

```text
ticket/
  source/            全部未编译源码、依赖和构建辅助文件
  dist/              只保留最终压缩包
  一键打包.cmd       双击运行
```

执行 `一键打包.cmd` 后，脚本会：

1. 检查 Node.js 22 或更高版本。
2. 从 `source/` 复制前台、检票端、后台和依赖到临时目录。
3. 生成内置静态资源映射。
4. 用 esbuild 打包 Node 入口，并生成 SEA blob。
5. 注入 Node 运行时，生成 `QingjiangRailway12306.exe`。
6. 打包成 `dist/qingjiang-railway-12306-win64.zip`。
7. 清空 `dist/` 中旧产物，只保留这一个压缩包。

压缩包内包含：

```text
qingjiang-railway-12306-win64/
  QingjiangRailway12306.exe
  config.toml
  使用说明.txt
```

运行电脑不需要安装 Node.js，目录中不会出现 `.js` 文件。

### 20.3 打包注意事项

- 需要 Node.js 22 或更高版本，SEA 只支持较新的 Node。
- 首次运行打包脚本时，`npx` 可能需要联网下载 esbuild 和 postject。
- `source/admin/dist` 是后台构建产物，打包时复制为 `admin/web`，不要删除。
- 不要把 `source/ticket/data` 打进分发包，脚本不会复制该目录。
- 打包后 `dist` 只保留一个 zip，不再保留解压目录。

### 20.4 Docker

适合正式演示机：

- Node 基础镜像。
- 三个服务同镜像。
- `config.toml` 挂载。
- `data/` 挂载。
- 8001 由静态服务或 Nginx 托管 `admin/dist`。

---

## 21. 常见问题

### 21.1 前台打不开

检查：

- 3000 是否在运行。
- 是否访问 `ticket.localhost:3000`。
- 是否有防火墙拦截。

### 21.2 后台接口 404

检查：

- 3000 是否在运行。
- Vite proxy 是否指向 3000。
- 是否误用了 `/api/api`。

### 21.3 后台登录后菜单不全

原因：token 对应角色权限不足。

处理：

- 退出重新登录。
- 检查 `/api/accounts/me` 返回的 `permissions`。

### 21.4 后台刷新后回登录页

原因：服务端重启导致内存 token 失效。

处理：重新登录。

### 21.5 检票端没有车次

检查：

- 是否完成首次设置。
- 车站是否正确。
- 当天是否有车次。
- 是否已到开检时间。
- `checkinMinutes` 是否合理。

### 21.6 二维码无效

检查：

- 订单是否已支付。
- 是否超过 `qr_ttl`。
- `ticket` 和 `check` 的 `qr_secret` 是否一致。
- 系统时间是否准确。

### 21.7 人脸不匹配

检查：

- 乘车人是否录入本人人脸。
- 光线是否充足。
- 摄像头是否选对。
- 是否戴帽子、口罩。
- 阈值 0.58 是否过严。

### 21.8 摄像头打不开

检查：

- 浏览器是否允许摄像头。
- 是否被其他程序占用。
- 是否使用 HTTPS 或 localhost。
- 设备名是否选对。

### 21.9 扫码小窗不显示

检查：

- 当前闸机类型是否为「检票闸机」；入站、出站不显示扫码小窗。
- 设置中「显示扫码画面」是否打开。
- `qj_show_scan_preview` 是否为 1。
- `body` 是否有 `scan-preview-on`。

### 21.10 设备一直显示停用或维修

检查：

- 后台设备状态是否 `normal`。
- 后台是否把设备设为 `disabled`（本机停用）。
- 后台是否把设备设为 `maintenance`（维修）。
- `stationId` 和 `gateNo` 是否匹配。
- 检票端轮询是否正常。
- `maintenanceText` 是否为空。

说明：检票端已不提供本机停用开关。旧浏览器里的 `qj_device_disabled` 会在启动时自动清零。

### 21.11 后台大屏时钟闪烁

已修复：时钟改为 iframe 内部自刷新。

### 21.12 管理脚本中文乱码

原因：`manage.ps1` 不是 UTF-8 BOM。

处理：重新以 UTF-8 BOM 保存。

### 21.13 `ticket.localhost` 无法解析

PowerShell / .NET 可能无法解析 `*.localhost`，浏览器通常可以。

验证时改用：

```text
http://localhost:3000
```

### 21.14 pnpm junction 失效

目录改名后 `node_modules` 内的 pnpm junction 可能失效。

处理：

```powershell
cd admin
Remove-Item -Recurse -Force node_modules
pnpm install
```

### 21.15 后台构建失败

检查：

- `<n-card>` 是否闭合。
- `useUserStore` 导入路径是否正确。
- 路由文件是否有语法错误。
- 运行 `pnpm exec vite build` 查看完整错误。

---

## 22. 安全边界与已知限制

### 22.1 演示系统定位

- 不连接真实 12306。
- 不接真实支付。
- 不适用于生产售票。
- 数据仅保存在本地 JSON。

### 22.2 已知限制

1. 动态二维码仍兼容未签名旧格式。
2. 二维码接口不校验登录态，仅校验订单号和状态。
3. 检票事件本地写入时未携带 `stationId`、`gateNo`。
4. `checkin-server.js` 的 `MAIN_PORT` 仍未使用。
5. 检票端每 5 秒心跳会写 `db.json`，可能频繁写盘。
6. 后台与检票端共用同一份 `db.json`，不是完全独立数据库。
7. 后台 token 存在内存，重启后失效。
8. 前台账号人脸接口仍保留，但购票不再强制。
9. 二维码密钥为演示密钥，正式环境必须更换。
10. 没有真实支付、短信、实名认证。

### 22.3 正式化建议

- 改用 SQLite / PostgreSQL。
- 使用 JWT 或持久化会话。
- 二维码增加用户态校验。
- 关闭旧未签名二维码。
- 事件补车站和闸机字段。
- 增加 HTTPS。
- 增加接口限流。
- 增加操作审计导出。
- 增加设备离线告警。

---

## 23. 验收清单

### 23.1 服务启动

- [ ] 3000 可访问。
- [ ] 3001 可访问。
- [ ] 8001 可访问。
- [ ] `manage.ps1 status` 三个服务均运行。

### 23.2 后台权限

- [ ] admin 看到 10 个菜单。
- [ ] operator 只看到总览、订单、关于。
- [ ] inspector 看到总览、检票管理、订单、关于。
- [ ] 服务端重启后后台自动回登录页。

### 23.3 前台

- [ ] 能查询车票。
- [ ] 查票打开新标签页。
- [ ] 新页面自动回填条件。
- [ ] 购票不要求账号人脸。
- [ ] 未录入人脸的乘车人不能购票。
- [ ] 订单二维码可刷新。
- [ ] 公告按策略显示。
- [ ] 页脚显示版权备案。
- [ ] 横竖屏正常。

### 23.4 后台

- [ ] 车站新增状态、检票口、站台、地址。
- [ ] 车站操作列是「编辑 / 更多」。
- [ ] 新增城市后无需刷新即可选择。
- [ ] 检票管理四个标签页可切换。
- [ ] 实时监控 WebSocket 连接。
- [ ] 人工复核可放行。
- [ ] 设备可新增、编辑、维修。
- [ ] 车站大屏可全屏。
- [ ] 日志可分页过滤。
- [ ] 系统设置在订单下方。
- [ ] 数据重置校验管理员密码。

### 23.5 检票端

- [ ] 首次打开强制设置。
- [ ] 设置后记住车站和编号。
- [ ] 扫码画面默认隐藏。
- [ ] 开启后右下角小窗。
- [ ] 活体检测能眨眼通过。
- [ ] 后台停用后 5 秒内显示“本机已停用”。
- [ ] 后台维修后 5 秒内显示维修文案。
- [ ] 恢复 normal 后恢复检票。
- [ ] 一人一闸。
- [ ] 横竖屏正常。
- [ ] 页脚版权备案。

### 23.6 数据

- [ ] `db.json` 正常写入。
- [ ] WebSocket 快照订单数正确。
- [ ] 动态二维码可识别。
- [ ] 日志有记录。
- [ ] 重置后数据恢复默认。

---

## 24. 开发约定

### 24.1 修改服务端

- 所有响应使用 `ok()` / `fail()`。
- 所有后台写操作使用 `requireAccountRole()`。
- 所有旅客写操作使用 `requireUser()`。
- 写操作后调用 `saveDB()`。
- 需要实时刷新的写操作调用 `broadcast()`。
- 审计动作调用 `addLog()`。

### 24.2 修改前端

- 接口统一走 `request()` 封装。
- 不在前端保存 `qr_secret`。
- 人脸特征不写日志。
- 新增静态资源要更新版本号。
- 修改样式注意横竖屏。

### 24.3 修改后台

- 新路由加到 `router/modules/ticket.ts`。
- 设置 `sort` 和 `permissions`。
- 新接口加到 `api/ticket/index.ts`。
- 新页面放在 `views/ticket/`。
- 构建前执行 `pnpm exec vite build`。

### 24.4 修改检票端

- 本地存储键统一 `qj_` 前缀。
- 设备状态轮询保持 5 秒。
- 摄像头启停必须尊重维修状态。
- 扫码和活体检测要考虑低性能设备。
- 修改后执行 `node --check public-checkin/app.js`。

### 24.5 提交前检查

```powershell
node --check ticket/server.js
node --check ticket/db.js
node --check check/checkin-server.js
node --check check/public-checkin/app.js
node --check ticket/public/app.js
```

后台：

```powershell
cd admin
pnpm exec vite build
```

---

## 25. 变更记录

详细变更见根目录：

```text
CHANGELOG.md
```

本轮重点：

- 动态二维码
- WebSocket
- 检票管理
- 日志管理
- 人工复核
- 设备停用
- 车站大屏
- 活体检测
- 扫码小窗
- 页脚备案
- 横竖屏
- 项目拆分
- 一键启停
- 配置端口

---

## 附录 A：常用命令

```powershell
# 查看服务状态
.\manage.ps1 status

# 启动
.\manage.ps1 start

# 停止
.\manage.ps1 stop

# 检查端口
netstat -ano | findstr :3000
netstat -ano | findstr :3001
netstat -ano | findstr :8001

# 语法检查
node --check ticket\server.js
node --check check\checkin-server.js

# 后台构建
cd admin
pnpm exec vite build
```

---

## 附录 B：默认演示数据

如果数据库未被重置，通常包含：

- 城市：北京、南京、上海
- 车站：南京南、上海虹桥
- 车次：G1145 等
- 设备：QJ-01 / 南京南 / normal

具体以 `ticket/data/db.json` 为准。

---

## 附录 C：接口认证速查

| 场景 | 请求头 |
| --- | --- |
| 后台接口 | `x-token` |
| 旅客接口 | `x-token` |
| 人机验证接口 | `x-captcha` |
| 检票事件上报 | 请求体 `secret` |
| 动态二维码 | 无 |

---

## 附录 D：端口速查

| 服务 | 端口 | 配置 |
| --- | --- | --- |
| 前台/主服务 | 3000 | `[ticket] port` |
| 检票端 | 3001 | `[check] port` |
| 后台 | 8001 | `[admin] port` |

---

## 附录 E：文件路径速查

| 内容 | 路径 |
| --- | --- |
| 数据库 | `ticket/data/db.json` |
| 主服务 | `ticket/server.js` |
| 前台脚本 | `ticket/public/app.js` |
| 前台样式 | `ticket/public/style.css` |
| 检票服务 | `check/checkin-server.js` |
| 检票脚本 | `check/public-checkin/app.js` |
| 检票样式 | `check/public-checkin/style.css` |
| 后台菜单 | `admin/src/router/modules/ticket.ts` |
| 后台接口 | `admin/src/api/ticket/index.ts` |
| 后台设置页 | `admin/src/views/ticket/settings/index.vue` |
| 后台检票页 | `admin/src/views/ticket/checkin/index.vue` |
| 后台日志页 | `admin/src/views/ticket/logs/index.vue` |
| 后台车站页 | `admin/src/views/ticket/stations/index.vue` |

---

## 附录 F：版本说明

当前文档对应 2026-11-14 版本。

历史版本与本轮变更见 `CHANGELOG.md`。

---

（完）
