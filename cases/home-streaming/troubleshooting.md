# 串流故障恢复与防复发

[返回案例](README.md) · [完整搭建配置](setup.md) · [源文覆盖与边界](source-coverage.md)

依据 2026-10-01 原方案记录整理；以下命令供读者在自己的 Windows 环境中按需执行，本次没有连接设备或运行这些命令。占位符定义见 [搭建配置](setup.md)，先替换为本机真实值；不要公开替换后的地址、MAC 或用户名。

## 三、这次故障的根因与修复记录

### 1. 过期的电视地址

早期记录为 `<OLD_CLIENT_LAN_IP>`，实际固定地址为 `<CLIENT_LAN_IP>`。原页面记载旧地址没有 ARP，Moonlight 使用旧主机条目时显示电脑离线。区分“客户端地址已变”和“应填入 Moonlight 的主机地址”：Moonlight 主机栏始终使用 `<HOST_LAN_IP>`。

### 2. 主机网络类别为 Public

`<HOST_ETHERNET_ADAPTER>` 曾被 Windows 标记为 Public，影响局域网发现和防火墙判断；原记录称已改为 Private。仅在确认可信的家庭网络按需调整，因为 Private 会改变发现及相应防火墙规则的适用范围。

### 3. 地址和端口混淆

- `<HOST_LAN_IP>`：有线 Windows 主机。
- `<CLIENT_LAN_IP>`：投影客户端，不是 Moonlight 要添加的主机。
- TCP 47989：默认 HTTP 端口，不填入 Moonlight 主机栏；原文“RTSP”标签已按 [官方资料](https://docs.lizardbyte.dev/projects/sunshine/v0.21.0/about/advanced_usage.html#port) 纠正。
- TCP 47990：Web UI 端口，也不是 Moonlight 主机地址。

### 4. RDP 会话干扰

RDP 会创建另一个 Windows 会话；原案例曾导致 Sunshine、Steam 和 VDD 不在同一会话。页面记载通过三项 Console Session Guard 任务处理：

- `Sunshine - Console Session Guard (Logon)`
- `Sunshine - Console Session Guard (Startup)`
- `Sunshine - Console Session Guard (RemoteConnect)`

客厅串流时不要重新连接 RDP。以上只是任务名称记录；源文没有脚本实现、任务 XML、触发器具体条件、权限设置或测试结果，不能称为本仓库已发布、可直接部署或已复现的恢复方案。

### 5. 音频初始化时机

早期日志出现：

```text
Couldn't create Device Enumerator
There will be no audio
Unable to initialize audio capture
```

原案例重启 SunshineService 后，新会话出现：

```text
Audio capture format is [F32 48000 2.0]
Opus initialized: 48 kHz, 2 channels, 96 kbps
```

这支持主机音频捕获及 Opus 初始化恢复，不单独证明客厅扬声器实际发声。已有这两行但客户端无声时，先检查 Moonlight 静音、Android TV 音频输出与投影仪音量，不先修改 Sunshine。

## 四、故障恢复 Runbook

### 第一步：服务、会话和监听端口

在自己的主机打开管理员 PowerShell，执行只读检查：

```powershell
Get-Service SunshineService
query session
Get-NetTCPConnection -State Listen |
  Where-Object { $_.LocalPort -in @(47989,47990) }
```

预期检查状态（不是本次执行输出）：

```text
SunshineService = Running
console = <WINDOWS_USER> / Active
47989、47990 = Listening
```

### 第二步：电视链路

先替换占位符再执行。PowerShell 示例把地址置于引号中，避免把尖括号误当作语法：

```powershell
arp -a '<CLIENT_LAN_IP>'
ping '<CLIENT_LAN_IP>'
```

将 ARP 结果与 `<CLIENT_MAC>` 对照。若 ARP 为 Incomplete 或无条目，按原记录依次检查：

1. 电视是否连接家庭 5 GHz SSID。
2. 是否误连访客网络。
3. 在可信家庭网中，按需关闭 AP/client isolation；这会允许客户端相互访问，不在公共或访客网照搬。
4. 子网掩码是否为 `255.255.255.0`、网关是否为 `<LAN_GATEWAY>`；这是原案例 `/24` 网络的值。
5. 必要时让电视忘记 Wi-Fi 后重新连接；此操作会中断网络并要求重新连接。

### 第三步：清理 Moonlight 旧条目

1. 删除旧 Sunshine/PC 条目；确认无需保留旧配对后再执行。
2. 添加 `<HOST_LAN_IP>`。
3. 重新配对 PIN。
4. 选择 `Steam Big Picture`。
5. 不添加 `<CLIENT_LAN_IP>`，不写端口。

### 第四步：查看 Sunshine 日志并分支处理

原环境日志路径与过滤命令如下；安装目录不同应按自己的实际路径调整，不假定文件已存在：

```powershell
Get-Content 'C:\Program Files\Sunshine\config\sunshine.log' -Tail 300 |
  Select-String 'CLIENT CONNECTED|Audio capture format|Opus initialized|There will be no audio|Unable to initialize'
```

| 日志表现 | 原案例处理顺序 |
| --- | --- |
| 没有新的 `CLIENT CONNECTED` | 检查网络、电视 IP、Wi-Fi 隔离与 Moonlight 旧条目 |
| 有 `CLIENT CONNECTED`，无画面 | 检查 VDD、console 会话和 `output_name` |
| 有 `There will be no audio` | 按需重启 SunshineService，再重新连接；重启会中断已有串流 |
| 有音频初始化但电视无声 | 检查 Moonlight、Android TV 和投影仪音频 |
| 有 NvENC 超时 | 检查 GPU/驱动与无线抖动，先降低分辨率、码率；源文没有提供调整后的具体数值 |

过滤命令只覆盖前四类连接/音频关键词，未包含 NvENC；检查编码超时时应查看未过滤的日志，不把过滤结果当作不存在超时的证明。分支用于定位方向，不表示日志出现一次就能确定所有根因。`CLIENT CONNECTED` 不能单独证明画面、声音、输入和稳定性通过。

## 五、长期防复发

- 在路由器为 `<CLIENT_MAC>` 保留 DHCP 地址 `<CLIENT_LAN_IP>`，避免旧地址记录继续使用；按自己网络范围配置。
- 主机始终使用网线，在可信家庭网保持 `<HOST_ETHERNET_ADAPTER> = Private`。
- 电视与主机处于同一家庭 LAN。
- 日常使用 `Steam Big Picture`，`Desktop` 留作维护，不作为日常游戏入口。
- 电视串流期间不连接 RDP。
- 不直接运行第二个 `Sunshine.exe`。
- 不同时叠加 VDD、GameViewer 等虚拟显示设备。
- 原页面记载 Wi-Fi 曾测到约 **10–349 ms、平均约 162 ms**；没有测量工具、次数、时长或原始输出，不能包装成端到端延迟或本次独立基准测试。若以稳定 1440p/4K 为目标，原方案优先改善 AP 覆盖或有线回程；不代表该目标已实现。
- NAS 使用“NAS 归档 + 本地活动库”策略；未确认 NAS 地址、容量与 SMB 路径前，不迁移现有 `D:\SteamLibrary`。这里保留的是库路径记录，不提供私人 NAS 地址或共享路径。

恢复后回到 [逐项验收标准](setup.md)，记录实际结果，尤其补测连续运行与断开重连；本次文档整理不补写未提供的成功结论。
