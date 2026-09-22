# ROV Qt 通信链路教程

本文面向第一次接触本工程通信代码的开发者，说明当前实现的通信思路、代码入口、固件升级数据流和常见排错方式。本文描述的是当前代码已经实现的 USB CDC + CAN 网关 + STM32G431 Bootloader 链路，不代表 Dashboard 已经接入真实 ROV 遥测。

## 1. 先记住一条原则

> **页面不认识协议，协议不认识页面。**

页面只提交业务请求，或接收已经解码的业务结果；串口、帧头、CAN ID、命令码和 CRC 都留在通信层。当前的依赖方向是：

```text
UI 页面
   ↓ 结构化 Request
Service / Controller
   ↓ 业务状态、payload
Protocol
   ↓ 原始协议帧
Transport
   ↓ USB CDC 字节
STM32H750 CAN_To_Uart
   ↓ CAN / CAN FD
STM32G431 Bootloader
```

这样做的好处是：更换 USB、TCP 或 Nano 接入方式时，不需要重写页面；协议字段变化时，也不会把十六进制细节扩散到 QWidget。

## 2. 一次通信经过哪些代码

### 2.1 发送方向

以固件页点击“读取版本”为例：

```text
FirmwarePage
    ↓ BootloaderService::sendHostCommand()
BootloaderProtocol
    ↓ 生成 8 字节 Host CONTROL payload
BootloaderCommunicationService
    ↓ 封装成 AA55 CAN 网关帧
SerialTransport::writeBytes()
    ↓
USB CDC → H750 → CAN → G431
```

页面只表达“目标节点发送 `GET_VERSION`”，不自己拼接 `01 01 ... CRC8`。

连续固件数据走另一条路径：

```text
FirmwarePage
    ↓
BootloaderDownloadController
    ↓ 按窗口准备 56 字节有效数据
BootloaderService::startDataWindow()
    ↓ 每个逻辑块编码为 AA59 DATA_BLOCK
BootloaderCommunicationService
    ↓ Credit 允许多少块就发送多少块
SerialTransport
```

### 2.2 接收方向

USB CDC 不保证一次 read 就得到完整的一帧，也可能一次 read 得到多帧。因此接收流程必须是增量的：

```text
SerialTransport::bytesReceived(QByteArray)
    ↓
BootloaderCommunicationService::processReceivedBytes()
    ↓ 累积 m_receiveBuffer，按 AA + family + length 找完整帧
AA55 / AA58 / AA59 Decoder
    ↓ 校验帧尾、长度、CRC、版本和字段
结构化结果
    ↓ Qt signal
FirmwarePage / DownloadController
```

解析器不能依赖单次串口读取的边界，必须覆盖半帧、粘包、噪声、错误帧和错误后恢复。

## 3. 三类协议的分工

| 协议族 | 主要用途 | 当前处理方式 | 不能做什么 |
| --- | --- | --- | --- |
| `AA55` | CAN 网关短帧、Bootloader 控制、节点响应 | 增量拆包，CRC8，多数为 8 字节 CAN payload | 不负责连续大数据的节奏控制 |
| `AA58` | 上位机与 H750 公共链路心跳 | 独立 20 字节解析，CRC16-CCITT | 不转换为 CAN，不代表 G431 已响应 |
| `AA59` | 连续固件数据的网关流控 | 增量拆包，CRC16，`BEGIN/DATA_BLOCK/END`，Credit/ACK | 不代表 G431 Flash 已提交 |

### 3.1 AA55：零散控制帧

`AA55` 适合短小、需要单独确认的命令：

- `ENTER_BOOT`；
- `ERASE`；
- `WRITE`；
- `WRITE_END`；
- `VERIFY`；
- `JUMP_APP`；
- `GET_VERSION`、`GET_INFO`、`GET_STATUS`。

Bootloader 的 Host CONTROL 和 Node RESPONSE 都由 `BootloaderProtocol` 编码/解析，CAN 网关外层由 `CanGatewayProtocol` 处理。两层协议不要混在一个类里。

### 3.2 AA58：链路心跳

心跳每约 1000 ms 发送一次，`SystemHeartbeatDecoder` 只负责验证并产生 `SystemHeartbeat`。Firmware 页面用心跳 watchdog 判断公共 USB CDC 链路是否在线，约 2.5 秒没有有效心跳就显示离线。

