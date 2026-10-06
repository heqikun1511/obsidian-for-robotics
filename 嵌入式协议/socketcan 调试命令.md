[[CAN]]

先确保 `vcan0` 存在：

```
ip link show vcan0
```

若不存在：

```
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
```

若没有 `candump` / `cansend`，安装：

```
sudo apt update
sudo apt install can-utils
```


## 终端 1：观察原始 CAN 总线

```
candump -tz -x vcan0
```

参数含义：

```
-tz  显示时间戳
-x   显示字节 payload
```



## 终端 2：实时 DBC 解析

先安装仅用于开发验证的 Python 工具：

```
python3 -m pip install --user cantools python-can
```

进入协议包目录：

```
cd /home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_can_protocol
```

运行实时监听和解析：

```
python3 - <<'PY'
import can
import cantools

dbc = cantools.database.load_file(
    "test/data/fsac_virtual_vehicle.dbc"
)

bus = can.Bus(interface="socketcan", channel="vcan0")

print("正在监听 vcan0，按 Ctrl+C 退出")

while True:
    message = bus.recv()

    print(
        f"RAW  id=0x{message.arbitration_id:03X} "
        f"data={message.data.hex(' ').upper()}"
    )

    try:
        decoded = dbc.decode_message(
            message.arbitration_id,
            message.data
        )
        print("DBC ", decoded)
    except KeyError:
        print("DBC  未定义 CAN ID")
    except Exception as error:
        print("DBC  解码失败:", error)
PY
```


## 终端 3：手工模拟车辆发送反馈帧

发送虚拟 DBC 中的车辆反馈帧 `0x100`：

```
cansend vcan0 100#9A1085FF1E0705A6
```

你应在终端 1 看到：

```
vcan0  100   [8]  9A 10 85 FF 1E 07 05 A6
```

终端 2 应看到类似：

```
RAW  id=0x100 data=9A 10 85 FF 1E 07 05 A6

DBC {
  'VehicleSpeedKph': 42.5,
  'SteeringAngleDeg': -12.3,
  'BrakePressureBar': 15.0,
  'DriveEnabled': 1,
  'VehicleMode': 'Emergency',
  'AliveCounter': 5,
  'FeedbackChecksum': 166
}
```

发送虚拟自动驾驶命令帧 `0x200`：

```
cansend vcan0 200#FA0024FA0100095A
```





# 有关虚拟环境

python3 -m venv /home/amoy/.venvs/fsac-can-tools



source /home/amoy/.venvs/fsac-can-tools/bin/activate

安装依赖：

```
python -m pip install --upgrade pip
python -m pip install python-can cantools
```

验证：

```
python -c "import can, cantools; print(can.__version__)"
```