### 5 MAC指令
为了实现网络管理，在服务器和终端MAC层之间设计了一系列MAC指令交互。MAC层指令对应用、应用服务器及终端上运行的应用均不可见。

在一帧数据中可以包含任意MAC指令集，这些指令集要么放在FOpts字段中，要么当FPort为0时作为单独的数据帧打包在FRMPayload字段中。若放在FOpts中时，MAC指令是不加密的且不超过15字节；而如果打包在FRMPayload中的话，则需加密且不超过最大FRMPayload长度。

>**注意**：不愿被截获破解的MAC指令要放在FRMPayload中作为数据帧发送。

一个MAC指令由1字节的指令标识（CID）附加指令相关字节组成，相关字节可为空。

|    CID     |     Command      | 发送方为终端 | 发送方为网关 |                     简单描述                     |
| ---------- | ---------------- |:------------:|:------------:| ------------------------------------------------ |
|    0x02    |   LinkCheckReq   |      x       |              |          终端用来确认与服务器的连接情况          |
|    0x02    |   LinkCheckAns   |              |      x       |   LinkCheckReq指令的应答，同时包含接收信号强度   |
|    0x03    |    LinkADRReq    |              |      x       | 要求终端改变数据传输速率、发送功率、重复率或信道 |
|    0x03    |    LinkADRAns    |      x       |              |                LinkRateReq的应答                 |
|    0x04    |   DutyCycleReq   |              |      x       |            设置设备的最大总发射占空比            |
|    0x04    |   DutyCycleAns   |      x       |              |                DutyCycleReq的应答                |
|    0x05    | RXParamSetupReq  |              |      x       |                 设置接收时隙参数                 |
|    0x05    | RXParamSetupAns  |      x       |              |              RXParamSetupReq的应答               |
|    0x06    |   DevStatusReq   |              |      x       |                   终端状态请求                   |
|    0x06    |   DevStatusAns   |      x       |              |         返回终端的状态，即电量和解调范围         |
|    0x07    |  NewChannelReq   |              |      x       |                创建或修改无线信道                |
|    0x07    |  NewChannelAns   |      x       |              |               NewChannelReq的应答                |
|    0x08    | RXTimingSetupReq |              |      x       |                设置接收时隙的时间                |
|    0x08    | RXTimingSetupAns |      x       |              |              RXTimingSetupAns的应答              |
| 0x80至0xFF |       专用       |      x       |      x       |              预留作专用网络指令扩展              |

表4 MAC指令

>**注意**：MAC指令的长度没有明确给出，必须隐式告知MAC处理方。因此未知MAC指令无法忽略掉，一碰到就会终止MAC指令处理进程。基于此建议按照LoRaWAN协议来处理MAC命令。这样所有基于LoRaWAN协议的MAC命令都可以被处理，即使是更高版本的命令。

>**注意**：服务器调整的参数（如RX2、信道）只保留至下一次终端入网时。因此每次成功入网之后会重置这些参数为默认值，服务器端决定是否按需要重新调整参数。

#### 5.1 链路检查指令（LinkCheckReq, LinkCheckAns）
终端通过LinkCheckReq指令可以验证与服务器的连接性，该指令无消息内容。

服务器通过一个或多个网关接收到LinkCheckReq时，回复LinkCheckAns指令。

|     Size (bytes)      |   1    |   1   |
| -------------------- | ------ | ----- |
| LinkCheckAns Payload | Margin | GwCnt |

解调余量（Margin）为取值大小0-254的8bit无符号整型，表示上次成功收到LinkCheckReq指令的链路余量（dB）。值为0代表在解调下限（0 dB或无余量）接收到指令帧，而值为20表示网关在比解调下限高20dB的信号强度下收到指令帧。值255为保留值。

网关数量（GwCnt）为成功收到上一帧LinkCheckReq指令的网关个数。

#### 5.2 链路ADR指令（LinkADRReq, LinkADRAns）
服务器通过**LInkADRReq**指令要求终端进行速率更改。

|    Size (bytes)    |        1         |   2    |     1      |
| ------------------ | ---------------- | ------ | ---------- |
| LinkADRReq Payload | DataRate_TXPower | ChMask | Redundancy |

|       Bits       |  [7..4]  | [3..0]  |
| ---------------- | -------- | ------- |
| DataRate_TXPower | DataRate | TXPower |

所请求的数据传输速率（**DataRate**）和发射功率（**TXPower**）与地域相关且经编码的，详见《LoRaWAN地域相关参数手册》第7章。

信道掩码（**ChMask**）通过将相应的比特位置0，来对上行链路可用的信道进行编码，如下表所示。

| Bit# | Usable channels |
| ---- | --------------- |
|  0   |    Channel 1    |
|  1   |    Channel 2    |
|  ..  |       ..        |
|  15  |   Channel 16    |
表5 信道状态表