心跳在线只说明：

```text
上位机 ←→ H750 USB CDC 链路基本可达
```

它不说明：

- 某个 G431 节点在线；
- 节点已经进入 Bootloader；
- Flash 擦除或写入已经成功。

### 3.3 AA59：连续数据流控

AA59 把连续数据拆成三类消息：

```text
BEGIN
   ↓ 获得初始 Credit
DATA_BLOCK × N
   ↓ 收到累计 FLOW_ACK 后补充 Credit
END
```

当前实现使用 CRC16-CCITT。解码器支持 USB CDC 的半帧和粘包；服务层会去重重复 ACK、检查块编号范围、检查 Credit 合法性，并在等待确认超时后结束失败。

## 4. 固件升级完整流程

### 第一步：准备镜像

`BootloaderDownloadController` 读取 `.bin` 文件，计算 CRC-32/MPEG-2，记录目标节点、总字节数和逻辑包数。每个 Bootloader DATA 逻辑包固定 64 字节，其中 56 字节是有效固件数据，尾包用 `FF` 补齐。

### 第二步：进入 Bootloader

发送：

```text
ENTER_BOOT
```

APP 收到该命令后可能直接复位，不回复普通 ACK。因此上位机要记录“已发送，等待复位”，不能因为暂时没有响应就把它判断成失败。

### 第三步：擦除并建立窗口

下载控制器按顺序发送 `ERASE` 和 `WRITE`。`WRITE` 的返回值决定节点一次可以接收多少个逻辑包，这个窗口属于 G431 Flash 提交层。

### 第四步：通过 AA59 发送数据

对当前 Bootloader 窗口：

1. 用 AA59 `BEGIN` 告诉 H750 即将发送哪一段数据；
2. 初始 Credit 允许发送有限数量的 `DATA_BLOCK`；
3. 每收到累计 `FLOW_ACK`，再释放新的 Credit；
4. AA59 会话确认完整后，才继续处理节点 Flash 窗口；
5. 如果网关确认到了但节点窗口未提交，查询 `WINDOW_STATUS`，只重发当前未提交窗口。

### 第五步：结束、校验和跳转

只有节点确认所有逻辑包已经提交，才继续：

```text
WRITE_END → VERIFY → JUMP_APP
```

其中：

- AA59 ACK 代表网关流控层的确认；
- `WINDOW_STATUS` 代表节点窗口提交情况；
- `VERIFY` 代表节点对镜像的校验结果；
- `JUMP_APP` 才代表请求启动 APP。

这四种结果必须分开显示和记录，不能用一个“发送成功”覆盖全部状态。

## 5. 两层流控为什么不能混用

当前下载同时存在两个窗口：

```text
┌──────────────────────────────┐
│ AA59 网关窗口                 │
│ USB → H750 的接收 Credit      │
│ 由 FLOW_ACK 推进               │
└──────────────┬───────────────┘
               │ H750 转发
               ▼
┌──────────────────────────────┐
│ G431 Bootloader 窗口          │
│ CAN → Flash 的写入提交窗口    │
│ 由 WRITE / WINDOW_STATUS 推进 │
└──────────────────────────────┘
```

如果只看 AA59 ACK，可能出现“USB 和 H750 都收到了，但节点 Flash 还没有提交”的假成功。因此 `BootloaderDownloadController` 保留两套状态，并在网关流控完成后继续等待节点窗口结果。

## 6. Classic CAN 和 CAN FD 的关系

当前页面允许选择两种数据面：

| 部分 | Classic CAN | CAN FD+BRS |
| --- | --- | --- |
| 控制帧 | 固定使用标准 CAN 8 字节 | 不切换 |
| 固件 DATA 逻辑块 | H750 拆成 `0x100~0x107` 八个物理片段 | 1 帧承载 64 字节 |
| AA59 流控 | 两者共用 | 两者共用 |

因此，切换 CAN FD 只改变固件数据面的物理发送方式，不改变 Host CONTROL 的格式，也不改变页面业务命令。

## 7. 代码文件地图

