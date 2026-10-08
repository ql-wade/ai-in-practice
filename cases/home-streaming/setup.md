# 书房到客厅：搭建与配置记录

[返回案例](README.md) · [故障恢复](troubleshooting.md) · [源文覆盖与边界](source-coverage.md)

本文依据 2026-10-01 保存的原方案正文整理。设备状态、版本与成功日志均是历史记录，本次只做文档整理及官方端口核验，没有重新访问设备或现场复测。配置不是最新版本推荐；验收标准不是已通过清单。

## 一、整体拓扑

```text
书房 Windows 11 / RTX 3090
        │ <HOST_ETHERNET_ADAPTER>：<HOST_LAN_IP>
        ▼
家庭路由器 / AP
        )))) 5 GHz Wi-Fi
        ▼
客厅 Z9 Pro：<CLIENT_LAN_IP>
        │
        ▼
Android TV Moonlight → 投影画面和声音
```

视频路径：

```text
游戏/Steam → VDD 1920×1080@60 → Sunshine DDX
→ RTX 3090 NVENC（HEVC，必要时 H.264）
→ Moonlight MediaCodec → 投影仪
```

音频路径：

```text
Windows 默认播放端点
AI Noise-Canceling Speaker (ASUS Utility)
→ Sunshine WASAPI loopback → Opus
→ Moonlight Android TV → 投影仪音频输出
```

Moonlight 主机栏只填写 `<HOST_LAN_IP>`。投影客户端地址、Web UI 端口和其他串流端口都不是这里要填写的主机地址。

## 二、最终配置（历史记录）

### 占位符与替换

公开稿不含真实地址、MAC、Windows 用户名或唯一设备 ID。先在自己的可信家庭网中确认下表取值，再替换配置和命令中的同名占位符；尖括号文本不能原样执行。主机与客户端必须使用不同的对应地址。

| 占位符 | 含义 |
| --- | --- |
| `<HOST_LAN_IP>` | 有线 Windows 主机 IPv4 地址 |
| `<CLIENT_LAN_IP>` | 客厅投影客户端当前固定 IPv4 地址 |
| `<OLD_CLIENT_LAN_IP>` | 已作废的客户端地址，仅供排障对照 |
| `<LAN_GATEWAY>` | 家庭 LAN 网关 |
| `<LAN_SUBNET_CIDR>` | 家庭 LAN 网段的 CIDR 表示，用于规则范围 |
| `<CLIENT_MAC>` | 投影客户端 MAC，用于 ARP 对照与 DHCP 保留 |
| `<WINDOWS_USER>` | 当前 console 会话的 Windows 用户 |
| `<VDD_DEVICE_ID>` | 本机 VDD 输出设备 ID，与 `output_name` 对应 |
| `<VDD_DISPLAY_NAME>` | 本机 VDD 显示名称 |
| `<HOST_ETHERNET_ADAPTER>` | 主机有线网络适配器名称 |

### 主机和会话

| 项目 | 原记录最终值 |
| --- | --- |
| 主机 IP | `<HOST_LAN_IP>/24` |
| 网关 | `<LAN_GATEWAY>` |
| Windows 网络类型 | Private |
| Sunshine 服务 | `SunshineService`，Automatic / Running |
| Sunshine 子进程 | console Session 1 |
| GPU | NVIDIA GeForce RTX 3090 |
| NVIDIA 驱动 | 616.56 |
| Sunshine | v2026.516.143833 |
| 客厅设备 | Z9 Pro，`<CLIENT_LAN_IP>` |
| VDD | VDD by MTT / `<VDD_DISPLAY_NAME>` |
| VDD 设备 ID | `<VDD_DEVICE_ID>` |

原页面记录 SunshineService 自动运行、VDD 和 NVENC 正常，TCP 47989/47990 正在监听，Z9 Pro 已固定地址；最近成功会话出现 `CLIENT CONNECTED`、`Audio capture format` 和 `Opus initialized`。这些是历史观察，不是本次重新测量，也不单独证明画面、声音、输入及长期稳定性全部通过。

### Sunshine 关键项：完整 17 项

下面保留历史配置。根据自己的设备替换 `output_name`，不要把原记录参数当成适用于所有设备的默认值。

```ini
address_family = ipv4
encoder = nvenc
adapter_name = NVIDIA GeForce RTX 3090
capture = ddx
hevc_mode = 3
av1_mode = 1

output_name = <VDD_DEVICE_ID>
dd_configuration_option = ensure_primary
dd_resolution_option = manual
dd_manual_resolution = 1920x1080
dd_refresh_rate_option = manual
dd_manual_refresh_rate = 60

max_bitrate = 35000
stream_audio = enabled
install_steam_audio_drivers = disabled
origin_web_ui_allowed = lan
upnp = disabled
```

### Steam 入口

保留两个应用入口：`Desktop` 用于维护和排障，`Steam Big Picture` 用于客厅日常使用。Sunshine 中 Big Picture 的启动与退出入口分别是：

```text
steam://open/bigpicture
steam://close/bigpicture
```

### 防火墙与端口：纠正原记录标签

