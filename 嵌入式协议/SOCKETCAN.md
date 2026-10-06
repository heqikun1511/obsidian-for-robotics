---
tags:
  - CAN
  - linux
---
SocketCAN 是 Linux 内核提供的 CAN 总线通信框架。它把 CAN 控制器抽象成类似网卡的网络接口，例如：
[[CAN]]
```
can0
can1
can2
vcan0
```

应用程序不需要直接调用 PCAN、Kvaser 等厂商的私有 API，而是像操作 UDP/TCP Socket 一样，通过 Linux Socket API 收发 CAN 帧。SocketCAN 使用 Linux 网络栈，支持多个程序同时监听同一 CAN 接口、内核级报文过滤、时间戳、错误帧、CAN FD、ISO-TP、J1939 等功能。


CAN ID通常不是“设备地址”。它更多表示“这是什么消息”，比如：

```
0x101  车辆速度
0x102  转向角
0x201  AMI状态
0x301  VCU故障
```

但 CANopen 会基于 CAN ID 构造节点相关的 COB-ID，例如：

```
0x600 + Node ID    SDO请求
0x580 + Node ID    SDO响应
```


# 用虚拟CAN学习，不需要硬件

SocketCAN提供 `vcan` 虚拟接口，特别适合开发ROS 2节点、CAN解析器和单元测试。

加载模块：

```
sudo modprobe vcan
```

创建接口：

```
sudo ip link add dev vcan0 type vcan
sudo ip link set vcan0 up
```

检查：

```
ip -details link show vcan0
```

删除：

```
sudo ip link del vcan0
```

注意：`vcan0`没有真实波特率，也没有物理层错误、ACK、终端电阻或 bus-off，因此只能测试软件收发逻辑，不能验证真实总线可靠性。

# 用candump接收报文

打开终端1：

```
candump vcan0
```

终端2发送：

```
cansend vcan0 123#11223344
```

终端1会看到类似：

```
vcan0  123   [4]  11 22 33 44
```

真实接口同理：

```
candump can2
```