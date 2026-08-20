# UART转发板及第二组吐珠电机协议 V1.0

## 1. 串口参数

安卓板、UART转发板、原PANTAO主板之间均使用：115200、8N1、无校验、无硬件流控。

- USART2：连接原PANTAO主板，PA2=TX、PA3=RX
- USART3：连接安卓板，PB10=TX、PB11=RX

## 2. 原球盘协议透明转发

原安卓与PANTAO主板之间的协议不修改。正常模式下，除转发板自身协议外，数据原样双向转发。

固定14字节格式：`AA ResendID ID Code1 Code2 Data1 Data2 Data3 Data4 ACK Expand CRC_H CRC_L 55`。

CRC16规则与原PANTAO主板一致：初值`0xFFFF`、多项式`0xA001`、计算Byte0~Byte10，CRC高字节放Byte11、低字节放Byte12。

## 3. 转发板设备类型

```c
#define Relay_to_Android 0x04
#define Android_to_Relay 0x05
```

`Code1=0x05`由UART转发板本地处理，不再发送给原PANTAO主板。

## 4. Android -> 第二组吐珠电机

### 4.1 按数量吐珠

```text
Code1 = 0x05
Code2 = 0x01
Data1 = 0x00
Data2 = 0x00
Data3 = 数量高8位
Data4 = 数量低8位
ACKbyte = 0x01
ExpandCode = 0x00
```

数量：`((uint16_t)Data3 << 8) | Data4`。如果已有未完成数量，新数量继续累加。

收到合法命令后，转发板立即把收到的完整14字节原帧原样返回给安卓作为ACK。同一个`ID + Code2`在5秒内重复到达时，只重复ACK，不重复执行。

示例：ID=`0x21`，吐10颗：

```text
AA 00 21 05 01 00 00 00 0A 01 00 02 40 55
```

### 4.2 独立停止电机2

```text
Code1 = 0x05
Code2 = 0x02
Data1~Data4 = 0
ACKbyte = 0x01
ExpandCode = 0x00
```

只停止并清空电机2，不影响电机1，也不发送给原PANTAO主板。

示例，ID=`0x22`：

```text
AA 00 22 05 02 00 00 00 00 01 00 F0 47 55
```

## 5. 转发板 -> Android

### 5.1 电机2剩余珠数

```text
Code1 = 0x04
Code2 = 0x01
Data3:Data4 = 剩余珠数
ACKbyte = 0x00
ExpandCode = 0x00
```

发送时机：收到新吐珠命令后、每检测到1颗有效出珠后、独立停止并清零后。

示例：ID=`0x01`，剩余9颗：

```text
AA 00 01 04 01 00 00 00 09 00 00 9F E9 55
```

### 5.2 电机2吐珠超时

```text
Code1 = 0x04
Code2 = 0x02
Data3:Data4 = 超时后剩余珠数
ACKbyte = 0x00
ExpandCode = 0x00
```

## 6. 电机2控制参数

第二组电机与原PANTAO电机1硬件相同：85%正转、约2.56kHz PWM、连续3000ms无有效出珠则超时、先断电1ms、反转300ms、最多重新正转3次；有效光眼低电平脉宽必须大于100us。

硬件定义：PA6/TIM3_CH1->SS6285L FI，PA7/TIM3_CH2->SS6285L BI，PB0->HOOLLE_OUTPUT2。

## 7. 原协议特殊命令

### 7.1 StopAllDevice

原`Code1=0x01 / Code2=0xFF`帧继续原样发送给PANTAO主板，同时本地停止并清空电机2。

### 7.2 PANTAO串口Bootloader OTA

原进入Bootloader命令为`Code1=0x01 / Code2=0xF0 / Data1~4=42 4F 54 41`（ASCII `BOTA`）。转发板先完整转发BOTA帧，再停止电机2并进入RAW透明模式。RAW模式下不解析、不修改、不插入本地帧，所有字节双向原样转发。检测到PANTAO应用重新发送合法的`Code1=0x00`普通14字节帧后恢复正常模式。