原文把 TCP 47989 标为“RTSP 串流”，这是错误标签。依据 [Sunshine 官方端口表](https://docs.lizardbyte.dev/projects/sunshine/v0.21.0/about/advanced_usage.html#port)，默认端口族中 TCP 47989 是 HTTP，RTSP 是 TCP 48010；TCP 47984 是 HTTPS，47990 是 Web UI。该表为官方历史版本文档，2026-10-08 用于核对端口标签，不用于把本案例版本称作最新推荐。

| 协议 | 默认端口 | 核验后的用途 |
| --- | --- | --- |
| TCP | 47984 | HTTPS（原文笼统写作配对/控制） |
| TCP | 47989 | HTTP；不是 RTSP |
| TCP | 47990 | Web UI |
| TCP | 48010 | RTSP（原文只写 Sunshine TCP） |
| UDP | 47998 | 视频 |
| UDP | 47999 | 控制 |
| UDP | 48000 | 音频 |
| UDP | 48002 | 麦克风，官方该版本表标为未使用 |
| UDP | 5353 | mDNS 发现；属于原案例保留规则，不是上表的串流端口族 |

原案例保留的 UDP 放行范围是 **47998–48010**，这是原环境的规则范围，不表示其中每个端口都有独立用途，也不表示所有环境都需要开放整个范围。原记录新增规则 `Sunshine LAN TCP 47989-47990`，仅放行 `<LAN_SUBNET_CIDR>`；原有 Sunshine TCP/UDP 与 mDNS 规则保留。本次没有读取这些规则的实际定义或修改防火墙。

若更改 Sunshine 基础端口，其他端口会随偏移变化；以上表格仅描述默认端口族。端口标签修正不改变原案例检查 47989/47990 是否监听的方法。

### 按记录搭建时的顺序与操作影响

1. 确认主机有线、投影连接家庭 5 GHz Wi-Fi，两端属于同一家庭 LAN；按自己的地址填写占位符。
2. 对照主机、GPU、驱动、VDD 与 console 会话状态，再填写 17 项配置；源文没有安装包操作、VDD 安装步骤或版本升级过程，不能据此假定已完成安装。
3. 对照服务 Automatic / Running 和端口监听，保留 `Desktop` 与 `Steam Big Picture` 入口。
4. 在 Moonlight 删除作废条目，添加 `<HOST_LAN_IP>` 并重新配对 PIN，选择 `Steam Big Picture`。
5. 按 [故障恢复步骤](troubleshooting.md) 读回服务、链路和日志，再做下面的验收。

网络类别改为 Private、局域网防火墙放行和关闭 AP/client isolation 都会改变设备间可见性或访问范围，只适用于自己确认可信的家庭网，按需执行。不要在公共或访客网络照搬，不关闭整个防火墙，也不扩大到公网。重启 SunshineService 会中断现有串流；调整配置、删除旧条目和重配对前确认没有正在使用的会话。本仓库只说明步骤，不执行这些操作。

## 六、最终验收标准（待逐项记录结果）

以下完整保留源文验收目标。历史记录支持部分状态观察，但没有逐项验收报告，尤其没有 30 分钟连续运行结果。复现时自行填写日期、方法、实际结果，未测项目保持“未验证”。

| 验收项目 | 本次文档整理的证据状态 |
| --- | --- |
| Z9 Pro `<CLIENT_LAN_IP>` 可被主机 ARP 发现 | 原文给出检查方法，未附原始 ARP 输出 |
| Moonlight 只使用主机 `<HOST_LAN_IP>` | 原文指定正确地址，未附客户端截图 |
| Sunshine 日志出现 `CLIENT CONNECTED` | 原文记录最近成功会话出现；本次未复测 |
| VDD 输出为 1920×1080@60 | 配置与视频路径如此记录；未附独立输出测量 |
| HEVC 1080p60 连续运行 30 分钟 | 验收目标，未提供通过记录 |
| 日志出现 `Audio capture format` 和 `Opus initialized` | 原文记录出现；只支持主机捕获与编码初始化 |
| Steam Big Picture 可直接启动 | 已给启动入口，未附独立验收记录 |
| 断开/重连后不会进入错误 RDP 会话 | 验收目标；Guard 实现与测试结果未提供 |

## 七、原记录提到、未随文提供的本地文件

| 原环境相对路径（普通文本） | 所指内容 |
| --- | --- |
| `outputs/串流方案-书房-客厅.md` | 详细架构文件 |
| `work/Sunshine-ConsoleSessionGuard.ps1` | Windows 会话脚本 |
| `work/Sunshine-ConsoleSessionGuard.Tests.ps1` | Session Guard 测试 |
| `outputs/Moonlight-AndroidTV-v12.1-nonRoot.apk` | Moonlight Android TV APK |

源页面未提供附件引用或文件内容，无法验证这些文件存在、下载或具体实现。它们不是本仓库附件，不提供假下载链接，也不补造脚本、测试或 APK。文件名中的 v12.1 只是历史记录，不是最新版推荐。

## 八、官方资料

- [Sunshine 官方项目](https://github.com/LizardByte/Sunshine)
- [Sunshine 官方配置文档](https://github.com/LizardByte/Sunshine/blob/master/docs/configuration.md)
- [Sunshine Windows 音频源码](https://github.com/LizardByte/Sunshine/blob/master/src/platform/windows/audio.cpp)
- [Sunshine Windows 服务源码](https://github.com/LizardByte/Sunshine/blob/master/tools/sunshinesvc.cpp)
- [Moonlight 官方设置指南](https://github.com/moonlight-stream/moonlight-docs/wiki/Setup-Guide)
- [Apollo](https://github.com/ClassicOldSong/Apollo)：原资料列出的备选项目，实际主链路仍是 Sunshine + Moonlight。

端口标签另参见上文的官方版本端口表。官方资料链接用于查阅；本次只核验端口标签，没有据此替换原历史配置。
