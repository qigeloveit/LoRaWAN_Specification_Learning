### 3 物理层消息格式
LoRa术语上分为上行消息和下行消息。
#### 3.1 上行消息
**上行消息** 是由终端经一个或多个网关发往服务器的消息。

上行消息通过LoRa无线分组显示模式传输，包含了LoRa物理帧头（PHDR）和头部CRC字段（PHDR_CRC）。CRC负责保证payload的完整性。

其中，PHDR、PHDR_CRC和payload的CRC字段均由射频收发机完成插入。

|Preamble |PHDR |PHDR_CRC |PHYPayload |CRC|
|---|---|---|---|---|
