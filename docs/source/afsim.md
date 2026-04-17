# AFSim
## 分布式仿真
### DIS和XIO
下面按“接收 → 处理 → 发送”的顺序，把 **DIS** 和 **XIO** 两条链路分别串起来（分布式场景里通常两者并行存在）。

#### **DIS：实体状态同步链路**

- **接收（网络 → PDU 对象）**
  - UDP Socket 收包后写入 GenIO 的接收缓冲：`GenUDP_IO::Receive` 会重置接收缓冲并调用 `ReceiveBuffer(...)` 把数据读进缓冲区（[GenUDP_IO.cpp#L42-L61]）。
  - DIS 的 UDP Device 从 GenIO 缓冲中取数据，够一个 PDU 就解析：`WsfDisUDP_Device::GetPdu` 里调用 `mGenIOPtr->Receive(0)`，然后 `DisPdu::Create(*mGenIOPtr, aPduFactoryPtr)` 生成具体 PDU（[WsfDisUDP_Device.cpp#L126-L158]）。

- **处理（PDU 对象 → 仿真对象/状态）**
  - 每个仿真步由 `WsfDisInterface::AdvanceTime` 拉取并处理所有待处理 PDU：过滤后直接调用 `pduPtr->Process()`（[WsfDisInterface.cpp#L1203-L1275]）。
  - 以最关键的 Entity State 为例：`WsfDisEntityState::Process()` 会
    - 忽略影子站点（`cSHADOW_SITE`）的 PDU；
    - 查本地是否已有对应实体；没有则创建外部平台，有则更新平台状态（[WsfDisEntityState.cpp#L68-L134]）。
  - “新实体 → 本地外部平台（你说的 Shadow/外部实体落地）”的核心入口是 `WsfDisInterface::AddExternalPlatformP`：选择平台类型、创建平台、必要时剥离组件、挂 `WsfDisMover`，然后 `Simulation.AddPlatform(...)`（[WsfDisInterface.cpp#L506-L646]）。
  - “ShadowPlatform（调试影子）”是可选逻辑：满足 shadow 配置时 `AddShadowPlatform(...)` 额外创建 `_shadow` 平台并设置 `cSHADOW_SITE`）。

- **发送（仿真状态 → 网络）**
  - 发送最终落在 Device 的 `PutPduP`：设置时间戳/仿真时间，`aPdu.Put(*mGenIOPtr)` 序列化进发送缓冲，然后 `mGenIOPtr->Send()` 发 UDP 包（[WsfDisUDP_Device.cpp#L160-L174]）。
  - 触发“什么时候发”通常是：心跳、阈值（位置/姿态变化）、以及特定事件（开火/爆炸等）。具体策略在 `WsfDisPlatform/WsfDisInterface` 的更新逻辑里决定，然后丢给 Device 发。

#### **XIO：AFSIM 内部生态通信链路（心跳发现 + TCP/UDP 数据）**

- **接收（网络 → XIO Packet 对象）**
  - XIO 在 `Initialize` 里启动连接管理（监听 TCP、建立 UDP 连接等）（[WsfXIO_Interface.cpp#L155-L192]）。
  - UDP/Broadcast/Multicast 目标连接通过 `ConnectToTarget` 建立：内部创建 `GenUDP_IO`，包装成 `PakUDP_IO`，交给 `mThreadedIO` 异步收包（[WsfXIO_Interface.cpp#L194-L273]）。
  - 每帧调用 `WsfXIO_Interface::AdvanceTime`，会执行 `ProcessMessages(aSimTime)` 处理所有已收包（[WsfXIO_Interface.cpp#L496-L541]）。

- **处理（XIO Packet → 回调/发布订阅/请求响应）**
  - `ProcessMessages` 从线程 I/O 中 `Extract` 出 packet 列表，对“同步包”做时间戳换算并可能缓存延迟处理，然后用 `PakProcessor::ProcessPacket(&pkt, true)` 分发给具体 packet 类型处理器（[WsfXIO_Interface.cpp#L544-L579]）。
  - packet 类型注册在 `WsfXIO_PacketRegistry::RegisterPackets`，未注册的包不能收发（[WsfXIO_PacketRegistry.cpp#L18-L66]）。
  - XIO 的“业务处理”主要靠回调绑定：构造函数里把心跳、初始化、服务发现等 packet 绑定到对应 handler（[WsfXIO_Interface.cpp#L83-L99]）。

- **发送（发布/请求/心跳 → 网络）**
  - `WsfXIO_Interface::Send/SendToAll*` 会统一打上 `ApplicationId` 和 `TimeStamp`，然后交给 `mThreadedIO.Send(...)`（[WsfXIO_Interface.cpp#L353-L412]）。
  - 心跳：`SendHeartbeat()` 构造 `WsfXIO_HeartbeatPkt` 并通过 UDP 广播给所有人（[WsfXIO_Interface.cpp#L646-L654]），接收端 `HandleHeartbeat` 收到后可触发 TCP 连接建立（[WsfXIO_Interface.cpp#L656-L679]）。
  - 发布订阅：发布时会把任意可序列化对象打包到 `GenBuffer`（大端），用 `PakO` 序列化，然后发出去（[WsfXIO_Publisher.hpp#L90-L100]）。

## **把两条链路放到“分布式仿真”里怎么理解**
- **DIS**：负责“不同仿真器/不同进程之间的实体状态同步”（标准 PDU、UDP、多播/广播、`pdu->Process()` 更新平台）。
- **XIO**：负责“AFSIM 生态内部的控制/查询/发布订阅”（心跳发现 + TCP/UDP，Packet 注册分发，应用级消息）。

如果你告诉我你关心的“数据”具体是哪类（实体位置、轨迹 Track、指令控制、传感器报告等），我可以把对应的 packet/PDU 类型再精确到具体类名和处理函数链路。