# msd_lite — 轻量级 IPTV 组播转 HTTP 流媒体守护进程

[![Build-macOS-latest](https://github.com/wzy1985/msd_lite/actions/workflows/build-macos-latest.yml/badge.svg)](https://github.com/wzy1985/msd_lite/actions/workflows/build-macos-latest.yml)
[![Build-Ubuntu-latest](https://github.com/wzy1985/msd_lite/actions/workflows/build-ubuntu-latest.yml/badge.svg)](https://github.com/wzy1985/msd_lite/actions/workflows/build-ubuntu-latest.yml)

> 本项目是基于 [rozhuk-im/msd_lite](https://github.com/rozhuk-im/msd_lite) 的 fork，在原有功能基础上新增了 **控制面 / 数据面接口分离**（`dataIfName`）能力，详见下文"控制面 / 数据面分离"一节。

---

## 项目简介

msd_lite 是 Multi Stream daemon（msd）的轻量精简版本，用于通过 HTTP 在网络上组织 IPTV 流媒体转发。

它的工作方式：在服务器上接收 **UDP 组播流**（包括 RTP 流），再以 **HTTP 单播流**的形式分发给局域网内的播放器、电视、机顶盒等客户端。这样客户端无需直接加入组播组，只需一个普通的 HTTP 播放地址即可收看 IPTV 直播。

```
组播源 ──► msd_lite（接收组播 → 内部缓冲）──► HTTP 单播 ──► 播放器 / 电视 / 机顶盒
```

## 核心特性

- 开源免费，采用 **BSD-2-Clause** 许可证；
- 仅接收 UDP 组播，支持 RTP 流；
- 线程运行期**无死锁**；
- 发送端始终启用**零拷贝发送**（Zero Copy on Send, ZCoS）；
- 无需轮询即可向客户端发送数据（不依赖 `fUsePollingForSend`）；
- 不解析 MPEG2-TS 流内容，向新加入的客户端"智能"发送 MPEG2-TS 流头；
- 支持自定义 HTTP 响应头（如 DLNA 头），便于接入电视 / 播放器生态；
- **控制面 / 数据面接口分离**（本 fork 新增）：组播订阅指令与组播数据接收可使用不同网卡接口。

## 工作原理与控制面 / 数据面分离

组播流的接收涉及两个平面：

| 平面 | 作用 | 默认使用的接口 | 可配置项 |
|---|---|---|---|
| **控制面（Control Plane）** | 通过 IGMP/MLD 发送组播**加入 / 离开订阅指令** | `ifName` | 配置：`ifName`；URL 参数：`ifname` / `ifindex` |
| **数据面（Data Plane）** | 实际**接收组播 UDP 数据包** | 与 `ifName` 相同 | 配置：`dataIfName`；URL 参数：`data_ifname` / `data_ifindex` |

**默认行为（`dataIfName` 为空时）：** 组播订阅指令与组播数据接收均使用 `ifName` 指定的接口，与原有版本行为完全一致。

**当 `dataIfName` 不为空时：**
- 组播订阅指令（IGMP/MLD join/leave）仍使用 `ifName` 指定的接口；
- 组播数据（UDP 数据包）接收改用 `dataIfName` 指定的接口；
- 当数据面接口与控制面接口不同时，程序会**同时在两个接口上执行组播加入**，确保数据能够到达。

**适用场景示例：** 控制面订阅接口（如承载 IGMP 的上行 / VLAN 接口）与组播数据到达接口（如拨号口、专用组播口）位于不同网卡的网络拓扑，通过本特性可避免数据面与订阅面互相干扰，或在不支持 IGMP 转发的链路上灵活部署。

## 编译与安装

**前置依赖：** `git`、`cmake`、C 编译器（Linux 下为 `build-essential`，macOS 下为 Xcode Command Line Tools）。

**注意：** 仓库包含子模块 `src/liblcb`，克隆时必须使用 `--recursive` 参数：

```bash
git clone --recursive https://github.com/wzy1985/msd_lite.git
cd msd_lite
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_VERBOSE_MAKEFILE=true ..
make -j 8
```

如果已经克隆但未初始化子模块，请先执行：

```bash
git submodule update --init --recursive
```

## 运行测试

```bash
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_TESTS=1 ..
cmake --build . --config Release -j 16
ctest -C Release --output-on-failure -j 16
```

## 命令行参数

```
msd_lite [-d] [-v] [-c file]
         [-p PID file] [-u uid|usr -g gid|grp]

 -h           显示本帮助
 -d           以守护进程（daemon）方式运行
 -c file      指定配置文件
 -p PID file  将进程 PID 写入指定文件
 -u uid|user  以指定用户运行
 -g gid|group 以指定用户组运行
 -v           详细日志（verbose）
```

## 配置文件说明

默认配置位于仓库 `conf/msd_lite.conf`，为 XML 格式（**尺寸单位：KB，时间单位：秒**）。根节点为 `<msd>`，主要分节如下：

| 配置节点 | 说明 |
|---|---|
| `log.level` | syslog 日志级别，0（emerg）～7（debug） |
| `threadPool.threadsCountMax` | 线程池最大线程数，`0` 表示自动 |
| `threadPool.fBindToCPU` | 是否将线程绑定到 CPU |
| `HTTP.bindList` | HTTP 服务监听地址列表，支持 IPv4 / IPv6 |
| `HTTP.hostnameList` | 允许的主机名（Host）列表，`*` 表示任意 |
| `hubProfileList.hubProfile` | 流分发 Hub 参数模板（见下表） |
| `sourceProfileList.sourceProfile` | 组播源接收参数模板（见下表） |

### hubProfile 参数

| 参数 | 说明 |
|---|---|
| `fDropSlowClients` | 是否断开慢速客户端 |
| `fSocketHalfClosed` | 是否对客户端启用 `shutdown(SHUT_RD)` |
| `fSocketTCPNoDelay` | 是否对客户端启用 `TCP_NODELAY` |
| `fSocketTCPNoPush` | 是否对客户端启用 `TCP_NOPUSH` / `TCP_CORK` |
| `precache` | 预缓存大小（KB），分发流前先缓存的数据量 |
| `ringBufSize` | 流接收环形缓冲区大小，需为 `sndBlockSize` 的整数倍 |
| `skt.sndBuf` | 客户端套接字最大发送块大小，须大于 `sndBlockSize` |
| `skt.sndLoWatermark` | 发送块大小，须为 4 的整数倍 |
| `skt.congestionControl` | TCP 拥塞控制算法（会覆盖请求参数、服务器设置及系统默认值） |
| `headersList.header` | 自定义 HTTP 响应头（在流数据之前发送，可用于 DLNA 等） |

### sourceProfile 与 multicast 参数

| 参数 | 说明 |
|---|---|
| `skt.rcvBuf` | 组播接收套接字缓冲区大小 |
| `skt.rcvLoWatermark` | 关闭轮询时的实际最小客户端发送块 |
| `skt.rcvTimeout` | 组播接收超时（影响状态统计） |
| `multicast.ifName` | **控制面接口**：组播订阅（IGMP/MLD join/leave）使用的网卡名称 |
| `multicast.dataIfName` | **数据面接口**（本 fork 新增）：实际接收组播 UDP 数据的网卡名称，留空时回退为 `ifName` |
| `multicast.rejoinTime` | 每隔 X 秒执行一次 IGMP/MLD leave+join（刷新订阅），`0` 表示不执行 |

**配置示例**（订阅走 `vlan777`，数据接收走 `eth0`）：

```xml
<multicast>
    <ifName>vlan777</ifName>
    <dataIfName>eth0</dataIfName>
    <rejoinTime>0</rejoinTime>
</multicast>
```

## HTTP 播放接口

服务启动后，客户端通过 HTTP **GET / HEAD** 请求流地址。请求必须满足：

- 仅支持 GET / HEAD 方法（其他方法返回 400）；
- `Host` 头必须匹配 `HTTP.hostnameList` 中配置的主机名（否则返回 403）；
- 未识别的路径返回 404。

| 路径 | 说明 |
|---|---|
| `/stat` | 全局统计信息（纯文本） |
| `/hubstat` | 各流 Hub 的分发统计信息 |
| `/udp/<组播地址>[?参数]` | 请求 UDP 组播流，转为 HTTP 单播流分发 |
| `/rtp/<组播地址>[?参数]` | 同上，适用于 RTP 流 |

组播地址中的 UDP 端口可省略，**默认端口为 1234**。

### 查询参数

| 参数 | 所属平面 | 说明 |
|---|---|---|
| `ifname` | 控制面 | 组播订阅接口名（优先于 `ifindex`） |
| `ifindex` | 控制面 | 组播订阅接口索引 |
| `data_ifname` | 数据面 | 数据接收接口名（本 fork 新增） |
| `data_ifindex` | 数据面 | 数据接收接口索引（本 fork 新增） |
| `rejoin_time` | — | 覆盖配置中的 `rejoinTime` |

**播放地址示例**（订阅接口 `vlan777`，数据接收接口 `eth0`）：

```
http://msd-host:7088/udp/239.255.0.1:1234?ifname=vlan777&data_ifname=eth0
```

若只指定订阅接口、不指定数据面参数，则数据接收自动回退到订阅接口（与原有行为一致）：

```
http://msd-host:7088/udp/239.255.0.1
```

## 作为系统服务运行

- **FreeBSD：** 仓库提供 `os/freebsd/msd_lite` rc 脚本。将示例配置复制到配置目录并修改接口名后，在 `/etc/rc.conf` 中加入 `msd_lite_enable="YES"`，然后执行 `service msd_lite restart`。
- **Linux：** 可配合 systemd 编写服务单元运行（项目未内置，按发行版习惯自行配置即可）。

## 许可证

本项目采用 **BSD 2-Clause License**（详见仓库 `LICENSE` 文件），允许源码 / 二进制形式的分发与修改（包括商用），分发时需保留原版权声明。

原项目作者：Rozhuk Ivan `<rozhuk.im@gmail.com>`（2011–2026）。

## 参考链接

- 原项目：[rozhuk-im/msd_lite](https://github.com/rozhuk-im/msd_lite)
- 原项目文档：[msd_lite 官方 wiki](http://www.netlab.linkpc.net/wiki/en:software:msd:lite)
- 作者捐赠页：请参阅原项目主页