**ChMask** 字段中某个bit置1意味着对应的信道可以用于上行传输，前提是该信道支持终端当前使用的数据传输速率；而某个bit置0则表示避免使用该信道。

|      Bits       |  7  |   [6:4]    | [3:0] |
| --------------- | --- | ---------- | ----- |
| Redundancy bits | RFU | ChMaskCntl | NbRep |

在Redundancy中**NbRep**字段为每条上行消息的重复发送次数。该字段仅对"unconfirmed"上行帧有效，默认为1，有效范围为[1:15]。如果终端收到NbRep==0，则使用默认值。网络管理员可通过该字段控制节点的上行链路冗余，从而保证给定的服务质量。终端在重传期间与正常帧一样跳频，每次重传之后会等待，直到接收窗口失效。

信道掩码控制（**ChMaskCntl**）字段负责控制ChMask的掩码位。当网络中含超过16个信道时，该字段只能设为非0值。它控制ChMask使用哪16个信道。同时它也可以用于统一打开或关闭使用某种特定调制方式的信道。该字段的具体使用和地域相关，具体见《LoRaWAN地域相关参数手册》第7章。

信道频率与地域相关，定义见第6章。终端收到LinkADRReq指令后，回复LinkADRAns指令。

|    Size(bytes)     |   1    |
| ------------------ | ------ |
| LinkADRAns Payload | Status |

|    Bits     | [7:3] |     2     |       1       |        0         |
| ----------- | ----- | --------- | ------------- | ---------------- |
| Status bits |  RFU  | Power ACK | Data rate ACK | Channel mask ACK |

LinkADRAns Status字段各bit位含义如下：

|                  |                                  Bit=0                                   |      Bit=1       |
| ---------------- | ------------------------------------------------------------------------ | ---------------- |
| Channel mask ACK |              要使用的信道未定义，忽略该指令，终端状态不改变              | 信道掩码设置成功 |
|  Data rate ACK   | 所请求的速率未定义或者在给定信道掩码下不支持，忽略该指令，终端状态不改变 |   速率设置成功   |
|    Power ACK     |           所请求的的功率设备不支持，忽略该指令，终端状态不改变           |   功率设置成功   |

以上三个字段任意一个值为0，指令就执行失败，节点保持之前的状态不变。

#### 5.3 终端发送占空比（DutyCycleReq, DutyCycleAns）
**DutyCycleReq** 指令用于限制终端的最大总发送占空比。总发送占空比指的是所有子频带上的发送占空比之和。

|     Size(bytes)      |     1     |
| -------------------- | --------- |
| DutyCycleReq Payload | MaxDCycle |

允许的最大终端发送占空比为：

$$总发送占空比 = \frac{1}{2^{MaxDCycle}}$$

MaxDutyCycle的有效范围为[0:15]。0表示除了当地法规限制外"没有占空比限制"。

值为255意味着终端应立即进入静默状态，相当于远程关闭设备。

终端通过DutyCycleAns指令回复DutyCycleReq。DutyCycleAns指令无payload。

#### 5.4 接收窗口参数（RXParamSetupReq, RXParamSetupAns）
RXParamSetupReq指令用于更新第二个接收窗口（RX2）的频率和数据传输速率，同时也可以用于设置RX1时隙上下行链路数据传输速率的偏移。

|     Size(bytes)     |     1      |     3     |
| ------------------- | ---------- | --------- |
| RX2SetupReq Payload | DLsettings | Frequency |

|    Bits    |  7  |     6:4     |     3:0     |
| ---------- | --- | ----------- | ----------- |
| DLsettings | RFU | RX1DRoffset | RX2DataRate |

RX1DRoffset字段用来设置终端设备上行和下行第一个接收窗口（RX1）数据速率之间的偏移，默认值为0。基站在某些地域有最大功率密度限制，所以使用偏移，并且还可以用来均衡上行和下行的无线链路余量。

**RX2DataRate** 定义第二个接收窗口使用的数据传输速率，其用法和**LinkADRReq**指令 一样（例如，0 表示 DR0/125kHz）。**Frequency** 是第二个接收窗口使用的信道频率，其频率编码规则的定义见**NewChannelReq**命令。

终端设备收到**RXParamSetupReq**后回复**RXParamSetupAns**指令。其payload为单字节的status。

|     Size(bytes)     |   1    |
| ------------------- | ------ |
| RX2SetupAns Payload | Status |

Status含义如下：

|    Bits     | 7:3 |        2        |         1         |      0      |
| ----------- | --- | --------------- | ----------------- | ----------- |
| Status bits | RFU | RX1DRoffset ACK | RX2 Data rate ACK | Channel ACK |

|                   |              Bit=0              |          Bit=1          |
| ----------------- | ------------------------------- | ----------------------- |
|    Channel ACK    |           频率不可用            |     RX2信道设置成功     |
| RX2 Data rate ACK |       数据传输速率不可用        | RX2数据传输速率设置成功 |
|  RX1DRoffset ACK  | RX1的上下行数据传输速率偏移无效 |   RX1DRoffset设置成功   |