| 文件 | 职责 |
| --- | --- |
| `src/communication/transport/SerialTransport.*` | Windows USB CDC 枚举、打开关闭、轮询读取、写入原始字节 |
| `src/communication/protocol/CanGatewayProtocol.*` | AA55 外层 CAN 网关帧和 CRC8 |
| `src/communication/protocol/SystemHeartbeatProtocol.*` | AA58 心跳和 CRC16-CCITT |
| `src/communication/protocol/CanFlowControlProtocol.*` | AA59 帧、ACK、Credit 和 CRC16 |
| `src/communication/service/BootloaderCommunicationService.*` | 接收路由、串口生命周期、AA59 流控状态和超时 |
| `src/communication/bootloader/BootloaderProtocol.*` | Bootloader Host/Node/Peer payload 和 CRC |
| `src/communication/bootloader/BootloaderTypes.h` | 命令、状态和结构化响应类型 |
| `src/communication/bootloader/BootloaderService.*` | 用业务参数发送控制命令或数据窗口 |
| `src/communication/bootloader/BootloaderDownloadController.*` | 固件下载状态机和失败恢复 |
| `src/pages/firmware/FirmwarePage.*` | 固件页 UI、请求触发、状态和日志显示 |

## 8. 新增功能时应该怎么放

### 新增一个协议帧

1. 在 `src/communication/protocol/` 新增独立类型和 decoder；
2. 处理半帧、粘包、错误帧、CRC 和恢复；
3. 在 `BootloaderCommunicationService::processReceivedBytes()` 中增加明确的协议族路由；
4. 用值类型 signal 把结果送到 Service；
5. 加纯字节流测试；
6. 页面只接收 Service/Contract 的业务结果。

### 新增一个页面按钮

1. 页面把用户意图定义成结构化 Request；
2. Service 做范围检查、单位转换和协议编码；
3. Transport 只接收已经编码完成的字节；
4. UI 等待真实反馈，不在点击后直接伪造成功状态。

### 排查一条命令

按下面顺序定位：

```text
页面是否发出了结构化请求？
    ↓
Service 是否生成了正确 payload？
    ↓
Transport 是否成功写出字节？
    ↓
H750 是否返回正确协议族？
    ↓
Decoder 是否通过长度、帧尾和 CRC？
    ↓
Service 是否把结果转换成页面状态？
```

不要一开始就在页面里打印或修改协议字节；先确认是哪一层没有产生预期结果。

## 9. 本地构建和测试

Fluent-Qt 是当前公共控件的必要第三方源码依赖，必须位于：

```text
third_party/Fluent-Qt/CMakeLists.txt
```

依赖安装、固定版本和完整构建命令见 [`docs/toolchain.md`](../toolchain.md)。准备好工具链和依赖后：

```powershell
. .\toolchain\env\activate.ps1
& .\toolchain\env\build-gui.ps1
ctest --test-dir build\gui --output-on-failure
```

当前协议测试目标是 `rov_can_flow_protocol_test`。真实 USB 设备、CAN 总线和 G431 Bootloader 联调不属于这个单元测试。

## 10. 当前边界和未完成部分

当前已经接入的是真实固件升级通信链路：USB CDC、AA55、AA58、AA59、Bootloader 控制和 Legacy 单节点下载。

以下内容仍不能当作已经完成：

- Dashboard 姿态、推进器和六自由度控制的真实 ROV Service；
- Jetson Nano/TCP Transport；
- 8 节点自治升级、Provider/Coordinator 修复轮次；
- 自动缺包修复；
- 真实设备全流程联调结果。

开发时如果看到 Preview 数值或页面绿色状态，要先确认它来自真实 Snapshot 还是演示数据，不能把 UI 演示当成硬件通信成功。

## 11. 相关文档

- [`src/communication/README.md`](../../src/communication/README.md)：通信层边界；
- [`src/communication/protocol/README.md`](../../src/communication/protocol/README.md)：协议层边界；
- [`docs/bootloader/README.md`](../bootloader/README.md)：Bootloader 命令、下载流程和实机测试；
- [`docs/ui-contract.md`](../ui-contract.md)：页面与数据层之间的契约边界；
- [`CONTRIBUTING.md`](../../CONTRIBUTING.md)：多人协作和合并要求。
