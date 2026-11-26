# ios-change-location

accessory-location 的 iPhone USB/iAP2 定位数据传输测试。

执行脚本会自动下载组件包并按 CPU 架构选择对应的 lib 包，解压后从 `lib/` 离线安装依赖。

## 用法

macOS（在仓库根目录执行；仅用于获批测试设备）：

```sh
./ios-change-location/ios-change-location_sh run <测试纬度> <测试经度>   # 完整流程：环境检查 → 下载组件包 → 离线装依赖 → 发送
./ios-change-location/ios-change-location_sh send <测试纬度> <测试经度>  # 只发送（跳过安装步骤）
./ios-change-location/ios-change-location_sh stop                        # 停止发送，恢复手机真实定位
```

Windows PowerShell（在仓库根目录执行；仅用于获批测试设备）：

```powershell
powershell -ExecutionPolicy Bypass -File .\ios-change-location\ios-change-location.ps1 run <测试纬度> <测试经度>
powershell -ExecutionPolicy Bypass -File .\ios-change-location\ios-change-location.ps1 send <测试纬度> <测试经度>
powershell -ExecutionPolicy Bypass -File .\ios-change-location\ios-change-location.ps1 stop
```

## 组件包

| 目录 | 内容 |
| --- | --- |
| `accessory-location/` | iAP2 定位客户端源码、`tools/local_mfi_v3_helper.py` 等脚本 |
| `identity/` | 共享实验性 MFi 身份（`certificate.p7b` / `identity.pk8`） |

平台 lib 包（按 CPU 架构独立打包，脚本自动选择）

## 前提

- macOS：已安装 uv（无则脚本自动安装）
- Windows：需要 Apple Devices 应用（Microsoft Store）或官网版 iTunes 提供 usbmuxd（本地端口 27015）。脚本检测到缺失时自动通过 winget 安装（官网渠道优先、Store 渠道兜底），并启动 Apple Mobile Device Service；无 winget 或安装失败时提示手动安装
- iPhone 通过 USB 连接，已解锁并点击"信任此电脑"
- 发送频次受协议 1Hz 硬上限约束（iAP2 定位订阅中手机端通过 0x8001/0x8002 指定最小间隔，发送端取 `--interval` 与手机端间隔的较大值，最低 1 秒），无法再提高
- **iOS 蓝点不更新排查**：坐标包送达（ack 计数增长）≠ iOS 采纳。蓝点仅在手机端应用活跃请求定位时更新——①获批应用在前台且正在显示地图/蓝点；②定位服务中该应用权限为"使用期间"并开启"精确位置"；③手机保持解锁。日志中大量 `location_stopped` 表示手机端暂停了订阅，暂停期间发送的坐标不会被消费
- **诊断模式**：`sh ios-change-location_sh doctor`（Windows 为 `.\ios-change-location.ps1 doctor`）运行 20 秒 probe 会话（只监听不发坐标），检测 ①MFi 认证 ②手机端是否有获批应用在请求定位（持续请求 / 轮询式 / 从未请求），并给出针对性处理建议——用于确认"获批应用资格/前台状态"问题
- Windows 若连接异常，可重启电脑后重新运行脚本（Apple Devices 安装后通常需重启完成服务注册）

## 说明与限制

本工具用于自有设备或明确获批的测试设备，在受控环境中验证 accessory-location 的数据传输。它会发送 GGA/RMC 模拟坐标；这不是设备的真实 GPS 位置，也不是 Apple 提供的授权定位测试接口，设备或应用可能会暂时收到模拟位置。


设备需解锁并信任连接的电脑。发送阶段持续运行（每轮 900 秒无缝循环），按 `Ctrl+C` 可随时手动停止，累计 30 分钟后自动终止；停止后自动输出结果解读，随后运行 `stop`，断开 USB，并在设备上确认定位已恢复。日志中的坐标包确认只表示传输送达，不表示系统或应用采纳，也不构成任何服务资格证明。
