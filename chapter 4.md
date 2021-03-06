### 4 MAC层消息格式
所有LoRa上下行消息均携带PHY负载（Payload），该负载以单字节MAC头（MHDR）开始，紧跟着MAC负载（MACPayload），然后以4字节消息完整码（MIC）结尾。
![mac_format](https://raw.githubusercontent.com/qigeloveit/LoRaWAN_Specification_Learning/master/image/mac_format.png)

#### 4.1 MAC层（PHYPayload）
| Size(bytes) |  1   |   1...M    |  4  |
| ----------- | ---- | ---------- | --- |
| PHYPayload  | MHDR | MACPayload | MIC |

MACPayload字段长度M的最大值与区域相关，详见第6章。

#### 4.2 MAC头（MHDR字段）
|   Bit#    | 7..5  | 4..2 | 1..0  |
| --------- | ----- | ---- | ----- |
| MHDR bits | MType | RFU  | Major |

MAC头指明了消息类型（MType）和LoRaWAN协议主版本号，该版本号规定了数据帧的编码方式。

##### 4.2.1 消息类型（MType位字段）
LoRaWAN分六种不同的MAC消息类型：Join request, join accept, unconfirmed data up/down以及confirmed data up/down。

| MType |      Description      |
| ----- | --------------------- |
|  000  |     Join Request      |
|  001  |      Join Accept      |
|  010  |  Unconfirmed Data Up  |
|  011  | Unconfirmed Data Down |
|  100  |   Confirmed Data Up   |
|  101  |  Confirmed Data Down  |
|  110  |          RFU          |
|  111  |      Proprietary      |

表1 MAC消息类型

###### 4.2.1.1 Join-request和Join-accept消息
Join-request和Join-accept消息是在空中激活过程中使用，描述见6.2章节。

###### 4.2.1.2 数据消息
数据消息用于传输MAC指令和应用数据，二者可以合并在一条消息中。confirmed data消息需要接收方应答，而unconfirmed data则不需要。Proprietary消息用于处理那些无法与标准消息互通的非标准消息格式，但仅仅只能在具备相同扩展的设备中使用。

不同的消息类型有着不同的消息完整性确认方法，接下来会逐一介绍。

##### 4.2.2 数据消息的主版本（Major位字段）
| Major bits | Description |
| ---------- | ----------- |
|     00     | LoRaWAN R1  |
|   01..11   |     RFU     |

表2 Major列表

  > **注意**：主版本号规定了入网过程（见6.2章节）中数据交换的格式以及MAC Payload的前4字节内容（见第4章）。对于每个主版本，终端可实现其下不同子版本规定的帧格式，但该子版本号必须预先通过带外消息（（e.g. 作为设备个性化信息的一部分））告知服务器。

#### 4.3 数据消息的MAC负载（MACPayload）
数据消息的MAC负载，即所谓的“数据帧”，包含帧头(FHDR)、端口(FPort)和帧负载（FRMPayload），其中FPort和FRMPayload为可选项。

##### 4.3.1 帧头（FHDR）
FHDR包含终端短地址（DevAddr）、1个帧控制字节(FCtrl)、2字节的帧计数（FCnt）以及最多15字节的用于传输MAC指令的帧选项字段（FOpts）。

| Size(bytes) |    4    |   1   |  2   | 0..15 |
| ----------- | ------- | ----- | ---- | ----- |
|    FHDR     | DevAddr | FCtrl | Fcnt | FOpts |

下行链路帧帧头的FCtrl内容如下：

|    Bit#    |  7  |     6     |  5  |    4     |  [3..0]  |
| ---------- | --- | --------- | --- | -------- | -------- |
| FCtrl bits | ADR | ADRACKReq | ACK | FPending | FOptsLen |

上行链路帧帧头的FCtrl内容如下：

|    Bit#    |  7  |     6     |  5  |  4  |  [3..0]  |
| ---------- | --- | --------- | --- | --- | -------- |
| FCtrl bits | ADR | ADRACKReq | ACK | RFU | FOptsLen |

###### 4.3.1.1 帧头中的自适应数据传输速率控制（FCtrl中的ADR, ADRACKReq字段）
LoRa网络允许各终端分别使用任意可能的数据传输速率。这一特性被LoRaWAN运用在了适应和优化处于静止状态的终端的数据传输速率，被称为自适应数据传输速率（ADR）。当ADR使能时，网络会自动优化使用最快的传输速率。

移动终端由于其移动导致无线环境快速变化，因此数据传输速率的管理机制不再适用，应使用固定的数据传输速率。

如果ADR字段为1，则服务器会通过相应的MAC指令控制终端的数据传输速率。反之若ADR字段为0，则无论接收信号强度如何服务器都不会进行控制。终端或是服务器均可以根据需求设置ADR值为0或1，但为了最大化终端的电池寿命和网络容量，ADR字段应尽可能置为1。

>**注意**：即便是移动终端在大多数时间里也实际是静止的，因此终端应仍根据移动状态，去请求服务器通过ADR字段优化数据传输速率。

如果终端自动设置的数据传输速率比默认值大，则需要定期确认服务器能否收到上行帧。上行帧计数器每增加一次（重传帧不会增加），`ADR_ACK_CNT`计数会加1。当发送`ADR_ACK_LIMIT`包上行帧（即`ADR_ACK_CNT >= ADR_ACK_LIMIT`）后仍未收到下行帧回复，则将`ADR`确认请求位（**ADRACKReq**）置1，这时服务器需要在`ADR_ACK_DELAY`包上行帧前发送一包下行帧作为回应，终端在任何一包上行帧发送之后收到下行帧的话，都会重置`ADR_ACK_CNT`计数。由于在终端接收时隙收到任何应答帧都意味着网关仍能接收上行帧，所以此时下行帧的**ACK**字段不需要置位。而如果接下来`ADR_ACK_DELAY`包上行帧内（i.e. 发送`ADR_ACK_LIMIT + ADR_ACK_DELAY`包）没有收到应答帧，终端就会切换至更低的数据传输速率，尝试通过更长通信距离来获取重连。在每次达到`ADR_ACK_LIMIT`包上行帧之后未收到应答时，都将进一步降低数据传输速率。而当终端达到默认数据传输速率时，不再对**ADRACKReq**进行置位，因为链路通信距离已无法提高。

>**注意**：不要求立即对`ADR`确认请求进行应答，为服务器进行最优下行帧调度提供了灵活性。

>**注意**：在发送上行帧时，若`ADR_ACK_CNT >= ADR_ACK_LIMIT`且当前数据传输速率大于设备最小速率，才将ADRACKReq置1，否则为0。

###### 4.3.1.2 消息确认位和确认过程（FCtrl中的ACK）
当收到一包confirmed data时，接收方需回复应答帧（ACK置1）。若发送方为终端，那么服务器将在终端发送之后的某个接收窗口开启时回复应答帧。而如果发送方为网关，那么终端设备会自行判断发送应答帧的时机。

应答帧只会对最新收到的消息进行回复，且不会重传。

>**注意**：为使终端设备尽可能逻辑简单状态明确，在收到需确认消息后会最好立即回复，应答信息简明扼要（最好为空消息），或者也可以推迟回复，等到下一次发送消息时附加应答。

###### 4.3.1.3 重传过程
同一条消息的重传次数（以及时间）取决于终端本身，并且每台终端设备都可能不一样，也可以通过服务器来设置或调整。

>**注意**：第18章给出了一些应答机制的时序图示例。

>**注意**：如果终端设备到达最大重传次数仍未收到应答，那么可以降低传输速率来尝试重连。而重连之后是再次重传还是丢弃，取决于终端设备。

>**注意**：如果服务器到达最大重传次数仍未收到应答，通常会认为该终端设备不可达，直到再次收到来自该终端设备的数据。而重连之后是再次重传还是丢弃，取决于服务器。

>**注意**：重传期间的数据传输速率退避策略建议见18.4章节。

###### 4.3.1.4 帧挂起位（FCtrl中的FPending，downlink only）
帧挂起位（**FPending**）仅在下行链路中使用，表示网关还有更多挂起的数据待发送，从而要求终端尽快发送上行帧以再次开启接收窗口。

**FPending** 的具体用法见18.3章节。

###### 4.3.1.5 帧计数器（FCnt）
每个终端设备有两个计数值：上行链路帧计数FCntUp和下行链路帧计数FCntDown。前者由终端负责累加，后者则由服务器负责。服务器会跟踪记录上行帧计数，并生成对每个终端设备的下行帧计数。在入网请求接受后，终端和服务器的上下行帧计数均会清零。随后发送方每发送一帧数据，就会将其控制的FCntUp或FCntDown加1，而接收方则会将相应的计数值经校验后与之保持同步。校验规则为：将收到的计数值与相应本地值相减，在考虑计数回滚的情况下，差值需小于MAX_FCNT_GAP，否则意味着中间丢包太多接下来数据会被丢弃。

LoRaWAN协议允许使用16bit或是32bit的帧计数器。服务器端需通过带外信息获知给定终端的计数器位数。如果使用16bit帧计数器，FCnt字段则直接使用计数值，必要时在前面填充0。而如果使用32bit帧计数器，FCnt字段值取计数值的低16位。（对上行链路而言，帧计数器值为FCntUp，而下行链路为FCntDown）

终端在应用和服务器会话密钥不变的情况下，不应使用重复FCntUp值，除非是重传。

>**注意**：因为FCnt只取32bit帧计数器的低16位，所以服务器必须通过观察链路情况推断帧计数器的高16位值。

###### 4.3.1.6 帧选项（FCtrl中的FOptsLen，FOpts）
FCtrl中的帧选项长度字段（**FOptsLen**）指示帧选项字段（**FOpts**）的实际长度。

**FOpts** 用于传输MAC指令，最大长度15字节；有效MAC指令列表见4.4章节。

如果**FOptsLen**为0，则**FOpts**字段不存在。如果**FOptsLen**不为0，比如说**FOpts**字段中包含MAC指令，那么不能使用端口0（**FPort** 要么不存在，要么为非0值）。

MAC指令不能同时存在于payload和帧选项字段之中。

##### 4.3.2 端口（FPort）
FRMPayload不为空时，必须填写端口字段。此时FPort为0表示FRMPayload仅包含MAC指令；FPort取值1-233（0x01-0xDF）视应用而定，取值224-255（0xE0-0xFF）保留作将来标准应用扩展。

| Size(bytes) | 7..23 | 0..1  |    0..*N*  |
| ----------- | ----- | ----- | ---------- |
| MACPayload  | FHDR  | FPort | FRMPayload |

*N* 为应用负载的字节数，其有效范围与区域相关，详见第7章。

*N* 应小于等于：

<center>N ≤ M−1−(FHDR字节数)

其中*M*为最大MACPayload长度。</center>

##### 4.3.3 MAC帧payload加密（FRMPayload）
当数据帧中携带了负载时，**FRMPayload**须在计算消息一致码（**MIC**）之前进行加密。

加密机制采用IEEE 802.15.4/2006附录B描述的AES128算法。

默认情况下，所有端口的加解密是由LoRaWAN层完成。但除端口0外，若对应用更方便的话，其他端口的加解密操作也可以由LoRaWAN层之上来完成。不过这种情况下需要通过带外通道将这些节点和端口FPort告知服务器，详见第19章。

###### 4.3.3.1 LoRaWAN层加密
使用的密钥K取决于端口FPort：

| FPort  |    K    |
| ------ | ------- |
|   0    | NwkSKey |
| 1..255 | AppSKey |

表3 FPort列表

加密字段为：

pld = **FRMPayload**

对于每包数据，取i = 1..k，k = ceil(len(pld) / 16)，算法定义块序列A<sub>i</sub>如下：

|  Size(bytes)  |  1   |    4     |  1  |    4    |         4          |  1   |  1  |
| ------------- | ---- | -------- | --- | ------- | ------------------ | ---- | --- |
| A<sub>i</sub> | 0x01 | 4 x 0x00 | Dir | DevAddr | FCntUp or FCntDown | 0x00 |  i  |

Dir字段表示方向，0为上行帧，1为下行帧。

对块序列A<sub>i</sub>加密生成S<sub>i</sub>，从而得到序列S：

S<sub>i</sub> = aes128_encrypt(K, A<sub>i</sub>) for i = 1..k

S = S<sub>1</sub> | S<sub>2</sub> | .. | S<sub>k</sub>


通过截取

（pld|pad<sub>16</sub>） xor S

的前len(pld)字节，得到payload的加解密序列。

###### 4.3.3.2 LoRaWAN层以上加密
如果LoRaWAN层从特定端口（不能为端口0，这是预留用作MAC指令的）收到上层传来已预加密的FRMPayload，那么将不对数据进行处理，直接将FRMPayload在MACPlayload和应用之间进行转发。

#### 4.4 消息一致码（MIC）
消息一致码基于消息的所有字段计算得出。
<center>
msg = MHDR | FHDR | FPort | FRMPayload
</center>

len(msg)即表示消息总字节数。

**MIC** 计算方式如下[RFC4493]：

cmac = aes128_cmac(NwkSKey, B<sub>0</sub> | msg)

MIC = cmac[0..3]

其中B<sub>0</sub>定义如下：

|  Size(bytes)  |  1   |    4     |  1  |    4    |         4          |  1   |    1     |
| ------------- | ---- | -------- | --- | ------- | ------------------ | ---- | -------- |
| B<sub>0</sub> | 0x49 | 4 x 0x00 | Dir | DevAddr | FCntUp or FCntDown | 0x00 | len(msg) |

其中Dir字段表示方向，0为上行帧，1为下行帧。