表7 RX2SetupAns status字段含义

以上三个字段任意一个值为0，指令就执行失败，节点保持之前的状态不变。

#### 5.5 终端状态（DevStatusReq, DevStatusAns）
服务器通过**DevStatusReq**指令获取终端状态信息，该指令无payload。终端收到**DevStatusReq**会回复**DevStatusAns**指令。

|     Size(bytes)      |    1    |   1    |
| -------------------- | ------- | ------ |
| DevStatusAns Payload | Battery | Margin |

**Battery** 上报格式如下：

| Battery |           描述           |
| ------- | ------------------------ |
|    0    |     终端接入外部电源     |
| 1.。254 | 电量，1最小，254表示最大 |
|   255   |       无法测量电量       |

表8 电量解析

**Margin** 为上一次成功收到**DevStatusReq**指令时的解调信噪比（dB）进行取整得到的值。其值为6bit的有符号整型，范围为[-32,31]。

|  Bits  | 7:6 |  5:0   |
| ------ | --- | ------ |
| Status | RFU | Margin |

#### 5.6 创建/修改信道（NewChannelReq, NewChannelAns）
NewChannelReq指令用于创建信道或者修改已有信道的参数。该指令设置新信道的中心频率，以及该信道可用的数据速率范围：

|      Size(bytes)      |    1    |  3   |    1    |
| --------------------- | ------- | ---- | ------- |
| NewChannelReq Payload | ChIndex | Freq | DrRange |

信道索引（ChIndex）是新信道或者待修改信道的索引值。LoRaWAN规范中已经为不同的地域和频段内置了默认信道，这些信道适用所有设备，而且不能通过NewChannelReq（见第6章）进行修改。如果默认的信道数量是N，那默认的信道就是 0 ~ N-1，而 ChIndex 取值范围就是 N ~ 15。终端设备至少要能处理 16 个不同的信道。在某些地区终端允许存储超过16个信道。

频率（Freq）为24bits无符号整型，实际信道频率是 100 × Freq（Hz），其中100MHz以下的值保留未来使用。如此可设置100MHz~1.67GHz之间以 100 Hz为步长的所有信道频率。 Freq为0表示不使用该信道。终端设备必须对频率进行检查，保证它的射频硬件支持将要使用的频率，否则返回错误。

数据速率范围（DrRange）指明此信道的数据传输速率范围。该字段由两个4bit的索引组成：

|  Bits   |  7:4  |  3:0  |
| ------- | ----- | ----- |
| DrRange | MaxDR | MinDR |

根据章节5.2的定义，MinDR字段指定此信道最低数据速率，例如：0 表示指定 DR0/125 kHz。同理MaxDR指定最高数据速率。例如：DrRange = 0x77表示信道只能使用 50kbps GFSK，DrRange = 0x50表示数据速率范围是DR0 / 125 kHz 到 DR5 / 125 kHz。

创建的信道一旦设置成功，可以马上用来通信。

终端收到NewChannelReq后回复NewChannelAns命令，其payload如下：

|      Size(bytes)      |   1    |
| --------------------- | ------ |
| NewChannelAns Payload | Status |

Status含义如下：

|  Bits  | 7:2 |         1          |          0           |
| ------ | --- | ------------------ | -------------------- |
| Status | RFU | Data rate range ok | Channel frequency ok |

|                      |              Bit=0               |         Bit=1          |
| -------------------- | -------------------------------- | ---------------------- |
|  Data rate range ok  | 数据传输速率超出当前终端允许范围 | 终端兼容该数据速率范围 |
| Channel frequency ok |        设备无法使用该频率        |        频率可用        |

表9 表7 NewChannelAns status字段含义

以上两个字段任意一个值为0，指令就执行失败，节点保持之前的状态不变。

#### 5.7 TX和RX之间延时设置（RXTimingSetupReq, RXTimingSetupAns）
RXTimingSetupReq指令可配置上行发送结束到打开第一个接收窗口之间的延时。第二个接收窗口在第一个接收窗口之后一秒打开。

|       Size(bytes)        |    1     |
| ------------------------ | -------- |
| RXTimingSetupReq Payload | Settings |

Delay字段表示时延，由两个4bit的索引组成：

|   Bits   | 7:4 | 3:0 |
| -------- | --- | --- |
| Settings | RFU | Del |

延迟单位是秒，Del=0 表示1秒：

| Del | Delay[s] |
| --- | -------- |
|  0  |    1     |
|  1  |    1     |
|  2  |    2     |
|  3  |    3     |
| ..  |    ..    |
| 15  |    15    |

表10 Del映射表

终端收到RXTimingSetupReq后回复RXTimingSetupAns指令，该指令无payload。
